# Zapier設定手順書（擬似環境）

## 1. 目的
- 本書は `Doc/02_specification.md` と `Doc/03_detailed_design.md` に基づく Zapier 設定手順を示す。
- 本運用では Notion には `人の名前さん_内容`（Title）を記録する。

## 2. 事前準備

### 2.1 必要アカウント
- Google（Forms / Drive / Sheets）
- Notion
- Zapier

### 2.2 Driveフォルダ
- フォルダA（受付）: `01_フォーム受付_未処理`
- フォルダB（最終）: `02_請求書補完1_確定`
- 任意: エラー隔離フォルダ

### 2.3 スプレッドシート
- フォーム回答シート（フォーム連携で自動作成）
- `zap_handoff`（Zap1→Zap2受け渡し）
- `processing_log`（処理ログ）

### 2.4 シート列定義
- `zap_handoff`:
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
- `processing_log`:
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

## 3. 重要ルール

### 3.1 文字列
- 名前は最終的に `〜さん` 1回だけに正規化する。
- 内容の禁止文字 `\\ / : * ? " < > |` は `-` へ置換する。

### 3.2 ファイル名
- 短縮名: `人の名前さん_内容.拡張子`
- 詳細名: `YYYY/MM/DD_人の名前さん_内容_金額.拡張子`

### 3.3 一意キー
- `reception_id = フォーム回答ID`

## 4. Zap1 設定（受付→短縮名→受け渡し）

### 4.1 Zap名
- `Zap 1 - Rename Short + Handoff`

### 4.2 Trigger
1. App: Google Drive
2. Event: `New File in Folder`
3. Folder: フォルダA
4. Test で `ID` と `Original Filename` が取れることを確認

### 4.3 Filter（対象拡張子のみ）
- `Original Filename` の末尾が以下のいずれか
  - `.pdf`
  - `.jpg`
  - `.jpeg`
  - `.png`
  - `.heic`

### 4.4 回答行特定（Lookup Spreadsheet Row）
1. Worksheet: フォーム回答シート
2. Lookup column: 添付URL列（または補助列 `file_id_for_zap`）
3. Lookup value: Trigger の `file_id`（またはID抽出値）
4. 補助列を使う場合はフォーム回答シートに `file_id_for_zap` を用意する

### 4.5 正規化・短縮名生成
- Formatter / Code で以下を作る
  - `name_norm`
  - `content_norm`
  - `amount_norm`
  - `short_name`

### 4.6 Driveリネーム（短縮名）
1. App: Google Drive
2. Event: `Update File or Folder Metadata`
3. File: Trigger の `ID`（動的マッピング）
4. Name: `short_name`

### 4.7 `zap_handoff` 追記
1. App: Google Sheets
2. Event: `Create Spreadsheet Row`
3. Worksheet: `zap_handoff`
4. マッピング
  - `created_at`: Variables > System > Current time（ISO）
  - `file_id`, `reception_id`, `invoice_date`, `name_norm`, `content_norm`, `amount_norm`, `short_name`
  - `detail_name_base`: 空欄可（Zap2で生成する場合）
  - `notion_page_id`: 空欄

### 4.8 `processing_log` 追記（Zap1成功ログ）
1. Worksheet: `processing_log`
2. マッピング
  - `processed_at`: Current time（ISO）
  - `reception_id`, `file_id`, `short_name`
  - `final_file_name`: 空欄
  - `drive_url_a`: 取得できるDriveリンク
  - `drive_url_b`: 空欄
  - `notion_page_id`: 空欄
  - `zap_name`: `Zap 1`（手入力可）
  - `result`: `SUCCESS`
  - `error_message`: 空欄

## 5. Zap2 設定（Notion作成→詳細名→フォルダB移動）

### 5.1 Zap名
- `Zap 2 - Notion + Rename Detail + Move Final`

### 5.2 Trigger
1. App: Google Sheets
2. Event: `New Spreadsheet Row`
3. Worksheet: `zap_handoff`
4. Test で1行取得を確認

### 5.3 Code by Zapier（詳細名とTitle生成）
- Input Data
  - `invoice_date = Trigger.invoice_date`
  - `name_norm = Trigger.name_norm`
  - `content_norm = Trigger.content_norm`
  - `amount_norm = Trigger.amount_norm`
  - `reception_id = Trigger.reception_id`
  - `short_name = Trigger.short_name`
- コード（確定版）

```javascript
const invoiceDate = (inputData.invoice_date || "").trim();
const nameRaw = (inputData.name_norm || "").trim();
const content = (inputData.content_norm || "").trim();
const amount = String(inputData.amount_norm || "").trim();
const receptionId = String(inputData.reception_id || "").trim();
const shortName = (inputData.short_name || "").trim();

const extMatch = shortName.match(/\.([A-Za-z0-9]+)$/);
const ext = extMatch ? extMatch[1].toLowerCase() : "pdf";

const nameBase = nameRaw.replace(/(?:さん)+$/u, "");
const name = `${nameBase}さん`;

const short_title = `${name}_${content}`;
const detail_name = `${invoiceDate}_${name}_${content}_${amount}.${ext}`;
const detail_name_with_id = `${invoiceDate}_${name}_${content}_${amount}_${receptionId}.${ext}`;

output = { short_title, detail_name, detail_name_with_id, ext };
```

### 5.4 Notion作成（Name必須）
1. App: Notion
2. Event: `Create Data Source Item`
3. Data Source: 対象DB（例: `タスク管理` など）
4. **Name**: `Step2 -> Short Title` を必ず設定
5. Content: 空欄可
6. Content Format: `Plain text`

> 注意: `Name` を設定しないと、Notion行は作成されてもタイトル空欄になる。

### 5.5 Driveリネーム（詳細名）
1. App: Google Drive
2. Event: `Update File or Folder Metadata`
3. File: `Trigger.file_id`（動的）
4. Name: `Step2.detail_name`

### 5.6 フォルダB移動
1. App: Google Drive
2. Event: `Move File`
3. File: `Trigger.file_id`（動的）
4. Folder: フォルダB

### 5.7 `processing_log` 追記（Zap2成功ログ）
1. App: Google Sheets
2. Event: `Create Spreadsheet Row`
3. Worksheet: `processing_log`
4. マッピング
  - `processed_at`: Current time（ISO）
  - `reception_id`: Trigger
  - `file_id`: Trigger
  - `short_name`: Trigger
  - `final_file_name`: Step3/Step4 の最終Name
  - `drive_url_a`: 空欄
  - `drive_url_b`: 以下のどちらか
    - Stepのリンク項目
    - 文字列連結 `https://drive.google.com/open?id=` + `Trigger.file_id`
  - `notion_page_id`: 空欄
  - `zap_name`: `Zap 2`
  - `result`: `SUCCESS`
  - `error_message`: 空欄

## 6. Notionが選択肢に出ない場合
1. Zapier `App Connections` で Notion接続を新規作成/再接続
2. 認可画面 `Allow access` で表示されるページにチェックして許可
3. Stepを開き直し `Data Source` を `Refresh`
4. それでも出ない場合は、認可画面で見えているDB（例: `タスク管理`）で先に接続確認し、後でDB整理する

## 7. よくある詰まりどころ
- `Move File` の `File` に候補が出ない
  - Static選択ではなく、右の `+` から動的に `Trigger.file_id` を入れる
- `Lookup Spreadsheet Row` が見つからない
  - フォーム回答シートに補助列 `file_id_for_zap` を作ってID照合
- `Current time` が見つからない
  - `Variables` > `System` > `Current time: ... (ISO)` を使用
- Notion追加されるが `Name` 空欄
  - Stepの `Name` フィールドに `Short Title` を設定する

## 8. 最終テスト観点
1. フォルダBに `YYYY/MM/DD_人の名前さん_内容_金額.拡張子` が存在
2. Notionに `人の名前さん_内容` が追加
3. `processing_log` に `zap_name = Zap 2`, `result = SUCCESS` が追加

## 9. 公開前チェック
1. Zap1 / Zap2 とも `Publish` 済みか
2. Zap History で直近 Run が `Successful` か
3. テスト用の重複行・空欄行を必要に応じて整理したか
