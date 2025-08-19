# ER図

## 概要

本ドキュメントは、「DevFlow Engine」のデータモデルをER図で表現する。
DynamoDBのシングルテーブルデザインを採用しているため、エンティティ間のリレーションはテーブル内のアイテムとして表現される。

## ER図 (Mermaid)

```mermaid
erDiagram
    SESSIONS {
        string session_id PK "セッションID"
        string user_id "ユーザーID (GSI-PK)"
        string repository_url "リポジトリURL"
        string repository_owner "リポジトリオーナー"
        string repository_name "リポジトリ名"
        string branch_name "ブランチ名"
        string ecs_task_arn "ECSタスクARN"
        number pr_number "PR番号"
        string pr_url "PR URL"
        string editor_url "Editor URL"
        string status "ステータス"
        string initial_prompt "初回プロンプト"
        list_map history "対話履歴"
        number github_installation_id "GitHub AppインストールID"
        number created_at "作成日時 (GSI-SK)"
        number updated_at "更新日時"
        number ttl "TTL"
        string error_message "エラーメッセージ"
    }

    HISTORY_ITEM {
        string prompt "プロンプト"
        string response "応答"
        number timestamp "タイムスタンプ"
        string pr_url "PR URL"
    }

    SESSIONS ||--o{ HISTORY_ITEM : "has"
```

## 説明

- **SESSIONSエンティティ:** `devflow-sessions`テーブルの各アイテムを表す。
- **HISTORY_ITEMエンティティ:** `history`属性内の各対話履歴オブジェクトを表す。これは`SESSIONS`エンティティ内にネストされたデータ構造である。
- **リレーション:** `SESSIONS`は複数の`HISTORY_ITEM`を持つことができる (1対多)。
