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

`sealed class`は、複雑な状態を端的に、かつ型の整合性を取りながら移動させるのに適当な方法です。

```dart

@freezed
sealed class AuthenticationState with _$AuthenticationState {
    const factory AuthenticationState.notLogin() = NotLogin;
    const factory AuthenticationState.logined({
        required User user,
    }) = Logined,
}

@freezed
sealed class SignupState with _$SignupState {

    /// サインイン前
    const factory SignupState.provisional({
        @Default('') String userId,
        @Default('') String password,
        @Default('') String mailAddress,
    }) = Provisional;

    /// メール認証中
    const factory SignupState.mailAuthentication({
        @Default('') String userId,
        @Default('') String password,
        @Default('') String mailAddress,
    }) = MainAuthentication,

    /// 本登録
    const factory SignupState.definitive({
        required Provisional data,
        @Default('') String name,
        @Default('') String address,
        @Default('') Sexual sex,
        // ...
    }) = Definitive

    const factory SignupState.authenticateTwoAuth({
        required Provisional data,
        @Default('') String name,
        @Default('') String address,
        @Default('') Sexual sex,
        // ...
        @Default('') String totpSecret,
    })
}

```

この状態をたとえばSignupStateNotifierが、それぞれ副作用的操作に応じて状態を遷移させていくことで、より宣言的に記述することができます。

```dart
Widget build(BuildContext context) {
    final signupState = ref.watch(signupProvider);

    return switch(state) {
        // あまり頻繁に見る構文ではありませんが、このように表現できます。便利
        Definitive(:final name) || AuthenticateTwoAuth(:final name) => Text('$name'),
        Provisional() || MailAuthentication() => throw StateError('invalid state.'),
    }
}
```

現状はファクトリメソッドに対するスプレッド構文が無かったり、sealed classに対して特定のフィールドがあるか記述する方法が`is` `as` `switch`などに限られるといった構文上制約は強いです。このため、以前の状態からさらに進んだ文脈の状態を記述するためには、この点は冗長に書かざるを得ません。

しかしながら、このように直和型の表現で状態遷移を行えば、この型では必ずこの値はあるはずとか、なければそれは仕様バグというものを見つけたり、不整合な状態を防いだりすることができるようになります。

そしてウィジェットはどの状態であればどれを出す、というのをより宣言的に記述することができるのです。




## 非同期処理の状態管理

多くの副作用には非同期処理を伴います。そして世のアプリケーションは頻繁にこの非同期処理というものに苦しめられてきました。なぜなら、非同期処理の状態をより汎用的に扱う方法がなかったからです。

これはRiverpod抜きにしたDartのFutureもそうです。Futureはそもそもsealed classが登場するよりずっと前から存在するものですから、型安全的に状態を網羅することが不得意です。

多くの場合非同期を伴う副作用は、その副作用自体がシステム全体で冪等として存在しない限りは、何度も実行されたくないです。これまでは、そのためにローディング中の状態をViewModelで保持したり、そもそもこの状態管理から逃げて[useDebounced](https://pub.dev/documentation/flutter_hooks/latest/flutter_hooks/useDebounced.html)的手法で逃げたりしながらモーダルインジケーターという悪しき風習に委ねる方法が取られてきました。

非同期処理に対する状態も「状態」の一種である以上は、適正に管理しなければ不具合を引き起こす要因となります。

さて、Riverpodではこの非同期の副作用を、より汎用的に簡単に扱うことができます。特にRiverpod 3.0からはmutationが試験的に導入されたため、多くのパターンの副作用に伴う非同期状態を簡単に扱えるようになりました。

```dart
@riverpod
class BookNotifier extends _$BookNotifier {
    @override
    Future<List<Book>> build() async {
        return await ref.read(apiClientProvider).getBooks();
    }

    @mutation
    Future<void> loadNext() async {
        final currentState = await future;
        final lastBookId = currentState.last.id;
        state = AsyncData([
            ...currentState,
            await ref.read(apiClientProvider).getBooks({untilId: lastBookId}),
        ]);
    }
}
```

たったこれだけのコードですが、このNotifierが示す状態には、
- 最初のBooksの初期取得（読込中）
- 最初のBooksの初期取得エラー
- 取得結果（完了時）
- 最後尾以降の読み込みが行われていない状態
- 最後尾の読み込みが行われている状態
- 最後尾の読み込みがエラーとなった状態
- 最後尾の読み込みが完了した状態（＝取得結果の更新とは別に）

の8状態が網羅されています。この8状態を`AsyncValue`と`MutationState`によって表現可能なため、

```dart
class BookList extends HookConsumerWidget {
    @override
    Widget build(BuildContext context, WidgetRef ref) {
        final book = ref.watch(bookNotifierProvider);
        final loadNext = ref.watch(bookNotifierProvider.loadNext);
        final scrollController = useScrollController();

        useEffect(() {
            void listener() {
                if(controller.position.pixels != controller.position.maxScrollExtent) return;
                if(loadNext.state is PendingMutation) return;
                loadNext.call();
            }
            controller.addListener(listener);
            return () => controller.removeListener(listener);
        });

        return switch(book) {
            AsyncLoading() => Center(child: CircularProgressIndicator()),
            AsyncError() => Column(children: [
                Text('エラーが発生しました。'),
                ElevatedButton(
                    onPressed: () => ref.invalidate(bookNotifierProvider),
                    child: Text('再読み込み')
                ),
            ]),
            AsyncData(:final value) => ListView.builder(
                controller: scrollController,
                itemCount: value.length + (loadNext.state is PendingMutation ? 1 : 0),
                itemBuilder: (BuildContext context, int index) {
                    if (index == value.length) {
                        return const Center(child: CircularProgressIndicator());
                    }
                    return ListTile(title: Text(value[index].title));                   
                }
            )
        };
    }
}
```

このように非常に宣言的に記述可能になります。

フロントエンドにおけるほとんどの処理は非同期処理でありながら、従来の多くのライブラリやフレームワークではこれを上手に扱うことを苦手としていました。しかし、RiverpodではFutureProviderやAsyncNotifierProviderを上手に活用することで、進行状況、エラー、完了状態を型安全的に、かつ宣言的に扱うことができるのです。さらにローディングやエラー時の制御、重複実行の防止も簡単に行うことができ、複雑な非同期処理もシンプルに状態管理として扱うことができます。

特にMutationに関してはRiverpod 3.0からの実験的な機能ですが、これまではAsyncValueの状態を自分で保持する必要があったため、これを不要にして本質的な副作用処理に集中できるようになる機能です。

### 共通エラー処理

ところで、共通エラー処理というものについても多くの人々が悩まされてきたものだと思います。これもいくつかの方法が考えられます。

すべての副作用を伴うエラー処理がダイアログを出すだけであれば、

```dart
ref.listen(bookNotifierProvider.loadNext, (_, next) {
    switch(next) {
        MutationError(:final error, :final stackTrace):
            commonError(error, stackTrace);
        default:
    }
});
ref.listen(bookNotifierProvider, (_, next) {
    switch(next) {
        AsyncError(:final error, :final stackTrace)
            commonError(error, stackTrace);
    }
});
```
のように、各Mutationや各AsyncErrorをlistenすればよいという話になります。Mutation<T>やAsyncValue<T>型自体のユーティリティにしてしまうことも考えられます。

```dart

void commonError<T>(
    BuildContext context, AsyncValue<T>? previous, AsyncValue<T> after) {
  if (after is! AsyncError) return;
  showDialog(
      context: context,
      builder: (context) {
        return AlertDialog(
          title: const Text('エラー'),
          content: Text(after.error.toString()),
          actions: [
            TextButton(
              onPressed: () => Navigator.of(context).pop(),
              child: const Text('OK'),
            ),
          ],
        );
      });
}

ref.listen(bookNotifierProvider,
    (previous, next) => commonError(context, previous, next));

```

もし、特定のエラー時は透過的に処理を行うエラーハンドリングがある場合、たとえばセッション切れの場合にログイン画面に戻すような場合を考えると、

```dart
```dart
// エラー状態を管理するクラス
@freezed
sealed class AppErrorState with _$AppErrorState {
    const factory AppErrorState.none() = None;
    const factory AppErrorState.error({
        required Object error,
        required StackTrace stackTrace,
        required Completer<void> completer,
    }) = Error;
}

@riverpod
class AppErrorNotifier extends _$AppErrorNotifier {
    @override
    AppErrorState build() => const AppErrorState.none();


    // guard関数: エラー時に自動で状態をセットし、UI操作を待機
    Future<T?> guard<T>(Future<T> Function() action) async {
        try {
            final result = await action();
            clear();
            return result;
        } catch (e, st) {
            // Completerを生成し、エラー状態に保持
            final completer = Completer<void>();
            state = AppErrorState.error(
                error: e,
                stackTrace: st,
                completer: completer,
            );
            // エラー時、UI側でcompleterが完了するまで待機
            await completer.future;
            state = const AppErrorState.none();

            // ここで例えばダイアログ表示後の状態変更、ログアウト処理などを行う
            if(e is SessionErrorException) {
                await ref.read(accountProvider).logout();
            }


            // 最終的なMutationやAsyncErrorへと昇華させる
            rethrow;

        }
    }
}

// guardを使ったProvider例
Future<void> guardedBooks(Ref ref) async {
    try {
        return await ref.read(appErrorProvider.notifier).guard(() async {
            await ref.read(apiClientProvider).getBooks();
        });
    } catch(e,s) {
        // 固有のエラー処理ロジック

        // ここもウィジェットに伝播させるためにrethrow
        rethrow;
    }
}

// ウィジェット側: エラー時にダイアログを表示し、完了時にcompleterを完了させる
ref.listen(appErrorProvider, (prev, next) async {
    if (next is! Error) return;
    await showDialog(
        context: context,
        builder: (context) => AlertDialog(
            title: const Text('エラー'),
            content: Text(next.error.toString()),
            actions: [
                TextButton(
                    onPressed: () {
                        Navigator.of(context).pop();
                        next.completer.complete();
                    },
                    child: const Text('OK'),
                ),
            ],
        ),
    );
});
```

この方法であれば、ウィジェット側にもAsyncErrorやMutationErrorとして状態が伝播し、さらに共通エラーハンドリングのようなことも行うことができ、さらにそれを操作待ちに依存させたりすることも可能となります。エラーの状態を配列で持てば、より堅牢になるでしょう。

いずれにせよ、ウィジェットやUIは状態の写像であるという原理原則のもと、何の状態を監視するべきか、何の状態を見るべきかが明確であれば、このような共通エラーハンドリングも容易に行えることでしょう。




## まとめ

本章では、Riverpodを用いた状態管理において「状態の正規化」と「責務の分離」がなぜ重要か、またどのように実践できるかを解説しました。状態を意味論的な単位で分割し、各機能ごとに独立したProviderやNotifierを設計することで、再利用性や保守性が向上します。また、状態のライフサイクルや副作用の管理、非同期処理の扱いについても、Riverpodの仕組みを活用することで、より宣言的かつ安全に実装できることを示しました。アプリケーションの規模や要件に応じて、適切な粒度で状態を管理することが、長期的な開発効率や品質向上につながります。