# @Environment(\\.keyPath)実践入門

<div class="author-info">
星野恵瑠<br />
Twitter: @lovee<br />
Bluesky: @lovee.bsky.social<br />
Misskey: @lovee@misskey.io<br />
GitHub: @el-hoshino
</div>

## はじめに

ご存知の通り、SwiftUIはState-Drivenのアプリフレームワークです。それはつまり「State」、すなわち「状態」の管理が非常に重要だということです。ところがその状態管理に、多くの人は`@State`や`@Binding`を使っていることが多いと見受けますが、`@Environment`を活用している人は少ないように感じます。この記事は、そんな`@Environment`を布教したいと考え、執筆に至りました。

ちなみに、`@Environment`といっても、実はこれは2種類あって、最初のSwiftUIから使える`@Environment(\.keyPath)`のものと、iOS 17から使えるようになった`@Environment(ObservableType.self)`のものです。本記事は敢えて前者のみ扱い、特別に言及しない限り`@Environment`といえば`@Environment(\.keyPath)`のことを指します。

<div class="column">

`@Environment(ObservableType.self)`を敢えて扱わないのはいくつか理由があります。内訳はまた本記事の最後に述べさせていただきたいですが、根本的にはこのObservableTypeの方にしかないメリットが全く感じないからです。両方使っていい場面もあれば、KeyPathを使った方がいい場面もありますが、逆にObservableTypeを使った方がいい場面が一つもない上、むしろ使い方をミスるとビルド時に検出できずランタイムエラーでクラッシュまでします。だったらもうKeyPathだけ使えばええんちゃう？と思ってしまうのです。

</div>

## `@Environment`とは

`@Environment`は、状態を環境変数として扱うためのプロパティラッパーです。実は普段のアプリ開発で意識せずに使ってる人も多いかなと思います。例えば、`@Environment(\.colorScheme)`を使えば、今がダークモードかライトモードかを取得できます。他にも、`@Environment(\.locale)`を使えば、端末のロケールを取得できます。なので、`@Environment`は実は非常に身近な存在なのです。

でも、`@Environment`がこれだけだと思ってしまっては困ります！実はこの`@Environment`、うまく活用すれば非常に強力なツールになります！

### `@Environment`の状態伝搬のイメージ

`@Environmnet`の活用法を紹介する前に、まずは基礎を押さえておきたいと思います。我々はどんなときに`@Environment`を使いたくなりますか。`@Environment`はどんな課題を解決しようとしているのか。

通常の`@State`などの場合、状態の伝搬は明示的にイニシャライザーを通じて伝播していきます。例えば下記のようなカウンターを表示するコードを考えてみましょう。

```swift
struct CountView: View {
    var count: Int

    var body: some View {
        Text("\(count)")
    }
}

struct ContentView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            CountView(count: count)
            Button("Increment") {
                count += 1
            }
        }
    }
}
```

このコードでは、`CountView`として必要な`count`状態は、親の`ContentView`が`@State`として保持しており、そして`CountView`を生成するときにその`count`を渡しています。このように、`@State`は親から子へと状態を伝播していくわけです。

ところが、もし`ContentView`が直接`CountView`を持つわけではなく、間に別のビューが挟まる場合、この状態の伝搬は少し面倒になります。例えば下記のような構成を考えてみましょう。

```swift
struct ChildView: View {
    var body: some View {
        CountView(count: /* ここに何を書く？ */) // ここでcountを渡したい
    }
}

struct ContentView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            ChildView() // ここでCountViewではなくChildViewを持つ
            // ...
        }
    }
}
```

そうです、正攻法でやると、孫の`CountView`に`count`を渡すためには、`ChildView`にも`count`を渡す必要があります。

```swift
struct ChildView: View {
    var count: Int // ここにcountを保持する

    var body: some View {
        CountView(count: count)
    }
}
```

なんだ大したことないじゃん、と思うかもしれませんが、今回はあくまで仕組みのために単純化した例です。実際の開発では、間に挟まってるビューが一つだけとは限らないし、どこにどれだけの状態を必要としているのかもわかりません。そうなると、状態の伝搬は非常に面倒になります。ここで`@Environment`が登場します。

`@Environment`は、状態の伝搬を自動で行ってくれるプロパティラッパーです。`@Environment`を使えば、直接の子供だけでなく、自分の孫、曾孫、ひ孫、ひひ孫、……といった、どこまで行っても状態が伝搬されるのです。これにより、どの子供がどの状態を必要としているのかを意識せずに、状態を伝搬させることができるのです。先ほどの`@Environment(\.colorScheme)`を思い出してみましょう。これがまさにアプリの大元が持っているカラースキーム状態を、アプリ内の全てのビューに伝搬させている例です。

### `@Environment`のメリット

筆者が思う`@Environment`のメリットは、主に3つあると考えています。

1. **状態の伝搬が楽**

   先ほども述べた通り、`@Environment`を使えば、状態の伝搬を自動で行ってくれます。これにより、間にビューを挟みたいとき、いちいち状態を持さずに済むので、コードがシンプルになります。

1. **共有されるのはゲッターのみ**

   `@Environment`は、基本的にゲッターのみを公開されます。これにより、状態の変更は状態を保持しているビューでのみ行われるため、状態変更のトレースが行いやすいです。

1. **必ず値が存在する**

   もし`@Environment`で指定した状態の型がOptionalじゃなければ、値が必ずあるので、実行時に値が存在しないことによるクラッシュが発生しません。

### `@Environment`のデメリット

もちろん`@Environment`も万能ではありません、デメリットもあります。

1. **状態が必ずモジュール全体に渡って共有されます**

   実際のアプリ開発では、例えば特定の機能だけに関係するすべてのビューにこの状態を共有したい、もしくは単純に特定のビューのすべての子供にこの状態を共有したい、といったことも多々あると思います。ところが残念ながら、`@Environment`はあくまで「環境変数」の一種なので、使い出したらその状態は必ずモジュール全体がアクセス可能になってしまいます。

1. **…他にあるかな？**

   ぶっちゃけ、上記のように狭い範囲での状態共有に不向き以外、デメリットあまりない気がしますよね…

## `@Environment`の基礎的な使い方

というわけで、`@Environment`の得手不得手を理解したところで、実際に使い方を見ていきましょう。

### カスタム`@Environment`の作り方

まずは、`@Environment`を使うために、カスタム`@Environment`環境変数を作る方法を見ていきます。`@Environment`は`\.keyPath`の形で使いますが、これは`EnvironmentKey`というプロトコルを満たす型を使うことで値の読み書きを実現しています。なので、まずは`EnvironmentKey`を満たす型を作ります。今回は例としてカウンターを表示するので、`count`を扱う`CountKey`という型を作ってみましょう。

```swift
struct CountKey: EnvironmentKey {
    static var defaultValue: Int = 0
}
```

`CountKey`は、`defaultValue`というプロパティを持っています。これは何も設定されてない時に返されるデフォルト値です。今回は`Int`型のデフォルト値を`0`にしています。

次に、この`CountKey`を使って、`@Environment`を作ります。

```swift
extension EnvironmentValues {
    var count: Int {
        get { self[CountKey.self] }
        set { self[CountKey.self] = newValue }
    }
}
```

`EnvironmentValues`は、`@Environment`で使うためのプロパティをアクセスする型です。この`EnvironmentValues`に、`count`というプロパティを追加することで、`@Environment`で`\.count`というKeyPathが使えるようになります。そしてこの`count`は、`CountKey`を使って値の書き込みと読み取りができるようになります。

<div class="column">

上記のコードはXcode 15までの実装ですが、執筆締め切り現在の2024年6月20日では、Xcode 16 Betaからは`@Entry`を使ってより簡単にカスタム`@Environment`を作れるようになりました。`@Entry`を使った`\.count`の実装は下記のようになります。

```swift
extension EnvironmentValues {
    @Entry var count: Int = 0
}
```

そうです、これだけです。`@Entry` Macroのおかげで、`EnvironmentKey`の定義やデフォルト値の設定、そして`EnvironmentValues`へプロパティー追加する時のゲッターとセッターの実装がすべて自動的に行うようになったので、非常に簡単にカスタム`@Environment`を作れるようになりました。またこの機能はあくまでSwiftの言語機能として追加されたものなので、Xcode 16以降のバージョンであれば、iOS 18以前でも使えて、とても便利です。

ただしあくまでBeta機能なので、正式リリースされる時には具体的な実装が変わる可能性があります。その際は公式ドキュメントを参照してください。

</div>

### `@Environment`を使った状態の書き込み方

`\.count`という`@Environment`用のKeyPathを作りましたので、値の書き込みは`.environment(_:_:)` Modifierが使えます。このModifierは2つの引数を取ります。1つ目の引数は`@Environment`用のKeyPathで、2つ目の引数は書き込む値です。例えば今回は`\.count`にカウントを書き込むので、先ほどのカウンターの表示の例を流用するとこんな風に使えます。

```swift
struct ContentView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            ChildView()
            // ...
        }
        .environment(\.count, count) // ←ここでcountを書き込む
    }
}
```

これで、`VStack`以下の全てのビューに`\.count`環境変数を`count`に設定しました。

### `@Environment`を使った状態の読み取り方

次は環境変数の読み取り方ですが、これは`@Environment(\.keyPath)`の形で使うので、`\.count`を読み取るなら`@Environment(\.count)`と書けばOKです。同じく先ほどのカウンターの表示の例を流用すると、`CountView`で`\.count`を読み取るならこんな感じです。

```swift
struct CountView: View {
    @Environment(\.count) var count : Int //←ここで型の宣言はしてもしなくてもOKです

    var body: some View {
        Text("\(count)")
    }
}
```

ここで注意して欲しいのは、`\.count`環境変数をセットしたのは`ContentView`で、`ContentView`が直接持つ子供`ChildView`はこの`\.count`について一切意識していないということです。しかし`CountView`は`ChildView`の子供なので、つまり`CountView`は`ContentView`の孫にあたります。だから`CountView`は`ContentView`の`\.count`環境変数を読み取ることができるのです。

<div class="column">

`@Environment`で指定した環境変数の型は、KeyPath指定した時点で既に決まっているため、基本的に型の宣言は任意です。筆者の場合は、型を宣言することで、コードの可読性が上がると感じているのと、そもそも他のプロパティーと違って型宣言を省略するのは気持ち悪く感じているので、基本的には型宣言をしています。

</div>

全部のコードを載せるとこんな感じです。

```swift
struct CountKey: EnvironmentKey {
    static var defaultValue: Int = 0
}

extension EnvironmentValues {
    var count: Int {
        get { self[CountKey.self] }
        set { self[CountKey.self] = newValue }
    }
}

/* Xcode 16以降の場合、上記のすべての実装は下記のように簡潔にまとめられます。
extension EnvironmentValues {
    @Entry var count: Int = 0
}
*/

struct ContentView: View {
    @State private var count = 0

    var body: some View {
        VStack {
            ChildView()
            Button("Increment") {
                count += 1
            }
        }
        .environment(\.count, count)
    }
}

struct ChildView: View {
    var body: some View {
        CountView()
    }
}

struct CountView: View {
    @Environment(\.count) var count

    var body: some View {
        Text("\(count)")
    }
}
```

## `@Environment`のより高度な使い方

ここまでで、`@Environment`の基礎的な使い方を見てきました。しかし、筆者がこれだけ`@Environment`を推したいのはもちろんこれだけではありません。これからは、`@Environment`のより高度な使い方を見ていきましょう。

### KeyPathの連結

`@Environment`はKeyPathを受け取るってことは、当然ながらKeyPathを連結したKeyPathも使えるわけです。これは複雑な状態の塊から、必要な部分だけを取り出したいときに非常に便利です。例えば`CGRect`の環境変数があって、その中の`midX`だけを使いたい時と考えましょう。もしこの連結を知らなかったら、こんな風に書いているのではないかと思います：

```swift
struct ContentView: View {
    @Environment(\.rect) var rect: CGRect
    var midX: CGFloat { rect.midX }
    // ...
}
```

しかしKeyPathの連結を使えば、こんな風に書けます：

```swift
struct ContentView: View {
    @Environment(\.rect.midX) var midX: CGFloat
    // ...
}
```

### 敢えてセッターを公開したい

メリットの章に書いた通り、`@Environment`は基本的にゲッターのみ共有されますが、実は見方を変えれば、「セット処理」を「変数」とみなせば、そのセット処理自身のゲッターを公開することで、実質セッターのみ公開しているようなこともできます。

ただし気をつけないといけないのは、「処理」のことを言ったら、「Closure」を思い浮かぶ人が多いかと思いますが、SwiftUIの仕組みの都合で、Closureのような匿名な参照型を使うと無駄なレンダリングが発生する可能性があります。そのため、Swift 5.2から導入された`callAsFunction`を使った`struct`で処理を書きましょう。例えばリストから選択されたIndexを、リスト内の各子ビューに設定させたい時、こんな感じで使えます。

```swift
struct SetSelectedIndexAction {
    private var selectedIndex: Binding<Int?> // ←誤用を防ぐために全てのプロパティーを外部から隠す
    init(selectedIndex: Binding<Int?>) {
        self.selectedIndex = selectedIndex
    }
    func callAsFunction(_ index: Int) {
        selectedIndex.wrappedValue = index
    }
}

struct SetSelectedIndexKey: EnvironmentKey {
    static var defaultValue: SetSelectedIndexAction = .init(selectedIndex: .constant(nil))
}

extension EnvironmentValues {
    var setSelectedIndex: SetSelectedIndexAction //...省略
}

struct ContentView: View {
    @State private var selectedIndex: Int?

    var body: some View {
        VStack {
            ForEach(0..<10) { i in
                MyCell(index: i)
                    .background(selectedIndex == i ? Color.red : Color.clear)
            }
        }
        .environment(\.setSelectedIndex, .init(selectedIndex: $selectedIndex))
    }
}

struct MyCell: View {
    var index: Int
    @Environment(\.setSelectedIndex) var setSelectedIndex

    var body: some View {
        Button("Select") {
            setSelectedIndex(index)
        }
    }
}
```

上記の例では、リスト内の子ビュー`MyCell`は、実際に`selectedIndex`の値を知りませんが、`setSelectedIndex`を使って`selectedIndex`を変更することができます。これにより、`ContentView`が持つ`selectedIndex`の値を、`MyCell`が変更できるようになります。

もちろん現実では`selectedIndex`の設定は他にいくらでもやり方はありますし、そもそも設計としてこれはあまりいい例として言いにくいです。これはあくまで使い方の例として、イメージしやすいように書いたものですが、これで`@Environment`の使い方の幅が広がることを感じていただければ幸いです。また他に、Closureの代わりに`Binding`を使うことで、実質ゲッターとセッター両方公開するようなこともできます。

<div class="column">

個人的な宣伝になってしまいますが、実はSwiftUIでToastを簡単に表示するライブラリー`Tardiness`を作りました。そのライブラリーの中で、あらゆる画面からToastの文言をセットできるようにするために、この`callAsFunction`方式の`@Environment`を使いました。もし興味があれば、ぜひこちらのリポジトリーを見てみてください：<https://github.com/el-hoshino/Tardiness>。

</div>

### 同じ環境変数にビューヒエラルキーに応じて違う値を混在させる

`@Environment`を使えば、状態を環境変数としてアプリ全体に共有することになるから、アプリ全体が一つだけの状態を共有することになると思うかもしれませんが、実は違います。`@Environment`の状態はビューヒエラルキーで継承されており、すなわち途中で書き換えれば、そのビューヒエラルキー以下のビューにだけその書き換わった値が適用されるということです。

実はこの特性は、無意識のうちに使っている人も多いではないかと思います。画面遷移の後、前の画面に戻る時は`@Environment(\.dismiss)`を使っているではないでしょうか。そうです、この`\.dismiss`はまさにこの特性を利用しています。何重もの画面遷移をしても、`\.dismiss`を使えば、ちゃんと自分自身だけが消えて他のビューはそのまま残るのは、まさに自分自身が読み取った`\.dismiss`環境変数の値が自分自身に合わせて書き換えられたものだからです。

ではこの特性を使って、ビューヒエラルキーに応じて違う値を混在させる例を見てみましょう。

```swift
struct CopyrightKey: EnvironmentKey {
    static var defaultValue: String = "@lovee"
}

extension EnvironmentValues {
    var copyright: String //...省略
}

struct CopyrightView: View {
    @Environment(\.copyright) var copyright: String
    var body: some View {
        Text(copyright)
    }
}

struct ContentView: View {
    @Environment(\.copyright) var copyright: String
    var body: some View {
        VStack {
            Text(copyright)
            
            CopyrightView()
            
            CopyrightView()
                .environment(\.copyright, "星野恵瑠")
        }
        .environment(\.copyright, "el-hoshino")
    }
}
```

上記のコードで、`ContentView`を表示したら、このような結果になるでしょう。

```text
@lovee
el-hoshino
星野恵瑠
```

さて、なぜこうなるかを具体的にみていきましょう。

まずは最後の`CopyrightView`を見ましょう。このビューに直接`.environment`で`\.copyright`を`星野恵瑠`にセットしましたので、このビューは`星野恵瑠`を表示します。

次にその上の`CopyrightView`を見ましょう。このビューは特に何もセットされていません。なので自分自身の親、すなわち`VStack`を見ます。`VStack`には`.environment`で`\.copyright`を`el-hoshino`に設定されているので、このビューは`el-hoshino`を表示します。

最後に一番上の`Text(copyright)`を見ましょう。このビューは直接`ContentView`が持つ`\.copyright`を読み取っています。`ContentView`は`VStack`に対して`\.copyright`をセットしましたが、逆にいうと自分自身には何もセットしていないし仕組み上できないので、デフォルト値の`@lovee`が表示されます。

このように、ビューヒエラルキーに応じて違う値を混在させることができるのが`@Environment`の特性です。この特性を活かせば、幅広い表現ができるでしょう。例えば特定のビューヒエラルキーにだけダークモードだけ有効にしたい時、そのビューに`.environment(\.colorScheme, .dark)`をセットすれば実現できます。

## あとがき

いかがでしたでしょうか。`@Environment`を活用すると、いろんなことが楽にできることを感じていただけたかと思います。

最後に、この記事が敢えて`\.keyPath`の方だけ取り上げる理由を少し説明したいと思います。ここまで読んでわかっていただいた通り、`@Environment(\.keyPath)`は非常に強力でありながら、使い方もそんなに難しくないです。では`@Environment(ObservableType.self)`はどうでしょうか。少なくとも、筆者からしてみたらいくつか致命的な問題点があると言わざるを得ないです。

まずははじめにに書いた通りランタイムエラーのリスクがあります。`\.keyPath`の方はデフォルト値を利用したり、そもそもOptionalにすれば、ランタイムエラーは簡単に回避できるし、仮に設定し忘れを防ぎたい場合もデフォルト値でアサートを入れれば本番アプリに影響与えずに検出できますが、`ObservableType.self`の方はそうはいきません。設定し忘れたら終わりです。

次に抽象の型、つまり`protocol`をそのまま使えないのも大きな問題点だと思います。具象だけしか使えないと、開発環境だったりテストの時に、オブジェクトの置き換えがだいぶ難しくなります。

そして最後に`ObservableType.self`は、オブジェクトのインスタンスを入れるだけでセットするので、極端な例になりますが、サブクラスを作ってしまった時、代入したインスタンスはスーパークラスのインスタンスを指しているのかサブクラスのインスタンスを指してるのか曖昧になってしまうし、この作りはそもそも悪いのはもちろんですがこれを技術的に防げないのが問題だと思います。

その点、`\.keyPath`の方は、そもそもKeyPathを指定して使うので、そういった曖昧性もなければ、抽象も思うがまま使えます。何より、`ObservableType.self`が使える場面は、全て`\.keyPath`でも代替できちゃいます。じゃあ`\.keyPath`を使った方がいいじゃないでしょうか。

というわけで、`@Environment(\.keyPath)`の布教記事でした。ぜひ`@Environment`を活用して、SwiftUIの開発をもっと楽しんでください！
