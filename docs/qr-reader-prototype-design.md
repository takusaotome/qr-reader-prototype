# QRコード読み取りプロトタイプ 設計書

| 項目 | 内容 |
|------|------|
| 文書ID | QR-PROTO-DESIGN-001 |
| 対象 | `experiment/qr-reader-prototype/` |
| 作成日 | 2026-02-27 |
| 作成者 | 開発チーム |
| 文書性質 | 実装の規約書（将来の本番統合・保守時の準拠仕様） |

### 改版履歴

| 版 | 日付 | 内容 |
|----|------|------|
| 1.0 | 2026-02-27 | 初版作成 |
| 1.1 | 2026-02-27 | コードレビュー対応: try/finally, _startGeneration, destroy()インスタンスキャプチャ |
| 1.2 | 2026-02-27 | 設計書レビュー対応: 非同期競合仕様の具体化、状態機械のdestroy()規定、UI Controller層の分離記述、HTTPS前提条件の追加 |
| 1.3 | 2026-02-27 | 再レビュー対応: stop()失敗回復パスを動作フローに追記、destroy()コード例に同期例外の注記追加、未使用のerror→idle遷移を削除 |

---

## 1. 概要

QRコード読み取り機能の開発に先立ち、カメラ経由のQRコード読み取り機能を独立プロトタイプとして実装する。
本番プロジェクト（CakePHP 3.7 + jQuery 3.3.1、iOS Safari対象）へ後日統合することを前提に、技術スタックを揃えた形で作成する。

## 2. 技術選定と前提条件

| 項目 | 選定 | バージョン | 備考 |
|------|------|-----------|------|
| QRスキャンライブラリ | html5-qrcode | 2.3.8 | iOS Safari対応済み、`Html5Qrcode` ローレベルAPI使用 |
| jQuery | jQuery | 3.3.1 | 本番と同一バージョン |
| SRI (html5-qrcode) | — | — | `sha256-ZgsSQ3sddH4+aLi+BoXAjLcoFAEQrSE/FnsUtm+LHY4=` |
| SRI (jQuery) | — | — | `sha256-FgpCb/KJQlLNfOu91ta32o/NMZxltwRo8QtmkMRdAu8=` |

- **プロトタイプ**: CDN経由（バージョン固定 + SRI）
- **本番**: `webroot/js/lib/html5-qrcode.min.js` にローカル配置

### HTTPS要件（必須）

iOS Safari ではカメラアクセス（`getUserMedia`）に **HTTPS が必須**（localhost除く）。
本番環境のサーバーにSSL証明書が設定されていることが前提条件となる。
プロトタイプの実機テストでは ngrok でHTTPSトンネルを使用する。

## 3. ファイル構成

```text
experiment/qr-reader-prototype/
├── index.html                            # プロトタイプ本体（HTML + CSS + JS 単一ファイル）
├── README.md                             # 使い方・ngrok実機テスト手順・本番統合時の注意点
└── docs/
    └── qr-reader-prototype-design.md     # 本設計書
```

## 4. カメラ起動方式

背面カメラを2段フォールバックで取得する。カメラ選択UIは設けない（実運用では背面カメラ一択のため）。

```text
1. facingMode: { exact: "environment" }    ← 背面カメラ厳密指定
     ↓ 失敗時
   インスタンス再生成（Html5Qrcode.clear() → new Html5Qrcode()）
     ↓
2. facingMode: { ideal: "environment" }    ← 背面カメラ優先で再試行
     ↓ 失敗時
   onError(message, type) を発火 → エラー状態へ遷移
```

### インスタンス再生成の理由

html5-qrcode ライブラリは `start()` 失敗後に内部状態が遷移中のまま残る場合がある。
同一インスタンスで再度 `start()` を呼ぶと "Cannot transition to a new state, already under transition" エラーが発生するため、
フォールバック前に `clear()` → `new Html5Qrcode()` でインスタンスを再生成する。

## 5. QrScannerModule の設計

### 5.1 状態機械

```text
idle ──→ starting ──→ scanning ──→ stopping ──→ idle
  ↑          │            │            │
  │          ↓            ↓            ↓
  └─────── error ←────── error ←───── error

  ※ destroy() は任意の状態から idle へ強制遷移（下記「destroy() の特別規定」参照）
```

#### 通常遷移テーブル

| 現在の状態 | 許可される遷移先 | 遷移契機 |
|-----------|----------------|---------|
| `idle` | `starting` | `start()` 呼び出し |
| `starting` | `scanning` | カメラ起動成功 |
| `starting` | `error` | カメラ起動失敗 |
| `scanning` | `stopping` | `stop()` 呼び出し / QR検出後の自動停止 |
| `scanning` | `error` | 予期せぬエラー |
| `stopping` | `idle` | カメラ停止完了 |
| `stopping` | `error` | カメラ停止失敗 |
| `error` | `starting` | `start()` 呼び出し（エラーからの再試行） |

- 通常遷移はすべて `_setState()` を経由し、`onStateChange(newState)` を発火する
- 許可されない遷移は `_setState()` が拒否し、`console.warn` を出力

#### destroy() の特別規定

`destroy()` はページ離脱・バックグラウンド遷移時の緊急クリーンアップであり、**通常の状態遷移とは独立した強制リセット操作**として扱う。

- **任意の状態から `idle` へ強制遷移**する（`_setState()` は経由しない）
- `_state = 'idle'` を直接代入した後、`onStateChange('idle')` を明示的に発火する
- `_setState()` を経由しない理由: `destroy()` は `stopping` 中や `starting` 中など、通常遷移テーブルで `idle` への直接遷移が許可されていない状態からも呼ばれる必要があるため

### 5.2 非同期競合防止

#### 5.2.1 _startGeneration カウンター（Promiseの無効化）

`start()` 呼び出しごとにインクリメントされるカウンター。`destroy()` でもインクリメントする。
非同期の `.then()` / `.catch()` コールバック内で `isStale()` を判定し、
`destroy()` 後に到着した古いPromiseの結果が状態を上書きすることを防止する。

```javascript
var generation = ++this._startGeneration;
function isStale() { return generation !== self._startGeneration; }

// Promise callback 内
.then(function() {
  if (isStale()) return;  // destroy() 済みなら何もしない
  self._setState('scanning');
})
```

#### 5.2.2 destroy() のインスタンスキャプチャ（設計必須）

`destroy()` 内の `stop().then(…clear…)` は非同期で実行される。
この間に `start()` が呼ばれると `this._html5QrCode` が新インスタンスに差し替わり、
古い非同期コールバックが `this._html5QrCode.clear()` で**新しいインスタンスを誤って破棄する**競合が発生する。

**規定**: `destroy()` では、非同期処理の開始前に対象インスタンスをローカル変数にキャプチャし、
コールバック内ではローカル変数経由でのみ旧インスタンスを操作すること。

```javascript
destroy: function() {
  this._startGeneration++;

  // 【必須】現在のインスタンスをローカル変数にキャプチャ
  var instance = this._html5QrCode;
  if (instance) {
    if (this._state === 'scanning' || this._state === 'starting') {
      instance.stop().then(function() {
        instance.clear();     // ← this._html5QrCode ではなく instance を使用
      }).catch(function() {
        instance.clear();
      });
    } else {
      instance.clear();
    }
  }
  this._state = 'idle';
  this.onStateChange('idle');
  this._html5QrCode = null;
  this._hasHandledScan = false;
}
```

> **実装上の補足**: `starting` 状態でライブラリが `stop()` をサポートしない場合、`stop()` が同期的に例外を投げる可能性がある。実装では `instance.stop()` の呼び出し自体を `try/catch` で囲み、同期例外時にも `instance.clear()` を確実に実行すること。各 `clear()` 呼び出しも個別に `try/catch` で囲む。

#### 5.2.3 _hasHandledScan ロック（二重検出防止）

1回のスキャン成功後、`stop()` 完了まで後続のQR検出通知を無視する。

- `_handleScanResult()` 先頭で `_hasHandledScan === true` なら即 return
- `_handleScanResult()` 先頭で `_state !== 'scanning'` なら即 return
- `stop()` 完了時（`idle` 遷移時）に `false` にリセット

### 5.3 コールバック例外安全性

`_handleScanResult()` 内で `onScanSuccess()` を `try/finally` で囲む。
外部コールバックが例外を投げても `finally` ブロックで `stop()` が必ず実行される。

```javascript
_handleScanResult: function(decodedText) {
  if (this._hasHandledScan) return;
  if (this._state !== 'scanning') return;

  this._hasHandledScan = true;
  try {
    this.onScanSuccess(decodedText);
    // enableEnterFallback 処理
  } catch (e) {
    console.error('[QrScanner] onScanSuccess threw:', e);
  } finally {
    this.stop();  // 例外時も必ず実行
  }
}
```

**例外発生時の回復フロー**:
1. `onScanSuccess()` が例外 → `catch` がログ出力
2. `finally` で `stop()` 実行 → `scanning` → `stopping`
3. `stop()` 内の Promise 完了 → `_hasHandledScan = false` にリセット → `stopping` → `idle`
4. `idle` 状態に復帰 → 再スキャン可能

### 5.4 公開API

```javascript
var QrScannerModule = {
  // --- 設定 ---
  config: {
    enableEnterFallback: false,  // true時のみEnterキーイベントも発火
    qrboxSize: 250,              // QR読み取り枠サイズ（px）
    fps: 10                      // スキャンフレームレート
  },

  // --- コールバック（外部から差し替え可能）---
  onScanSuccess: function(decodedText) {},   // 読み取り成功時（主経路）
  onError: function(errorMessage, type) {},  // エラー（permission_denied / camera_not_found / unknown）
  onStateChange: function(state) {},         // idle / starting / scanning / stopping / error

  // --- メソッド ---
  start: function(targetElementId) {},  // idle / error 時のみ受付
  stop: function() {},                  // scanning 時のみ受付
  isActive: function() {},              // scanning or starting → true
  destroy: function() {}                // 任意状態から強制クリーンアップ（5.1 特別規定参照）
};
```

### 5.5 カメラ解放

| イベント | 用途 | 備考 |
|---------|------|------|
| `pagehide` | メインのカメラ解放 | iOS Safariで安定動作（必須） |
| `visibilitychange` | バックグラウンド遷移検出 | 補助的に併用 |

`beforeunload` はiOS Safariで発火が不安定なため使用しない。

## 6. 画面構成

### 6.1 画面要素

| # | 要素 | 説明 |
|---|------|------|
| 1 | ステータスバー | 状態機械に連動した色変化（5色）+ テキスト + パルスアニメーション |
| 2 | カメラプレビュー領域 | `html5-qrcode` が描画する `<video>` ストリーム（`#qr-reader`） |
| 3 | エラーメッセージ | 赤背景、5秒後にフェードアウト |
| 4 | スキャン開始/停止ボタン | 状態に応じて `disabled` 制御（`starting` / `stopping` 中は両方無効） |
| 5 | 読み取り結果テキストボックス | QR読み取り時に自動入力 + 手入力も可能（`readonly` なし） |
| 6 | 読み取り履歴 | デバッグ用。最大20件、タイムスタンプ付き、「クリア」ボタン。本番では除外する |

### 6.2 UI Controller層

QrScannerModule はUIに依存しない純粋なスキャンエンジンとして設計する。
UI Controller層はプロトタイプの `index.html` 内に同梱されるが、モジュールとは以下のコールバック接続のみで結合する。

```text
QrScannerModule                      UI Controller
─────────────────                    ──────────────
onStateChange(state)  ──────────→  updateStatusUI(state) + updateButtonState(state)
onScanSuccess(text)   ──────────→  setResult(text) + addHistory(text)
onError(msg, type)    ──────────→  showError(userMessage)
```

#### UI Controller の関数一覧

| 関数 | 責務 |
|------|------|
| `updateStatusUI(state)` | ステータスバーのCSS class切替 + テキスト更新 |
| `updateButtonState(state)` | 開始/停止ボタンの `disabled` 制御 |
| `setResult(text)` | テキストボックスへの値セット + `trigger('input')` + 緑ハイライト（2秒） |
| `addHistory(text)` | 履歴リストへの追加（最大20件、FIFO） |
| `showError(message)` | エラーメッセージの表示（5秒後フェードアウト） |

本番統合時は、UI Controller層をCakePHPのViewテンプレートに合わせて差し替える。
QrScannerModuleのコールバックに本番固有のハンドラ（Ajax送信、画面遷移等）をバインドする。

### 6.3 ステータスバーの色定義

| 状態 | 背景色 | テキスト色 | ドット色 | アニメーション |
|------|--------|-----------|---------|--------------|
| `idle` | `#e8e8e8` | `#666` | `#999` | なし |
| `starting` | `#fff3cd` | `#856404` | `#ffc107` | パルス |
| `scanning` | `#d4edda` | `#155724` | `#28a745` | パルス |
| `stopping` | `#fff3cd` | `#856404` | `#ffc107` | なし |
| `error` | `#f8d7da` | `#721c24` | `#dc3545` | なし |

### 6.4 ボタン状態制御

| 状態 | スキャン開始 | スキャン停止 |
|------|------------|------------|
| `idle` | 有効 | 無効 |
| `starting` | 無効 | 無効 |
| `scanning` | 無効 | 有効 |
| `stopping` | 無効 | 無効 |
| `error` | 有効 | 無効 |

## 7. 動作フロー

### 7.1 正常系

1. ユーザーが「スキャン開始」をタップ
   → QrScannerModule: `idle` → `starting`
   → UI Controller: ステータス「カメラ起動中...」（黄）、両ボタン無効
2. カメラ起動（`exact: "environment"` → 失敗時 `ideal: "environment"` にフォールバック）
3. カメラ起動成功
   → QrScannerModule: `starting` → `scanning`
   → UI Controller: ステータス「スキャン中」（緑）、停止ボタン有効
4. QRコード検出
   → QrScannerModule: `_hasHandledScan = true` → `onScanSuccess(decodedText)` → `stop()`
   → UI Controller: テキストボックスに値セット + 緑ハイライト + 履歴追加
5. カメラ停止完了
   → QrScannerModule: `scanning` → `stopping` → `idle`、`_hasHandledScan = false`
   → UI Controller: ステータス「待機中」（グレー）、開始ボタン有効

### 7.2 エラー系

**カメラ起動失敗**（`starting` → `error`）:
- カメラ権限拒否 → `onError(msg, 'permission_denied')` → UI Controller: エラーメッセージ表示
- カメラ未検出 → `onError(msg, 'camera_not_found')` → UI Controller: エラーメッセージ表示
- その他の起動エラー → `onError(msg, 'unknown')` → UI Controller: エラーメッセージ表示

**カメラ停止失敗**（`stopping` → `error`）:
- `stop()` 内の `Html5Qrcode.stop()` が reject → `_hasHandledScan = false` にリセット → `error` 状態に遷移
- UI Controller: ステータス「エラー」（赤）、開始ボタン有効
- ユーザーが「スキャン開始」をタップ → `error` → `starting` で復帰

### 7.3 ページ離脱・バックグラウンド遷移

- `pagehide` / `visibilitychange(hidden)` → `destroy()` → 強制 `idle` 化（セクション5.1 特別規定）

## 8. テスト環境

| 環境 | 手順 |
|------|------|
| PC確認 | `python3 -m http.server 8000` → `http://localhost:8000` |
| 実機確認 | 上記に加え `ngrok http 8000` → HTTPS URL に実機からアクセス |

iOS Safari のカメラアクセスにはHTTPSが必須（localhost除く）。セクション2の前提条件を参照。

## 9. 検証項目

### 9.1 基本機能

| # | 項目 | 期待結果 |
|---|------|---------|
| 1 | スキャン開始 | カメラ起動 + ステータス「スキャン中」（緑） |
| 2 | QR読み取り | テキストボックスに文字列 + 緑ハイライト + `onScanSuccess` 発火 |
| 3 | 手入力 | テキストボックスに直接入力可能 |
| 4 | スキャン停止 | カメラ停止 + ステータス「待機中」（グレー） |

### 9.2 エッジケース

| # | 項目 | 期待結果 |
|---|------|---------|
| 5 | 連打テスト | 開始→即停止→即開始でエラーなく遷移 |
| 6 | 二重検出防止 | 同一QRを映し続けて `onScanSuccess` が1回のみ |
| 7 | カメラ権限拒否 | `onError(msg, 'permission_denied')` + エラーメッセージ表示 |
| 8 | タブ切替 | `visibilitychange` で `destroy()` → ステータス「待機中」に戻る |
| 9 | ページ離脱 | `pagehide` で `destroy()` → カメラ解放 |

### 9.3 例外・競合

| # | 項目 | 期待結果 |
|---|------|---------|
| 10 | コールバック例外 | `onScanSuccess` が例外を投げても `stop()` が実行される |
| 10a | — 例外後の状態復帰 | `state` が `idle` に復帰し、`_hasHandledScan` が `false` にリセットされる |
| 10b | — 例外後の再スキャン | 「スキャン開始」が有効になり、再度スキャンを実行できる |
| 11 | destroy競合 | `starting` 中に `destroy()` → 古いPromiseが状態を上書きしない |
| 12 | destroy後の再開 | `destroy()` 後に `start()` → 新インスタンスで正常にスキャン開始 |

## 10. 本番統合時の注意点

### ライブラリ配置

```text
webroot/js/lib/html5-qrcode.min.js  ← CDNからダウンロードしてローカル配置（v2.3.8固定）
```

- SRI推奨。CDN参照は本番では使用しない
- jQuery 3.3.1 は本番既存のものをそのまま使用

### QrScannerModule の組み込み

1. `QrScannerModule` のコードを独立 `.js` ファイルに分離
2. CakePHPのViewテンプレートで読み込み
3. `onScanSuccess` にページ固有のハンドラを設定（例: Ajax送信、画面遷移）
4. UI Controller層は各画面のViewに合わせて個別実装

### Enterキーフォールバック

既存画面がEnterキーでフォーム送信する設計の場合:

```javascript
QrScannerModule.config.enableEnterFallback = true;
```

QR読み取り成功時に `onScanSuccess` に加えて `keydown(Enter)` イベントも発火する。

### 読み取り履歴

プロトタイプのデバッグ用機能。本番では履歴セクションのHTMLを除外し、`addHistory()` 呼び出しを削除する。

### カメラ解放

本番でも `pagehide` + `visibilitychange` イベントハンドラを必ず設定すること。
`beforeunload` はiOS Safariで不安定なため使用しない。
