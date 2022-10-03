---
title: "SwiftWasmではじめるWebアプリ開発"
emoji: "🌐"
type: "tech"
topics: [Swift, SwiftWasm, WebAssembly, Web]
published: true
---

この記事ではSwiftWasmでのWebアプリ開発のはじめかたを説明します。SwiftWasmの技術的背景について軽く説明し、ローカル環境でシンプルなアプリを作成するところまでを目標とします。

## SwiftWasmとは

> WebAssembly はモダンなウェブブラウザーで実行できる（略）ネイティブに近いパフォーマンスで動作する（略）アセンブリ風言語です。（略） C/C++ や Rust のような言語のコンパイル対象となって、それらの言語をウェブ上で実行することができます。


SwiftWasmはSwiftのWebAssemblyサポートを進めるプロジェクトです。

WebAssembly にコンパイルすることで、ブラウザ上でSwiftを実行することができます。

https://github.com/swiftwasm

## シンプルなアプリを作ってみる

### 0. SwiftWasmのインストール

SwiftWasmのアプリをビルドする際には [carton](https://github.com/swiftwasm/carton) というツールを使います。まずはcartonをインストールします。

```bash
$ brew install swiftwasm/tap/carton
```

Linux環境の場合、Dockerイメージを使うのがお手軽だと思います。

```bash
$ docker run -it --rm -p 8080:8080 -v $(pwd):$(pwd) -w $(pwd) ghcr.io/swiftwasm/carton
```

### 1. プロジェクトを作る

さて、プロジェクトを作っていきましょう。

以下のようにプロジェクトディレクトリを作成し、その中で `carton init` コマンドを実行します。 `--template tokamak` は後に使用するTokamakというUIフレームワーク向けにプロジェクトを生成するためのオプションです。

```bash
$ mkdir SampleApp
$ cd SampleApp
$ carton init --template tokamak
```

以下のようなプロジェクト構成になります。SwiftPMパッケージと同様の構成になっています。

```bash
$ tree
.
├── Package.swift
├── README.md
├── Sources
│   └── SampleApp
│       └── App.swift
└── Tests
    └── SampleAppTests
        └── SampleAppTests.swift
```

`App.swift`の内容は以下のようなSwiftUIに似たコードになっています。

```swift
import TokamakDOM

@main
struct TokamakApp: App {
    var body: some Scene {
        WindowGroup("Tokamak App") {
            ContentView()
        }
    }
}

struct ContentView: View {
    var body: some View {
        Text("Hello, world!")
    }
}
```



### 2. 動かしてみる

プロジェクトを作ったばかりですが、早速動かしてみましょう。プロジェクトディレクトリで `carton dev` コマンドを実行することでアプリを実行することが出来ます。 [http://127.0.0.1:8080](http://127.0.0.1:8080) にアクセスすると実際に動作している様子が確認できます。

> **Warning**
> Dockerコンテナ上で実行する場合はコンテナ外からのリクエストを受け付けるために `carton dev --host 0.0.0.0` を実行してください。

初回実行時にツールチェーンをインストールするので2~3分時間がかかるので注意してください。

```bash
$ carton dev
...
[ NOTICE ] Server starting on http://127.0.0.1:8080
```

正しく動作するとこのような表示になります。実にシンプルですね。

![](https://storage.googleapis.com/zenn-user-upload/0xztp3t9vwwkexk7e29x7xkk9o8b)


### 3. ToDoアプリにしてみる

このままではシンプルすぎるのでToDoアプリに転生させてみましょう。

プロジェクト作成時に指定した [Tokamak](https://github.com/TokamakUI/Tokamak) はSwiftUI互換のUIライブラリとして開発されており、SwiftUIとほぼ同じ書き味でアプリを組み立てることが出来ます。

楽しくなってきましたね。

```swift
import TokamakDOM

@main
struct TokamakApp: App {
    var body: some Scene {
        WindowGroup("Tokamak App") {
            ContentView()
        }
    }
}

struct Item {
    var isCompleted: Bool = false
    var text: String
}

struct ContentView: View {
    @State var newItem = ""
    @State var items = [Item]()

    func addNewItem() {
        items.append(Item(text: newItem))
        newItem = ""
    }

    var body: some View {
        VStack(alignment: .leading) {
            HStack {
                Button("+", action: addNewItem)
                TextField("New todo item", text: $newItem, onCommit: addNewItem)
            }
            List {
                ForEach(0..<items.count, id: \.self) { i in
                    Toggle(
                        items[i].text,
                        isOn: Binding(get: { items[i].isCompleted }, set: { items[i].isCompleted = $0 })
                    )
                }
            }
        }
    }
}
```


## まとめ

この記事は以下のリポジトリのREADME.mdに書いてる内容を日本語で紹介したに過ぎませんが、少しでもプロジェクトに興味を持ってもらえれば嬉しいです。

- https://github.com/TokamakUI/Tokamak
- https://github.com/swiftwasm/carton


現状本当に手が足りていないので助けてください
