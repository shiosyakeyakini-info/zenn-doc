---
title: "序章　なぜ宣言型UIを選ぶのか"
---

## スマホアプリ市場の変遷と宣言型UIの採用

スマホアプリ市場は非常に進化が速く、毎年OSのアップデートの影響を受けるという奇異な市場です。

UIひとつをとってもMaterial 2から3だったり、iOSも次期で一新されるとあったり、あるいはOSの新機能との統合がどうだ、セキュリティの向上の影響はInfo.plistでどうだと目まぐるしく変化します。

さらに、Androidに関して言えば使われるデバイスも余りに多種多様です。1台1台検証することは根本的に不可能です。Androidを搭載したスマホでない環境すらあるわけで（例えばMeta Questとか）、市場も非常に多様性に富んでいます。

このような市場においては、数年単位のスパンで意思決定するのが根本的に困難ですから、より変化についていくことが要求されます。市場がそもそもウォーターフォールモデルよりアジャイル的な手法を要求しているといえます。

そのような市場の要請のもと、現在ではUnityなどの一部を除くすべての基本的なUIフレームワークは宣言的UIを採用するに至りました。AndroidではJetpack Compose、iOSではSwift UI、クロスプラットフォームではReact Nativeも、そして当然Flutterも。

そもそも宣言型UIとはなんなのでしょうか。これを用いればどのようなメリットがあるのでしょうか。なぜ市場は宣言型UIで開発することを要求して、それが変化に対応できるアプリになるというのでしょうか。

本書はそのような視点で、宣言型UIの根本的な性質と、状態管理の原理原則、設計哲学について述べていきます。



## 命令的設計からの脱却と根本的変換

これまでのUI構築の歴史を振り返ってみましょうか。

### 原始的手続型パラダイム

原初的には、画面とは手続的に、直接操作するものでした。大昔のBASICのソースコードを考えてみましょう。

```basic
10 LET SELECT_INDEX = 3
20 FOR I = 1 TO 5
30   IF I = SELECT_INDEX THEN
40     PRINT "> 行"; I
50   ELSE
60     PRINT "  行"; I
70   END IF
80 NEXT I
90 LINE (5,60)-(120,60)
100 FOR I = 1 TO 5
110   X = 130
120   Y = 10 + (I - 1) * 10
130   PRINT TAB(18); "▶"
140 NEXT I
```

このように、BASICでは各命令を行番号とともに順序立てて記述し、プログラムの実行は上から下へと進みます。  
**どのような手順で何をするか**を逐次的に指定するため、手続き的（命令的）なプログラミングスタイルの典型例です。

かつてこの世界には、UIフレームワークも状態管理パラダイムもモジュール結合度の概念もなく、ただ上から下へ記述することによってUIが成立していました。

### イベント駆動型手続的パラダイム

時代はくだり、Visual Basic .NETや、JavaのSwing、AndroidのActivityといった概念は、オブジェクト指向的にUIを設計できるようにしました。MVC、MVVMといったバックエンドで使われてきたモデルをフロントエンドにも少しずつ応用しようと試行錯誤されてきました。

これはある意味で現在のUI設計の癌であり、未だに画面とはこのように記述されると狂信的に信奉している者も少なくないでしょう。しかしながら、**我々はもう、保守以外でこの時代に生きてはなりません**。

たとえばJava - Swingであれば、明示的にUIコンポーネントを生成し、一つ一つ操作を記述してきました。

```java
import javax.swing.*;
import java.awt.event.*;

public class CounterApp {
    public static void main(String[] args) {
        JFrame frame = new JFrame("カウンター");
        JButton button = new JButton("増やす");
        JLabel label = new JLabel("カウント: 0", SwingConstants.CENTER);

        final int[] count = {0}; // イベントリスナー内で使うため配列で定義

        button.addActionListener(new ActionListener() {
            public void actionPerformed(ActionEvent e) {
                count[0]++;
                label.setText("カウント: " + count[0]);
            }
        });

        frame.setLayout(new java.awt.BorderLayout());
        frame.add(label, java.awt.BorderLayout.CENTER);
        frame.add(button, java.awt.BorderLayout.SOUTH);

        frame.setSize(200, 120);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.setVisible(true);
    }
}
```

または、VBではどうだったでしょうか。これもやはり、根本的には直接UIを操作することに代わりはありません。

```vbnet
Public Class Form1
    Private count As Integer = 0
    Private label As New Label()
    Private button As New Button()

    Private Sub Form1_Load(sender As Object, e As EventArgs) Handles MyBase.Load
        label.Text = "カウント: 0"
        label.TextAlign = ContentAlignment.MiddleCenter
        label.Dock = DockStyle.Top
        label.Height = 40

        button.Text = "増やす"
        button.Dock = DockStyle.Bottom
        AddHandler button.Click, AddressOf Button_Click

        Me.Controls.Add(label)
        Me.Controls.Add(button)
        Me.Text = "カウンター"
        Me.Size = New Size(200, 120)
    End Sub

    Private Sub Button_Click(sender As Object, e As EventArgs)
        count += 1
        label.Text = "カウント: " & count.ToString()
    End Sub
End Class
```

これらはオブジェクト指向的にはUIと状態がいずれもモジュール結合度的に密結合です。また一部分を再利用することも、Viewだけではできたかもしれませんが、背後の状態と結合していたりして難しかった側面があります。これを解決しようと様々な試行錯誤を経て、「リアクティブ」という概念が生まれることになりました。

### リアクティブパラダイム

時代を経るにつれて、状態の流れは一方向であるべき、モジュール結合度が高いと保守性やテスト可能性を損なうといった考え方が普及しました。AngualrJSやWPF、Android(Java/Kotlin)のLiveDataがそのような流れに沿って、双方向のリアクティブ的UIによる画面構築を可能として、より効率的に状態とUIを管理することを目指しました。

たとえば、AndroidではLiveDataとBindingを用いると、以下のようになります。

**CounterViewModel.kt**
```kotlin
import androidx.lifecycle.LiveData
import androidx.lifecycle.MutableLiveData
import androidx.lifecycle.ViewModel

class CounterViewModel : ViewModel() {
    private val _count = MutableLiveData(0)
    val count: LiveData<Int> = _count

    fun increment() {
        _count.value = (_count.value ?: 0) + 1
    }
}
```

**activity_main.xml**
```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">
    <data>
        <variable
            name="viewModel"
            type="com.example.CounterViewModel" />
    </data>
    <LinearLayout
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:gravity="center">

        <TextView
            android:id="@+id/countText"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text='@{String.valueOf(viewModel.count)}'
            android:textSize="24sp" />

        <Button
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="増やす"
            android:onClick="@{() -> viewModel.increment()}" />
    </LinearLayout>
</layout>
```

このように、ViewModelの`count`をLiveDataとして公開し、`increment`メソッドで値を更新します。XMLではDataBindingを使い、`viewModel.count`を監視してUIを自動更新します。

これは状態と画面を分離して、お互いを知りすぎないように、意識させないようにする設計でした。これは従来より開発生産性を上げるものでしたが、それでも限界がありました。

たとえば単一のViewModelが無関係の責務の集合体になったり、画面全体を更新するためパフォーマンスの劣化を招いたりすることがありました。

### コンポーネント指向と宣言型UI

そしてさらに時代はくだり、React(2013年一般公開)を先駆けとして様々な「宣言型UI」を標榜するフレームワークが登場しました。WebではReactのほかにVue.js、AndroidはJetpack Compose、iOSもSwift UIがそうです。

ここには、これまで登場しなかった「関数型プログラミング」の観念を応用し、UIとはそもそもどのように構築されるべきであるかという常識を覆しました。しかしながら一方で、その抽象的観念についてこれず、あるいは応用がうまくいかず失敗した例も数多と存在します。

さらに、単に宣言型というだけでなく、その宣言型UIにおける状態管理、副作用管理も変遷を経て、より洗練された形で現在に至ります。これにはReact Hooks(2019年)もまた転換期といえるでしょうか。

| ライブラリ　| 登場年　| 概要　|
| --- | --- | --- |
| Recoil | 2020年5月 | Meta公式ライブラリ。「原子状態モデル」を提示　|
| Jotai | 2020年10月 | Recoilに触発されより軽量・直感的な設計 |
| Pinia | 2021年3月　| Vue 3における推奨 |
| Riverpod | 2021年9月 | Providerの後継。現在のFlutterでのデファクトスタンダード　|

これらは、これまでの無関係な機能と責務の集中に対して、いずれもより原子的な、小さい単位とサイクルに基づく状態管理を行うアプローチを提示するか、その概念に強い影響を受けています。

次の章では、この宣言型やそれに伴う状態管理において、特に発祥となるReact、あるいはさらに影響を受けたHaskellの精神を、Reactのみならずより普遍的な内容として捉えるための、数理的基礎理論を学んでいきましょう。Reactが発祥であるというのは事実であるものの、そこから影響を受けたすべての宣言型UIフレームワークにおいて、どれにでも適用できるはずの理論です。