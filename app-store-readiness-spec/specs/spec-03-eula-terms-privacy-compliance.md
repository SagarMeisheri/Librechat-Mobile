# Spec 03: EULA, Terms of Service & Privacy Compliance

## 1. Context & Problem Statement
* **Apple Guideline 1.2 (User-Generated Content)**: Requires apps with open-ended AI or community-driven content to bind users to an End User License Agreement (EULA) with explicit zero tolerance for objectionable content.
* **Apple Guideline 5.1.1 (Data Collection & Privacy)** & **Google Play User Data Policy**: Apps must provide accessible Privacy Policies both on the store listing and in-app, disclose data retention policies, and explain third-party AI model data transmission.

### Architectural Reality: Dual-Tier Legal Model
In a self-hosted client architecture, legal compliance operates across two distinct domains:
1. **Tier 1 (Client App EULA & Privacy)**: The agreement between the mobile user and the developer of Switchboard, establishing client terms, disclaimers, and zero-tolerance content rules.
2. **Tier 2 (Server-Specific Terms & Privacy)**: The agreement between the user and their individual LibreChat server administrator. LibreChat natively supports this via `interfaceConfig.termsOfService` / `privacyPolicy` in `librechat.yaml`, delivered via `GET /api/config`, and persisted server-side via `GET /api/user/terms` and `POST /api/user/terms/accept`.

`Librechat-Mobile` already contains `TermsScreen.kt` and `TermsViewModel.kt` for Tier 2, but lacks the first-launch Tier 1 EULA gating and the post-login bridge.

## 2. Objective
Implement a compliant dual-tier legal flow: enforce first-launch Client App EULA acceptance via `SettingsDataStore`, bridge post-login server terms gating via the existing `TermsScreen`, and provide comprehensive in-app legal disclosures in Settings.

---

## 3. Architecture & File Modifications

### Modified Files:
* `core/data/src/commonMain/kotlin/com/garfiec/librechat/core/data/datastore/SettingsDataStore.kt`
  * Add `isAppEulaAccepted: Flow<Boolean>` and `suspend fun setAppEulaAccepted(accepted: Boolean)`.
* `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/screen/ServerUrlScreen.kt`
  * Gate initial setup behind `AppEulaBottomSheet` if `isAppEulaAccepted` is `false`.
* `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/viewmodel/LoginViewModel.kt`
  * On `LoginOutcome.Success`, check `user.termsAccepted` and `startupConfig.interface?.termsOfService?.modalAcceptance`. If terms acceptance is pending, emit `requiresTermsAcceptance = true` to route to `TermsScreen`.
* `feature/settings/src/commonMain/kotlin/com/garfiec/librechat/feature/settings/screen/SettingsScreen.kt`
  * Add a dedicated "Legal & About" section presenting client policies, server policies (if provided by `/api/config`), and AI data disclosure.
* `feature/settings/src/commonMain/composeResources/values/strings.xml`
  * Add localization strings: `title_legal_privacy`, `item_app_eula`, `item_app_privacy`, `item_server_terms`, `item_server_privacy`, `item_ai_disclosure`, `dialog_ai_disclosure_title`, `dialog_ai_disclosure_body`.

### New Files:
* `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/components/AppEulaBottomSheet.kt`
  * First-run consent sheet displaying zero tolerance policy, links to Switchboard Terms & Privacy, and an "Agree & Continue" button.

---

## 4. Requirements & Implementation Details

### R3.1: Tier 1 - Mandatory First-Run Client EULA
* On initial app launch (before server validation):
  * Query `SettingsDataStore.isAppEulaAccepted`.
  * If `false`, display non-dismissible `AppEulaBottomSheet`.
  * Headline: *"Welcome to Switchboard"*.
  * Copy:
    > *"Switchboard is an open client for connecting to self-hosted LibreChat servers. By using this application, you agree to our Terms of Service and End User License Agreement (EULA).*  
    >  
    > **Zero Tolerance Policy:** You may not use this application to generate, distribute, or facilitate hate speech, sexually explicit material, harassment, copyright infringement, or unlawful conduct. Engaging in abusive behavior may result in permanent blocking.*"
  * Action: `[ Agree & Continue ]` calls `setAppEulaAccepted(true)` and unlocks the server connection UI.

### R3.2: Tier 2 - Server Terms of Service Integration
* When logging into a LibreChat server:
  1. If `startupConfig.interface?.termsOfService?.modalAcceptance == true` and `user.termsAccepted == false`:
  2. The login flow navigates to the existing `AuthRoute.Terms` (`TermsScreen.kt`).
  3. `TermsViewModel` fetches markdown terms via `GET /api/user/terms`.
  4. Tapping "I Accept" invokes `userRepository.acceptTerms()`, posting to `POST /api/user/terms/accept` to update the user's record in MongoDB.
  5. Upon 200 OK response, the user transitions into the chat workspace.

### R3.3: Settings "Legal & About" Section
In `SettingsScreen`, render the "Legal & About" group:
1. **Switchboard Client EULA & Terms**: Opens `https://switchboard.chat/terms`.
2. **Switchboard Privacy Policy**: Opens `https://switchboard.chat/privacy`.
3. **Connected Server Policies** *(Conditional)*:
   * If `startupConfig.interface?.termsOfService?.externalUrl` is present, display "Server Terms of Service".
   * If `startupConfig.interface?.privacyPolicy?.externalUrl` is present, display "Server Privacy Policy".
4. **AI Data Handling Disclosure**: Tapping opens a native modal:
   > *"**How your data is handled:**  
   > Switchboard connects directly to your self-hosted LibreChat server. Switchboard operates no centralized analytics, tracking, or intermediary servers.  
   >  
   > Your prompts, responses, and file attachments are sent directly to your connected server, which forwards them to the AI model providers configured by your server administrator (such as OpenAI, Anthropic, or Google). Please consult your server administrator's privacy policy for details regarding model retention and logging."*

---

## 5. Validation Scorecard

| Check ID | Verification Criteria | Test Method | Pass / Fail |
|:---:|---|---|:---:|
| **VAL-01** | Clean install presents `AppEulaBottomSheet` before server entry is allowed. | Clean install test | [ ] |
| **VAL-02** | Accepting EULA persists in `SettingsDataStore`; app relaunch does not re-prompt. | Restart regression test | [ ] |
| **VAL-03** | Server with `modalAcceptance: true` gates unaccepted users onto `TermsScreen`. | Mock backend test | [ ] |
| **VAL-04** | Tapping "I Accept" on `TermsScreen` hits `POST /api/user/terms/accept` successfully. | HTTP trace inspection | [ ] |
| **VAL-05** | Settings displays working external web links for Switchboard legal pages. | Intent verification | [ ] |
| **VAL-06** | AI Data Handling dialog renders clear third-party forwarding disclosures. | UI inspection | [ ] |

---

## 6. Backend Contract & Wire Schema Details

### 6.1 Server Terms & Privacy Config Contract: `GET /api/config`
* **Route**: `api/server/routes/config.js`
* **Configuration Source**: `librechat.yaml` under `interface` block:
  ```yaml
  interface:
    privacyPolicy:
      externalUrl: 'https://example.com/privacy'
      openNewTab: true
    termsOfService:
      externalUrl: 'https://example.com/tos'
      openNewTab: true
      modalAcceptance: true
      modalTitle: 'Terms of Service'
      modalContent: '# Terms and Conditions...'
  ```
* **Wire Response (`200 OK`)**:
  ```json
  {
    "interface": {
      "privacyPolicy": {
        "externalUrl": "https://example.com/privacy",
        "openNewTab": true
      },
      "termsOfService": {
        "externalUrl": "https://example.com/tos",
        "openNewTab": true,
        "modalAcceptance": true,
        "modalTitle": "Terms of Service",
        "modalContent": "# Terms and Conditions..."
      }
    }
  }
  ```

### 6.2 Terms Query Contract: `GET /api/user/terms`
* **Route**: `api/server/routes/user.js` (`getTermsStatusController`).
* **Authentication**: Bearer JWT (`requireJwtAuth`).
* **Wire Response (`200 OK`)**:
  ```json
  {
    "termsAccepted": false,
    "termsAcceptedAt": null
  }
  ```
  *(Or `"termsAccepted": true, "termsAcceptedAt": "2026-09-26T12:00:00.000Z"` if already accepted).*

### 6.3 Terms Acceptance Contract: `POST /api/user/terms/accept`
* **Route**: `api/server/routes/user.js` (`acceptTermsController`).
* **Authentication**: Bearer JWT (`requireJwtAuth`).
* **Backend Database Action**:
  Executes `db.acceptTerms(req.user.id)` which updates the MongoDB user document:
  ```javascript
  {
    $set: {
      termsAccepted: true,
      termsAcceptedAt: new Date()
    }
  }
  ```
* **Wire Response (`200 OK`)**:
  ```json
  {
    "message": "Terms accepted successfully",
    "termsAcceptedAt": "2026-09-26T12:34:56.789Z"
  }
  ```


