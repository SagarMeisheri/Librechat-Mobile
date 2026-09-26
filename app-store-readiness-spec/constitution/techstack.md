# Constitution: Tech Stack & Architecture

## 1. System Architecture Overview
The application is built on **Kotlin Multiplatform (KMP)** and **Compose Multiplatform (CMP)**, targeting Android (API 26+) and iOS (iOS 16+).

```text
├── app/                  # Android launcher & application container
├── iosApp/               # iOS native project wrapper (Swift / Xcode)
├── core/
│   ├── common/           # Constants, AppInfo, FeatureSupport, platform utilities
│   ├── model/            # Shared domain data models & serializable DTOs
│   ├── network/          # Ktor HTTP/SSE client & API endpoints
│   ├── data/             # Repositories, DataStores & Room persistence
│   ├── ui/               # Core design tokens, theme, and common components
│   └── logging/          # Diagnostic logger
└── feature/
    ├── auth/             # Server URL setup, OAuth, 2FA, login, terms acceptance
    ├── chat/             # Message stream, conversation UI, artifacts, voice, feedback/reporting
    ├── conversations/    # History drawer, search, fork/export
    ├── agents/           # Agent marketplace, MCP tools
    ├── files/            # Media library & attachments
    └── settings/         # Preferences, themes, legal & about, account deletion
```

## 2. Core Libraries & Conventions
* **Language & Runtime**: Kotlin 2.x with Coroutines & StateFlow.
* **UI Framework**: Compose Multiplatform (Material 3).
* **Dependency Injection**: Koin (`koinViewModel()`, Koin modules per feature).
* **Networking**: Ktor Client (`HttpClient`) with Server-Sent Events (SSE) streaming engine.
* **Persistence**:
  * **Relational / Caching**: AndroidX Room (multiplatform SQLite) scoped strictly per active account (`AccountScopedDaoRule`).
  * **App Preferences**: `SettingsDataStore` (`core/data/datastore/SettingsDataStore.kt`) backed by multiplatform DataStore Preferences.
* **Localization / Resources**: Compose Multiplatform Resources (`Res.string.*`). All user-facing strings must be declared in `strings.xml`.
* **Serialization**: `kotlinx.serialization` (`@Serializable`, `@SerialName`).

## 3. LibreChat Backend Parity & Contract Boundaries

### 3.1 Upstream Schema Integrity (`scripts/mirrors.json`)
* The mobile client maintains strict schema parity with LibreChat's TypeScript packages (`@librechat/data-provider`, `@librechat/data-schemas`).
* Models registered in `scripts/mirrors.json` (such as `FeedbackTag.kt`, `ArtifactsMode.kt`) are validated in CI via `python3 scripts/check-mirrors.py`.
* **Zero Ad-Hoc Enum Alterations**: Never add client-only enum keys to mirrored models. Any wire payload must pass backend Zod validation (e.g. `feedbackSchema.safeParse()`). Client-only concerns must use dedicated UI models and map into valid upstream payloads.

### 3.2 Dual-Tier Legal & Privacy Architecture
* **Tier 1 (Client App EULA)**: Required by Apple Guideline 1.2 for the distributed client application. Enforced locally before server configuration and persisted in `SettingsDataStore`.
* **Tier 2 (Server Terms & Privacy)**: Provided dynamically by the connected LibreChat instance via `GET /api/config` (`interface.termsOfService`, `interface.privacyPolicy`). Post-login acceptance is tracked on the user model and synchronized via `GET /api/user/terms` and `POST /api/user/terms/accept`.

### 3.3 Backend Contract Registry & Route Topology

| Capability | Backend Endpoint | HTTP Verb | Authentication | Backend Handler & Middleware | Data Contract / Schema |
|---|---|:---:|:---:|---|---|
| **Startup Config** | `/api/config` | `GET` | None | `api/server/routes/config.js` | `TStartupConfig` (unauth pre-login payload) |
| **Reviewer Login** | `/api/auth/login` | `POST` | None | `loginController` (`requireLocalAuth`) | `{ email, password }` -> JWT token & refresh cookie |
| **Abuse Reporting** | `/api/messages/:convoId/:msgId/feedback` | `POST` | Bearer JWT | `messages.js` (`feedbackSchema`) | `{ feedback: { rating, tag, text } }` |
| **Terms Query** | `/api/user/terms` | `GET` | Bearer JWT | `getTermsStatusController` | `{ termsAccepted: boolean, termsAcceptedAt: string? }` |
| **Terms Accept** | `/api/user/terms/accept` | `POST` | Bearer JWT | `acceptTermsController` | `{ message: string, termsAcceptedAt: string }` |
| **Account Deletion** | `/api/user/delete` | `DELETE` | Bearer JWT | `canDeleteAccount`, `deleteUserController` | Optional 2FA `{ token?, backupCode? }` |
| **Apple OAuth** | `/oauth/apple` | `GET` | None | `passport-apple` strategy | `code`, `id_token` -> OAuth callback redirect |

## 4. Platform-Specific Boundaries

### iOS Native Integration (`iosApp/`)
* **Bundle Identifier**: `com.garfiec.librechat.ios`
* **Display Name**: `Switchboard`
* **App Transport Security (ATS)**: Review builds MUST use HTTPS. Cleartext HTTP exceptions are restricted to local development/intranet connections.
* **IPv6 Requirement**: Apple conducts reviews on IPv6-only networks. All network calls must resolve domain names through high-level APIs (Ktor/Darwin/NSURLSession) and avoid hardcoded IPv4 literals.

### Android Native Integration (`app/`)
* **Application ID**: `com.garfiec.librechat`
* **Network Security**: Configured in `network_security_config.xml`. While cleartext is allowed for local self-hosters with a mandatory in-app confirmation dialog, store review testing must default to HTTPS.

## 5. Coding Standards & Agent Constraints
1. **No Monolithic Single-File Edits**: Features and UI dialogs must be partitioned into dedicated composables and ViewModels under `feature/<domain>/components/`.
2. **Account Isolation Guarantee**: All persistent data and queries must respect multi-tenant account scoping (`AccountScopedDaoRule`).
3. **Immutability & State Handling**: ViewModels expose immutable `StateFlow<UiState>`. User events are dispatched via public methods on ViewModels.
4. **String Extraction**: Do not hardcode UI text. Every label, button, and prompt must be added to `feature/<domain>/src/commonMain/composeResources/values/strings.xml`.
