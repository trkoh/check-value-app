# 配信導線セットアップ

3シナリオ（Mac画像 / Web上画像 / 屋外カメラ）で「PWA を1〜2タップで起動」できるようにする手順。サーバーは介さない。

公開URL: <https://odayakalife.dev/check-value-app/>

## S2 (Pinterest等 Web): ブックマークレット

現在見ているページの一番大きい `<img>` を抽出して、ヴァルール変換ツールに `?img=` で渡す。

### 使い方

ブックマークバーに以下のリンクをドラッグ:

[**ヴァルール確認**](javascript:(function()%7Bvar%20i%3DArray.from(document.images)%3Bif(!i.length)%7Balert('%E7%94%BB%E5%83%8F%E3%81%8C%E8%A6%8B%E3%81%A4%E3%81%8B%E3%82%8A%E3%81%BE%E3%81%9B%E3%82%93')%3Breturn%7Di.sort(function(a%2Cb)%7Breturn(b.naturalWidth*b.naturalHeight)-(a.naturalWidth*a.naturalHeight)%7D)%3Bvar%20u%3Di%5B0%5D.currentSrc%7C%7Ci%5B0%5D.src%3Bwindow.open('https%3A%2F%2Fodayakalife.dev%2Fcheck-value-app%2F%3Fimg%3D'%2BencodeURIComponent(u)%2C'_blank')%3B%7D)())

（GitHub上ではリンクのドラッグが効かない場合がある。プレーンHTMLで開けば確実にドラッグできる）

### ソース

```js
javascript:(function(){
  var imgs = Array.from(document.images);
  if (!imgs.length) { alert('画像が見つかりません'); return; }
  imgs.sort(function (a, b) {
    return (b.naturalWidth * b.naturalHeight) - (a.naturalWidth * a.naturalHeight);
  });
  var url = imgs[0].currentSrc || imgs[0].src;
  window.open('https://odayakalife.dev/check-value-app/?img=' + encodeURIComponent(url), '_blank');
})();
```

### 制限

- Pinterest のように `<img>` が CSS `background-image` で配置されている場合は拾えない。
- 画像URLが認証保護 / リファラ制限されている場合、PWA 側で読み込みが CORS で失敗する。
- 一部のサイトは CSP で `javascript:` URL を弾く（X / Twitter 等）。

## S1 (Mac画像): macOS Quick Action

Finder で画像を右クリック → "クイックアクション" → "ヴァルール確認" でブラウザに渡す。

### Shortcuts.app での作成手順

1. **Shortcuts.app** を開き、新規ショートカット作成
2. 設定アイコン → "詳細"
   - ✅ **共有シートに表示**
   - ✅ **クイックアクションとして使用** → "Finder" にチェック
   - 入力: **画像 / ファイル**
3. アクションを順に追加:
   - **「base64エンコード」** — 入力 = ショートカットの入力
   - **「テキスト」** — 内容: `data:image/jpeg;base64,` + 上のbase64出力（変数挿入）
   - **「URLを開く」** — 内容: `https://odayakalife.dev/check-value-app/?img=` の後にエンコード済みURLを連結
4. ショートカット名: **「ヴァルール確認」**

### 別案（簡易）

base64 が大きすぎてURLに収まらない場合は、画像をiCloudやimgurに一旦アップロードしてそのURLを渡すアクションに置き換える。ローカル完結を諦める。

または、ファイルをクリップボードにコピー → `?paste=1` で起動 → 起動後にユーザが Cmd+V する2タップフロー:

1. **「クリップボードにコピー」** — 入力 = ショートカット入力
2. **「URLを開く」** — `https://odayakalife.dev/check-value-app/?paste=1`

こちらの方が確実で、現実的にはこちらを推奨。

## S3 (屋外カメラ): iOS Shortcut → Action Button

iPhone 15+ の Action Button、または iPhone 標準のロック画面ショートカットから1タップでカメラ直結起動する。

### Shortcuts.app (iOS) での作成手順

1. **Shortcuts.app** で新規ショートカット
2. アクションを追加:
   - **「URLを開く」** — 内容: `https://odayakalife.dev/check-value-app/?capture=1`
3. ショートカット名: **「ヴァルール撮影」**
4. 設定 → 詳細 → **「ホーム画面に追加」** または、iPhone 設定 → アクションボタン → ショートカット → このショートカットを割り当て

### 動作

- PWA をホーム画面に追加している場合は、PWA がスタンドアロンで起動する
- 起動時に `?capture=1` を検知して getUserMedia を即起動 → カメラ画面 → 「撮影」タップで変換結果

### 共有シートからの呼び出し

写真.app の共有シートから渡したい場合:

1. ショートカットの設定で **「共有シートに表示」** ✅、入力: **画像**
2. アクション:
   - **「クリップボードにコピー」** — 入力 = ショートカットの入力
   - **「URLを開く」** — `https://odayakalife.dev/check-value-app/?paste=1`
3. 起動後の PWA で Cmd/Ctrl+V 相当（iOS Safari なら長押し → ペースト、または PWA 上の「ペースト」ボタン）

iOS Safari の Web Share Target API は未対応（WebKit bug #194593, 2026年5月時点）のため、上記のクリップボード経由がもっとも安定した回避策。

## URLパラメータ一覧

| パラメータ | 動作 |
|---|---|
| `?img=<URL>` | 任意の画像URLを入力に（外部URLはCORS許可が必要） |
| `?source=<URL>` | 旧パラメータ（同origin推奨、後方互換） |
| `?capture=1` | 起動と同時にカメラを開く |
| `?paste=1` | 起動時にペースト待機メッセージを表示 |
