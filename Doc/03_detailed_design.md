# 詳細検討書（擬似環境向け）

## 1. 文書の位置づけ
- 本書は `Doc/02_specification.md` を実装手順レベルに具体化した詳細設計である。
- レビュー指摘を反映し、Zap手順・重複防止・行特定ロジック・ログ項目を具体化する。
- 本プロジェクトの運用方針として、Notionへ記録する必須項目は `Title` のみとする。

## 2. 実装方針（確定）
- Notion作成時の登録値は `Title = 人の名前さん_内容`。
- 2段階リネームを維持する。
  - 1回目: 短縮名 `人の名前さん_内容.拡張子`
  - 2回目: 詳細名 `YYYYMMDD_人の名前さん_内容_金額.拡張子`
- Zap構成は以下の3本。
  - `Zap 1`: 添付ありの一次処理（短縮名化 + 重複チェック + Notion作成）
  - `Zap 2`: 添付ありの最終保管処理（詳細名化 + 同名回避 + フォルダB移動）
  - `Zap URL`: 添付なしURL提出時の重複チェック + Notion作成

## 3. 事前準備

### 3.1 外部サービス
- Googleフォーム（回答先スプレッドシート連携を有効化）
- Googleドライブ
  - フォルダA: フォーム添付アップロード先
  - フォルダB: 最終保管先
- Notion Database（Titleプロパティ必須）
- Zapier
- Googleスプレッドシート（フォーム回答シート + ログシート）

### 3.2 管理用ID
- `FOLDER_A_ID`
- `FOLDER_B_ID`
- `ERROR_FOLDER_ID`（任意）
- `NOTION_DATABASE_ID`

## 4. データ定義

### 4.1 入力データ（フォーム）
- 請求日: `YYYY-MM-DD`
- 人の名前
- 内容
- 金額
- 添付ファイル（任意）
- URL（任意）
- フォーム回答ID（Google Forms Response ID）

### 4.2 内部計算フィールド
- `reception_id`: フォーム回答ID（重複判定キー）
- `name_norm`: 敬称付与後の名前
- `content_norm`: 禁止文字置換・文字数調整後の内容
- `amount_norm`: 数値のみ抽出した金額
- `short_name`: `name_norm + "_" + content_norm + "." + ext`
- `short_title`: `name_norm + "_" + content_norm`
- `detail_name`: `yyyymmdd + "_" + name_norm + "_" + content_norm + "_" + amount_norm + "." + ext`

### 4.3 正規化手順（順序固定）
1. 前後空白除去
2. 名前末尾に `さん` がなければ付与
3. 内容の禁止文字 `\/:*?"<>|` を `-` に置換
4. 内容を80文字目安で切り詰め
5. 金額を数値のみへ変換
6. 生成ファイル名全体を255文字以内に調整

## 5. Zap詳細設計

### 5.1 Zap 1（添付あり: 短縮名化 + Notion作成）
- Trigger:
  - Google Drive `New File in Folder`（フォルダA）
- Filter（誤トリガー防止）:
  - 詳細名は除外: `^[0-9]{8}_.+_[0-9]+\\.[A-Za-z0-9]+$`
  - 対象拡張子のみ通過: `\\.(pdf|jpg|jpeg|png|heic)$`（大文字小文字無視）
- Steps:
1. `file_id` を取得
2. Google Sheets `Find Row` で回答行を特定
   - 条件: 回答シートの「添付URL」列に `file_id` を含む行
3. 回答行から `フォーム回答ID` を取得し `reception_id` とする
4. 正規化手順を実施し `short_name` / `short_title` を生成
5. Google Drive `Update File` で1回目リネーム
6. Notion `Search Database Item`（`Title == short_title`）で重複チェック
7. Filter: 検索0件のみ続行
8. Notion `Create Database Item`（`Title = short_title`）
9. Zapier Storage/管理シートへ受け渡しデータ保存
   - 保存項目: `file_id`, `reception_id`, `name_norm`, `content_norm`, `amount_norm`, `invoice_date`, `notion_page_id`
10. ログ記録

### 5.2 Zap 2（添付あり: 詳細名化 + フォルダB移動）
- Trigger:
  - Zapier Storage/管理シートの新規レコード（Zap 1完了データ）
- Steps:
1. 受け渡しデータから `detail_name` を生成
2. Google Drive `Find File` でフォルダBの同名有無を確認
3. 同名ありの場合は `detail_name + "_" + reception_id` に変更
4. Google Drive `Update File` で2回目リネーム
5. Google Drive `Move File` でフォルダBへ移動
6. ログ記録
- Note:
  - 本運用ではNotion更新を行わない（Titleのみ運用のため）。

### 5.3 Zap URL（添付なしURL提出）
- Trigger:
  - Google Forms `New Response`
- Filter:
  - 添付ファイルが空
  - URLが空でない
- Steps:
1. 回答行から `フォーム回答ID` を取得し `reception_id` とする
2. 名前・内容を正規化し `short_title` を生成
3. Notion `Search Database Item`（`Title == short_title`）で重複チェック
4. Filter: 検索0件のみ続行
5. Notion `Create Database Item`（`Title = short_title`）
6. ログ記録

## 6. 分岐条件
- 分岐A（添付あり）:
  - `添付ファイル != 空` の場合、`Zap 1` → `Zap 2`
- 分岐B（URLのみ）:
  - `添付ファイル = 空` かつ `URL != 空` の場合、`Zap URL`
- 分岐C（不正入力）:
  - `添付ファイル = 空` かつ `URL = 空` はエラーとしてログ記録

## 7. 重複・再実行対策
- `reception_id`（フォーム回答ID）を内部キーとして採用する。
- Zap 1 / Zap URLともにNotion作成前に `Search + Filter` を必須化する。
- 同一 `reception_id` の再実行時は `duplicate` ログを記録し作成スキップする。
- Zap 2は `file_id` 基準で再実行安全（idempotent）を確保する。

## 8. エラー処理設計
- リネーム失敗:
  - エラー内容と `file_id` をログ出力し停止
- 移動失敗:
  - リトライ後も失敗時は `needs_manual`
  - 必要に応じて `ERROR_FOLDER_ID` へ退避
- Notion作成失敗:
  - Zapier標準リトライ
  - 最終失敗時は `needs_manual`

## 9. ステータス定義

### 9.1 Notionステータス
- 本運用では未使用（Titleのみ運用）。

### 9.2 ログステータス
- `success` / `error` / `duplicate` / `needs_manual`

## 10. ログ設計

### 10.1 ログ保存先
- Googleスプレッドシート `processing_log` シート

### 10.2 ログ項目
- 処理日時
- reception_id（フォーム回答ID）
- file_id
- short_name
- final_file_name
- drive_url_a
- drive_url_b
- notion_page_id
- zap_name（Zap 1 / Zap 2 / Zap URL）
- result（success / error / duplicate / needs_manual）
- error_message

## 11. テスト設計

### 11.1 正常系
1. 添付あり1件を送信
2. フォルダAで短縮名へ変更される
3. Notionに `Title=人の名前さん_内容` が作成される
4. フォルダBに詳細名で保存される

### 11.2 例外系
1. 添付なしURLありを送信
2. Notionに `Title=人の名前さん_内容` が作成される
3. Driveのリネーム・移動処理は実行されない

### 11.3 境界系
1. 名前末尾が `さん` の入力
2. 内容に禁止文字を含む入力
3. 金額が `12,000円` の入力
4. 同一ファイル名がフォルダBに既存の状態
5. 同一回答の再送信（duplicate判定）

## 12. 未確定事項
- Zap 1からZap 2への受け渡し媒体（Storage or シート）
- ログシート列名の最終確定
- Notion重複判定キー（`Title` のみで十分か、将来 `reception_id` プロパティ追加するか）
