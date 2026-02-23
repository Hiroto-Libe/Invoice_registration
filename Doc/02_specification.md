# 仕様書（擬似環境向け）

## 1. 文書の目的
- `Doc/01_requirements.md` を実装可能な仕様に具体化する。
- 対象範囲は「Googleフォーム提出→Drive 2段階リネーム→Zapier経由でNotion登録」。

## 2. 前提と対象範囲

### 2.1 設計前提
- Zapierはトリガー時点のファイル情報を参照するため、後続リネームが先行ステップに反映されない場合がある。
- Notionタイトル用（短縮名）とDrive保管用（詳細名）を2段階で分離する。
- 本運用でNotionへ記録する項目は `Title` のみとする。

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
  - フォルダA: アップロード先
  - フォルダB: 最終保管先
- 自動処理: Zapier
- 記帳先: Notion Database（Titleプロパティのみ利用）
- ログ: Googleスプレッドシート

## 4. データ仕様

### 4.1 入力項目
- 請求日（必須, `YYYY-MM-DD`）
- 人の名前（必須）
- 内容（必須）
- 金額（必須）
- 添付ファイル（条件付き必須, PDF/画像, 最大100MB, 1ファイル）
- アクセス可能URL（任意）
- フォーム回答ID（必須）

### 4.2 入力バリデーション
- 添付ファイルまたはアクセス可能URLのいずれか1つ以上が必須。
- 添付ありは基本ルート、添付なしかつURLありはURL例外ルートへ分岐する。

### 4.3 正規化ルール
- 名前: 前後空白除去し、末尾が `さん` でなければ付与する。
- 内容: 前後空白除去後、禁止文字 `\/ : * ? " < > |` を `-` に置換する。
- 内容: 禁止文字置換後に最大80文字目安で切り詰める。
- 金額: `replace(/[^0-9]/g, "")` 相当で数値のみ抽出する。
- ファイル名長: 拡張子含む全体長を255文字以内にする。

## 5. 命名仕様

### 5.1 1回目リネーム（短縮名）
- 形式: `人の名前さん_内容.拡張子`
- 用途: Notionタイトル生成
- 実行場所: フォルダA

### 5.2 2回目リネーム（詳細名）
- 形式: `YYYYMMDD_人の名前さん_内容_金額.拡張子`
- 用途: Drive保管用正式名
- 実行タイミング: Notion作成後
- 実行場所: フォルダBへ移動時

### 5.3 重複回避
- フォルダBで同名存在チェックを行う。
- 同名ありの場合、末尾に受付ID（フォーム回答ID）を付与する。
- 同名チェックは `Google Drive: Find File` を使って実施する。

## 6. 処理フロー仕様

### 6.1 基本ルート（添付あり）
1. Googleフォーム送信でファイルがフォルダAに作成される。
2. `Zap 1` がフォルダA新規ファイルを検知し、短縮名へ1回目リネームする。
3. 同じ `Zap 1` 内でNotionに `Title` のみでページ作成する。
4. `Zap 2` が処理済みデータを受け取り、詳細名へ2回目リネームしてフォルダBへ移動する。
5. 処理結果をログに記録する。

### 6.2 URL例外ルート（添付なし）
1. `Zap URL` がGoogleフォーム新規回答をトリガーに起動する。
2. 添付が空かつURLありをFilterで判定する。
3. Notionに `Title` のみでページ作成する。
4. 処理結果をログに記録する。

## 7. Zapier実装仕様

### 7.1 Zap 1: 受付・一次処理（添付あり）
- Trigger: Google Drive `New File in Folder`（フォルダA）
- Filter:
  - 詳細名パターンは除外: `^[0-9]{8}_.+_[0-9]+\\.[A-Za-z0-9]+$`
  - 対象拡張子のみ通過: `\\.(pdf|jpg|jpeg|png|heic)$`
- Action:
  - Google Sheets `Find Row`: `file_id` で回答行特定
  - Formatter: 名前/内容/金額正規化
  - Google Drive `Update File`: 短縮名へ変更
  - Notion `Search Database Item`: `Title` 一致検索
  - Filter: 検索結果0件のみ続行
  - Notion `Create Database Item`: `Title = 人の名前さん_内容`
  - Storage/シート記録: Zap 2への受け渡しデータ保存

### 7.2 Zap 2: 最終保管処理（添付あり）
- Trigger: Storage/シートの新規レコード（Zap 1完了データ）
- Action:
  - Formatter: 詳細名生成
  - Google Drive `Find File`: フォルダB同名チェック
  - Formatter: 同名時は `_<受付ID>` を付与
  - Google Drive `Update File`: 詳細名へ変更
  - Google Drive `Move File`: フォルダBへ移動
  - ログ記録

### 7.3 Zap URL: URL例外ルート
- Trigger: Google Forms `New Response`
- Filter:
  - 添付ファイルが空
  - URLが空でない
- Action:
  - Formatter: 名前/内容正規化
  - Notion `Search Database Item`: `Title` 一致検索
  - Filter: 検索結果0件のみ続行
  - Notion `Create Database Item`: `Title = 人の名前さん_内容`
  - ログ記録

## 8. 二重処理防止
- 一意キーは `フォーム回答ID` を使用する。
- Notion作成前に `Search Database Item + Filter` で重複確認する。
- 既存時はNotion作成をスキップし、ログステータスを `duplicate` とする。
- Zap 2は `file_id` を基準に再実行安全（idempotent）を確保する。

## 9. エラー処理
- 1回目リネーム失敗: `error` をログ記録し停止。
- Notion作成失敗: Zapier標準リトライ、最終失敗時に `needs_manual` を記録。
- 2回目リネーム/移動失敗: `error` 記録。必要に応じてエラー隔離フォルダへ移動。
- URL例外ルート失敗: `needs_manual` を記録し再処理対象とする。

## 10. ステータス定義
- Notionステータス: 本運用では未使用（Titleのみ運用）。
- ログステータス: `success` / `error` / `duplicate` / `needs_manual`

## 11. ログ仕様（スプレッドシート）
- 記録項目:
  - 処理日時
  - 受付ID（フォーム回答ID）
  - file_id
  - short_name
  - final_file_name
  - drive_url_a
  - drive_url_b
  - notion_page_id
  - zap_name
  - result
  - error_message

## 12. 設定パラメータ
- Google Drive:
  - フォルダA ID（アップロード先）
  - フォルダB ID（最終保管先）
  - エラー隔離フォルダID（任意）
- Notion:
  - Database ID
  - Titleプロパティ名
- Zapier:
  - Trigger/Action接続アカウント
  - フィルタ正規表現
  - Storage/シート連携設定

## 13. 受入試験仕様（最小）
- テストデータ3件（PDF/画像混在）で確認する。
  - Notionタイトルが `人の名前さん_内容` で作成される。
  - Drive最終ファイル名が `YYYYMMDD_人の名前さん_内容_金額.拡張子` になる。
  - 金額に `,` や `円` が含まれても数値のみで命名される。
  - 禁止文字が置換される。
  - 同一提出の再処理で二重登録されない。
  - URLのみ提出でもNotionタイトルが作成される。
