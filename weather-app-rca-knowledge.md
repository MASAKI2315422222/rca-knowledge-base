# お天気アプリ（天気予報アプリ）RCA ナレッジベース

## アプリ概要

| 項目 | 内容 |
|---|---|
| **アプリ名** | 天気予報アプリ |
| **技術スタック** | Node.js / Express / axios |
| **ポート** | 3001 |
| **ヘルスチェック** | `GET /health` |
| **メインAPI** | `GET /api/weather?postalCode=XXXXXXX` |

---

## 外部依存サービス一覧

| サービス | URL | 用途 | タイムアウト |
|---|---|---|---|
| zipcloud | zipcloud.ibsnet.co.jp | 郵便番号→住所変換 | 10秒 |
| Open-Meteo Geocoding | geocoding-api.open-meteo.com | 住所→緯度経度変換 | 10秒 |
| Open-Meteo Forecast | api.open-meteo.com | 7日間天気予報取得 | 10秒 |
| RainViewer | api.rainviewer.com | 雨雲レーダー | 10秒 |
| OpenStreetMap | tile.openstreetmap.org | 地図タイル表示 | 10秒 |

---

## エラーコード一覧と根本原因パターン

### ZIPCLOUD_TIMEOUT
**発生条件：** zipcloud API への接続が10秒以内に完了しない  
**HTTP ステータス：** 504  
**根本原因パターン：**
- zipcloud.ibsnet.co.jp サービス障害・メンテナンス
- ネットワーク経路の遅延・輻輳
- DNS 解決の失敗

**確認手順：**
1. `curl https://zipcloud.ibsnet.co.jp/api/search?zipcode=1000001` で疎通確認
2. zipcloud の公式ステータスページを確認
3. サーバー側のネットワークログを確認

**再発防止策：**
- zipcloud の代替サービスをフォールバックとして追加
- タイムアウト値の見直し（現在10秒）
- リトライロジックの実装

---

### GEOCODING_TIMEOUT
**発生条件：** Open-Meteo Geocoding API への接続が10秒以内に完了しない  
**HTTP ステータス：** 504  
**根本原因パターン：**
- Open-Meteo サービス障害
- ネットワーク経路の遅延
- フォールバック検索パターンの多重実行による遅延

**確認手順：**
1. `curl "https://geocoding-api.open-meteo.com/v1/search?name=Tokyo&count=1"` で疎通確認
2. Open-Meteo ステータスページ確認（https://status.open-meteo.com）
3. フォールバックパターン数を確認（最大5パターン × 10秒 = 最大50秒の可能性あり）

**再発防止策：**
- フォールバックパターンごとのタイムアウトを短縮（例：3秒）
- Geocoding 結果のキャッシュ実装（同一郵便番号の再問い合わせ削減）

---

### FORECAST_TIMEOUT
**発生条件：** Open-Meteo Forecast API への接続が10秒以内に完了しない  
**HTTP ステータス：** 504  
**根本原因パターン：**
- Open-Meteo サービス障害
- 大量リクエストによるレート制限
- hourly データ（7日×24時間）のレスポンスサイズが大きい

**確認手順：**
1. `curl "https://api.open-meteo.com/v1/forecast?latitude=35.68&longitude=139.69&daily=weather_code&timezone=Asia/Tokyo"` で疎通確認
2. リクエストパラメータの hourly 項目数を確認（現在6項目）

**再発防止策：**
- hourly データの取得項目を必要最小限に絞る
- daily データのみ先に取得し、hourly は非同期で取得する設計に変更

---

### POSTAL_CODE_NOT_FOUND
**発生条件：** zipcloud が該当する郵便番号を返さない  
**HTTP ステータス：** 404  
**根本原因パターン：**
- 廃止・変更された郵便番号を入力
- 新設された郵便番号（zipcloud のデータ更新遅延）
- 入力値のバリデーション通過後の未登録番号

**確認手順：**
1. 日本郵便公式サイトで郵便番号を確認
2. zipcloud のデータ更新日を確認

---

### GEOCODING_NOT_FOUND
**発生条件：** Open-Meteo が該当住所の緯度・経度を返さない  
**HTTP ステータス：** 404  
**根本原因パターン：**
- 英語マッピングが未登録の都道府県（mapPrefectureToEn 関数の未対応）
- Open-Meteo のジオコーディングデータに存在しない地名
- 全フォールバックパターンが失敗

**確認手順：**
1. server.js の `mapPrefectureToEn` 関数で対象都道府県が定義されているか確認
2. フォールバック検索パターンのログを確認

**再発防止策：**
- 緯度・経度の別途取得手段を追加（Google Maps API など）

---

### INVALID_POSTAL_CODE
**発生条件：** 7桁の数字以外の入力  
**HTTP ステータス：** 400  
**根本原因パターン：**
- フロントエンドのバリデーション漏れ
- API を直接呼び出すクライアントの実装ミス

---

### INTERNAL_SERVER_ERROR
**発生条件：** 上記以外の予期しないエラー  
**HTTP ステータス：** 500  
**根本原因パターン：**
- axios のバージョン互換性問題
- Node.js のメモリ不足
- 依存ライブラリの予期しない例外

**確認手順：**
1. サーバーログの `[ERROR]` 行を確認（スタックトレース付き）
2. Node.js のメモリ使用量を確認
3. `npm audit` で脆弱性・互換性問題を確認

---

## 過去の RCA 分析事例

※ 分析事例は随時追記していく

---

## RCA 分析結果テンプレート

```markdown
## RCA 分析結果

**インシデント番号：** INCxxxxxxx
**発生日時：** YYYY-MM-DD HH:MM
**エラーコード：** （上記一覧より）

### 障害概要
（インシデントの説明）

### 根本原因
（特定または推定された根本原因）

### 影響範囲
- 影響ユーザー数：
- 影響機能：

### 対処内容
（実施した対処）

### 再発防止策
1. 短期：
2. 中期：
3. 長期：
```
