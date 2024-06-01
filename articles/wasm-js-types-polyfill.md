---
title: "Wasm Type Reflection JS APIのPolyfillを書いた"
emoji: "🩹"
type: "tech"
topics: [WebAssembly, JavaScript]
published: true
---

[WebAssembly Type Reflection JS API](https://github.com/WebAssembly/js-types/blob/main/proposals/js-types/Overview.md)と呼ばれるAPIのPolyfillを書いたので、その紹介です。WebAssemblyでスレッドを扱う際に少し便利になります。

https://github.com/swiftwasm/wasm-imports-parser

## WebAssembly Type Reflection JS APIとは

WebAssembly Type Reflection JS APIは、WebAssemblyモジュールからimport/exportされる関数のシグネチャやメモリサイズの情報を取得するためのJavaScript APIです。
WebAssemblyモジュール自体にはそのような情報がもともと含まれており、処理系の中でも使っていたのですが、JavaScriptからその情報を取得するためのAPIが無かったため、新たにAPIが提案されました。

https://github.com/WebAssembly/js-types

```js
const module = await WebAssembly.compile(buffer);
for (const importEntry of WebAssembly.Module.imports(module)) {
  console.log(importEntry.module, importEntry.name, importEntry.kind);
  console.log(importEntry.type); // <- New
}
```

新たに追加された `type` プロパティには `kind` に応じた以下のような型情報が含まれます。

```ts
type RefType = "funcref" | "externref"
type ValueType = "i32" | "i64" | "f32" | "f64" | "v128" | RefType
type GlobalType = {value: ValueType, mutable: boolean}
type MemoryType = {limits: Limits}
type TableType = {limits: Limits, element: RefType}
type Limits = {min: number, max?: number}  // see below
type FunctionType = {parameters: ValueType[], results: ValueType[]}
type ExternType =
  {kind: "function", type: FunctionType} |
  {kind: "memory",   type: MemoryType} |
  {kind: "table",    type: TableType} |
  {kind: "global",   type: GlobalType}
```

## スレッドとの関係

現在、WebAssemblyでスレッドを扱うためには、複数のWebAssemblyインスタンスを複数のWeb Workerで動かすという方法が一般的です。
この場合、全てのインスタンスで同じ線形メモリを共有する必要があるため、スレッドを有効にしたWebAssemblyモジュールは線形メモリを外部からimportします。

```wasm
(module
  (import "env" "memory" (memory 1 1 shared))
  ...
)
```

このようなモジュールをインスタンス化する準備として、インスタンスに注入するための線形メモリインスタンスを作成する必要があります。
この際、線形メモリの初期ページ数や最大ページ数を指定する必要があります。これらのサイズがWebAssemblyモジュールが期待するサイズを下回っていると、インスタンス化に失敗するため、事前にモジュールが期待するサイズを調べておく必要があります。

ここで、WebAssembly Type Reflection JS APIが役立ちます。モジュールがimportするメモリのサイズを取得することができるため、それに合わせて線形メモリを作成することができます。

```js
const module = await WebAssembly.compile(buffer);
const memoryType = WebAssembly.Module.imports(module).find(entry => entry.kind === "memory").type;
const memory = new WebAssembly.Memory({
    initial: memoryType.minimum, maximum: memoryType.maximum, shared: true
});
const instance = await WebAssembly.instantiate(module, {env: {memory}});
```

WebAssembly Type Reflection JS APIがない場合、Wasmバイナリを事前に解析しておいたり、Emscriptenのようにそもそもビルド時に静的なサイズを指定しておく必要があり、なかなか面倒です。

## WebAssembly Type Reflection JS APIの実装状況

- V8: 実装済み
    - https://chromestatus.com/feature/5725002447978496
    - Chrome 78から利用可能
    - Node.js 13.0.0から利用可能
        - おそらく[このコミット](https://github.com/nodejs/node/commit/f7f6c928c1c9c136b7926f892b8a2fda11d8b4b2)で追加されていそう
- SpiderMonkey: 実装済み
    - Firefox Nightlyで利用可能
    - https://bugzilla.mozilla.org/show_bug.cgi?id=1651725
- JavaScriptCore: 未実装
    - [`WebAssembly.Module.imports`の実装](https://github.com/WebKit/WebKit/blob/b50dcf22f189f2c47da11c0929f1204ba6ecac1f/Source/JavaScriptCore/wasm/js/WebAssemblyModuleConstructor.cpp#L103-L132)

## Polyfill

SwiftのWebAssemblyターゲットのスレッドサポートを進めるにあたって今すぐ使いたいので、Polyfillを書きました。

https://github.com/swiftwasm/wasm-imports-parser

簡単なWasmバイナリのパーサと、それを使ったPolyfillの２つのモジュールから構成されています。

Wasmバイナリのパーサは、Wasmバイナリのバイト列を受け取り、importされている関数やメモリの情報を取得します。
パーサといってもimportセクションまでしか読み込まないので、実装はとても軽量です。

自分のユースケースはimportされているメモリのサイズを取得するだけだったので、exportの情報の取得は実装してませんが、そちらもサポートするのは大して難しくないと思います。

```js
import { parseImports } from 'wasm-imports-parser';

const moduleBytes = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00,
    0x02, 0x06, 0x01, 0x00, 0x00, 0x02, 0x00, 0x01,
]);
const imports = parseImports(moduleBytes);
console.log(imports);
// > [
// >   {
// >     module: '',
// >     name: '',
// >     kind: 'memory',
// >     type: { minimum: 1, shared: false, index: 'i32' }
// >   }
// > ]
```

Polyfillの方は、`WebAssembly`グローバルオブジェクトを受け取って、`WebAssembly.Module.imports`を拡張した`WebAssembly`互換オブジェクトを返します。
もし実行エンジンがWebAssembly Type Reflection JS APIをサポートしている場合は、そのままの`WebAssembly`オブジェクトを返し、サポートしていない場合は前述のWasmバイナリのパーサを使って情報を取得するように拡張します。

```js
import { polyfill } from 'wasm-imports-parser/polyfill.js';

globalThis.WebAssembly = polyfill(globalThis.WebAssembly);

const moduleBytes = new Uint8Array([
    0x00, 0x61, 0x73, 0x6d, 0x01, 0x00, 0x00, 0x00,
    0x02, 0x06, 0x01, 0x00, 0x00, 0x02, 0x00, 0x01,
]);
const module = await WebAssembly.compile(moduleBytes);
const imports = WebAssembly.Module.imports(module);
console.log(imports);
// > [
// >   {
// >     module: '',
// >     name: '',
// >     kind: 'memory',
// >     type: { minimum: 1, shared: false, index: 'i32' }
// >   }
// > ]
```

V8との互換性テストをしている最中に気がついたのですが、`type`プロパティの内容がバージョンごとに揺れているようです。`index`プロパティがあったりなかったり、`shared`でない場合に`shared`プロパティが無かったりします。PolyfillはNode.js 22と互換な内容にするようにしました。テストが楽なので。

## まとめ

WebAssembly Type Reflection JS APIというちょっとニッチなAPIのPolyfillを書きました。Wasmで共有メモリを今すぐ扱う際にはその場しのぎの手段として便利だと思います。
