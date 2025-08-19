# データモデル定義書 (Pydantic)

## 1. 概要

本ドキュメントは、「DevFlow Engine」のバックエンドで使用するデータモデルをPydanticモデルとして定義する。
これにより、APIリクエスト/レスポンスの型安全性を保証し、バリデーションを強制する。

## 2. モデル定義

### 2.1. 基本モデル

#### `HistoryItem`

対話履歴の単一アイテムを表す。

```python
from typing import Optional
from pydantic import BaseModel, Field

class HistoryItem(BaseModel):
    prompt: str = Field(description="ユーザーが入力したプロンプト")
    response: str = Field(description="システムからの応答")
    timestamp: int = Field(description="タイムスタンプ (Unix time)")
    pr_url: Optional[str] = Field(None, description="関連するPull RequestのURL")
```

#### `SessionStatus`

セッションの状態を表すEnum。

```python
from enum import Enum

class SessionStatus(str, Enum):
    STARTING = "starting"
    RUNNING = "running"
    STOPPED = "stopped"
    ERROR = "error"
```

### 2.2. DynamoDBモデル

#### `Session`

`devflow-sessions`テーブルに格納されるアイテムの完全なスキーマを定義する。

```python
import uuid
from datetime import datetime, timedelta
from typing import List, Optional

class Session(BaseModel):
    session_id: str = Field(default_factory=lambda: str(uuid.uuid4()), description="セッションID (UUID v4)")
    user_id: str = Field(description="ユーザーID")
    repository_url: str = Field(description="GitHubリポジトリURL")
    repository_owner: str = Field(description="リポジトリオーナー")
    repository_name: str = Field(description="リポジトリ名")
    branch_name: str = Field(description="作業ブランチ名")
    ecs_cluster_arn: Optional[str] = Field(None, description="ECSクラスターARN")
    ecs_task_arn: Optional[str] = Field(None, description="ECSタスクARN")
    ecs_task_definition_arn: Optional[str] = Field(None, description="ECSタスク定義ARN")
    ecs_service_name: Optional[str] = Field(None, description="ECSサービス名")
    pr_number: Optional[int] = Field(None, description="Pull Request番号")
    pr_url: Optional[str] = Field(None, description="Pull RequestのURL")
    editor_url: Optional[str] = Field(None, description="Web EditorのURL")
    status: SessionStatus = Field(SessionStatus.STARTING, description="セッションの状態")
    initial_prompt: str = Field(description="初回プロンプト")
    history: List[HistoryItem] = Field(default_factory=list, description="対話履歴")
    github_installation_id: int = Field(description="GitHub AppインストールID")
    created_at: int = Field(default_factory=lambda: int(datetime.now().timestamp()), description="作成日時 (Unix time)")
    updated_at: int = Field(default_factory=lambda: int(datetime.now().timestamp()), description="更新日時 (Unix time)")
    ttl: int = Field(default_factory=lambda: int((datetime.now() + timedelta(hours=24)).timestamp()), description="TTL (24時間後)")
    error_message: Optional[str] = Field(None, description="エラーメッセージ")
```

### 2.3. APIリクエスト/レスポンスモデル

#### `StartSessionRequest` (POST /sessions)

```python
class StartSessionRequest(BaseModel):
    repository_url: str = Field(..., regex=r'^https://github\.com/[\w\-\.]+/[\w\-\.]+$', description="有効なGitHubリポジトリURL")
    prompt: str = Field(..., min_length=1, max_length=5000, description="初回プロンプト")
```

#### `StartSessionResponse` (POST /sessions)

```python
class StartSessionResponse(BaseModel):
    session_id: str = Field(description="発行されたセッションID")
    status: SessionStatus = Field(description="現在のセッション状態")
    message: str = Field(description="ユーザーへのメッセージ")
```

#### `ExecutePromptRequest` (POST /sessions/{session_id}/prompts)

```python
class ExecutePromptRequest(BaseModel):
    prompt: str = Field(..., min_length=1, max_length=5000, description="追加のプロンプト")
```

#### `ExecutePromptResponse` (POST /sessions/{session_id}/prompts)

```python
class ExecutePromptResponse(BaseModel):
    status: SessionStatus = Field(description="現在のセッション状態")
    message: str = Field(description="ユーザーへのメッセージ")
    pr_url: Optional[str] = Field(None, description="更新されたPull RequestのURL")
```

#### `GetSessionResponse` (GET /sessions/{session_id})

```python
class GetSessionResponse(Session):
    # Sessionモデルをそのまま利用するが、レスポンスとして定義
    pass
```

#### `GetEditorUrlResponse` (GET /sessions/{session_id}/editor)

```python
class GetEditorUrlResponse(BaseModel):
    editor_url: str = Field(description="署名付きのWeb EditorアクセスURL")
    expires_at: int = Field(description="URLの有効期限 (Unix time)")
```

#### `DeleteSessionResponse` (DELETE /sessions/{session_id})

```python
class DeleteSessionResponse(BaseModel):
    message: str = Field(description="完了メッセージ")
```

#### `ErrorResponse` (共通エラーレスポンス)

```python
from typing import Any

class ErrorDetail(BaseModel):
    field: Optional[str] = None
    value: Optional[Any] = None

class ErrorResponse(BaseModel):
    error: str = Field(description="エラーコード")
    message: str = Field(description="人間可読なエラーメッセージ")
    details: Optional[ErrorDetail] = Field(None, description="エラーの詳細")
    request_id: str = Field(description="AWSリクエストID")
    timestamp: int = Field(description="タイムスタンプ (Unix time)")
```
