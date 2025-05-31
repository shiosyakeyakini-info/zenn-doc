---
title: "第３章　状態管理の理論と実践"
---

## Riverpodにおける状態の正規化と責務の分離

ここでは、FlutterやRiverpodを用いて、どのように状態を分離するべきかを考察します。

ただ単に同じ画面にあるというだけでは同じ属性たる状態を持ちえません。何らか意味論的に同じものを示すグループ単位で管理されるべきです。

たとえば、一般的なコンテキストメニューひとつひとつはどうでしょうか。たまたまその文脈で必要な操作が寄せ集められているに過ぎず、本質的に「切り取り」と「コピー」は相互に依存することもない無関係の処理です。

これら依存関係にない機能を、同一のNotifierに寄せ集めてしまうと、他の箇所で同じコンテキストメニューを使用できなくなりますね。これはViewModel巨艦主義ともいえる、意味論的に無関係の寄せ集めを単一のクラスで状態管理してしまう過ちをを繰り返すこととなります。

下記の例では、たとえば「コピー」「貼り付け」の他に、翻訳とCopilotを押すと通信が始まり、ダイアログなりボトムモーダルシートなりで結果を表示する例を考えます。

これは、無関係の翻訳機能とCopilotのチャット機能、同一のコンテキストメニューにあるからという理由だけで同じモデルに押し込まれています。そしてその操作もです。明らかに責務が分離されていません。

```dart
@freezed
abstract class ContextMenuState with _$ContextMenuState {
    const factory ContextMenuState({
        @Default(Language.en) Language language,
        @Default(CopilotModel.chatGPT4o) CopilotModel copilotModel,
    }) = _ContextMenuState;
}


@riverpod
class ContextMenuNotifier extends _$ContextMenuNotifier {
    @override
    ContextMenuState build(String text) {
        return ContextMenuState();
    }

    @mutation
    Future<void> copy() {}

    @mutation
    Future<void> paste() {}

    @mutation
    Future<String> translate(Language language) {
        return ref.read(translateProvider).translate(
            language: language,
            text: state.text,
        )
    }

    @mutation
    Future<String> askCopilot(CopilotModel model) {
        return ref.read(copilotProvider(model)).ask(message: state.text);
    }
}

class TranslateMenu extends ConsumerWidget {
    @override
    Widget build(BuildContext context) {
        ref.listen(contextMenuProvider.translate, (_, next) {
            switch(next.state) {
                case SuccessMutation(:final value)
                    await showDialog(
                        builder: (context) => TranslateDialog(text: value),
                    );
            } 
        });

        final translate = ref.watch(contextMenuProvider.translate);

        return switch() {
            // ...
        }

    }
}
```

これらは、それぞれが単一の副作用処理を行うProviderとして定義されるのが本来は望ましいでしょう。コンポーネントを純粋関数に留めるという大方針を貫き、機能を分離するなら、ひとつひとつにNotifierがあるイメージになります。

```dart
@riverpod
class CopyNotifier extends _$CopyNotifier {
    @override
    void build(String text) {};

    @mutation
    Future<void> copy() {
        await Clipboard.setData(text);
    }
}

class CopyMenu extends ConsumerWidget {

    final String text;

    const CopyMenu({required this.text});

    @override
    Widget build(BuildContext context) {
        final copy = ref.watch(copyNotifierProvider(text).copy);

        ref.listen(copyNotifierProvider(text).copy, (_, next) {
            if(next is SuccessMutation) Navigator.of(context).pop();
        });

        return ListTile(
            onPressed: copy is PendingMutation ? null : copy.call()
        )
    }
}

class TranslateDialog extends ConsumerWidget {

    final Language language;
    final String text;

    const TranslateDialog({required this.language, required this.text});

    @override
    Widget build(BuildContext context) {
        // 責務の分離を考えれば、そもそも間にNotifierを噛ませる必要がなく、
        // 単に`translateProvider`を`watch`すれば済む話ですね。
        // 
        final translate = ref.watch(translateProvider(language: language, text: text));
        return switch(translate.state) {
            AsyncLoading(): Center(child: CircularProgresIndicator()),
            AsyncData(:final value) =>
        }
    }
}
```
これによって、いくつかは単にFutureProviderを`watch`するだけで済むようになります。ところでこの場合、通信を呼び出す契機は、もとのコンテキストメニューから、ダイアログに移っています。

これは仕様をも変えていることになりますが、責務の分離の観点と再利用性からすれば、どちらのほうが長期的に利するでしょうか。「ダイアログをどの文脈でも独立して使い回せる」という点では、ダイアログが初期表示を契機として通信を始める方が、コンポーネント単位では理にかなっています。

ただし、通信成功時のみダイアログを表示したい、エラー時はダイアログの遷移をさせないでほしい、といった場合には、FutureProviderではなくボタン押下に伴うNotifierによる副作用とすることも当然できます。その場合でも、コンテキストメニューの他のアイテムとは直接の依存も関係もないので、独立するのがよいでしょう。



しかしながら、FlutterはReactほど厳密に純粋関数を要求していません。上記の方法はコンポーネントを純粋に保つという点では正解ですが、小規模なアプリケーションであれば必ずしもそれは必要ありません。単純に`onPressed`で直接副作用を持ってしまっても構わないのです。ここはある種、コード規約とか、裁量による側面があります。


```dart
class CopyMenu extends ConsumerWidget {
    @override
    Widget build(BuildContext context) {
        return ListTile(
            onPressed: () async {
                await Clipboard.setData();
                if(!context.mounted) return;
                Navigator.of(context).pop();
            }
        ) 
    }
}
```

また、同期処理なら問題ありませんが、非同期処理の場合は二度押しのような重複した操作を行わせたくないという話もあるかもしれません。そのような場合は、適切に非同期がどういった状態か管理したほうがよいので、やはりNotifierを作成して副作用を押し込め、非同期の状態に対してウィジェットがどうあるべきか記述するのがよいでしょう。上記の例は簡単に実現できますが、非同期処理であるが故、同じ操作を二度連打したりすることができるかもしれません。

これはそのプログラムがどのように用いられるものかとか、開発の速度をより優先したいとか、あるいはリファクタリングとしてこうした点を見直そうとか、そういったところと天秤にかけられることになるでしょう。

### 複雑な入力フォームの場合を考える

では、複雑な状態を持つ例として路線検索の入力フォームを考えてみましょう。路線検索の入力フォームには、出発地、目的地、経路、IC優先、自由席・指定席優先、出発時刻、到着時刻、新幹線・特急利用の有無と多種多能な入力項目があります。さらに、出発地や目的地の入力補完、現在時刻の設定など、それぞれに多数の副作用も考えられます。

では、これらは同じ検索項目としての意味論を果たしているから、同じNotifierで管理するべきでしょうか？これらは最終的に検索対象として送信されるのですから、確かにそのような見方もできます。そんなに大きなフォームでなければ、そのように作成してもいいかもしれません。

ここで、RDBS的な思考でいえば、最終的にフォームへ送信する内容は、ある種複数のテーブルを結合したビューのように映ります。

```dart

@riverpod
class DepartureNotifier extends _$DepartureNotifier {
    @override
    String build() {}

    // 入力内容で更新
    String updateText(String text) => state = text;

    // 入力補完をする
    @mutation
    Future<void> completation() {
        ref.read(apiClientProvider).completation(CompletationRequest(text: state));
    }
}

// 検索のリクエスト
@riverpod
SearchRequest searchRequest(Ref ref) {
    return SearchRequest(
        departure: ref.watch(departureNotifierProvider),
        arrive: ref.watch(arriveNotifierProvider);
        via: ref.watch(viaNotifierProvider),
        priorityIC: ref.watch(icNotifierProvider),
        // ...
    );
}

// 検索対象
@riverpod
Future<void> search(Ref ref) async {
    final request = ref.watch(searchProvider);
    return await ref.read(apiClientProvider).search(request);
}
```

この分割の例はかなり極端ですが、非常に柔軟です。familyの引数にどこで使われるかのContext的な引数があれば、同じProviderをそのまま別の箇所でも使用できるでしょう。

```dart
// どの文脈で状態が保持されるかをEnumで明示
enum DepartureContext { front, sidebar }

@riverpod
class DepartureContext extends _$DepartureContext {
    String build(DepartureContext context) => "";
}
```

ところで、これはReactの状態管理ライブラリのJotaiやRecoilが目指すような、Atom的観念にも通ずるところがあります。というより、本質的には同じところに着地します。これはもしかしたらRiverpod/flutter_hooksの作者がReactを源流としているからかもしれませんね。

しかし、Riverpodは状態の責務の分離までかくあるべしという強い指針を持つかというとそうではないので、別に

```dart
@freezed
abstract class SearchState with _$SearchState {
    const factory SearchState({
        @Default('') String departure,
        @Default('') String arrive,
        [] List<String> via,
        @Default(true) bool priorityIC,
        @Default(true) bool reservedSeat,
        // ...
    }) = _SearchState;
}

@riverpod
class SearchStateViewModel extends _$SearchStateViewModel {
    SearchState build() => SearchState();
}
```

とすることが必ずしも悪いというわけでもありません。例えば簡易入力フォームの要件があとから出てきたとかであれば、それはそのときに状態に原子化をしてもよいのです。こちらはこちらで、見た目上、意味論的にも同じようにみえる塊がモデルとして投影されているので、複雑な依存関係を考慮せずに済むともいえます。

さらに、既存のMVVM/クリーンアーキテクチャからの移行という側面がもしあれば、全く異なる分散型依存管理を要求する原子的スタイルは、設計を白紙からやり直すようなレベルの大仕事になります。あるいは、クリーンアーキテクチャをFlutter&Riverpodに当てはめて使うのだという話をすれば、このような機能凝縮は避けられません。この点については次の章で詳しく見ていきます。

全員がReact&Recoil/JotaiやVue.js&Atomic Designの経験があり、状態管理の原子性について論じることができて、それに基づいて実装するのであれば、Atom的発想による機能責務はある意味で再利用性を求めた理想といえます。

しかしMVVMアーキテクチャやViewControllerの操作に慣れたスマホアプリ開発者に突然状態の原子性を要求しても混乱するかもしれませんし、Riverpodは別に状態の原子性を押し出した状態管理ライブラリというわけでもありません。実際RiverpodのDocsにAtomという言葉は一つも出てきません。ここはある種の含みだとも捉えられます。

ただし、コンポーネント単一の再利用性という側面を見れば、状態の原子性というものを意識したほうが、より再利用性の高いアプリケーションを構築する事が可能となります。

## 状態のライフサイクル

先ほど挙げた路線検索のNotifierはどのようなライフサイクルを含むでしょうか。

より原子的な設計に基づく状態管理のこの方法では、たとえば検索結果のUIにも入力フォームがあれば、そのフォームを編集した途端に検索処理が実行されることになります。

```dart

@riverpod
class DepartureNotifier extends _$DepartureNotifier {
    @override
    String build() {}

    // 入力内容で更新
    String updateText(String text) => state = text;

    // 入力補完をする
    @mutation
    Future<void> completation() {
        ref.read(apiClientProvider).completation(CompletationRequest(text: state));
    }
}

// 検索のリクエスト
@riverpod
SearchRequest searchRequest(Ref ref) {
    return SearchRequest(
        departure: ref.watch(departureNotifierProvider),
        arrive: ref.watch(arriveNotifierProvider);
        via: ref.watch(viaNotifierProvider),
        priorityIC: ref.watch(icNotifierProvider),
        // ...
    );
}

// 検索対象
@riverpod
Future<void> search(Ref ref) async {
    final request = ref.watch(searchRequestProvider);
    return await ref.read(apiClientProvider).search(request);
}
```

このフォームには、暗に以下の仕様があります。

- フォームや検索結果から一旦離れたら、すべての結果が失われる
- 検索結果ページでフォームの内容を編集すれば、即座に再検索が行われる

これは状態の変更の依存による伝播の性質です。現状、これはすべてが`ref.watch`による依存だからです。

先ほどUIは副作用を起こす契機を持つがUIは副作用を持たないということを述べました。つまりUIはそれが本当に副作用を持つかどうかを意識していないので、Provider側でビジネスロジックが起こすべき副作用と、そこでは起こす必要がない副作用をうまく調整する必要があります。

### 検索の再要求

たとえば、検索は行わせるが**ユーザーの操作によって明示的に変更が要求されたときだけ**副作用を発生させたいとします。この場合、状態の変更の有無にかかわらず検索結果はその状態を覚えておかないといけない、つまり状態の保持ロジックの変更となります。これはUIの都合だけという話ではなく、ビジネスロジックの変更と見做し得ます。

ではこれはRiverpodではどうなるのでしょうか。

```dart
@riverpod
Future<void> search(Ref ref) async {
    final request = ref.read(searchRequestProvider);
    return await ref.read(apiClientProvider).search(request);
}
```

その時点の`searchProvider`は読むだけと変化します。そしてUI側では、もちろん`ref.watch`はしますが、もう一つ、

```dart
ref.invalidate(searchProvider);
ref.refresh(searchProvider)
```

が登場します。これはプロバイダの明示的更新です。更新に含めて、プロバイダーの**破棄**を行います。この破棄という性質については、あとで詳しく解説します。

これを用いれば、当該のProviderが再評価されるので、

```dart
class SearchButton extends ConsumerWidget {
    @override
    Widget build(BuildContext context) {
        final isLoading = ref.watch(
            searchProvider.select((e) => e.isLoading || e.isRefreshing)
        );
        return ElevatedButton(
            onPressed: () => isLoading ? null : ref.invalidate(searchProvider),
            child: Text('再検索')
        )
    }
}
```

このようになります。ところで、このボタンは最初のトリガーとしての「検索」には使用できるでしょうか？そもそも、この`searchProvider`の読み込み中かどうか知るにはライフサイクル上既に存在していなければなりません。まだ検索していない画面でこのウィジェットを置いてしまうと、意図しないAPIリクエストが発生してしまいます。

つまり、既に検索が行われたあとまたは検索中の画面にしか置くことはできないということです。

### Notifierの再検索とinvalidate

Notifierであったらもっと話は難しくなります。なぜならNotifierは直接的に副作用を持つ主体であり、invalidateまたはrefreshで自身のProviderを破棄されたあとに、実行中の副作用があればどうなるのか、という問題が発生するからです。

簡単な例として、

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';

void main() => runApp(const ProviderScope(child: MyApp()));

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      debugShowCheckedModeBanner: false,
      theme: ThemeData(colorSchemeSeed: Colors.blue),
      home: const InvalidateExampleWidget(),
    );
  }
}

final invalidateExampleProvider = AsyncNotifierProvider.autoDispose<InvalidateExample, int>(InvalidateExample.new);

class InvalidateExample extends AutoDisposeAsyncNotifier<int> {
  @override
  Future<int> build() async {
    print('build called.');
    await Future.delayed(const Duration(seconds: 2));
    return 0;
  }

  Future<void> increment() async {
    final currentState = state.value ?? 0;
    await Future.delayed(const Duration(seconds: 4));
    state = AsyncValue.data(currentState + 1);
  }
}

class InvalidateExampleWidget extends ConsumerWidget {
  const InvalidateExampleWidget({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(invalidateExampleProvider);
    return Scaffold(
      appBar: AppBar(
        title: const Text('Invalidate Example'),
      ),
      body: Center(
          child: switch (count) {
        AsyncLoading() => const CircularProgressIndicator(),
        AsyncError() => const Text('Error occurred'),
        AsyncData(:final value) =>
          Column(
            mainAxisAlignment: MainAxisAlignment.center,
            children: [
              Text('Count: $value'),
              ElevatedButton(
                onPressed: () => ref.invalidate(invalidateExampleProvider),
                child: const Text('Invalidate'),
              ),
            ],
          ),
         AsyncValue() => SizedBox.shrink(),
      }),
      floatingActionButton: FloatingActionButton(
        onPressed: () async {
          await ref.read(invalidateExampleProvider.notifier).increment();
        },
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

これを[dartpad.dev](https://dartpad.dev/?sample=counter)にて実行してみましょう。

5回ほどインクリメントをしたあと、さらに+ボタンを押してinvalidateを押すとどうなるでしょうか？これはRiverpod 3.0以前かそれより前かで動作が異なります。

- Riverpod 2系では、invalidateの操作後に操作前に行った副作用が完了し、6に書き換わります。
- Riverpod 3系では、下記のようなエラーが見えることでしょう。

```
 Unhandled Exception: Cannot use the Ref of invalidateExampleProvider after it has been disposed. This typically happens if:
- A provider rebuilt, but the previous "build" was still pending and is still performing operations.
  You should therefore either use `ref.onDispose` to cancel pending work, or
  check `ref.mounted` after async gaps or anything that could invalidate the provider.
- You tried to use Ref inside `onDispose` or other life-cycles.
  This is not supported, as the provider is already being disposed.
```

2系では一度破棄したのに、破棄前の状態に戻ってしまいます。3系ではエラー内容に新たに導入された`if(ref.mounted)`で破棄されていないかを見てください、とあります。

この再更新(`ref.invalidate` `ref.refresh`)というのは破棄を伴うので、Notifierにおいては相当注意深く扱わなければなりませんし、破棄したあとに、他の副作用がどうあるべきかというのを排他制御的に見つめ直す必要があります。

ところで、NotifierでないFutureを返すProviderではこのref.mountedを気にする必要はないのでしょうか？これはそもそも、副作用を起こして状態を内部で変える手段を持たないのですから、原理的に起こり得ないはずです。[1]

[1]: たぶん……　Riverpodのソースを呼んだわけではないので絶対にないとはいいきれません



## 状態遷移図とフロー
ビジネスロジックの状態遷移的観点と、`sealed class`の活用を説明する。



## 非同期処理の状態管理

多くの場合非同期処理を伴うことと、それは抽象化すると大抵の場合同じ状態遷移となるから、
いわゆる初期表示だとか一覧画面の表示だとかいう例において、FutureProviderが大多数のケースで適切であることも説明する。
そこに何かしら操作が介在するのであれば、そのときにNotifierを用いればよい。

また、イミュータブルであることがスレッドセーフであることはここで詳しく説明する。