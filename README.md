# QRコード読み取りプロトタイプ

見積明細R2 No.6「QRコード読み取り（全3画面）」の技術検証用プロトタイプ。
CakePHP 3.7 + jQuery 3.3.1 環境への統合を前提に、カメラ経由のQR読み取り機能を単体で検証する。

## 技術スタック

| 項目 | バージョン | 備考 |
|------|-----------|------|
| html5-qrcode | 2.3.8 | QRスキャンライブラリ（CDN） |
| jQuery | 3.3.1 | 本番と同一バージョン（CDN） |
| 対象ブラウザ | iOS Safari | 背面カメラ固定 |

## ローカルでの確認方法

### PC（ブラウザ確認）

```bash
cd experiment/qr-reader-prototype
python3 -m http.server 8000
```

ブラウザで `http://localhost:8000` を開く。

> PC内蔵カメラが使われる。QRコードの画像をスマホ等に表示して読み取り可能。

### 実機（iPhone / iPad）確認 — ngrok

iOS Safari ではカメラアクセスに HTTPS が必須のため、ngrok を使用する。

#### 1. ngrok インストール（未導入の場合）

```bash
brew install ngrok
ngrok config add-authtoken <YOUR_TOKEN>
```

> トークンは https://dashboard.ngrok.com で取得。

#### 2. ローカルサーバー起動

```bash
cd experiment/qr-reader-prototype
python3 -m http.server 8000
```

#### 3. ngrok でHTTPSトンネル作成

別ターミナルで:

```bash
ngrok http 8000
```

表示された `https://xxxx.ngrok-free.app` のURLをiPhoneのSafariで開く。

#### 注意事項

- ngrok 無料プランは接続数・帯域に制限あり（検証用途には十分）
- 毎回URLが変わるため、QRコードでURLを端末に送ると便利

## カメラ起動方式

背面カメラを2段フォールバックで取得する:

1. `facingMode: { exact: "environment" }` — 背面カメラを厳密指定
2. 失敗時 → `facingMode: { ideal: "environment" }` — 背面カメラ優先で再試行

カメラ選択UIは設けない（実運用では背面カメラ一択のため）。

## 結果テキストボックス

- QRコード読み取り成功時に自動入力される
- 手入力も可能（QRコードが読み取れない場合の代替手段として）
- QR読み取り時は緑色ハイライトで視覚フィードバック

## QrScannerModule API

### コールバック

```javascript
// 読み取り成功時（主経路）
QrScannerModule.onScanSuccess = function(decodedText) {
  console.log('読み取り値:', decodedText);
};

// エラー発生時
// type: 'permission_denied' | 'camera_not_found' | 'unknown'
QrScannerModule.onError = function(errorMessage, type) {
  console.error(type, errorMessage);
};

// 状態変化時
// state: 'idle' | 'starting' | 'scanning' | 'stopping' | 'error'
QrScannerModule.onStateChange = function(state) {
  console.log('状態:', state);
};
```

### メソッド

```javascript
QrScannerModule.start('qr-reader');  // スキャン開始（idle時のみ）
QrScannerModule.stop();              // スキャン停止（scanning時のみ）
QrScannerModule.isActive();          // scanning or starting → true
QrScannerModule.destroy();           // 完全クリーンアップ
```

### 設定

```javascript
QrScannerModule.config.enableEnterFallback = true;  // Enterキー発火を有効化
```

### 状態遷移図

```
idle → starting → scanning → stopping → idle
  ↓                  ↓          ↓
error ←─────────── error ←─── error
```

- 不正な遷移は無視される（例: scanning中にstart()を呼んでも何も起きない）
- 各遷移時に `onStateChange` が発火する

## 検証チェックリスト

| # | 項目 | 確認方法 |
|---|------|----------|
| 1 | カメラ起動 | 「スキャン開始」タップ → プレビュー表示 + ステータス「スキャン中」 |
| 2 | QR読み取り | QRかざす → テキストボックスに値 + 緑ハイライト + コンソールログ |
| 3 | 手入力 | テキストボックスに直接入力できること |
| 4 | スキャン停止 | 「スキャン停止」タップ → カメラ停止 + ステータス「待機中」 |
| 5 | 連打テスト | 開始→即停止→即開始 → エラーなく遷移 |
| 6 | 二重検出防止 | 同一QRを映し続ける → `onScanSuccess` が1回のみ |
| 7 | カメラ権限拒否 | 権限拒否 → エラーメッセージ表示 |
| 8 | タブ切替 | バックグラウンドへ → カメラ解放 |
| 9 | ページ離脱 | 別ページへ遷移 → カメラ解放 |

## 本番統合時の注意点

### ライブラリ配置

```
webroot/js/lib/html5-qrcode.min.js  ← CDNからダウンロードしてローカル配置
```

- バージョン 2.3.8 を固定。SRI（Subresource Integrity）推奨
- CDN参照は本番では使用しない

### jQuery

本番の jQuery 3.3.1 をそのまま使用。追加読み込み不要。

### QrScannerModule の組み込み

1. `QrScannerModule` のコードを独立 `.js` ファイルに分離
2. CakePHPのViewテンプレートで読み込み
3. `onScanSuccess` にページ固有のハンドラを設定（例: Ajax送信、画面遷移）

### Enter キーフォールバック

既存画面がEnterキーでフォーム送信する設計の場合:

```javascript
QrScannerModule.config.enableEnterFallback = true;
```

これにより、QR読み取り成功時に `onScanSuccess` に加えて `keydown(Enter)` イベントも発火する。

### 読み取り履歴

プロトタイプのデバッグ用機能。本番では履歴セクションのHTMLを除外し、`addHistory()` 呼び出しを削除する。

### カメラ解放

本番でも以下のイベントハンドラを必ず設定すること:

- `pagehide` — iOS Safariで確実に動作するカメラ解放
- `visibilitychange` — バックグラウンド遷移時の補助

`beforeunload` はiOS Safariで不安定なため使用しない。
