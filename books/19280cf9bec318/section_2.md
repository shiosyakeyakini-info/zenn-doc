---
title: "第２章　UIは関数である"
---

## UIは純粋関数たりえるか

第1章では、関数型プログラミングについて論じました。  

第2章では、これらの概念がUIにおいてどのように適用されるか、すなはち**UIを関数として扱うとはどういうことか**を考察します。

UIにおいて純粋関数性を語るとは、「入力が同じであれば、常に同じUI（Widgetツリー）が返る」ことを意味します。  
Flutterにおいてこれは非常に自然な発想であり、StatelessWidgetやConsumerWidgetのように、`build`メソッドに与える引数が固定されていれば、あるいはBuildContextから導出される値が固定されていれば、UIも同じく一定であるべきという思想です。

たとえば、以下のようなシンプルなUIは参照透過です。

```dart
class Test extends StatelessWidget {
    final String text;
    const Test({required this.text});

    @override
    Widget build(BuildContext context) {
        return Text(text, style: TextStyle(color: Colors.red));
    }
}
```

この例では、`text`に `"Hello"` を与えれば、常に同じWidgetツリーが返されます。  
これは純粋関数的であり、テストや再利用が容易です。

## 非純粋なUIの例

一方で、StatefulWidgetのように状態を内部で保持する場合、参照透過性が破られる可能性があります。  
たとえば以下の例では、同じ引数であっても毎回異なるUIが生成されるため、冪等ではありません。

```dart
class BuildCounter extends StatefulWidget {
    const BuildCounter({Key? key}) : super(key: key);

    @override
    State<BuildCounter> createState() => _BuildCounterState();
}

class _BuildCounterState extends State<BuildCounter> {
    int _buildCount = 0;

    @override
    Widget build(BuildContext context) {
        _buildCount++;
        return Text('Build回数: $_buildCount');
    }
}
```

このようなWidgetは、状態と描画ロジックが結びついており、入力と出力の対応が崩れています。最初の状態では冪等ですが、描画が繰り返されるごとにウィジェットの返る値が変わってしまいます。つまり条件付き冪等です。

そのため、これは**純粋関数ではないUI**です。

多くの場合、内部状態を持つウィジェットは純粋関数的といえなくなります。それは`FocusNode`であったり、`TextEditingController`であってもです。

## FlutterにおけるUIと副作用の分離

純粋関数性を保つためには、UIの描画処理と副作用的な処理（通信、通知、永続化など）を分離しなければなりません。

たとえば以下の例では、副作用（HTTP通信）を`Provider`の内部仕様として封じ込め、UIはその出力（AsyncValue）に応じて描画を行うのみです。

```dart
@riverpod
Future<User> userDetail(Ref ref, String id) {
    return ref.read(apiClient).getUserDetail(UserDetailRequest(id: id));
}

class UserDetail extends ConsumerWidget {
    @override
    Widget build(BuildContext context, WidgetRef ref) {
        final userDetail = ref.watch(userDetailProvider(id));
        return switch (userDetail) {
            AsyncData(:final value) => Text('Hello, ${value.name}'),
            AsyncError() => Text('Oops'),
            AsyncLoading() => CircularProgressIndicator(),
        };
    }
}
```

このように、UIが「状態を受け取って描画する」だけであれば、それは純粋関数的な構造に近づきます。

ここで、UIによる`ref.watch`が`userDetailProvider`の監視開始に伴って通信を開始するのであれば、それは副作用といえるのではないか？という疑問が湧きます。

しかしながら、`userDetail`が実際に副作用を持つのか持たないのかはProviderの内部仕様であって、ウィジェットはそのことを**知る必要はありません**。あくまでこれはインターフェイスであり、たとえば下記のようなテストコードにおいて、

```dart
testWidget('test userDetail', (tester) {
    tester.pumpWidget(
        ProviderScope(
            overrides: [
                userdetailProvider.overrideWith(
                    (ref, id) => AsyncData(User(name: 'Tom'))
                )
            ],
            child: UserDetail()
        )
    );
    expect(find.byText('Hello, Tom', findOneWidgets));
});
```

これはProviderも副作用を持たないですが、UserDetailウィジェットは想定通りに動作します。

このProviderの依存は単なる入力であるという理屈は、Providerだけでなく、`MediaQuery.of(context)`や`Theme.of(context)`のようなInheritedWidgetの依存に対しても同じことがいえます。

もっと厳密にいえば、BuildContextからWidgetRefもMediaQueryもThemeも決定できるため、BuildContextに依存する入力（BuildContextという入力から導出されるべき入力）であるといえます。


## Flutterの非純粋な機構：副作用を持つ操作

一方で、以下のようなコードは副作用を持ちます。

```dart
ElevatedButton(
    onPressed: () {
        ScaffoldMessenger.of(context).showSnackBar(
            const SnackBar(content: Text('ボタンが押されました！')),
        );
    }
)
```

この例はユーザーの操作に応じて、外部に対してUIを操作する処理が実行されるため、**副作用がある**といえます。  
Flutterにおいては、`showDialog` や `showSnackBar` のようなUI制御系APIは命令的であり、**宣言的UIの外側にある命令的API**です。あるいはNavigatorなども該当します。

ここで重要なのは、Haskellほど過激にモナド化を要求しているわけでも、Reactほど過激にコンポーネントに対して純粋関数であることを要求しているわけでも、Jetpack ComposeのようにDialogもコンポーザブル関数にあるべきという論をふりかざしているわけでもなく、Flutter自体も命令的なAPIを提供しているということです。

これは、薬にも毒にもなり得ます。なぜなら宣言的パラダイムであることに対しては逆行しており、それが「Flutterが初心者でも容易に構築できる薬」にも、「大規模なアプリケーションで命令的所作が`! context.mounted`地獄に陥る毒」にもなるということです。


### 副作用の契機と主体

あるいはこのような例も考えてみましょう。

```dart
@riverpod
class CounterNotifier extends _$CounterNotifier {
    @override
    int build() => 0;

    void increment() => state++;
}

class CounterButton extends ConsumerWidget {
    const CounterButton({super.key});

    @override
    Widget build(BuildContext context, WidgetRef ref) {
        final count = ref.watch(counterProvider);
        return Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
                Text('カウント: $count'),
                ElevatedButton(
                    onPressed: () => ref.read(counterNotifierProvider.notifier).increment(),
                    child: const Text('増やす'),
                ),
            ],
        );
    }
}
```

これは副作用を持つでしょうか？誰が副作用を持つでしょうか？

副作用を直接的に持つのは`CounterNotifier`です。これは明確に、incrementという操作を行えば冪等ではなくなります。

ウィジェット単体でみればどうでしょうか？このウィジェットは**副作用を発生させる契機は持ちますが、副作用そのものは持っていません**。実際に副作用を起こすかどうかはCounterNotifierに委ねられているからです。これは、ウィジェットから見れば副作用がモナド化されている構図となります。

そうすればこのCounterButtonメソッドは、副作用を持つ契機自体はあっても、副作用そのものは発生させる方法を持っていなから、実務的には純粋関数的といえます。

ところで……Reactでは副作用は必ず隔離されていなければならないといいます。この理論をそのままflutter_hooksに適用して、たとえば極端な例として、

```dart
class Sample extends HookWidget {
    @override
    Widget build(BuildContext context) {
        final state = useState("");
        useEffect(() {
            print(state.value);
            return null;
        }, [state]);
        final controller = useTextEditingController();
        useEffect(() {
            void listener() => state.value = controller.text;
            controller.addListener(listener);
            return controller.removeListener;
        });

        return TextField(
            controller: controller,
        );
    }
}
```
「`print`は外部に作用する副作用だから、`useEffect`にて隔離されなければならない」という理屈に基づいて、副作用を分離しましたが、これは理論的に正しいでしょうか？現実的でしょうか？それは**宿題**にしておきます。

## 合成関数の集大成たるUIとイベントハンドリングのための高階関数


ここまでで見てきたとおり、FlutterにおけるUIは、純粋関数的に「状態 → UI」という写像を記述するものであり、
その実体は、`Text(...)`, `Column(...)`, `ElevatedButton(...)` といった**関数の合成結果**に他なりません。

ウィジェットのネストは、見た目にはツリー構造に見えますが、その本質は関数合成の糖衣構文（シンタックスシュガー）に過ぎません。たくさんの関数（ウィジェット）が合成した結果として一つのウィジェットツリーが生成されているといえます。
この意味で、UIツリーとは**合成関数の出力**として再構築されるものであり、再描画とは関数評価の再実行に等しいと捉えることができます。関数を再評価しているだけですから、実際にその部分、その領域として再描画するかは別次元にあります。実際、全く同じ結果となるUIツリーをUIとして描画するために計算しなおし、再描画する必要はないかもしれません。それはFlutterの内部で決定されることです。

また、UIは純粋な写像に留まらず、**副作用の契機を高階関数として受け取る**場面も存在します。
`onPressed: () => ...` のような構文は、UIコンポーネントが副作用を持つのではなく、**副作用を外部から注入される契機を提供するだけである**ことを意味します。この中で`showDialog`をしたりするならば話は別ですが、Notifierに対して抽象的な操作を発生させる分には、単なる副作用の契機でしかないということは先に述べたとおりです。この契機となるものをウィジェットが抽象的に提供する手段が、高階関数であるといえます。

つまるところ、**UIは合成関数による出力であると同時に、副作用的操作を高階関数によって注入可能なインタフェースである**という二重構造を備えています。
このように捉えることで、我々はウィジェットを、**写像的構造と副作用的契機の境界点**として正しく設計することが可能になるのです。


ところで、Flutterは必ずしも宣言的に高階関数を渡す手法を取るわけでもありません。

```dart
class Sample extends HookWidget {
    @override
    Widget build(BuildContext context) {
        final controller = TextEditingController();
        useEffect(() {
            void listener() => print(controller.text);
            controller.addListener(listener);
            return () => controller.removeListener(listener);
        });

        return TextField(
            controller: controller,
        );
    }
}
```

これはリスナーに対する関数は確かに高階関数ですが、明らかに手続的操作に基づいて登録されています。なぜでしょうか？

```dart
class Sample extends HookWidget {
    @override
    Widget build(BuildContext context) {
        return TextField(
            onChanged: [
                (text) => print(text),
            ]
        );
    }
}
```

のような形を取ることも設計上できたはずですね。実際、onSubmittedはウィジェットの引数として高階関数を取ります。あるいはCheckboxなどもonChangedを高階関数として取ります。これにはそもそも、FocusNodeやTextEditingController、あるいはNavigatorがなぜ宣言的な操作から離れて存在するのかという存在論的な問いへと繋がります。
