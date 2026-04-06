---
name: wmill-native-trigger
description: Guidance for adding native trigger services to Windmill. Use when implementing or modifying native trigger integrations across the backend and frontend.
license: MIT
metadata:
  author: windmill-labs
  version: "1.0.0"
---

# Skill: Adding Native Trigger Services

This skill provides comprehensive guidance for adding new native trigger services to Windmill. Native triggers allow external services (like Nextcloud, Google Drive, etc.) to trigger Windmill scripts/flows via webhooks or push notifications.

## Architecture Overview

The native trigger system consists of:

1. **Database Layer** - PostgreSQL tables and enum types
2. **Backend Rust Implementation** - Core trait, handlers, and service modules in the `windmill-native-triggers` crate
3. **Frontend Svelte Components** - Configuration forms and UI components

### Key Files

| Component                         | Path                                                                         |
| --------------------------------- | ---------------------------------------------------------------------------- |
| Core module with `External` trait | `backend/windmill-native-triggers/src/lib.rs`                                |
| Generic CRUD handlers             | `backend/windmill-native-triggers/src/handler.rs`                            |
| Background sync logic             | `backend/windmill-native-triggers/src/sync.rs`                               |
| OAuth/workspace integration       | `backend/windmill-native-triggers/src/workspace_integrations.rs`             |
| Re-export shim (windmill-api)     | `backend/windmill-api/src/native_triggers/mod.rs`                            |
| TriggerKind enum                  | `backend/windmill-common/src/triggers.rs`                                    |
| JobTriggerKind enum               | `backend/windmill-common/src/jobs.rs`                                        |
| Frontend service registry         | `frontend/src/lib/components/triggers/native/utils.ts`                       |
| Frontend trigger utilities        | `frontend/src/lib/components/triggers/utils.ts`                              |
| Trigger badges (icons + counts)   | `frontend/src/lib/components/graph/renderers/triggers/TriggersBadge.svelte`  |
| Workspace integrations UI         | `frontend/src/lib/components/workspaceSettings/WorkspaceIntegrations.svelte` |
| OAuth config form component       | `frontend/src/lib/components/workspaceSettings/OAuthClientConfig.svelte`     |
| OpenAPI spec                      | `backend/windmill-api/openapi.yaml`                                          |
| Reference: Nextcloud module       | `backend/windmill-native-triggers/src/nextcloud/`                            |
| Reference: Google module          | `backend/windmill-native-triggers/src/google/`                               |

### Crate Structure

The native trigger code lives in the `windmill-native-triggers` crate (`backend/windmill-native-triggers/`). The `windmill-api` crate re-exports everything via a shim:

```rust
// backend/windmill-api/src/native_triggers/mod.rs
pub use windmill_native_triggers::*;
```

All new service modules go in `backend/windmill-native-triggers/src/`.

---

## Core Concepts

### The `External` Trait

Every native trigger service implements the `External` trait defined in `lib.rs`:

```rust
#[async_trait]
pub trait External: Send + Sync + 'static {
    type ServiceConfig: Debug + DeserializeOwned + Serialize + Send + Sync;
    type TriggerData: Debug + Serialize + Send + Sync;
    type OAuthData: DeserializeOwned + Serialize + Clone + Send + Sync;
    type CreateResponse: DeserializeOwned + Send + Sync;

    const SUPPORT_WEBHOOK: bool;
    const SERVICE_NAME: ServiceName;
    const DISPLAY_NAME: &'static str;
    const TOKEN_ENDPOINT: &'static str;
    const REFRESH_ENDPOINT: &'static str;
    const AUTH_ENDPOINT: &'static str;

    async fn create(&self, w_id, oauth_data, webhook_token, data, db, tx) -> Result<Self::CreateResponse>;
    async fn update(&self, w_id, oauth_data, external_id, webhook_token, data, db, tx) -> Result<serde_json::Value>;
    async fn get(&self, w_id, oauth_data, external_id, db, tx) -> Result<Self::TriggerData>;
    async fn delete(&self, w_id, oauth_data, external_id, db, tx) -> Result<()>;
    async fn exists(&self, w_id, oauth_data, external_id, db, tx) -> Result<bool>;
    async fn maintain_triggers(&self, db, workspace_id, triggers, oauth_data, synced, errors);
    fn external_id_and_metadata_from_response(&self, resp) -> (String, Option<serde_json::Value>);

    async fn prepare_webhook(&self, db, w_id, headers, body, script_path, is_flow) -> Result<PushArgsOwned>;
    fn service_config_from_create_response(&self, data, resp) -> Option<serde_json::Value>;
    fn additional_routes(&self) -> axum::Router;
    async fn http_client_request<T, B>(&self, url, method, workspace_id, tx, db, headers, body) -> Result<T>;
}
```

Key design points:

- **`update()` returns `serde_json::Value`** - the resolved service_config to store
- **`maintain_triggers()`** - periodic background maintenance
- **No `list_all()` in the trait** - services that need it implement it privately

### Create Lifecycle: Two Paths

**Path A: Short (Google pattern)** - `service_config_from_create_response()` returns `Some(config)`:

1. `create()` registers on external service
2. `external_id_and_metadata_from_response()` extracts the ID
3. `service_config_from_create_response()` builds the config directly
4. Stores trigger in DB -- done

**Path B: Long (Nextcloud pattern)** - `service_config_from_create_response()` returns `None`:

1. `create()` registers on external service
2. `external_id_and_metadata_from_response()` extracts the ID
3. `update()` is called to fix the webhook URL with the now-known external_id
4. `update()` returns the resolved service_config
5. Stores trigger in DB

### OAuth Token Storage (Three-Table Pattern)

| Table                    | What's Stored                                                                                                                     |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| `workspace_integrations` | `oauth_data` JSON with `base_url`, `client_id`, `client_secret`, `instance_shared` flag; `resource_path` pointing to the variable |
| `variable`               | Encrypted `access_token` (at the path stored in `resource_path`)                                                                  |
| `account`                | `refresh_token`, keyed by `workspace_id` + `client` (service name)                                                                |

---

## Step-by-Step Implementation Guide

### Step 1: Database Migration

Create migration: `backend/migrations/YYYYMMDDHHMMSS_newservice_trigger.up.sql`

### Step 2: Update windmill-common Enums

Add variants to `TriggerKind` and `JobTriggerKind` enums.

### Step 3: Backend Service Module

Create `backend/windmill-native-triggers/src/newservice/` with `mod.rs` and `external.rs`.

### Step 4: Update lib.rs Registry

Add service module and `ServiceName` enum variant with all match arms.

### Step 5: Update handler.rs Routes

Add service to `generate_native_trigger_routers()`.

### Step 6: Update sync.rs

Add service to `sync_all_triggers()`.

### Step 7: Frontend Service Registry

Add to `NATIVE_TRIGGER_SERVICES` in `frontend/src/lib/components/triggers/native/utils.ts`.

### Step 8: Frontend Trigger Form Component

Create `frontend/src/lib/components/triggers/native/services/newservice/NewServiceTriggerForm.svelte`.

### Step 9: Frontend Icon Component

Create `frontend/src/lib/components/icons/NewServiceIcon.svelte`.

### Step 10: Update NativeTriggerEditor

Ensure dynamic loading of form components.

### Step 11: Workspace Integration UI

Add service to `supportedServices` map.

### Step 12: Update triggers/utils.ts

Update `triggerIconMap`, `triggerDisplayNamesMap`, `triggerTypeOrder`, `getLightConfig()`, `getTriggerLabel()`, `jobTriggerKinds`, `countPropertyMap`, `triggerSaveFunctions`.

### Step 13: Update TriggersBadge Component

Import icon, add to `baseConfig` with `countKey`, add to `allTypes` array.

### Step 14: Update TriggersWrapper.svelte

Add `{:else if}` case for new service.

### Step 15: Update AddTriggersButton.svelte

Add availability state and dropdown entry.

### Step 16: Update TriggersEditor.svelte

Add service to `nativeTriggerServices` map in `deleteDeployedTrigger()`.

### Step 17: Update OpenAPI Spec

Add to `JobTriggerKind` enum and regenerate types.

## Testing Checklist

- [ ] Database migration runs successfully
- [ ] `cargo check -p windmill-native-triggers --features native_trigger` passes
- [ ] `npx svelte-check --threshold error` passes (in frontend/)
- [ ] Service appears in workspace integrations list
- [ ] OAuth flow completes successfully
- [ ] Can create, view, update, and delete triggers
- [ ] Webhook receives and processes payloads
- [ ] Background sync works correctly
- [ ] Error handling works (expired tokens, service unavailable)

## Reference Implementations

- **Nextcloud** (self-hosted, update+get pattern): `backend/windmill-native-triggers/src/nextcloud/`
- **Google** (cloud, unified service, short create): `backend/windmill-native-triggers/src/google/`
