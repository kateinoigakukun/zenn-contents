---
title: "令和最新版 (2025/08) 古いChromiumのビルドをダウンロードする方法"
emoji: "🐣"
type: "tech"
topics: [Chrome]
published: true
---

特定のバージョンの Google Chrome からしか届かないクラッシュレポートの調査などで、古い Chromium を入手したい場面がある。ここでは現時点での手順をまとめておく。

最近のバージョンであれば [Chrome for Testing](https://googlechromelabs.github.io/chrome-for-testing/) から簡単にダウンロードできるが、提供されているのは Chrome 113 以降のみ。それ以前のビルドを試したい場合は、以下の方法を使う必要がある。

## Lexicon

* **Branch Base Position**: Chrome のリリースブランチが作成されたときの Chromium リポジトリ内のコミット位置。
* **Chromium Dash**: Chromiumの各バージョンやリリース情報、Branch Base Position を確認できる公式ダッシュボード。
* **Chromium Continuous Builds**: Chromium の main ブランチのスナップショットビルドを各プラットフォーム向けに公開しているアーカイブ。

## ステップ1. 対象StableバージョンのBranch Base Positionを調べる

[Chromium Dash](https://chromiumdash.appspot.com/releases) の "Branches" タブから確認できる。

例:
* Chrome 140 → Branch Base Position `1496484`
* Chrome 130 → Branch Base Position `1356013`

![](/images/chromium-dash-branches.png)

## ステップ2. Chromium のスナップショットをダウンロード

1. [Chromium Continuous Builds](https://commondatastorage.googleapis.com/chromium-browser-snapshots/index.html) を開く
2. 自分の環境に合ったプラットフォームディレクトリに移動
3. ステップ1で調べた Branch Base Position に**近い**ビルドを探す
   * 安定版の Branch Base Position と完全一致するスナップショットが存在するとは限らない
4. 該当ビルドの成果物をダウンロード
   * macOS の場合は `chrome-mac.zip` を取得。署名されていないため、展開前に
     ```sh
     xattr -d com.apple.quarantine chrome-mac.zip
     ```
     で quarantine フラグを削除する