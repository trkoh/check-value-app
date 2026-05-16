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

### Phase 0: スパイク
- 静的HTMLで `?img=URL` 受けて linear-Y のグレースケール表示
- iPhone Safariでホーム画面追加 → `getUserMedia`動作確認

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

## 5. オープン質問

1. ホスティング先の好み: Cloudflare Pages / Vercel / GitHub Pages?
2. ドメイン: 既存所有あり？それともサブパスでOK？
3. (B)のレイテンシ要件: PWA起動ラグ1〜2秒は許容範囲？それともネイティブ必須？
4. デザイン/ブランド: 最小限OK？それともこだわる？

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
