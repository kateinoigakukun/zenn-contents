---
title: "SwiftWasm 5.3 リリースノート"
emoji: "✨"
type: "tech"
topics: [Swift, SwiftWasm, WebAssembly]
published: true
---

SwifWasmの最初の安定したツールチェーンがリリースされました。この文書は[SwiftWasm Blogのリリースノート](https://blog.swiftwasm.org/posts/5-3-released/)を日本語訳したものです。

## Overview


今回のリリースはSwiftWasmツールチェーンの最初の安定バージョンリリースです。

macOS用の署名済み.pkgインストーラーとして提供されており、またIntelベースのUbuntu 18.04と20.04用のswiftenv互換アーカイブとDockerイメージしても提供されています。

このバージョンではWebAssembly上で安定したSwift機能を提供することに重点が置かれています。


Swift for WebAssemblyコンパイラ、標準ライブラリとコアライブラリ、JavaScript相互運用ライブラリ、UIライブラリ、ビルドツール、CIサポートが含まれています。


Swift Package Managerでライブラリを提供している場合、わずかな変更でWebAssembly対応出来る可能性があります。ぜひ試してみてください。


## 標準ライブラリとコアライブラリ

Swift標準ライブラリはWebAssembly上でフルに利用可能になっています。

標準ライブラリは現在、WebAssembly用のシステムインターフェースであるWASIに依存しており、[wasi-libc](https://github.com/WebAssembly/wasi-libc)とベースにビルドされています。将来的にWASIへの依存はオプショナルにする予定です。


### Foundation / XCTest

Foundation と XCTest は一部機能が制限されていますが、WebAssemblyでも利用可能になっています。利用できないAPIに関してはドキュメントを参照してください。

- https://book.swiftwasm.org/getting-started/foundation.html
- https://book.swiftwasm.org/getting-started/testing.html


## JavaScriptの相互運用ライブラリ

[JavaScriptKit](https://github.com/swiftwasm/JavaScriptKit)は、WebAssemblyを介してJavaScriptを操作するためのSwiftライブラリです。

このライブラリを使えば、JavaScriptのあらゆるAPIをSwiftから利用することができます。ブラウザアプリでのJavaScriptKitの使用例を簡単に紹介します。

```swift
import JavaScriptKit

let document = JSObject.global.document
var divElement = document.createElement("div")
divElement.innerText = "Hello, world"
_ = document.body.appendChild(divElement)
```

詳細は https://book.swiftwasm.org/getting-started/javascript-interop.html をご覧ください。


## UI library

[Tokamak UI framework](https://tokamak.dev) は、SwiftUIのAPIをクロスプラットフォーム向けに実装したものです。現在はWebAssembly/DOMターゲットとmacOS/Linux上での静的HTMLレンダリングをサポートしています。

[Tokamakを使って簡単なWebアプリを作るためのガイド](https://book.swiftwasm.org/getting-started/browser-app.html)をご覧ください。



## All-in-one builder, test runner, and bundler for SwiftWasm

[carton](https://github.com/swiftwasm/carton)はSwiftWasm向けに設計されたビルドツールです。webpack.jsのようなものですが、設定や複雑な依存関係はありません。また、ツールチェーンを自動的にインストールしてくれるので、SwiftWasmを始めるのに一番簡単な方法です。


## CI support

SwiftWasmプロジェクトでは、SwiftWasmツールチェーンを使って継続的インテグレーションを行うための[GitHub Action](https://github.com/swiftwasm/swiftwasm-action)も提供しています。
