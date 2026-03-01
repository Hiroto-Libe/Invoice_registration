# 詳細検討書（擬似環境向け）

## 1. 文書の位置づけ
- 本書は `Doc/02_specification.md` を実装手順レベルに具体化した詳細設計である。
- 実運用で確定したZap画面設定・詰まりどころ・回避策を反映する。

## 2. 実装方針（確定）
- Notionへ記録する項目は `Title` のみ。
- タイトル形式は `人の名前さん_内容`。
- 2段階リネームを維持。
  - 1回目（Zap1）: 短縮名
  - 2回目（Zap2）: 詳細名
- Zap構成:
  - `Zap 1`: 添付あり一次処理（短縮名化 + handoff + ログ）
  - `Zap 2`: 添付あり最終処理（Notion + 詳細名化 + 移動 + ログ）
  - `Zap URL`: 添付なしURL提出（Notion + ログ）

## 3. 事前準備

### 3.1 外部サービス
- Googleフォーム（回答先スプレッドシート連携）
- Googleドライブ
  - フォルダA: `01_フォーム受付_未処理`
  - フォルダB: `02_請求書補完1_確定`
- Notion Database（Title列あり）
- Zapier
- スプレッドシート
  - `zap_handoff`
  - `processing_log`

### 3.2 補助列（フォーム回答シート）
- `file_id_for_zap` を追加する。
- 添付URLからファイルIDを抽出して Lookup 用キーとする。

## 4. データ定義

### 4.1 内部キー
- `reception_id`: フォーム回答ID（重複判定キー）
- `file_id`: DriveファイルID（添付ありルート）

### 4.2 正規化フィールド
- `name_norm`
- `content_norm`
- `amount_norm`
- `short_name`
- `short_title`
- `detail_name`
- `detail_name_with_id`

### 4.3 正規化ロジック（順序固定）
1. 前後空白除去
2. 名前末尾の `さん` を正規化（最終的に1個）
3. 内容禁止文字置換
4. 内容を80文字目安に調整
5. 金額を数値のみ抽出

## 5. Zap 1 詳細設計

### 5.1 トリガー
- Google Drive `New File in Folder`（フォルダA）

### 5.2 ステップ
1. Filter（許可拡張子）
2. Google Sheets `Lookup Spreadsheet Row`（フォーム回答行特定）
3. Formatter/Code（短縮名生成）
4. Google Drive `Update File or Folder Metadata`
   - `File = Trigger.ID`
   - `Name = short_name`
5. Google Sheets `Create Spreadsheet Row`（`zap_handoff`）
6. Google Sheets `Create Spreadsheet Row`（`processing_log`）

### 5.3 `processing_log`（Zap1）
- `zap_name = Zap 1`
- `result = SUCCESS`
- `final_file_name`, `drive_url_b`, `notion_page_id` は空欄可

## 6. Zap 2 詳細設計

### 6.1 トリガー
- Google Sheets `New Spreadsheet Row`（`zap_handoff`）

### 6.2 Code by Zapier（確定版）
- Input Data:
  - `invoice_date`
  - `name_norm`
  - `content_norm`
  - `amount_norm`
  - `reception_id`
  - `short_name`

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

### 6.3 Notion作成
- Notion `Create Data Source Item`
- `Data Source`: 対象DB
- **`Name = Step2.short_title` を必須設定**
- `Content` は空欄可
- `Content Format = Plain text`

### 6.4 詳細名リネーム
- Google Drive `Update File or Folder Metadata`
- `File = Trigger.file_id`
- `Name = Step2.detail_name`

### 6.5 フォルダB移動
- Google Drive `Move File`
- `File = Trigger.file_id`
- `Folder = フォルダB`

### 6.6 `processing_log`（Zap2）
- `processed_at = Current time (ISO)`
- `reception_id`, `file_id`, `short_name`
- `final_file_name`
- `drive_url_b`
  - 取得リンク項目、または
  - `https://drive.google.com/open?id=` + `file_id`
- `zap_name = Zap 2`
- `result = SUCCESS`

## 7. Zap URL 詳細設計
1. Trigger: Google Forms `New Response`
2. Filter: 添付空かつURLあり
3. `short_title` 生成
4. Notion作成（Titleのみ）
5. `processing_log` 記録

## 8. 重複・再実行設計
- `processing_log.reception_id` で重複判定。
- 既存時は `duplicate` 記録してNotion作成をスキップ。
- `file_id` 基準で再実行時も同一対象を処理可能。

## 9. エラー処理
- `result` に `error` / `needs_manual` を記録。
- 失敗時は `error_message` にZapエラー本文を保存。
- 必要に応じて隔離フォルダへ移動。

## 10. 検証項目（最終）
1. フォルダBに詳細名ファイルが存在
2. Notionに `人の名前さん_内容` が追加
3. `processing_log` に `Zap 2 / SUCCESS`
4. `Name` 空欄行が発生しない（`Name` マッピング設定済み）

## 11. 運用メモ
- Notion Data Source が候補に出ない場合:
  - `App Connections` の Notion接続を再作成
  - 認可対象ページを見直し
  - ステップ側 `Data Source` を `Refresh`
- `Move File` の `File` は Staticではなく動的マッピングで設定する。
