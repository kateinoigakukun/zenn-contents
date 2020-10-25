---
title: "macOSのツールチェーンの仕組み"
emoji: "🤖"
type: "tech"
topics: [macOS]
published: false
---

macOSにはシステムにインストールされた複数のバージョンのツールチェーンを切り替えて使うための仕組みが備わっています。しかし、そのメカニズムについて記述された文書は少なく、雰囲気で `xcode-select` コマンドを使っている方も多いと思います。
この記事では、macOSにおけるツールチェーンの役割と仕組みについて紹介します。


## ツールチェーンとは

一般的にツールチェーンとはコマンドやライブラリ、ヘッダなどをひと纏めにしたツール群のことを指すことが多いです。

例えばC言語のソースコードから実行可能なバイナリへビルドするためには、

1. `clang`や`gcc`などのコンパイラでオブジェクトファイルへ変換
2. `ld64`や`lld`などのリンカでオブジェクトファイルと`libc`を実行可能バイナリとしてリンク

という操作が必要になります。
ここで登場した、コンパイラやリンカ、標準ライブラリなどのツール郡はバラバラに配布することもできますが、ひと纏めになっている方がツール間のバージョンの固定もできて都合が良いですね。
このようにそれぞれが連携しながら動くようなツール群を一つのパッケージとしたものがツールチェーンです。

### ツールチェーンの例

具体的なmacOS向けのツールチェーンには以下のようなものがあります。
他にもLLVMなどもツールチェーンとして考えることができますが、今回は後述するmacOSの`xcrun`と連携可能なツールチェーンのみを取り上げています。


| 名前 | インストール場所 |
|:-|:-|
| Xcode内蔵ツールチェーン | `/Applications/Xcode.app/Contents/Developer/Toolchains` |
| [OSS版 Swift](https://swift.org/download/#releases) | `/Library/Developer/Toolchains` |
| [Swift for Tensorflow](https://github.com/tensorflow/swift/blob/master/Installation.md#releases) | `/Library/Developer/Toolchains` |
| [SwiftWasm](https://github.com/swiftwasm/swift/releases) | `/Library/Developer/Toolchains` |


これらのツールチェーンは全て`xctoolchain`というバンドル形式で配布されており、以下のようなディレクトリ構造となっています。

```
├── ToolchainInfo.plist or Info.plist
└── usr
    ├── bin # コマンド
    │   ├── clang
    │   ├── swift
    │   └── ...
    ├── include # ヘッダ
    │   └── clang/c++/v1
    │       ├── stdlib.h
    │       └── ...
    ├── lib # ライブラリ
    │   ├── arc/libarclite_macosx.a
    │   └── ...
    └── ...
```


## macOSとの連携

さて、上に挙げたようなツールチェーンに内包されているコマンド達は、`/usr/bin/`にインストールされている同名のコマンドから動的にツールチェーンを切り替えながら実行することができます。

（この仕組みについて詳しく言及されている文書を今の所見つけることができていないため、ここからは`xcrun`や`xcode-select`の`man`ページの断片的な記述と、実際の挙動をベースに紹介します。）

具体的には現在アクティブになっているDeveloper Directoryと、`TOOLCHAINS`という環境変数によって`/usr/bin/`配下の特定のコマンドで実行されるバイナリが切り替わります。
`xcode-select`のmanページ最下部に動的に切り替わるコマンドの一覧がありますが、ここに無いいくつかのコマンドも同様の挙動をすることがあります。

### Developer Directory

Developer Directoryとは普段我々が `xcode-select` コマンドで切り替えているmacOS内部のパラメータです。Developer Directoryは普通Xcodeアプリ内部の`/Applications/Xcode.app/Contents/Developer`または`/Library/Developer/CommandLineTools`を指しています。 `DEVELOPER_DIR`環境変数で一時的に上書きすることが可能です。

`xcode-select --print-path`コマンドで現在の値を確認でき、`xcode-select --switch`コマンドでパスを変更できます。

```
$ xcode-select --print-path
/Applications/Xcode-12.2-beta3.app/Contents/Developer

$ DEVELOPER_DIR=/Library/Developer/CommandLineTools xcode-select --print-path
/Library/Developer/CommandLineTools

$ sudo xcode-select --switch /Applications/Xcode-12.2-beta1.app/Contents/Developer

$ xcode-select --print-path
/Applications/Xcode-12.2-beta1.app/Contents/Developer
```

後述する`TOOLCHAINS`環境変数がセットされていない場合、`/usr/bin/`配下のコマンドを起動すると、Developer Directoryをベースにコマンドが探索され、同名のコマンドが見つかるとそれが実行されます。

### `TOOLCHAINS`環境変数

`TOOLCHAINS`環境変数にはxctoolchainバンドルのBundle Identifierまたはツールチェーンのエイリアスを指定することができます。Bundle Identifierやエイリアスはxctoolchain中の`Info.plist`ファイルに記述されています。

```xml
$ cat /Library/Developer/Toolchains/swift-tensorflow-RELEASE-0.9.xctoolchain/Info.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Aliases</key>
	<array>
		<string>Swift for TensorFlow</string>
	</array>
	<key>CFBundleIdentifier</key>
	<string>com.google.swift.20200507</string>
	<key>CompatibilityVersion</key>
	<integer>2</integer>
	<key>CompatibilityVersionDisplayString</key>
	<string>Xcode 8.0</string>
	<key>CreatedDate</key>
	<date>2020-05-08T05:57:25Z</date>
	<key>DisplayName</key>
	<string>Swift for TensorFlow 0.9 Release 2020-05-07</string>
	<key>OverrideBuildSettings</key>
	<dict>
		<key>ENABLE_BITCODE</key>
		<string>NO</string>
		<key>SWIFT_DEVELOPMENT_TOOLCHAIN</key>
		<string>YES</string>
		<key>SWIFT_DISABLE_REQUIRED_ARCLITE</key>
		<string>YES</string>
		<key>SWIFT_LINK_OBJC_RUNTIME</key>
		<string>YES</string>
		<key>SWIFT_USE_DEVELOPMENT_TOOLCHAIN_RUNTIME</key>
		<string>YES</string>
	</dict>
	<key>ReportProblemURL</key>
	<string>https://bugs.swift.org/</string>
	<key>ShortDisplayName</key>
	<string>Swift for TensorFlow 0.9 Release</string>
	<key>Version</key>
	<string>5.0.20200507</string>
</dict>
</plist>
```


`TOOLCHAINS`環境変数が設定されている場合、`/usr/bin/`配下のコマンドを起動すると、`/Library/Developer/Toolchains`配下に配置されているxctoolchainの中にBundle Identifierかエイリアスが一致するものがあれば、その中の`usr/bin/`ディレクトリの同名のコマンドを実行します。xctoolchainが見つからない場合、Developer Directoryをベースとした探索にフォールバックします。


### 切り替えの優先度

<!--
digraph {
  check_toolchains_env[label = "TOOLCHAINS環境変数が定義されているか"]
  is_match_xctoolchain[label = "マッチするxctoolchainが/Library/Developer/Toolchains/にあるか"]
  is_exist_cmd_in_xctoolchain[label = "コマンドが存在するか"]
  run_xctoolchain_bin[label = "見つかったコマンドを実行"]
  check_developer_dir[label = "Developer Directoryにコマンドが存在するか"]
  run_dev_dir_bin[label = "見つかったコマンドを実行"]
  fail[label = "失敗"]

  check_toolchains_env -> is_match_xctoolchain[label = "YES"]
  check_toolchains_env -> check_developer_dir[label = "NO"]
  is_match_xctoolchain -> is_exist_cmd_in_xctoolchain[label = "YES"]
  is_match_xctoolchain -> check_developer_dir[label = "NO"]
  is_exist_cmd_in_xctoolchain -> run_xctoolchain_bin[label = "YES"]
  is_exist_cmd_in_xctoolchain -> check_developer_dir[label = "NO"]
  check_developer_dir -> run_dev_dir_bin[label = "YES"]
  check_developer_dir -> fail[label = "NO"]
}
-->

以上のルールをまとめたフローの図です。

![](https://storage.googleapis.com/zenn-user-upload/0a84uesbsoo7j60ul0k1j6vyzrwg)

`xcrun --find`コマンドによって実際にどのコマンドが解決されたか調べることができます。

```
$ xcrun --find swift
/Applications/Xcode-12.2-beta3.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swift

$ DEVELOPER_DIR=/Library/Developer/CommandLineTools xcrun --find swift
/Library/Developer/CommandLineTools/usr/bin/swift

$ TOOLCHAINS=com.google.swift.20200507 xcrun --find swift
/Library/Developer/Toolchains/swift-tensorflow-RELEASE-0.9.xctoolchain/usr/bin/swift
```


具体例として、Swift for TensorFlow向けのプロジェクトをビルドしたい場合、以下のように環境変数を設定しながら`swift`コマンドを実行します。

```
$ TOOLCHAINS=com.google.swift.20200507 /usr/bin/swift build
```

## 動的にコマンドが切り替わる仕組み

`/usr/bin/`のコマンドは具体的なバイナリへのシンボリックリンクになっているわけではなく、上のルールに沿ってコマンドを解決し引数をプロキシして実行するプログラムとなっています。

実際にディスアセンブルすると、`/usr/lib/libxcselect.dylib` というライブラリの`_xcselect_invoke_xcrun`関数を呼ぶだけの薄いラッパーであることがわかります。

```
$ objdump --macho --disassemble /usr/bin/swift
/usr/bin/swift:
(__TEXT,__text) section
_main:
100000f77:	55	pushq	%rbp
100000f78:	48 89 e5	movq	%rsp, %rbp
100000f7b:	8d 47 ff	leal	-1(%rdi), %eax
100000f7e:	48 8d 56 08	leaq	8(%rsi), %rdx
100000f82:	48 8d 3d 29 00 00 00	leaq	41(%rip), %rdi ## literal pool for: "swift"
100000f89:	89 c6	movl	%eax, %esi
100000f8b:	31 c9	xorl	%ecx, %ecx
100000f8d:	e8 00 00 00 00	callq	0x100000f92 ## symbol stub for: _xcselect_invoke_xcrun
```

また、殆どの場合問題になることはないと思いますが、コマンド解決の結果は`TOOLCHAINS`環境変数とDeveloper Directoryのペアをキーとしてキャッシュされることに注意してください。 `xcrun --no-cache` コマンドでキャッシュをリフレッシュすることができます。

このように実行されるコマンドを動的に切り替えるためのプログラムをShimと言います。`rbenv`などの複数のバージョンのツールを切り替えるためのツールでよく用いられる言葉です。

[Shim (computing) - Wikipedia](https://en.wikipedia.org/wiki/Shim_(computing))

## 宣伝

これで自由自在にツールチェーンの切り替えができるようになったと思うので、[SwiftWasm](https://swiftwasm.org/)のツールチェーンで実際に試してみてください😁

## 参照

- [Xcode Overview: Using Alternative Toolchains](https://developer.apple.com/library/archive/documentation/ToolsLanguages/Conceptual/Xcode_Overview/AlternativeToolchains.html)
- [Technical Note TN2339: Building from the Command Line with Xcode FAQ](https://developer.apple.com/library/archive/technotes/tn2339/_index.html)
- [llvm-project/CMakeLists.txt at llvmorg-11.0.0 · llvm/llvm-project](https://github.com/llvm/llvm-project/blob/llvmorg-11.0.0/llvm/tools/xcode-toolchain/CMakeLists.txt)