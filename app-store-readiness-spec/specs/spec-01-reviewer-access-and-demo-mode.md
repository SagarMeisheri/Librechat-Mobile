# Spec 01: Reviewer Access & Zero-Friction Demo Mode

## 1. Context & Problem Statement
Under **Apple App Store Review Guideline 2.1 (App Completeness)** and Google Play testing standards, an application that requires an external server configuration but offers no accessible demonstration backend or functional credentials will be summarily rejected. Reviewers test apps inside sandboxed, ephemeral test environments; they will not setup their own LibreChat server.

Currently, on first launch, `ServerUrlScreen.kt` presents an empty URL text input. If a reviewer enters nothing or invalid data, they remain stuck.

## 2. Objective
Provide a seamless, zero-configuration path allowing store reviewers and first-time evaluators to test the full range of native app capabilities (chat streaming, code syntax highlighting, artifacts, LaTeX math, and responsive tablet layout) with a single tap.

---

## 3. Architecture & File Modifications

### Modified Files:
* `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/screen/ServerUrlScreen.kt`
  * Add "Explore with Public Demo" secondary button.
* `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/viewmodel/ServerUrlViewModel.kt`
  * Add `connectDemoServer()` action that injects pre-configured demo server URL and initiates validation.
* `feature/auth/src/commonMain/composeResources/values/strings.xml`
  * Add resource strings: `btn_try_demo_server`, `demo_server_description`.
* `core/model/src/commonMain/kotlin/com/garfiec/librechat/core/model/config/AppBuildConstants.kt` (or config object)
  * Store the canonical demo server URL (e.g., `https://demo.switchboard.chat`).

---

## 4. Requirements & Implementation Details

### R1.1: Demo Server Provisioning (Infrastructure)
* The demo instance must be hosted over **HTTPS** on a public domain with valid, trusted SSL/TLS certificates.
* The domain must resolve properly on **IPv6-only networks** (mandated by Apple review testing).
* Pre-configured test accounts must exist:
  * Username: `appreview@switchboard.chat`
  * Password: `[ProvidedInReviewNotes]`
* Pre-seeded conversations must be available in the account demonstrating:
  1. Multi-turn markdown & code formatting.
  2. Inline Mermaid flowchart rendering.
  3. Interactive artifact cards (HTML/SVG).
  4. Math equation typesetting with LaTeX.
* Default model on this demo server must have strict safety filters activated to resist reviewer jailbreak testing.

### R1.2: UI Quick-Connect Affordance
* On `ServerUrlScreen`, directly below the Primary "Connect" button, render a visually distinct secondary text/outlined button:
  ```text
  [ Connect ]
  
  — or —
  
  [ Try with Demo Server ]
  ```
* Tapping `Try with Demo Server`:
  1. Populates the URL field with `https://demo.switchboard.chat`.
  2. Dispatches server validation automatically.
  3. Navigates smoothly to `LoginScreen` upon successful handshake.

### R1.3: Store Review Information Notes
Prepare exact text for the **App Review Information** section in App Store Connect & Google Play Console:
```text
SWITCHBOARD REVIEWER INSTRUCTIONS:
Switchboard is a third-party native client for self-hosted LibreChat instances. 
To facilitate immediate testing without configuring your own backend:
1. Tap "Try with Demo Server" on the launch screen (or use Server URL: https://demo.switchboard.chat).
2. On the login screen, enter:
   - Email: appreview@switchboard.chat
   - Password: [SECURE_PASSWORD]
3. Several sample conversations demonstrating offline viewing, markdown formatting, LaTeX equations, and diagram artifacts are already populated in this account.
```

---

## 5. Validation Scorecard

| Check ID | Verification Criteria | Test Method | Pass / Fail |
|:---:|---|---|:---:|
| **VAL-01** | First launch displays "Try with Demo Server" button cleanly across phones and tablets. | Visual UI inspection (Android + iOS) | [ ] |
| **VAL-02** | Tapping demo button populates HTTPS URL and advances to login without network errors. | Manual Device Test | [ ] |
| **VAL-03** | Backend responds under 500ms on IPv6-only cellular/Wi-Fi connection. | `curl -6 https://demo.switchboard.chat/api/config` | [ ] |
| **VAL-04** | Pre-seeded conversations render without crashes or layout clipping. | Chat inspection | [ ] |
| **VAL-05** | Entering a custom self-hosted URL still functions normally without regression. | Regression test | [ ] |
