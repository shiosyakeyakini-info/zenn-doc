---
title: "第４章　Flutterにおける構造的転換"
---

## MVCやMVVMのフロントエンドにおける限界
MVCやMVVMがフロントエンドにこれまで活用されてきた経緯があるが、
宣言型アーキテクチャでは必ずしも適切でないことを説明する。

## View層が存在しないFlutterアーキテクチャ
Flutterにおいては、View層とは与えられた引数をもとにSkiaやImpellerが描画するレイヤーであるから、
そもそもView層が存在しておらず、ウィジェットが最初からController的な立ち位置であることを説明する。

ReactやVue.jsはDOM操作からの延長上にあるが、ActivityまたはFragment&ViewModel(Java,Kotlin),ViewController(Obj-C,Swift)に対しては
Flutterはそもそもの描画機構がRenderObjectにより抽象化され、実際に描画するのはSkiaやImpellerであるから、
スマホアプリのフレームワークとしては、これまでのものとも連続性がなく、根本的に異なることも説明する。


## Flutterが完全に宣言的でもない理由
`showDialog`や`showSnackBar`などのUI操作は、副作用を伴う命令的操作である。
Reactなどのように、UIは100%宣言的に実装することを要求するものではない。
小規模なアプリケーションではこれらの命令的操作は許容されるが、
大規模なアプリケーションでは破綻することもある。
Jetpack Composeなどのように、Dialogも宣言的に実装する例もあり、
これはFlutterにおいても応用可能である。

## RiverpodがFlutter界の状態管理デファクトスタンダード担った理由を考える

第3章でも見てきましたが、Riverpodは

- Jotai/Recoil的「形式的原子性」を追求する
- Pinia的 意味論毎の「ストア管理」に舵を切る
- 従来のMVVMへそのまま移行するためにViewModel的に設計する

このどれもがRiverpodで実現することができますし、Riverpodはこのどれかを強要しているわけでもありません。

コンポーネントや状態に対する「原子性」を理論的な最小単位まで分解してそれに基づいてProviderを設計することも、極端な原子性は認知負荷が高すぎるから意味論的集合に留めた単位でProviderを設計することも、既存のAndroidのMVVMパターンからそのまま容易に移行するためにそのMVVMのモデルを移植し、部分的に分割していくことも、どれもRiverpodでは「可能」です。

さらに、Scoped Providerを用いてウィジェットツリーをスコープとした、InheritedWidgetや旧来の「Provider」パッケージの概念を再評価することもできます。
