# 仕様書（擬似環境向け）

## 1. 文書の目的
- `Doc/01_requirements.md` を実装可能な仕様に具体化する。
- 対象範囲は「Googleフォーム提出 → Drive 2段階リネーム → Zapier経由でNotion登録」。

## 2. 前提と対象範囲

### 2.1 設計前提
- Notionへの記録項目は `Title`（`人の名前さん_内容`）のみ。
- 添付ありは2段階リネーム。
  - 1回目: 短縮名
  - 2回目: 詳細名
- 実運用確定フロー:
  - Zap1: 受付・短縮名・受け渡し・ログ
  - Zap2: Notion作成・詳細名化・フォルダB移動・ログ

### 2.2 対象
- フォーム入力受付（添付あり/URLのみ）
- Driveの1回目リネーム（短縮名）
- Notionレコード作成（Titleのみ）
- Driveの2回目リネーム（詳細名）と最終保管フォルダ移動
- 処理ログ記録

### 2.3 対象外
- URL先の自動クロール/解析
- 会計計上・承認ワークフロー
- Notionへの追加項目保存（請求日、金額、URL、ステータス）

## 3. システム構成（最小）
- 入力: Googleフォーム
- ファイル保管: Googleドライブ
  - フォルダA: `01_フォーム受付_未処理`
  - フォルダB: `02_請求書補完1_確定`
- 自動処理: Zapier
- 記帳先: Notion Database（Titleプロパティを使用）
- ログ: Googleスプレッドシート（`zap_handoff`, `processing_log`）

## 4. データ仕様

### 4.1 入力項目
- 請求日（必須）
- 人の名前（必須）
- 内容（必須）
- 金額（必須）
- 添付ファイル（条件付き必須, PDF/画像, 最大100MB, 1ファイル）
- アクセス可能URL（任意）
- フォーム回答ID（必須）

### 4.2 入力バリデーション
- 添付ファイルまたはアクセス可能URLのいずれか1つ以上を必須。
- 添付ありは基本ルート、添付なしかつURLありはURL例外ルートへ分岐。

### 4.3 正規化ルール
- 名前: 前後空白除去後、末尾の `さん` は最終的に1個に正規化。
- 内容: 前後空白除去後、禁止文字 `\\ / : * ? " < > |` を `-` に置換。
- 内容: 最大80文字目安で切り詰め。
- 金額: 数値のみ抽出。
- ファイル名長: 拡張子含む全体長255文字以内。

## 5. 命名仕様

### 5.1 1回目リネーム（短縮名）
- 形式: `人の名前さん_内容.拡張子`
- 用途: 一時保管・視認性向上
- 実行場所: フォルダA

### 5.2 2回目リネーム（詳細名）
- 形式: `YYYY/MM/DD_人の名前さん_内容_金額.拡張子`
- 用途: Drive最終保管用正式名
- 実行場所: フォルダB移動前

### 5.3 重複回避
- フォルダB同名チェックを行う。
- 同名あり時は末尾に `_<reception_id>` を付与。

## 6. 処理フロー仕様

### 6.1 基本ルート（添付あり）
1. Googleフォーム送信でファイルがフォルダAに作成。
2. `Zap 1` がフォルダA新規ファイルを検知し、短縮名へ1回目リネーム。
3. `Zap 1` が `zap_handoff` / `processing_log` へ記録。
4. `Zap 2` が `zap_handoff` を受け、Notionに `Title` を作成。
5. `Zap 2` が詳細名へ2回目リネームし、フォルダBへ移動。
6. `Zap 2` が `processing_log` へ記録。

### 6.2 URL例外ルート（添付なし）
1. `Zap URL` がフォーム回答をトリガーに起動。
2. 添付空・URLありを判定。
3. Notionに `Title` のみ作成。
4. `processing_log` に結果記録。

## 7. Zapier実装仕様

### 7.1 Zap 1: 受付・一次処理（添付あり）
- Trigger: Google Drive `New File in Folder`（フォルダA）
- Filter: 許可拡張子のみ通過
- Action:
  - Google Sheets `Lookup Spreadsheet Row` で回答行特定
  - Formatter/Codeで `name_norm`, `content_norm`, `amount_norm`, `short_name` 生成
  - Google Drive `Update File or Folder Metadata`（短縮名）
  - Google Sheets `Create Spreadsheet Row`（`zap_handoff`）
  - Google Sheets `Create Spreadsheet Row`（`processing_log`, `zap_name=Zap 1`, `result=SUCCESS`）

### 7.2 Zap 2: Notion + 最終保管（添付あり）
- Trigger: Google Sheets `New Spreadsheet Row`（`zap_handoff`）
- Action:
  - Code by Zapier で `short_title`, `detail_name` 生成
  - Notion `Create Data Source Item`
    - `Name = short_title`（必須）
  - Google Drive `Update File or Folder Metadata`（詳細名）
  - Google Drive `Move File`（フォルダB）
  - Google Sheets `Create Spreadsheet Row`（`processing_log`, `zap_name=Zap 2`, `result=SUCCESS`）

### 7.3 Zap URL: URL例外ルート
- Trigger: Google Forms `New Response`
- Filter: 添付空かつURLあり
- Action:
  - `short_title` 生成
  - NotionにTitle作成
  - `processing_log` 記録

## 8. 二重処理防止
- 一意キーは `reception_id`（フォーム回答ID）。
- Notion作成前に `processing_log.reception_id` を参照して重複確認。
- 既存時は作成をスキップし `result=duplicate` を記録。

## 9. エラー処理
- リネーム失敗: `result=error` を記録して停止。
- Notion作成失敗: Zapier標準リトライ、最終失敗で `needs_manual`。
- 2回目リネーム/移動失敗: `result=error`、必要時エラー隔離。

## 10. ステータス定義
- ログステータス: `SUCCESS` / `error` / `duplicate` / `needs_manual`

## 11. ログ仕様（`processing_log`）
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

## 12. 受入試験仕様（最小）
- 添付あり1件で以下を確認。
  - Notion: `人の名前さん_内容`
  - Drive: `YYYY/MM/DD_人の名前さん_内容_金額.拡張子`
  - `processing_log`: `Zap 1/SUCCESS`, `Zap 2/SUCCESS`
- 添付なしURLあり1件でNotionタイトル作成を確認。
