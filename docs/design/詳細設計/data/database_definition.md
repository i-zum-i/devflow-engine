# データベース定義書

## 1. 概要

本ドキュメントは、「DevFlow Engine」で使用するデータベースの物理設計を定義する。

- **データベースシステム:** Amazon DynamoDB
- **テーブル数:** 1
- **データモデル:** シングルテーブルデザイン

## 2. テーブル一覧

| 論理テーブル名 | 物理テーブル名 | 概要 |
| :--- | :--- | :--- |
| セッション管理テーブル | `devflow-sessions` | ユーザーのセッション情報、コンテナの状態、対話履歴などを一元管理する。 |

## 3. devflow-sessions テーブル定義

- **プライマリキー:**
    - **パーティションキー (PK):** `session_id` (String)
- **グローバルセカンダリインデックス (GSI):**
    - **インデックス名:** `user_id-created_at-index`
    - **パーティションキー (PK):** `user_id` (String)
    - **ソートキー (SK):** `created_at` (Number)
- **Time to Live (TTL):**
    - **TTL属性:** `ttl`
    - **期間:** 24時間 (セッション作成・更新から24時間後にアイテムは自動的に削除される)

### 3.1. カラム一覧

| 論理名 | 物理名 | 型 | PK | GSI-PK | GSI-SK | 必須 | 説明 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| セッションID | `session_id` | String | ○ | | | ○ | セッションの一意なID (UUID v4形式) |
| ユーザーID | `user_id` | String | | ○ | | ○ | ユーザーの一意なID |
| リポジトリURL | `repository_url` | String | | | | ○ | 対象のGitHubリポジトリURL |
| リポジトリオーナー | `repository_owner` | String | | | | ○ | リポジトリのオーナー名 |
| リポジトリ名 | `repository_name` | String | | | | ○ | リポジトリ名 |
| ブランチ名 | `branch_name` | String | | | | ○ | 作業用に作成されたブランチ名 |
| ECSクラスターARN | `ecs_cluster_arn` | String | | | | | 実行中のECSクラスターのARN |
| ECSタスクARN | `ecs_task_arn` | String | | | | | 実行中のECSタスクのARN |
| ECSタスク定義ARN | `ecs_task_definition_arn` | String | | | | | 使用したECSタスク定義のARN |
| PR番号 | `pr_number` | Number | | | | | 作成されたPull Requestの番号 |
| PR URL | `pr_url` | String | | | | | 作成されたPull RequestのURL |
| Editor URL | `editor_url` | String | | | | | Web Editor (code-server) へのアクセスURL |
| ステータス | `status` | String | | | | ○ | セッションの状態 (`starting`, `running`, `stopped`, `error`) |
| 初回プロンプト | `initial_prompt` | String | | | | ○ | セッション開始時のプロンプト |
| 対話履歴 | `history` | List<Map> | | | | ○ | プロンプトと応答の履歴リスト |
| GitHub AppインストールID | `github_installation_id` | Number | | | | ○ | リポジトリにインストールされたGitHub AppのID |
| 作成日時 | `created_at` | Number | | | ○ | ○ | 作成タイムスタンプ (Unix time) |
| 更新日時 | `updated_at` | Number | | | | ○ | 更新タイムスタンプ (Unix time) |
| TTL | `ttl` | Number | | | | ○ | アイテムの有効期限 (Unix time) |
| エラーメッセージ | `error_message` | String | | | | | `status`が`error`の場合のエラー詳細 |
