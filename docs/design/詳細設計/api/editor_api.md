# API詳細設計書: /sessions/{session_id}/editor

## 1. エンドポイント

`GET /sessions/{session_id}/editor`

## 2. 概要

実行中のセッションに対応するWeb Editor (code-server) へのアクセスURLを取得する。
セキュリティのため、URLには有効期限付きの署名（またはトークン）が含まれる。

## 3. リクエスト

### 3.1. パスパラメータ

| 名前 | 型 | 必須 | 説明 |
| :--- | :--- | :--- | :--- |
| `session_id` | String | ○ | 対象のセッションID (UUID v4)。 |

### 3.2. ヘッダー

| 名前 | 型 | 必須 | 説明 |
| :--- | :--- | :--- | :--- |
| `X-API-Key` | String | ○ | 認証用のAPIキー |
| `X-User-ID` | String | ○ | リクエスト元のユーザーを識別するID |

## 4. レスポンス

### 4.1. 成功 (200 OK)

```json
{
  "editor_url": "https://xxx.execute-api.us-east-1.amazonaws.com/editor?token=abc123",
  "expires_at": 1692349200
}
```

| フィールド | 型 | 説明 |
| :--- | :--- | :--- |
| `editor_url` | String | 署名付きのWeb EditorアクセスURL。 |
| `expires_at` | Number | URLの有効期限 (Unix time)。 |

### 4.2. エラーレスポンス

| HTTPステータス | エラーコード | 説明 |
| :--- | :--- | :--- |
| 404 Not Found | `session_not_found` | 指定された`session_id`が存在しない。 |
| 409 Conflict | `session_not_running` | 対象のセッションが `running` 状態ではない。 |
| 503 Service Unavailable | `container_not_ready` | コンテナは起動しているが、Web Editorがまだ利用可能な状態になっていない。 |

## 5. シーケンス図

```mermaid
sequenceDiagram
    participant FE as フロントエンド
    participant APIGW as API Gateway
    participant Lambda as get_editor_url
    participant DynamoDB
    participant ECS

    FE->>APIGW: GET /sessions/{id}/editor
    APIGW->>Lambda: リクエストを転送
    Lambda->>DynamoDB: セッション情報を取得
    Lambda->>ECS: DescribeTasks (コンテナのIPアドレス等を取得)
    alt コンテナ準備完了
        Lambda->>Lambda: 署名付きURLを生成
        Lambda->>DynamoDB: editor_urlを更新
        Lambda-->>APIGW: 200 OK (URLを返す)
        APIGW-->>FE: レスポンスを転送
        FE->>FE: 新しいタブでEditorを開く
    else コンテナ準備中
        Lambda-->>APIGW: 503 Service Unavailable
        APIGW-->>FE: エラーレスポンスを転送
    end
```
