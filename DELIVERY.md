が# 配信導線セットアップ

「画像が目の前にある」状態から、ヴァルール確認ツールに最短手数で渡すための導線をシナリオ別にまとめる。

公開URL: <https://odayakalife.dev/check-value-app/>

各シナリオの記述構成:

- **状況**: いつ使うか
- **操作**: 実際に押す/する手順
- **結果**: 何が起こるか
- **仕組み / 根拠**: なぜ動くのか（仕様・ドキュメント・実装上の依拠）
- **動かない条件**: 既知の制限

検証ステータス記号:

- ✅ 実装側で検証済み（HTTPレスポンス、ブラウザ標準API、ローカル動作）
- 📖 仕様/公式ドキュメントの記述に基づく（実機未検証）
- ⚠️ 推測を含む。実機で要確認

---

## まず確認: 何もセットアップ無しでも動く経路

セットアップが面倒な場合、以下だけで全シナリオを概ねカバーできる:

| シナリオ             | 操作                                                                                                                  | 手数 |
| -------------------- | --------------------------------------------------------------------------------------------------------------------- | ---- |
| Mac のローカル画像   | ブラウザで本ツール開く → 画像ファイルをページにドラッグ                                                               | 1    |
| クリップボードに画像 | ブラウザで本ツール開く → Cmd/Ctrl+V                                                                                   | 1    |
| Web ページの画像     | 画像を**右クリック → 「画像をコピー」**（"アドレス" ではない方）→ 本ツールのタブで **Cmd/Ctrl+V**                       | 2    |
| iPhone でカメラ      | ホーム画面の PWA アイコンタップ → 「カメラ」ボタン → 撮影                                                             | 3    |

Shortcuts/Quick Action はこれを **1-2 タップに短縮するための最適化**。必須ではない。

---

## S2: Web ページの画像 — 右クリック→画像をコピー→ペースト ✅

### 状況
Chrome / Safari / Firefox で Web ページを見ていて、表示されている画像のヴァルールを確認したい。Pinterest の個別ピン詳細ページ、Wikipedia の記事中の絵画画像、ブログ記事の挿絵など。

### 操作 (毎回)
1. 確認したい画像が表示されているページで、画像を**右クリック → 「画像をコピー」**（"画像のアドレスをコピー" ではない方）
2. 本ツールのタブを開く（または新規タブで <https://odayakalife.dev/check-value-app/> ）
3. **Cmd+V**（Mac）または **Ctrl+V**（Windows/Linux）

### 結果
クリップボードの画像が即読み込まれ、ヴァルール変換結果が表示される。

### 仕組み / 根拠
- 「画像をコピー」はブラウザがピクセルデータを直接クリップボードに転送する経路。元サイトの CORS 設定や CDN の `Access-Control-Allow-Origin` ヘッダに依存しない
- ペーストは HTML 標準の `paste` イベントの `clipboardData.items` から `image/*` を取り出す。本ツールの `index.html` の `paste` ハンドラで処理

### 動かない条件
- ❌ 画像が CSS の `background-image` で配置されているサイト（Pinterest のグリッド一覧画面、Instagram のタイル等）では右クリックメニューに「画像をコピー」が出ない → **個別画像詳細ページ** に遷移してから右クリック
- ⚠️ Firefox 一部の経路でクリップボード経由の画像取得がブラウザの設定で制限されるケースあり。基本は Chrome / Safari で確実

### なぜブックマークレットを採用しなかったか
当初 javascript: ブックマークレットを試したが、以下の制約で安定動作しない:
- Pinterest 等の CSP / DOM 構造で誤ピックや無反応が頻発
- 多くの画像 CDN が CORS を返さないため `?img=URL` 経由が失敗
- `execCommand('copy')` で画像データをクリップボードへ転送する代替実装も、サイト・ブラウザ依存で挙動が不安定

結論として、ブラウザ標準の **右クリック→画像をコピー** が一番ロバスト。ユーザー側で2タップ余分に必要だが、確実性を優先した。

---

## S1: Mac の画像 ✅ / 📖

### 状況 A: ブラウザでもう本ツールを開いている

画像ファイルが Finder にある、またはクリップボードに画像がある。

#### 操作

- ファイルなら → ブラウザのページに **ドラッグ&ドロップ**
- クリップボードなら → ページ上で **Cmd+V**

#### 結果

即座に変換結果が表示される。

#### 根拠 (✅)

- D&D は HTML 標準の `drop` イベント（[HTML Living Standard §6.10](https://html.spec.whatwg.org/multipage/dnd.html)）。`index.html` の `window.addEventListener('drop', ...)` で処理
- ペーストは `paste` イベントの `clipboardData.items` から `image/*` を取り出す。Chrome / Safari / Firefox いずれも実装済み（[MDN: ClipboardEvent](https://developer.mozilla.org/en-US/docs/Web/API/ClipboardEvent)）

### 状況 B: 本ツールを開かずに、Finder から1タップ起動したい — macOS Shortcuts.app Quick Action 📖

#### セットアップ (初回1回)

1. **Shortcuts.app** を起動（macOS 12 Monterey 以降に標準搭載）
2. 新規ショートカット作成（左下「+」）
3. 右側のアクション検索で以下を順に追加:
   - **「クリップボードにコピー」** — "ショートカットの入力" を選択
   - **「URLを開く」** — `https://odayakalife.dev/check-value-app/?paste=1`
4. ショートカット名を「ヴァルール確認」に変更
5. 設定アイコン（"i" マーク）→ "詳細":
   - ✅ **共有シートに表示** をオン、入力タイプ: **画像** / **ファイル**
   - ✅ **クイックアクションとして使用** をオン、**Finder** にチェック

#### 毎回の操作

1. Finder で画像ファイルを右クリック
2. **「クイックアクション」→「ヴァルール確認」**
3. デフォルトブラウザで本ツールが開く、上部に「Cmd+V で画像をペースト」と表示
4. **Cmd+V** を押す → 変換結果が表示

#### 根拠 (📖)

- macOS Shortcuts.app の "Quick Action として Finder に表示" 機能は [Apple Shortcuts ユーザガイド](https://support.apple.com/guide/shortcuts-mac/intro-to-quick-actions-apdf22b0444c/mac) に記載
- "クリップボードにコピー" と "URL を開く" は標準アクションとして提供されている
- 本ツールの `?paste=1` パラメータは「起動時にペースト待機メッセージを表示する」だけ。**iOS/macOS Safari は `navigator.clipboard.read()` にユーザージェスチャ（クリック等）を要求する**ため自動読込はできない。ユーザが Cmd+V を1回押す必要がある

#### 動かない条件 / 既知の限界

- ⚠️ クリップボードの画像は「テキスト形式の画像」として扱われるか「ファイル形式」として扱われるかでブラウザの挙動が異なる。Safari は両対応、Firefox 一部の経路で取得できないケースあり
- ⚠️ Quick Action からブラウザを起動した直後、ブラウザがフォアグラウンドになっていてもページが完全に読み込まれる前に Cmd+V を押すとペーストイベントが拾えないことがある。1秒待ってから Cmd+V

---

## S3: 外で撮影してヴァルール確認 📖

### 状況

散歩中、街角で「描きたい」と思う風景に出会った。スマホを取り出して、その場で明暗の組み立てを確認したい。

### 経路 A: 何もセットアップ無し ✅

1. iPhone Safari で <https://odayakalife.dev/check-value-app/> を開く
2. 共有ボタン → 「**ホーム画面に追加**」
3. 以降、ホーム画面のアイコンをタップ → 「**カメラ**」ボタン → 撮影

**手数**: アイコンタップ + カメラボタン + 撮影 = 3タップ

#### 根拠 (✅)

- iOS Safari の "ホーム画面に追加" は [Apple Web App ガイド](https://developer.apple.com/library/archive/documentation/AppleApplications/Reference/SafariWebContent/ConfiguringWebApplications/ConfiguringWebApplications.html) に記載
- `<meta name="apple-mobile-web-app-capable" content="yes">` でスタンドアロン起動になる（`index.html` で設定済み）
- `<link rel="apple-touch-icon" ...>` で `icons/apple-touch-icon.png` をホーム画面アイコンに使う

### 経路 B: 1タップに短縮したい — iOS Shortcuts.app 📖

カメラを直接開きたい場合。Action Button（iPhone 15 Pro 以降）またはロック画面のショートカットから1タップで起動できる。

#### セットアップ (初回1回)

1. **ショートカット.app** を起動
2. 新規ショートカット作成（右上「+」）
3. アクション追加: **「URLを開く」** — `https://odayakalife.dev/check-value-app/?capture=1`
4. ショートカット名「ヴァルール撮影」
5. 設定 → **アクションボタン**（iPhone 15 Pro 以降）→ "ショートカット" → "ヴァルール撮影" を選択
6. （アクションボタンが無い機種）ショートカット.app から **ホーム画面に追加**、または iPhone のコントロールセンターにショートカット を追加

#### 毎回の操作

アクションボタン長押し（または上記で追加した起動方法）→ 一瞬で PWA が起動、自動でカメラが開く → タップで撮影

#### 根拠 (📖)

- アクションボタンへのショートカット割当ては [Apple Support: iPhone のアクションボタン](https://support.apple.com/ja-jp/HT213882) に記載
- `?capture=1` パラメータは `index.html` のブート処理で検知され、`navigator.mediaDevices.getUserMedia({video: {facingMode: 'environment'}})` を即呼び出してカメラモーダルを開く（実装済み、コードは [index.html](./index.html) の `openCamera()` 周辺）

#### 動かない条件

- ⚠️ iOS Safari で `getUserMedia` を使う際、HTTPS かつユーザージェスチャ後の呼び出しが必要。`?capture=1` でブート時に自動起動するため、ホーム画面 PWA からの起動なら「アイコンタップ」がジェスチャ扱いになる。Safari の通常タブから直接 URL を打って `?capture=1` を渡した場合、最初の `getUserMedia` 呼び出しが拒否される可能性あり → その場合は手動で「カメラ」ボタンを再タップ

### 経路 C: 撮影済み写真を渡す（共有シート経由） 📖

カメラロールにすでにある写真を選んで送りたい。

#### セットアップ

1. ショートカット.app で新規作成
2. アクション: **「クリップボードにコピー」** + **「URLを開く」** `https://odayakalife.dev/check-value-app/?paste=1`
3. 設定 → "共有シートに表示" オン、入力タイプ「画像」
4. ショートカット名「ヴァルール確認」

#### 毎回の操作

写真.app で画像を表示 → 共有ボタン → 一覧から「ヴァルール確認」→ PWA が起動 → 画面の「ペースト」ボタンをタップ

#### 根拠 (📖)

- iOS Safari の Web Share **Target** API は未対応（[WebKit Bug #194593](https://bugs.webkit.org/show_bug.cgi?id=194593) で 2019 年から議論中、2026 年 5 月時点で WebKit のスタンスは "neutral"）。**そのため共有シート → PWA 直接渡しは不可**
- 回避策として、共有シート受け取りが可能な「ショートカット」を中継し、クリップボード経由で渡す。これが現状唯一の安定経路

#### 動かない条件

- ⚠️ iOS の共有シートから来た「画像」がショートカットでクリップボードに入る形式は HEIC か JPEG か実機依存。本ツールは `image/*` を受けるので両対応のはずだが、未検証

---

## URL パラメータ仕様 ✅

`index.html` が解釈するクエリパラメータ:

| パラメータ      | 動作                                                              | 実装                                       |
| --------------- | ----------------------------------------------------------------- | ------------------------------------------ |
| `?img=<URL>`    | 指定 URL の画像を crossOrigin=anonymous で読み込む                | `loadImage(src, 'anonymous')`              |
| `?source=<URL>` | 旧パラメータ、`?img=` 同等だが crossOrigin 無し（同 origin 推奨） | 後方互換                                   |
| `?capture=1`    | ブート時に `openCamera()` を呼ぶ                                  | `getUserMedia({facingMode:'environment'})` |
| `?paste=1`      | ブート時にステータスバーに「Cmd/Ctrl+V でペースト」表示           | `setStatus(...)`                           |

実装は `index.html` のスクリプト末尾、`URLSearchParams` 解析と `useImageSource(initialSrc, ...).then(...)` のブロック。

---

## 検証されていない・する必要があること

- ⚠️ macOS Shortcuts.app での Quick Action 構築（記載手順は Apple ドキュメントから組み立てたが、Quick Action 入力からクリップボードへの実装は実機で要検証）
- ⚠️ iOS Shortcuts.app での「URL を開く」アクションが、ホーム画面 PWA を `start_url` でなく直接 `?capture=1` 付き URL で起動するか（manifest の `scope` 内なら PWA として、外なら Safari タブとして起動の挙動）
- ⚠️ Action Button への割当てが iPhone 15 Pro 以前の機種で代替手段で代替できるか
- ⚠️ S3 の共有シート経由フローで HEIC ファイルがクリップボード→ブラウザペーストで通るか

上記は実機検証してフィードバックを Issue にコメントしてもらうと、このドキュメントを精度化できる。
