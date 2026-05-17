# check-value-app

画家のための「ヴァルール（明暗の組み立て）確認」用ウェブツール。
カラー画像を1タップでグレースケール化し、構図の明暗を即座に確認できる。

**▶ 使う: https://odayakalife.dev/check-value-app/compare.html**

---

## なぜ作ったか

絵を描くときに「参照画像のヴァルール（明暗関係）を確認したい」場面は頻繁にある。
従来のフローはこれ:

1. ペイントアプリで参照画像を開く
2. 最上位に「白カラーレイヤー」を作る
3. ブレンドモードを Color に設定する
4. ヴァルールを確認する

手順が多すぎる。しかも参照画像の場所がブラウザ・カメラロール・ライブラリと散らばっていて、毎回コピーや移動が発生する。
「見つけた瞬間に確認」を成立させるツールが存在しないか、あっても入口が限られていた（既存ツール調査: [RESEARCH.md §2](./RESEARCH.md#2-競合既存ツール)）。

## 想定ユーザーと困りごと

| ユーザー | 困りごと |
|---|---|
| 写実画家・イラストレーター（Macで作業） | 参照画像をブラウジング中、毎回ペイントアプリ起動→レイヤー作成…が面倒 |
| 外でスケッチする人 | 街中で「描きたい」と思った景色のヴァルールを、その場でカメラからすぐ確認したい |
| Pinterest 等で構図探しする人 | 気になる画像のヴァルールを、Pinterest を離れずに即確認したい |

## できること

### 現状
- カラー画像を、ペイントアプリの「白カラーレイヤー + Color ブレンド」と同等のグレースケールに変換
- ファイル選択 / デフォルトサンプル表示
- 計算式の根拠を画面上で確認可能（折りたたみ式の参考リンク付き）

### 計画中（[Issue一覧](https://github.com/trkoh/check-value-app/issues)参照）
- PWA化（ホーム画面追加、オフライン動作）
- 入力多様化: ドラッグ&ドロップ / クリップボード貼り付け / `getUserMedia` でカメラ直結
- 配信導線:
  - macOS Quick Action（Finder右クリックから起動）
  - iOS Shortcut / Action Button（外で1タップ撮影→即確認）
  - ブックマークレット（Pinterest 等のページから1タップ）
- 明暗段階化（Notan / 2・3・5・9段階posterize）
- ペイントアプリへの送り返し（`navigator.share`）

## 使い方

### 即試す
1. https://odayakalife.dev/check-value-app/compare.html を開く
2. デフォルトでサンプル画像（Joaquín Sorolla, public domain）の変換結果が表示される
3. 「入力」のファイル選択で任意の画像を読み込めば即座に変換

### URLパラメータ
- `?source=URL` で任意の画像URLをデフォルト入力にできる
  - 例: `?source=samples/photoshop.PNG`

## 技術スタック

- **フロントエンド**: 素のHTML / CSS / JavaScript（フレームワーク不使用）
- **画像処理**: Canvas 2D API でクライアントサイド完結
- **アルゴリズム**: ITU-R BT.709 輝度係数 `Y = 0.2126R + 0.7152G + 0.0722B`
- **ホスティング**: GitHub Pages（カスタムドメイン `odayakalife.dev` 配下）
- **開発環境**: VSCode + Claude Code（devcontainer / Docker）

### 設計上の特徴
- **サーバー無し**: すべてクライアントサイド処理。画像は外部送信されない
- **プライバシー**: 個人の参照画像が外部サーバーを通らない
- **オフライン対応予定**: PWA化により回線なしでも動作（Phase 1）

## アルゴリズム選定の経緯

[RESEARCH.md §1](./RESEARCH.md#1-アルゴリズム) で 5 方式（HSL desat / Rec.601 Y' / Rec.709 Y' / Linear-Y / L\*）を比較検証した結果、画家ワークフローとの互換性を最重視して BT.709 (Rec.709 Y') を採用。
将来 posterize / Notan 機能を追加する際は、内部的に CIE L\* に切替予定（知覚均等性のため）。

## プロジェクト構成

| パス | 内容 |
|---|---|
| `compare.html` | 単一エントリーポイント。入力→ヴァルール出力 |
| `samples/` | テスト・デフォルト用画像 |
| `RESEARCH.md` | アルゴリズム / 競合 / 配信機構の調査ログ |
| `.devcontainer/` | 開発環境定義（Docker, firewall, Claude Code 設定） |

## 進捗

GitHub Issues で管理:

- 🎯 [#5 プロジェクトトラッキング](https://github.com/trkoh/check-value-app/issues/5)
- ✅ [#1 Phase 0: アルゴリズム比較検証](https://github.com/trkoh/check-value-app/issues/1) — BT.709 採用決定
- ⏳ [#2 Phase 1: PWA最小機能](https://github.com/trkoh/check-value-app/issues/2)
- ⏳ [#3 Phase 2: 配信導線](https://github.com/trkoh/check-value-app/issues/3)
- ⏳ [#4 Phase 3: 差別化機能](https://github.com/trkoh/check-value-app/issues/4)

## ライセンス

未定（個人プロジェクト）

サンプル画像 `samples/JoaquinSorolla.jpg` は Joaquín Sorolla y Bastida（1863–1923）作品のためパブリックドメイン。
