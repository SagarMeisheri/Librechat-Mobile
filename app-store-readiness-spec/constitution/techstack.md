# Constitution: Tech Stack & Architecture

## 1. System Architecture Overview
The application is built on **Kotlin Multiplatform (KMP)** and **Compose Multiplatform (CMP)**, targeting Android (API 26+) and iOS (iOS 16+).

```text
├── app/                  # Android launcher & application container
├── iosApp/               # iOS native project wrapper (Swift / Xcode)
├── core/
│   ├── model/            # Shared domain data models & serializable DTOs
│   ├── network/          # Ktor HTTP/SSE client & API endpoints
│   ├── data/             # Repositories & Room persistence
│   ├── ui/               # Core design tokens, theme, and common components
│   └── logging/          # Diagnostic logger
└── feature/
    ├── auth/             # Server URL setup, OAuth, 2FA, login
    ├── chat/             # Message stream, conversation UI, artifacts, voice
    ├── conversations/    # History drawer, search, fork/export
    ├── agents/           # Agent marketplace, MCP tools
    ├── files/            # Media library & attachments
    └── settings/         # Preferences, themes, account deletion
```

## 2. Core Libraries & Conventions
* **Language & Runtime**: Kotlin 2.x with Coroutines & StateFlow.
* **UI Framework**: Compose Multiplatform (Material 3).
* **Dependency Injection**: Koin (`koinViewModel()`, Koin modules per feature).
* **Networking**: Ktor Client (`HttpClient`) with Server-Sent Events (SSE) streaming engine.
* **Persistence**: AndroidX Room (multiplatform SQLite) scoped strictly per active account.
* **Localization / Resources**: Compose Multiplatform Resources (`Res.string.*`). All user-facing strings must be declared in `strings.xml`.
* **Serialization**: `kotlinx.serialization` (`@Serializable`, `@SerialName`).

## 3. Platform-Specific Boundaries

### iOS Native Integration (`iosApp/`)
* **Bundle Identifier**: `com.garfiec.librechat.ios`
* **Display Name**: `Switchboard`
* **App Transport Security (ATS)**: Review builds MUST use HTTPS. Cleartext HTTP exceptions must only be leveraged for local development/intranet connections.
* **IPv6 Requirement**: Apple conducts reviews on IPv6-only networks. All network calls must resolve domain names through high-level APIs (Ktor/Darwin/NSURLSession) and avoid hardcoded IPv4 literals.

### Android Native Integration (`app/`)
* **Application ID**: `com.garfiec.librechat`
* **Network Security**: Configured in `network_security_config.xml`. While cleartext is allowed for local self-hosters with a mandatory in-app confirmation dialog, store review testing must default to HTTPS.

## 4. Coding Standards & Agent Constraints
1. **No Monolithic Single-File Edits**: Features and UI dialogs must be partitioned into dedicated composables and ViewModels under `feature/<domain>/components/`.
2. **Account Isolation Guarantee**: All persistent data and queries must respect multi-tenant account scoping (`AccountScopedDaoRule`).
3. **Immutability & State Handling**: ViewModels expose immutable `StateFlow<UiState>`. User events are dispatched via public methods on ViewModels.
4. **String Extraction**: Do not hardcode UI text. Every label, button, and prompt must be added to `feature/<domain>/src/commonMain/composeResources/values/strings.xml`.
