---
title: "VRChat想定アバターをFlutterでぐりぐりしてみた" # 記事のタイトル
emoji: "😸" # アイキャッチとして使われる絵文字（1文字だけ）
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["VRChat", "Flutter"] # タグ。["markdown", "rust", "aws"]のように指定する
published: false # 公開設定（falseにすると下書き）
---

Flutter GPUというものがFlutter 3.24以降、プレビュー版として使用できるようになっていました。そこで、VRChat想定アバターをFlutterウィジェットで表示できるようにいろいろ試行錯誤してみました。

一体誰向けの記事なんでしょうね？私にもわかりませんが、どうしてもVRChat想定アバターをFlutterで描画しないと明日三途の川を渡りきってしまう余命宣告をされた人向けとかだと思います。

## VRChat想定アバターをVRM形式に変換する

ここはFlutterではなくVRChat特有の、それもアバター改変のドメイン知識が必要です。ここでは、アバター改変についてある程度理解がある(Modular Avatarでアバター改変をしたことがある)前提で進めます。ご理解が無い方は、とりあえずVRChatをTrusted Userになるまでフレンドを作って遊びましょう。

何かしらのアバターを例示しなければいけないので、ここでは[onntama shop](https://kakeudonn.booth.pm/)さんの[オリジナル3Dモデル【Rize】](https://kakeudonn.booth.pm/items/5496642)をFlutterの世界に持ち込むことにします。


まず、今のところ、いくつか試行錯誤してみましたが、Modular Avatarで他の服をSetup OutfitしたアバターをVRM形式で出力して後の工程で表示させてもあまりうまくいきませんでした。そこで、とりあえずBOOTHで購入したアバターをそのまま表示させることをとりあえず目標とします（これはいくつかのアバターで確認しましたがだいたいどれもうまくいきました）。

まず、あらかじめ、~~VCC~~ALCOMで下記のレポジトリを追加しておきます。

次に、**AAO Avatar Optimizer**のコンポーネントは一時的に削除しておきます。原因はわかりませんが、VRChatーというかUnityにおける最適化がどうもFlutterGPUと相性が悪そうでした。変なメッシュができたり服と素体が分離して壊れたりしたので、一時的に切っておきます。Setup Outfitを忘れたままアップロードしたアバターより見た目がグロテスクなので画像はありませんが、だいたいそれ以上の何かを想像してもらえればいいです。

この状態でアバターのヒエラルキーでアバターを選択し、Modular AvatarからManual Bakeを行います。これを対象にUniVRMで出力します。

安心と切り分けの根拠を得るために、[UniVRMTest](https://vrm.dev/vrm/how_to_view_vrm/)でVRM変換したアバターが、VRMとして動いていそうかを確認します。

## FlutterGPUを使えるための環境を構築する

さて、ここからFlutterのお話です。2025年12月27日現在ではmasterチャネルでないとだめとかがあるので、以下の環境構築のもと進めます。

- fvmでバージョン管理をします。masterチャネルの更新の追いつきも楽になります。
- `fvm flutter config --enable-native-assets`を叩きます。これが必須です。

さらに、今回ではflutter_sceneを使用するのですが、ここでは`.model`という拡張子のファイルを使用します。どうやらこれは`flutter_scene`のための独自フォーマットのようです。

幸いにも`.vrm`を`.model`に変換できる（.vrmがglTFの拡張の形式になっているからですね）のですが、そのためにCMakeを要求するので、下記の手順で入れておきます。WindowsやLinuxの方は適宜読み替えてください。

- `brew install cmake`

なお、`.model`に変換する過程でVRM特有の情報は失われます。VRMに沿って「MToonシェーダー相当の描画をしたい」といった要件の場合、[gltf_loader](https://pub.dev/packages/gltf_loader/versions/0.4.0)を使ってVRM特有の情報を得たうえで、flutter_sceneを自分でフォークして頑張るくらいの気概が必要そうでした。「SpringBoneで髪が揺れないと困る」くらいだったら未検証ですができるかもしれませんね。

逆に言えば、それ以外の用途ではglTFの部分しか使えないので、そもそもアバター自体もglTFとして出力してもいいですが、メタデータ（アバター名や作者、その他の付随情報）の取得などは先述のgltf_loaderで簡単にできるので、VRMでいいと思います。

## 依存関係を追加する

以下のように依存関係を構成します。

```yaml
dependencies:
  flutter_scene: ^0.9.2-0
  flutter_gpu:
    sdk: flutter
  gltf_loader: ^0.4.1 # VRM特有の情報が必要な場合
```

`flutter_scene`はFlutterGPUを使うためのほぼ唯一のパッケージです。`.model`を渡すとモデルを表示してくれます。

ここではflutter_hooksとriverpodも使っていきます。便利なので。

## VRMを更にmodelに変換する

前述の通り、flutter_sceneは`.model`という独自形式のファイルを使用するため、以下のようにコマンドを叩いて、VRMモデルをさらに変換します。直接`.model`にするUnityEditor拡張があってもいいかもしれませんが、需要がなさすぎるのでやりませんでした。

```bash
fvm dart run flutter_scene_importer:import \
  --input /絶対パス/to/your-avatar.vrm \
  --output /絶対パス/to/your-avatar.model
```

## 実装する

### ビジネスロジック部分

シーンの作成、モデルの読み込み、カメラの設定を行います。ビジネスロジックなので、Riverpodで書くとよいと思います。

ここではモデルパスを渡すようにしましたが、別に貰わずに直接`AssetBundle`から読みに行ったりしてもいいと思います。

```dart
/// シーンとカメラをまとめたデータクラス
class SceneData {
  const SceneData({required this.sceneInstance, required this.cameraInstance});
  final scene.Scene sceneInstance;
  final scene.Camera cameraInstance;
}

/// シーンを読み込むプロバイダー
@riverpod
Future<SceneData> sceneLoader(Ref ref, String modelPath) async {
  // Sceneの静的リソースを初期化
  await scene.Scene.initializeStaticResources();

  // モデルをロード
  final model = await scene.Node.fromAsset(modelPath);

  // シーンを作成してモデルを追加
  final newScene = scene.Scene();
  newScene.add(model);

  // カメラを設定（モデルの正面から見る位置）
  final camera = scene.PerspectiveCamera(
    position: vm.Vector3(0, 1, 3),
    target: vm.Vector3(0, 1, 0),
  );

  return SceneData(sceneInstance: newScene, cameraInstance: camera);
}
```

### ウィジェット部分

線とか円とか書ける`CustomPainter`がありますね。この`CustomPainter`にレンダリングします。`CustomPainter`はBuildContextに依存できない(Refを持てない)ので、素直にシーンとカメラをコンストラクタ引数で渡してあげます。

```dart
/// Sceneをレンダリングするカスタムペインター
class _ScenePainter extends CustomPainter {
  _ScenePainter({
    required this.sceneInstance,
    required this.cameraInstance,
  });

  final scene.Scene sceneInstance;
  final scene.Camera cameraInstance;

  @override
  void paint(ui.Canvas canvas, ui.Size size) {
    final viewport = ui.Offset.zero & size;
    sceneInstance.render(cameraInstance, canvas, viewport: viewport);
  }
  
  // 常に再描画（アニメーション対応）
  @override
  bool shouldRepaint(_ScenePainter oldDelegate) => true; 
}
```

残り、モデルを表示するウィジェットを作ります。`TickerProviderStateMixin`のために`StatefulWidget`を作るのは億劫なので、HookConsumerWidgetを使います。

```dart

class VrmViewer extends HookConsumerWidget {
  const VrmViewer({
    super.key,
    required this.modelPath,
    this.width,
    this.height,
  });

  /// 表示する.modelファイルのアセットパス
  final String modelPath;

  final double? width;
  final double? height;

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final scene = ref.watch(sceneLoaderProvider(modelPath));

    // フレーム更新用のTickerを使用して再描画をトリガー
    final tickerProvider = useSingleTickerProvider();
    useEffect(() {
      final ticker = tickerProvider.createTicker((_) {});
      ticker.start();
      return ticker.dispose;
    }, [tickerProvider]);

    return SizedBox(
      width: width,
      height: height,
      child: switch (scene) {
        AsyncData(:final value) => CustomPaint(
            painter: _ScenePainter(
              sceneInstance: value.sceneInstance,
              cameraInstance: value.cameraInstance,
            ),
            child: Container(),
          ),
        AsyncError(:final error) => Center(
            child: Text(
              'モデルのロードに失敗しました: $error',
              style: const TextStyle(color: Colors.red),
              textAlign: TextAlign.center,
            ),
          ),
        _ => const Center(child: CircularProgressIndicator()),
      },
    );
  }
}
```

### ビルド

アプリをビルドする際、デスクトップの場合は単に`flutter run`ではだめで、`fvm flutter run --enable-impeller`が必要です。FlutterGPUはImpellerでしか動作しません。

### 将来

実際にFlutterで表示してみて気付いたことはいくつかありますが、

**[liltoon](https://lilxyzw.github.io/lilToon/)が偉大すぎる**。あまりに自然に私達（あなたも勝手に巻き添えにします）はトゥーンシェーダーの世界観を見慣れ過ぎて目が肥えていましたが、Unityのレンダリングパイプラインどころかカスタムシェーダーが[issue](https://github.com/bdero/flutter_scene/issues/22)に上がっているに留まるこのflutter_sceneの貧弱なPBRマテリアル描画を見てしまうと、ああ、liltoonって最早単なるシェーダーに留まらずある種の世界だと感じました。このissueが解消されたとき、flutter_sceneで動作するGLSLとDartで書かれたトゥーンシェーダーを習作として書いてもいいかもと思いました。

逆に言えば、その時に初めてFlutterGPUがVRChatの文脈で「実用」に耐えうるものになるのではという期待を寄せることができます。
