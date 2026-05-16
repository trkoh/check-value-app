# 調査結果まとめと方向性提案

## 背景

画家用ヴァルール（valeur, 明暗の相対関係）確認ツール。

**ペインポイント**: ペイントアプリで白カラーレイヤーを最上位に重ねる現状フローが面倒。

**3つのユースケース**:
- (A) ブラウザで画像を見つけた瞬間
- (B) 実物を撮影したい瞬間
- (C) すでにライブラリ/アプリにあるリファレンス

---

## 1. アルゴリズム

### 結論
**単純desaturation (HSL L / HSV V) は使用しない**。同じ "見かけの明度" として扱えない色（純青と純黄など）が同一グレーに潰れる。

| 色 | RGB | HSL L | linear Y (Rec.709) |
|---|---|---|---|
| 純黄 | 255,255,0 | 128 | 0.928 |
| 純青 | 0,0,255 | 128 | 0.072 |

HSLでは同じグレー、linear Yでは **13倍差**。

### 推奨デフォルト
**linear-Y**: sRGB → linear → `0.2126 R + 0.7152 G + 0.0722 B` (Rec.709) → sRGBに戻す

### オプション
- **CIE L\*** (9段value scale や notan posterize と相性◎、知覚均等)
- **Posterize 2/3/5/9 段** (L\*空間)
- **Squint mode**: Gaussian blur + L\* (構図確認用、「目を細める」効果の数値モデル)
- **Color2Gray** (Gooch 2005) — 等輝度色対比保存。"情報重視"モード
- **Lu/Xu/Jia "Real-time Contrast Preserving Decolorization"** (OpenCV `cv::decolor` で実装済み)

### コード例（推奨デフォルト）
```js
function srgbToLinear(c) {
  const x = c / 255;
  return x <= 0.04045 ? x / 12.92 : Math.pow((x + 0.055) / 1.055, 2.4);
}
function linearToSrgb(c) {
  return c <= 0.0031308 ? 12.92 * c : 1.055 * Math.pow(c, 1/2.4) - 0.055;
}
function toValueGray(r8, g8, b8) {
  const Y = 0.2126 * srgbToLinear(r8) + 0.7152 * srgbToLinear(g8) + 0.0722 * srgbToLinear(b8);
  const v = Math.round(linearToSrgb(Y) * 255);
  return [v, v, v];
}
```

---

## 2. 競合・既存ツール

| ツール | 価格 | プラットフォーム | ライブカメラ | 共有受け | 強み | 弱み |
|---|---|---|---|---|---|---|
| **Mark II Artist's Viewfinder** | $39.99 | iOS | ◎ B&W luminance mode | × | プロ向け、Exposure lock等 | 高価、機能過多 |
| **Notanizer** | $1.99 | iOS | × | × | 確立、Feb update でOS 26対応継続 | ライブカメラなし |
| **Value Study** | $1.99/年 or $6.99永久 | iOS/Mac/Android | × | × | クロスプラットフォーム、histogram/eyedropper | Web版なし |
| **Art Reference** | $14.99永久 | iOS | × | × | 7-value scale + 参考画像集 | 自前画像のフロー弱い |
| **tonalvaluetool.com** | 無料 | Web | × | × | ローカル処理、Drag&Drop、2-16階調 | 共有導線なし、カメラなし |
| **iPhone標準カメラ Noirフィルタ** | 無料 | iOS | ◎ | × | ゼロ実装、(B)に最強のベースライン | 後加工不可、posterize/notanなし |

### ギャップ分析
- **どの既存ツールも「共有シート/Shortcut から1タップで → ヴァルール画像」の高速導線を提供していない**
- ライブカメラ強者 (Mark II Viewfinder) はB特化＆高価、A/Cが弱い
- 写真アプリ系 (Notanizer / Value Study) は逆にB(ライブ)が弱い
- Webツール (tonalvaluetool.com) は近いが、入口がブラウザに固定
- **iPhone標準カメラのNoirフィルタが事実上のベースライン** — これを上回るUXが要件

### ポジショニング案
**"Web-first / zero-install / 入口多様化型の高速 valeur viewer"**

差別化要素:
- PWA + iOS Shortcut共有 + ライブカメラ + perceptually-correct algo
- 結果画像を共有シートでペイントアプリに送り返す（地獄フロー根絶）

---

## 3. 配信機構

### 重要前提（2026-05時点で検証済み）
**iOS Safari の Web Share Target API は依然 未対応**。WebKit bug [#194593](https://bugs.webkit.org/show_bug.cgi?id=194593) は2019年からNEW、最終コメント2026-04-01、最終更新2026-05-11。WebKitのpositionは"neutral since 2023"。

→ **PWAを共有シートのターゲットにする教科書フローは iOS では不可**。
→ **回避策**: iOS Shortcuts で「共有シート → URL組み立て → Safari/PWA起動」のブリッジ（2026年現在も唯一安定して動く方法）

### 候補比較
| 手段 | (A) | (B) | (C) | 召喚コスト | 実装 | 配布 |
|---|---|---|---|---|---|---|
| **PWA core** | ○ | ○ | ○ | 1-2タップ | 小 | 不要 |
| **iOS Shortcut + 共有シート** | ◎ | △ | ◎ | 2-3タップ | 小 | iCloudリンク |
| **ブックマークレット (Safari)** | ◎ | × | × | 1タップ | 極小 | 不要 |
| **getUserMedia (PWA内)** | × | ◎ | × | 1タップ | 小 | 不要 |
| **iPhone標準カメラ Noir (競合)** | × | ◎ | × | 数タップ | ゼロ | ベースライン |
| **iOSアクションボタン → Shortcut** | × | ◎ | × | 長押し1回 | 中 | Shortcuts設定 |
| **Photo Editing Extension** | × | × | ◎ | 写真App内3タップ | 中 | App Store |
| **Native iOS app** | △ | ◎ | ◎ | 1タップ | 大 | App Store |
| **macOS Quick Action** | × | × | ◎ | 右クリック | 小 | 配布なら署名 |
| **Safari Web Extension** | ◎ | × | △ | 拡張から | 中 | App Store審査 |

### 推奨スタック（"ナウい開発フロー"準拠）
**PWA を中核 + 入口を多様化**:

1. **PWA core** (TypeScript + Canvas/WebGL): 単一URLでカメラ/ファイル/D&D/paste/`?img=`パラメータを受付
2. **iOS Shortcut** (iCloudリンク): 共有シート → URL組み立て → PWA起動
3. **ホーム画面アイコン**: (B)用、`getUserMedia`即起動
4. **ブックマークレット**: macOS/iPad Safariで(A)を最短化
5. **(将来) SwiftUI 薄ラッパー**: アクションボタン割当で(B)をネイティブ低遅延化

採用理由:
- 配布審査ゼロ、AI/vibe coding親和性最大、(A)(B)(C)全カバー
- iOS/iPadOS/macOS/Web 同時カバー
- 不満が定量化されてからネイティブ拡張する段階戦略

---

## 4. 次の一歩 (MVP案)

### Phase 0: アルゴリズム比較検証
- 5アルゴリズム横並びの比較ツール `compare.html` を作る
  - (a) HSL desaturation（悪い例、参照用）
  - (b) Rec.601 Y' (gamma)
  - (c) Rec.709 Y' (gamma) ← Photoshop/Procreateの "Color" ブレンド想定
  - (d) Linear-Y → sRGB（現推奨）
  - (e) L\* (CIELAB)
- ユーザーが現状ペイントアプリで作成したヴァルール画像 (= ground truth) と並べて目視比較
- **目的**: 「理論的に正しい」と「ユーザーの目に馴染む」のずれを早期に発見
- 完了条件: 1つ（または複数の組み合わせ）を確定し、§1「推奨デフォルト」を更新

### Phase 1: PWA最小機能
- linear-Y / L\* の切替
- posterize 2/3/5/9
- ファイル選択 / paste / D&D / camera
- PWA manifest (ホーム画面追加対応)

### Phase 2: 配信導線
- iOS Shortcut 作成・iCloudリンク
- ブックマークレット (1行JS)
- インストール手順ページ

### Phase 3: 差別化
- `navigator.share({files})` で processed画像をペイントアプリに送り返す
- Squint mode
- 必要なら Color2Gray / OpenCV.js (lazy load)

---

## 5. 具体的ユースケースと現案（2026-05-15追記）

### 即時適用したい3シナリオ

| # | シナリオ | デバイス | 入口 | 既存(A)(B)(C)との対応 |
|---|---|---|---|---|
| S1 | Mac上で見つけた「いい感じ」画像 | macOS | Finder / ブラウザ / 画像アプリ | (C) ローカルライブラリ |
| S2 | Pinterest等のWebで見つけた画像 | Mac/iOS Safari | ブラウザ | (A) ブラウザ内発見 |
| S3 | 外で歩いていて見つけた風景 | iPhone | カメラ | (B) 実物撮影 |

**要件**: いずれも「見つけた→即適用」のフリクションを最小化。理想は1–2タップ。

### 現案（叩き台）
- バックエンドにアップロード用APIを立てる
- Mac/iOS Shortcut から画像を POST → 処理済画像URLを返す → ブラウザで表示
- メリット: 入口を Shortcut に統一できる
- 懸念:
  - サーバー運用コスト（個人ツールで永続稼働は重い）
  - アップロード往復のレイテンシ（特にS3でモバイル回線時）
  - 通信できない場所で使えない（外でのS3で致命的）
  - プライバシー（個人の参照画像をサーバー経由）
  - 既存の[[Phase 1-3]]提案（PWA + クライアント処理）と矛盾する

### 調査タスク
代替/補強手段を網羅的に洗い、各シナリオで最短フローを再選定する。
→ **[§6 アップロード回避型アーキテクチャの再調査](#6-アップロード回避型アーキテクチャの再調査2026-05-15) 参照**

---

## 6. アップロード回避型アーキテクチャの再調査（2026-05-15）

### 結論先出し
**サーバーアップロード方式は不要**。linear-Y 変換は Canvas で <50 行のクライアント処理で完結する。サーバーを介する利点は「Shortcutに導線を一本化できる」程度で、デメリット（運用コスト、レイテンシ、オフライン不可、プライバシー、回線依存）の方が大きい。

→ **既存の[[Phase案]]（PWA + クライアント処理）を再確認し、各シナリオに合わせて入口を強化する方が筋がよい。**

### アップロード方式の評価（叩き台の検証）

| 観点 | サーバーアップロード | PWA クライアント処理 |
|---|---|---|
| 実装 | API + ホスティング + 認証 | 静的ファイルのみ |
| 月次コスト | サーバー代 | 0円（Cloudflare Pages等） |
| レイテンシ | 往復＋処理 | 即時（数十ms） |
| オフライン | × | ◎（PWA cache） |
| プライバシー | 画像が外部送信 | ローカル完結 |
| S3外出時(モバイル回線) | △〜× | ◎ |
| 入口統一 | ◎ Shortcut一本 | △ シナリオごとに最適化 |

→ 唯一のメリット「入口統一」は、シナリオごとに最適入口を用意すれば不要。

### シナリオ別: アップロード回避フロー

#### S1: Mac上で見つけた画像
1. **ドラッグ&ドロップ** (ブラウザの PWA タブに画像をD&D) — 最速、ゼロ設定
2. **コピペ** (画像をコピー → PWAで⌘V) — Finder/プレビュー/ブラウザいずれからも
3. **macOS Quick Action** (Shortcuts.app):
   - 右クリック→「Quick Actions」→「Open in Valeur」
   - Shortcutが画像をクリップボードにコピー＋PWAをデフォルトブラウザで起動
   - PWAは起動時にクリップボードから自動読込（`navigator.clipboard.read()`、ユーザージェスチャ1回必要）
4. **Safari Web Extension** (将来): 画像右クリック「View in Valeur」で新タブ起動＋画像受渡し（要Apple Developer、優先度低）

#### S2: Pinterest等Web上の画像
1. **ブックマークレット**: ブックマークバーから1タップ→現在ページの`<img>`を抽出してPWAへ渡す（一行JS、配布不要）
2. **長押し→「画像をコピー」→PWAで⌘V/ペースト** (Safari標準動作)
3. **iOS共有シート→Shortcut**:
   - Pinterest内「共有」→ カスタムShortcut「Open in Valeur」
   - 注意: Pinterestは「Copy link」だとpinページURLしか取れない。`navigator.share()`経由なら画像URLが取れることがあるがpinに依存
   - 確実なのは「Save image」でカメラロール保存→PWAのファイル選択でPhotosピッカー
4. **将来の Safari Web Extension** をiOS/macOS共通で

#### S3: 外で撮った風景
1. **PWA + getUserMedia**: ホーム画面アイコン1タップ→`?capture=1`でカメラ即起動→撮影→即変換表示
2. **iOS Action Button** (iPhone 15+):
   - 設定→Action Button→「Shortcut」→「Valeur起動」Shortcutを割当
   - 長押し1回でPWA起動、URLパラメータ`?capture=1`でカメラモード直行
3. **Camera Control button** (iPhone 16+) も同様に割当可
4. **ロック画面 Control Center カスタムコントロール** (iOS 18+): スワイプ→タップでValeur起動

### 各方式の召喚コスト比較

| シナリオ | サーバー案 | 回避案最短 | 削減タップ |
|---|---|---|---|
| S1 (Mac) | Shortcut起動→画像選択→アップロード待ち | ブラウザにD&D | -3 |
| S2 (Pinterest iOS) | 共有シート→アップロードShortcut→待ち | コピー→ペースト or ブックマークレット | 同等〜-1 |
| S3 (歩行中) | カメラ→保存→Shortcut→アップロード→結果 | Action Button→撮影→即表示 | -2、かつオフライン可 |

### 重要な技術ポイント (2026-05時点)

1. **iOS Safari の Web Share Target API は依然未対応** ([WebKit #194593](https://bugs.webkit.org/show_bug.cgi?id=194593))。PWAを共有シートのターゲット化は不可。
2. **Clipboard API**: iOS Safari 13.4+ で `navigator.clipboard.read()` 対応（画像含む）。ユーザージェスチャ必須なのでPWA起動後「Tap to load」を1回押す導線が必要。
3. **macOS Shortcuts.app** は Quick Action として Finder 右クリック / Services に登録可能。画像受け取り→JS実行→URL起動が標準ブロックで可能。
4. **iOS Action Button** は Shortcut呼び出しが可能で、結果としてのPWA起動レイテンシは1-2秒。標準Camera比でやや遅いが許容範囲。
5. **Pinterest からの画像取得**: pin URL ↔ 画像URL は1対1でなく、`originals/`バリアントを取るには pin メタデータ参照が必要。Webブックマークレットなら DOM から直接 `<img src>` を読める。
6. **iOS Safari Web Extension** は Apple Developer Program ($99/年) 必須。配布障壁が高いので初期は不採用。

### 推奨アーキテクチャ (修正版)

```
┌─────────────────────────────────────────┐
│  PWA (静的ホスティング、ゼロサーバー)   │
│   - linear-Y / L* / posterize           │
│   - File / Paste / D&D / getUserMedia   │
│   - ?img= / ?capture= / ?paste=1 受付   │
└─────────────────────────────────────────┘
        ↑       ↑          ↑          ↑
     ↓ S1 ↓   ↓ S2 ↓     ↓ S3 ↓   ↓ 共通 ↓
   ┌──────┐ ┌────────┐ ┌────────┐ ┌──────┐
   │ Mac  │ │ ブック │ │ Action │ │ ホーム│
   │QuickA│ │マークレ│ │Button  │ │画面ｱｲ│
   │ction │ │ット    │ │+Short  │ │コン  │
   └──────┘ └────────┘ └────────┘ └──────┘
```

**個人運用ならホスティングは Cloudflare Pages / GitHub Pages で十分**。HTTPS必須（`getUserMedia`/`clipboard.read`が要求）。

### サーバー案を採用する余地が残るケース

- **複数デバイス間で履歴を共有したい**（Mac で処理 → iPhone でも見たい）
- **重い処理を将来追加したい**（Color2Gray, decolor 等を WASM ではなく Cloud Function に逃がしたい）
- **画像をリンクで他人にシェアしたい**

→ いずれも初期MVPでは不要。必要になってから [[Phase 4以降]] で薄いストレージAPIを追加すればよい。

### 次アクション候補

1. **Phase 0 を最速で着手**: 静的HTMLで `?img=URL` 受け linear-Y表示 → ホスティング → 自分のiPhoneで Add to Home Screen
2. **Mac側 Quick Action** をShortcutsで試作（右クリック→クリップボード経由でPWAへ）
3. **ブックマークレット** をPinterestで実地テスト
4. **Action Button割当** で S3 体感レイテンシを測定 → ネイティブ要否を判断

---

## 7. 意思決定ログ

### 確定済 (2026-05-15)
- [x] **ホスティング**: GitHub Pages
- [x] **ドメイン**: `trkoh.github.io/check-value-app/` (サブパス)
- [x] **デザイン**: ミニマル（機能優先）
- [x] **配信戦略**: アップロード型サーバーは不採用、PWA + クライアント処理 ([§6](#6-アップロード回避型アーキテクチャの再調査2026-05-15))
- [x] **アルゴリズム検証アプローチ**: Phase 0で5方式を実画像で比較し決定 ([§4 Phase 0](#phase-0-アルゴリズム比較検証))

### 未決
- [ ] **アルゴリズムのデフォルト**: Phase 0完了時に確定
- [ ] **(B)シナリオ S3 のレイテンシ**: PWA起動1-2秒が許容範囲か。Phase 1完了後に実機で測定して判断
- [ ] **ペイントアプリへの送り返しUX**: Phase 3で`navigator.share({files})`の挙動を検証してから決定

---

## 参考リンク

**Algorithm**:
- [Color2Gray (Gooch 2005)](https://www.cs.northwestern.edu/~ago820/color2gray/)
- [Lu/Xu/Jia Decolorization project](http://www.cse.cuhk.edu.hk/leojia/projects/color2gray/)
- [OpenCV `decolor`](https://docs.opencv.org/4.x/d4/d32/group__photo__decolor.html)
- [Poynton Color FAQ](https://poynton.ca/notes/colour_and_gamma/)

**Competitive**:
- [Notanizer](https://apps.apple.com/us/app/notanizer/id1169086981)
- [Value Study](https://valuestudy.app/en/)
- [Mark II Artist's Viewfinder](https://www.artistsviewfinder.com/)
- [tonalvaluetool.com](https://tonalvaluetool.com/)
- [Art Reference & Value Study](https://apps.apple.com/us/app/artist-reference-value-study/id6752580673)

**Delivery**:
- [WebKit bug #194593 (Web Share Target未対応)](https://bugs.webkit.org/show_bug.cgi?id=194593)
- [Web Share Target spec](https://w3c.github.io/web-share-target/)
- [Apple Action Button guide](https://support.apple.com/guide/iphone/use-and-customize-the-action-button-iphe89d61d66/ios)
- [Apple Shortcuts user guide](https://support.apple.com/guide/shortcuts/)
