---
name: tauri
description: Canonical Tauri IPC implementations — command signatures, thiserror enums, State management, async discipline, and TypeScript IPC types. Use as reference when writing Tauri commands, Rust backend logic, or IPC contracts. For Rust language-level questions, consult rust-skills first.
license: MIT
---

# Tauri IPC: Canonical Patterns

Target: Tauri v2. For Rust language patterns (ownership, error handling, concurrency, etc.), always consult rust-skills first. This skill covers Tauri-specific IPC patterns only.

## Command signature

```rust
#[tauri::command]
async fn get_user(
    state: tauri::State<'_, AppState>,
    user_id: String,
) -> Result<User, AppError> {
    let db = state.db.lock().await;
    db.find_user(&user_id)
        .ok_or_else(|| AppError::UserNotFound(user_id))
}
```

## Error enum with thiserror + Serialize

```rust
#[derive(Debug, thiserror::Error, serde::Serialize)]
#[serde(tag = "code", content = "message", rename_all = "SCREAMING_SNAKE_CASE")]
pub enum AppError {
    #[error("User not found: {0}")]
    UserNotFound(String),

    #[error("Database error: {0}")]
    Database(String),

    #[error("Invalid input: {0}")]
    Validation(String),

    #[error("Internal error")]
    Internal(String),
}

// Tauri v2: convert AppError into an InvokeError the frontend can parse
impl From<AppError> for tauri::ipc::InvokeError {
    fn from(err: AppError) -> Self {
        serde_wasm_bindgen::to_value(&err)
            .map(tauri::ipc::InvokeError::from)
            .unwrap_or_else(|_| tauri::ipc::InvokeError::from("unknown error"))
    }
}
```

## State ownership: Rust holds, frontend references by opaque ID

```rust
use std::collections::HashMap;
use tokio::sync::Mutex;

pub struct AppState {
    pub db: DatabasePool,
    pub sessions: Mutex<HashMap<String, Session>>,
}

#[tauri::command]
async fn create_session(
    state: tauri::State<'_, AppState>,
    config: SessionConfig,
) -> Result<String, AppError> {
    let id = uuid::Uuid::new_v4().to_string();
    let session = Session::new(&config).map_err(|e| AppError::Internal(e.to_string()))?;
    state.sessions.lock().await.insert(id.clone(), session);
    Ok(id) // opaque string ID — frontend never sees the Session object
}

#[tauri::command]
async fn get_session_status(
    state: tauri::State<'_, AppState>,
    session_id: String,
) -> Result<SessionStatus, AppError> {
    let sessions = state.sessions.lock().await;
    let session = sessions
        .get(&session_id)
        .ok_or_else(|| AppError::UserNotFound(session_id))?;
    Ok(session.status())
}
```

## CPU-bound work in spawn_blocking

```rust
#[tauri::command]
async fn process_audio(
    state: tauri::State<'_, AppState>,
    input_id: String,
) -> Result<String, AppError> {
    let session = {
        let sessions = state.sessions.lock().await;
        sessions
            .get(&input_id)
            .cloned()
            .ok_or_else(|| AppError::UserNotFound(input_id))?
    };

    let output_id = tokio::task::spawn_blocking(move || {
        let output = transcode_audio(&session.file_path)?;
        Ok::<_, AppError>(output.id)
    })
    .await
    .map_err(|e| AppError::Internal(e.to_string()))??;

    Ok(output_id)
}
```

## Async I/O via reqwest

```rust
#[tauri::command]
async fn fetch_external_data(url: String) -> Result<Data, AppError> {
    let response = reqwest::get(&url)
        .await
        .map_err(|e| AppError::Internal(e.to_string()))?;

    let data: Data = response
        .json()
        .await
        .map_err(|e| AppError::Internal(e.to_string()))?;

    Ok(data)
}
```

## TypeScript types matching the Rust error enum

```typescript
// Matches AppError's #[serde(tag = "code", content = "message")]
type AppError =
  | { code: "USER_NOT_FOUND"; message: string }
  | { code: "DATABASE"; message: string }
  | { code: "VALIDATION"; message: string }
  | { code: "INTERNAL"; message: string };

// Usage — invoke is typed via the declared return type
import { invoke } from "@tauri-apps/api/core"; // Tauri v2

async function getUser(id: string): Promise<User> {
  return invoke("get_user", { userId: id });
}
```

## Integration test

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn get_user_returns_user_when_found() {
        let state = test_state().await;
        let result = get_user(state, "user-1".into()).await;
        assert!(result.is_ok());
    }

    #[tokio::test]
    async fn get_user_returns_error_when_not_found() {
        let state = test_state().await;
        let result = get_user(state, "nonexistent".into()).await;
        assert!(matches!(result, Err(AppError::UserNotFound(_))));
    }
}
```
