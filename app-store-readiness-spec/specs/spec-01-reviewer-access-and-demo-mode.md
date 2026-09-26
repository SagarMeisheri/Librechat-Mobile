# Spec 01: Reviewer Access & Zero-Friction Demo Mode

## 1. Context & Problem Statement
Under **Apple App Store Review Guideline 2.1 (App Completeness)** and Google Play testing standards, an application requiring an external server configuration but offering no accessible demonstration backend or functional credentials will be rejected immediately. Reviewers test apps inside sandboxed, ephemeral test environments on strict review time limits; they will not configure a self-hosted LibreChat server.

Currently, on first launch, `ServerUrlScreen.kt` presents an empty URL text input. If a reviewer enters nothing or invalid data, they remain stuck. Furthermore, LibreChat requires JWT authentication (`requireJwtAuth`) on all conversation endpoints, meaning the app must provide working credentials without friction.

## 2. Objective
Provide a seamless, zero-configuration path allowing store reviewers and first-time evaluators to test the full range of native app capabilities (chat streaming, code syntax highlighting, artifacts, LaTeX math, and responsive tablet layout) with a single tap, backed by a production-hardened LibreChat demo instance.

---

## 3. Architecture & File Modifications

### Modified Files:
* `core/common/src/commonMain/kotlin/com/garfiec/librechat/core/common/Constants.kt`
  * Add `DemoServerConstants` holding `DEMO_SERVER_URL = "https://demo.switchboard.chat"`, `REVIEWER_EMAIL`, and obfuscated/prefilled credentials.
* `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/screen/ServerUrlScreen.kt`
  * Add "Explore with Public Demo" secondary button below the primary "Connect" button.
* `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/viewmodel/ServerUrlViewModel.kt`
  * Add `connectDemoServer()` action that injects `DemoServerConstants.DEMO_SERVER_URL` and triggers server handshake (`GET /api/config`).
* `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/screen/LoginScreen.kt`
  * When connected to the demo server, render a prominent single-tap **"Sign In with Reviewer Account"** action button above the standard email/password fields.
* `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/viewmodel/LoginViewModel.kt`
  * Add `loginAsReviewer()` helper that automatically populates reviewer credentials and dispatches `login()`.
* `feature/auth/src/commonMain/composeResources/values/strings.xml`
  * Add resource strings: `btn_try_demo_server`, `demo_server_description`, `btn_login_as_reviewer`.

---

## 4. Requirements & Implementation Details

### R1.1: Demo Server Backend Runbook (LibreChat Deployment)
The public demo backend must be deployed using the official LibreChat container with strict compliance environment settings:

```env
# Server Domain & Networking
HOST=0.0.0.0
PORT=3080
DOMAIN_SERVER=https://demo.switchboard.chat
DOMAIN_CLIENT=https://demo.switchboard.chat

# Store Compliance: Disable Social Logins to satisfy Apple Guideline 4.8
ALLOW_SOCIAL_LOGIN=false
ALLOW_SOCIAL_REGISTRATION=false

# Store Compliance: Email Login Enabled with Registration Closed
ALLOW_EMAIL_LOGIN=true
ALLOW_REGISTRATION=false

# Store Compliance: Allow Native In-App Account Deletion (Apple Guideline 5.1.1(v))
ALLOW_ACCOUNT_DELETION=true

# AI Safety: Enable OpenAI Moderation Filter on Incoming Prompts
OPENAI_MODERATION=true
OPENAI_MODERATION_REVERSE_PROXY=https://api.openai.com/v1/moderations

# Long-lived Sessions for Review Stability
SESSION_EXPIRY=2592000000
REFRESH_TOKEN_EXPIRY=2592000000
```

* **IPv6 Network Audit**: The domain `demo.switchboard.chat` must resolve with valid `AAAA` DNS records and respond over IPv6 without packet drop.
* **Pre-Seeded Mongo Data**: The database (`mongodb://.../LibreChat`) must be pre-populated for user `appreview@switchboard.chat` with 4 distinct pinned conversations:
  1. *Markdown & Syntax Highlighting*: Python, Kotlin, and TypeScript code blocks with copy actions.
  2. *Mermaid Diagrams*: Sequence and architecture diagrams rendering via native CMP canvas/SVG.
  3. *Artifacts*: Interactive HTML/SVG previews rendering in inline or sheet mode.
  4. *LaTeX Equations*: Complex mathematical formulas rendering via LaTeX parser.

### R1.2: Mobile UI Quick-Connect & 1-Tap Login
1. On `ServerUrlScreen`:
   * Render `[ Connect ]` (Primary).
   * Directly below, render `[ Try with Demo Server ]` (Outlined / Secondary).
   * Tapping initiates `connectDemoServer()`, which calls `/api/config` and transitions to `LoginScreen`.
2. On `LoginScreen`:
   * Check if current server URL equals `DemoServerConstants.DEMO_SERVER_URL`.
   * If true, display a highlighted card:
     ```text
     ┌──────────────────────────────────────────────┐
     │ 🚀 App Store Reviewer Mode                   │
     │ Instant testing credentials are configured.  │
     │ [ Sign In as Reviewer ]                      │
     └──────────────────────────────────────────────┘
     ```
   * Tapping `Sign In as Reviewer` fills the credentials and triggers `login()` without requiring manual typing.

### R1.3: Store Review Notes Copy
The **App Review Information** notes submitted in App Store Connect & Google Play Console:
```text
SWITCHBOARD REVIEWER TESTING INSTRUCTIONS:
Switchboard is a native client for self-hosted LibreChat instances.
To allow immediate, zero-configuration evaluation without hosting your own server:
1. On the opening screen, tap "Try with Demo Server" (or enter: https://demo.switchboard.chat).
2. On the login screen, tap "Sign In as Reviewer" (or enter email: appreview@switchboard.chat / password: [SECURE_PASSWORD]).
3. The account is pre-loaded with sample conversations demonstrating markdown, Mermaid diagrams, interactive artifacts, and LaTeX math.
```

---

## 5. Validation Scorecard

| Check ID | Verification Criteria | Test Method | Pass / Fail |
|:---:|---|---|:---:|
| **VAL-01** | First launch displays "Try with Demo Server" button cleanly across phones and tablets. | Visual UI inspection (Android + iOS) | [ ] |
| **VAL-02** | Demo backend returns `200 OK` on IPv6 network without TLS warnings. | `curl -6 -I https://demo.switchboard.chat/api/config` | [ ] |
| **VAL-03** | Tapping "Try with Demo Server" navigates to `LoginScreen` within 800ms. | Integration test | [ ] |
| **VAL-04** | "Sign In as Reviewer" button performs login and transitions to chat thread within 1 tap. | Device UI automation | [ ] |
| **VAL-05** | Pre-seeded conversations render without crashes or layout clipping. | Chat inspection | [ ] |
| **VAL-06** | Entering a custom self-hosted URL still functions normally without regression. | Regression test | [ ] |

---

## 6. Backend Contract & Integration Details

### 6.1 Handshake Contract: `GET /api/config`
* **Route Implementation**: `api/server/routes/config.js` (`buildPreLoginPayload()`).
* **Authentication**: None (Anonymous caller).
* **Wire Response Contract**:
  ```json
  {
    "appTitle": "Switchboard Demo",
    "socialLoginEnabled": false,
    "emailLoginEnabled": true,
    "registrationEnabled": false,
    "passwordResetEnabled": false,
    "socialLogins": [],
    "interface": {
      "privacyPolicy": {
        "externalUrl": "https://switchboard.chat/privacy",
        "openNewTab": true
      },
      "termsOfService": {
        "externalUrl": "https://switchboard.chat/terms",
        "openNewTab": true,
        "modalAcceptance": false
      }
    }
  }
  ```
* **Client Handshake Guard**: The client requires HTTP `200 OK` and non-empty `serverDomain`/`appTitle` to mark the server connection validated.

### 6.2 Reviewer Authentication Contract: `POST /api/auth/login`
* **Route Implementation**: `api/server/routes/auth.js` (`loginController` with Passport `local` strategy).
* **Wire Request**:
  ```json
  {
    "email": "appreview@switchboard.chat",
    "password": "[SECURE_PASSWORD]"
  }
  ```
* **Wire Response (`200 OK`)**:
  * Response Body: `{ "token": "<JWT_ACCESS_TOKEN>", "user": { "id": "...", "email": "appreview@switchboard.chat", ... } }`
  * Response Headers: Sets HttpOnly `refreshToken` cookie.
* **Error Response (`403 Forbidden`)**: Occurs if `ALLOW_EMAIL_LOGIN=false`. Demo server `.env` must guarantee `ALLOW_EMAIL_LOGIN=true`.

### 6.3 MongoDB Seed Contracts for Pre-Loaded Conversations
Pre-seeded conversations inserted into MongoDB (`mongodb://.../LibreChat`) must adhere to data schemas defined in `@librechat/data-schemas`:
* **`conversations` collection**:
  * `conversationId`: UUID string (e.g., `demo-convo-artifacts-01`)
  * `user`: ObjectId of `appreview@switchboard.chat`
  * `title`: "Interactive Artifacts & Code Preview"
  * `endpoint`: `openAI` or `custom`
  * `model`: `gpt-4o-mini`
* **`messages` collection**:
  * `messageId`: UUID string
  * `conversationId`: Matches conversation UUID
  * `sender`: "Switchboard Assistant"
  * `isCreatedByUser`: `false`
  * `text`: Markdown with fenced code blocks (```python, ```mermaid), LaTeX expressions (`$$E=mc^2$$`), or artifact tags.


