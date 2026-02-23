# Zapier設定手順書（擬似環境）

## 1. 目的
- 本書は `Doc/02_specification.md` と `Doc/03_detailed_design.md` に基づく、Zapierの画面設定手順を示す。
- 本運用では Notion には `Title` のみを記録する。

## 2. 事前準備

### 2.1 必要アカウント
- Google（Forms / Drive / Sheets）
- Notion
- Zapier

### 2.2 Driveフォルダ
- フォルダA: フォーム添付アップロード先
- フォルダB: 最終保管先
- 任意: エラー隔離フォルダ

### 2.3 スプレッドシート
- フォーム回答シート（Googleフォーム連携で自動作成）
- `zap_handoff` シート（Zap 1 → Zap 2 受け渡し用）
- `processing_log` シート（ログ用）

### 2.4 シート列定義
- `zap_handoff` 列:
  - `created_at`
  - `file_id`
  - `reception_id`
  - `invoice_date`
  - `name_norm`
  - `content_norm`
  - `amount_norm`
  - `short_name`
  - `detail_name_base`
  - `notion_page_id`
- `processing_log` 列:
  - `processed_at`
  - `reception_id`
  - `file_id`
  - `short_name`
  - `final_file_name`
  - `drive_url_a`
  - `drive_url_b`
  - `notion_page_id`
  - `zap_name`
  - `result`
  - `error_message`

## 3. 共通ルール

### 3.1 文字列正規化
1. 前後空白を除去
2. 名前末尾が `さん` でなければ付与
3. 内容の禁止文字 `\/:*?"<>|` を `-` に置換
4. 内容を80文字目安で切り詰め
5. 金額を数値のみへ変換

### 3.2 ファイル名
- 短縮名: `人の名前さん_内容.拡張子`
- 詳細名: `YYYYMMDD_人の名前さん_内容_金額.拡張子`

### 3.3 一意キー
- `reception_id = フォーム回答ID`

## 4. Zap 1 設定（添付あり: 短縮名化 + Notion作成）

### 4.1 Zap基本情報
- Zap名: `Zap 1 - Rename Short + Create Notion`

### 4.2 Trigger
1. App: Google Drive
2. Event: `New File in Folder`
3. Folder: フォルダA
4. Test Trigger で `file_id` とファイル名が取得できることを確認

### 4.3 Filter（誤トリガー防止）
1. Step: `Filter by Zapier`
2. 条件1: ファイル名が詳細名正規表現に一致しない  
   - `^[0-9]{8}_.+_[0-9]+\.[A-Za-z0-9]+$` を除外
3. 条件2: 拡張子が許可形式  
   - `pdf, jpg, jpeg, png, heic`

### 4.4 回答行特定
1. App: Google Sheets
2. Event: `Lookup Spreadsheet Row`
3. Spreadsheet: フォーム回答シートを含むファイル
4. Worksheet: フォーム回答シート
5. Lookup Column: 添付URL列
6. Lookup Value: Triggerで取得した `file_id`
7. 成功時に以下が取得できることを確認
   - `フォーム回答ID`
   - 請求日
   - 名前
   - 内容
   - 金額

### 4.5 Formatter（正規化）
1. 名前正規化（空白除去 + `さん` 付与）
2. 内容正規化（禁止文字置換 + 80文字切り詰め）
3. 金額正規化（数値のみ）
4. 日付整形（`YYYY-MM-DD` -> `YYYYMMDD`）
5. `short_title` と `short_name` を生成
6. `detail_name_base` を生成

### 4.6 1回目リネーム
1. App: Google Drive
2. Event: `Update File`
3. File: Triggerの `file_id`
4. Name: `short_name`

### 4.7 重複チェック
1. App: Google Sheets
2. Event: `Lookup Spreadsheet Row`
3. Worksheet: `processing_log`
4. Lookup Column: `reception_id`
5. Lookup Value: フォーム回答ID
6. Step: `Filter by Zapier`
7. 条件: 検索結果が空（未処理）の場合のみ続行

### 4.8 Notion作成
1. App: Notion
2. Event: `Create Database Item`
3. Database: 対象DB
4. マッピング:
   - `Title` = `short_title`

### 4.9 zap_handoff書き込み
1. App: Google Sheets
2. Event: `Create Spreadsheet Row`
3. Worksheet: `zap_handoff`
4. マッピング:
   - `created_at` = Zap実行時刻
   - `file_id`
   - `reception_id`
   - `invoice_date`
   - `name_norm`
   - `content_norm`
   - `amount_norm`
   - `short_name`
   - `detail_name_base`
   - `notion_page_id`

### 4.10 processing_log書き込み（成功時）
1. App: Google Sheets
2. Event: `Create Spreadsheet Row`
3. Worksheet: `processing_log`
4. マッピング:
   - `processed_at` = Zap実行時刻
   - `reception_id`
   - `file_id`
   - `short_name`
   - `final_file_name` = 空
   - `drive_url_a` = TriggerのファイルURL
   - `drive_url_b` = 空
   - `notion_page_id`
   - `zap_name` = `Zap 1`
   - `result` = `success`
   - `error_message` = 空

## 5. Zap 2 設定（添付あり: 詳細名化 + フォルダB移動）

### 5.1 Zap基本情報
- Zap名: `Zap 2 - Rename Detail + Move Final`

### 5.2 Trigger
1. App: Google Sheets
2. Event: `New Spreadsheet Row`
3. Worksheet: `zap_handoff`

### 5.3 同名チェック
1. App: Google Drive
2. Event: `Find File`
3. Folder: フォルダB
4. File Name: `detail_name_base`

### 5.4 詳細名確定
1. Step: `Formatter` または `Code by Zapier`
2. ルール:
   - 同名なし: `final_file_name = detail_name_base`
   - 同名あり: `final_file_name = detail_name_base + "_" + reception_id`

### 5.5 2回目リネーム
1. App: Google Drive
2. Event: `Update File`
3. File: `file_id`
4. Name: `final_file_name`

### 5.6 フォルダB移動
1. App: Google Drive
2. Event: `Move File`
3. File: `file_id`
4. Destination Folder: フォルダB

### 5.7 processing_log書き込み（成功時）
1. App: Google Sheets
2. Event: `Create Spreadsheet Row`
3. Worksheet: `processing_log`
4. マッピング:
   - `processed_at` = Zap実行時刻
   - `reception_id`
   - `file_id`
   - `short_name`
   - `final_file_name`
   - `drive_url_a` = 空
   - `drive_url_b` = 移動後ファイルURL
   - `notion_page_id`
   - `zap_name` = `Zap 2`
   - `result` = `success`
   - `error_message` = 空

## 6. Zap URL 設定（添付なしURL提出）

### 6.1 Zap基本情報
- Zap名: `Zap URL - Create Notion Title Only`

### 6.2 Trigger
1. App: Google Forms
2. Event: `New Form Response`

### 6.3 Filter
1. 添付ファイルが空
2. URLが空でない

### 6.4 Formatter
1. 名前正規化
2. 内容正規化
3. `short_title` 生成
4. `reception_id` 取得（フォーム回答ID）

### 6.5 重複チェック
1. App: Google Sheets
2. Event: `Lookup Spreadsheet Row`
3. Worksheet: `processing_log`
4. Lookup Column: `reception_id`
5. Filter: 未処理のみ続行

### 6.6 Notion作成
1. App: Notion
2. Event: `Create Database Item`
3. Mapping:
   - `Title` = `short_title`

### 6.7 processing_log書き込み（成功時）
1. App: Google Sheets
2. Event: `Create Spreadsheet Row`
3. Worksheet: `processing_log`
4. マッピング:
   - `processed_at` = Zap実行時刻
   - `reception_id`
   - `file_id` = 空
   - `short_name` = `short_title`
   - `final_file_name` = 空
   - `drive_url_a` = 空
   - `drive_url_b` = 空
   - `notion_page_id`
   - `zap_name` = `Zap URL`
   - `result` = `success`
   - `error_message` = 空

## 7. エラー時の取り扱い
- 各Zapの最終にエラーログ記録ステップを追加する。
- `result = error` または `needs_manual` を記録する。
- 必要に応じてDriveファイルをエラー隔離フォルダへ移動する。

## 8. テスト手順
1. 添付ありデータ1件で `Zap 1` と `Zap 2` を確認
2. 添付なしURLありデータ1件で `Zap URL` を確認
3. 同一回答IDで再送して `duplicate` 動作を確認
4. 禁止文字/金額表記ゆれを含むデータで命名を確認

## 9. 本番化前チェック
1. 3つのZapが `ON` になっている
2. 参照フォルダID・シート名・DBが本番値に置換済み
3. `processing_log` に `error` が残っていない

## 10. テスト用ダミーデータ

### 10.1 添付あり（正常系）
- 入力値:
  - フォーム回答ID: `RESP-20260223-001`
  - 請求日: `2026-02-23`
  - 名前: `田中`
  - 内容: `動画編集`
  - 金額: `50,000円`
  - 添付: `invoice_sample.pdf`
  - URL: 空
- 期待結果:
  - 短縮名: `田中さん_動画編集.pdf`
  - 詳細名: `20260223_田中さん_動画編集_50000.pdf`
  - Notion Title: `田中さん_動画編集`
  - `processing_log.result`: `success`

### 10.2 添付なしURLあり（URL例外系）
- 入力値:
  - フォーム回答ID: `RESP-20260223-002`
  - 請求日: `2026-02-23`
  - 名前: `佐藤さん`
  - 内容: `バナー制作`
  - 金額: `12000`
  - 添付: 空
  - URL: `https://example.com/invoice/abc`
- 期待結果:
  - Notion Title: `佐藤さん_バナー制作`
  - Driveリネーム/移動: 実行されない
  - `processing_log.result`: `success`

### 10.3 重複確認（duplicate）
- 手順:
  1. `10.1` を1回実行
  2. 同じ `フォーム回答ID = RESP-20260223-001` で再実行
- 期待結果:
  - 2回目はNotion新規作成されない
  - `processing_log.result`: `duplicate`

### 10.4 禁止文字・長文確認（境界系）
- 入力値:
  - フォーム回答ID: `RESP-20260223-003`
  - 請求日: `2026-02-23`
  - 名前: `鈴木`
  - 内容: `LP制作/デザイン:*?\"<>|_テストテキスト（80文字超になる長文を入力）`
  - 金額: `12,345円`
  - 添付: `sample_image.png`
- 期待結果:
  - 禁止文字が `-` に置換される
  - 内容が80文字目安で切り詰められる
  - 金額が `12345` として詳細名に反映される
