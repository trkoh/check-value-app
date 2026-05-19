# check-value-app

画家のための「ヴァルール（明暗の組み立て）確認」用ウェブツール。
カラー画像を1タップでグレースケール化し、構図の明暗を即座に確認できる。

**▶ 使う: https://odayakalife.dev/check-value-app/**

（旧URL `compare.html` も従来通り使える）

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

### 実装済み機能
- カラー画像を、ペイントアプリの「白カラーレイヤー + Color ブレンド」と同等のグレースケールに変換
- 入力多様化:
  - ファイル選択
  - ドラッグ&ドロップ
  - クリップボード貼り付け（ボタン / Cmd・Ctrl+V）
  - `getUserMedia` カメラ直結（背面カメラ優先、撮影→即変換）
  - URLパラメータ: `?img=URL` `?paste=1` `?capture=1`
- アルゴリズム切替: **BT.709 Y'**（デフォルト, Photoshop互換）/ **CIE L\***（知覚均等）
- 段階化: 連続グレー / **Posterize 2・3・4・5・6・9・16段** / **Notan 2値・3値**
- **Squint mode**: Gaussian blur で「目を細めた構図」を再現（0–32px）
- **共有 / 保存**: `navigator.share({files})` で処理済み画像をペイントアプリに直接送信、非対応環境では PNG ダウンロード
- **色を逆引き (Reverse value picker)**: 出力画像をクリック → そのグレー値（Oklab L）に合う色候補12色を表示、HEXコピー
- PWA化: ホーム画面追加対応、Service Workerでオフライン動作
- 計算式の根拠を画面上で確認可能（折りたたみ式の参考リンク付き）

### 配信導線（外部から1〜2タップで起動）
詳細手順は [DELIVERY.md](./DELIVERY.md) 参照:
- **S1 Mac画像**: macOS Quick Action（Finder右クリック → クリップボードへコピー → `?paste=1` で起動）
- **S2 Web**: 画像を右クリック → 「画像をコピー」→ 本ツールで Cmd/Ctrl+V
- **S3 屋外**: iOS Shortcut + Action Button（`?capture=1` でカメラ直結）

### 計画中
- iOS Shortcut の iCloud リンク配布
- Color2Gray（OpenCV.js, "情報重視" モード）
- SwiftUI ネイティブラッパー（必要性確認後）

## 使い方

### 即試す
1. https://odayakalife.dev/check-value-app/ を開く
2. デフォルトでサンプル画像（Joaquín Sorolla, public domain）の変換結果が表示される
3. ファイル選択 / ペースト / ドラッグ&ドロップ / カメラ のいずれかで任意の画像を投入

### iPhoneでホーム画面に追加
1. Safariで開く → 共有ボタン → "ホーム画面に追加"
2. ホーム画面のアイコンから起動するとフルスクリーンで動作（Service Workerによりオフラインでも起動可）

### URLパラメータ
| パラメータ | 動作 |
|---|---|
| `?img=URL` | 任意の画像URLを入力に（外部URLはCORS必要） |
| `?source=URL` | 旧パラメータ（同origin推奨、後方互換） |
| `?capture=1` | 起動と同時にカメラを開く |
| `?paste=1` | 起動時にペースト待機メッセージを表示 |

## 技術スタック

- **フロントエンド**: 素のHTML / CSS / JavaScript（フレームワーク不使用）
- **画像処理**: Canvas 2D API でクライアントサイド完結
- **アルゴリズム**: ITU-R BT.709 Y' `Y' = 0.2126R + 0.7152G + 0.0722B` (デフォルト) / CIE L\* (切替可)
- **PWA**: manifest.webmanifest + Service Worker（stale-while-revalidate）
- **ホスティング**: GitHub Pages（カスタムドメイン `odayakalife.dev` 配下）
- **開発環境**: VSCode + Claude Code（devcontainer / Docker）

### 設計上の特徴
- **サーバー無し**: すべてクライアントサイド処理。画像は外部送信されない
- **プライバシー**: 個人の参照画像が外部サーバーを通らない
- **オフライン動作**: PWAインストール後は回線なしでも起動・処理可能

## アルゴリズム選定の経緯

5 方式（HSL desat / Rec.601 Y' / Rec.709 Y' / Linear-Y / L\*）を比較検証した結果、画家ワークフローとの互換性を最重視して BT.709 (Rec.709 Y') を採用（[RESEARCH.md](./RESEARCH.md) は古い、要更新）。Posterize / Notan / Reverse picker は内部的に CIE L\* / Oklab を使用。

## プロジェクト構成

| パス | 内容 |
|---|---|
| `index.html` | メインエントリ（PWA / 全機能） |
| `compare.html` | 旧シンプル版（後方互換用に保持） |
| `manifest.webmanifest` | PWA マニフェスト |
| `sw.js` | Service Worker（オフラインキャッシュ） |
| `icons/` | PWA / apple-touch アイコン |
| `samples/` | テスト・デフォルト用画像 |
| `DELIVERY.md` | 配信導線（右クリック&ペースト / iOS Shortcut / Mac Quick Action）手順 |
| `RESEARCH.md` | アルゴリズム / 競合 / 配信機構の調査ログ（古い、要更新） |
| `.devcontainer/` | 開発環境定義（Docker, firewall, Claude Code 設定） |

## 進捗

GitHub Issues で管理:

- 🎯 [#5 プロジェクトトラッキング](https://github.com/trkoh/check-value-app/issues/5)
- ✅ [#1 Phase 0: アルゴリズム比較検証](https://github.com/trkoh/check-value-app/issues/1) — BT.709 採用決定
- ✅ [#2 Phase 1: PWA最小機能](https://github.com/trkoh/check-value-app/issues/2) — 実装済み（iPhone 実機検証は別途）
- ✅ [#3 Phase 2: 配信導線](https://github.com/trkoh/check-value-app/issues/3) — 手順書（[DELIVERY.md](./DELIVERY.md)）
- ✅ [#4 Phase 3: 差別化機能](https://github.com/trkoh/check-value-app/issues/4) — Share / Squint 実装（Color2Gray・SwiftUI は将来）
- ✅ [#7 Reverse value picker](https://github.com/trkoh/check-value-app/issues/7) — Oklab パレットで実装
- ⏸ [#6 サンプル画像コミット](https://github.com/trkoh/check-value-app/issues/6) — Procreate ground truth が手元になく保留
- ⏸ [#8 RESEARCH 統合](https://github.com/trkoh/check-value-app/issues/8) — RESEARCH.md 自体が古く保留

## ライセンス

未定（個人プロジェクト）

サンプル画像 `samples/JoaquinSorolla.jpg` は Joaquín Sorolla y Bastida（1863–1923）作品のためパブリックドメイン。
