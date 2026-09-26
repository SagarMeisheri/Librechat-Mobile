# Spec 03: EULA, Terms of Service & Privacy Compliance

## 1. Context & Problem Statement
* **Apple Guideline 1.2 (User-Generated Content)**: Requires apps with open-ended or community-driven content to have an End User License Agreement (EULA) where users explicitly agree that objectionable content and abusive conduct are prohibited.
* **Apple Guideline 5.1.1 (Privacy & Data Collection)** & **Google Play User Data Policy**: Apps must:
  1. Provide a visible Privacy Policy link both within the app and on the store listing.
  2. Disclose whether user prompts or uploaded files are transmitted to third-party AI APIs (e.g., OpenAI, Google, Anthropic).
  3. Clearly state data retention and local storage policies.

Currently, `Librechat-Mobile` has no EULA acceptance screen on first run and lacks an accessible "Legal & Privacy" section in Settings.

## 2. Objective
Implement a compliant onboarding EULA consent dialog, embed legal links into the Settings hierarchy, and provide explicit disclosure regarding third-party AI data transmission.

---

## 3. Architecture & File Modifications

### Modified Files:
* `feature/settings/src/commonMain/kotlin/com/garfiec/librechat/feature/settings/screen/SettingsScreen.kt`
  * Add "Legal & About" section displaying Privacy Policy, Terms of Service, and Third-Party AI Data Disclosure.
* `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/screen/ServerUrlScreen.kt`
  * Integrate EULA acceptance prompt before advancing past server setup.
* `core/data/src/commonMain/kotlin/com/garfiec/librechat/core/data/preferences/UserPreferencesRepository.kt`
  * Persist `eulaAcceptedTimestamp: Long?` to avoid re-prompting returning users.
* `feature/settings/src/commonMain/composeResources/values/strings.xml`
  * Add strings: `title_legal_privacy`, `item_privacy_policy`, `item_terms_of_service`, `item_ai_disclosure`, `eula_zero_tolerance_notice`, `btn_accept_and_continue`.

### New Files:
* `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/components/EulaConsentBottomSheet.kt`
  * Modal sheet presented upon initial setup detailing zero tolerance for objectionable content and requiring explicit user agreement.

---

## 4. Requirements & Implementation Details

### R3.1: Mandatory First-Run EULA Consent
* Before completing initial server validation or login on a fresh install:
  * Present `EulaConsentBottomSheet`.
  * Headline: *"Welcome to Switchboard"*.
  * Body text:
    > *"Switchboard is an open-source client for connecting to self-hosted LibreChat servers. By using this application, you agree to comply with our Terms of Service and End User License Agreement (EULA).*  
    >  
    > **Zero Tolerance Policy:** You may not use this application to generate, distribute, or facilitate hate speech, sexually explicit material, harassment, copyright infringement, or illegal acts. Violation of these terms may result in account termination by your server administrator.*"
  * Provide clickable links: `[Terms of Service]` and `[Privacy Policy]`.
  * Primary Action Button: `[ Agree & Continue ]`. The user cannot proceed to chat until agreed.
  * Record acceptance locally in persistent preferences.

### R3.2: Settings "Legal & Privacy" Section
* In `SettingsScreen`, add a grouped section **"Legal & About"**:
  1. **Privacy Policy**: Opens hosted web policy URL (`https://switchboard.chat/privacy` or GitHub Pages).
  2. **Terms of Service & EULA**: Opens hosted web terms URL (`https://switchboard.chat/terms`).
  3. **AI Data Handling Disclosure**: Tapping opens an informational dialog:
     > *"**How your data is handled:**  
     > Switchboard connects directly to your self-hosted LibreChat server. The app itself operates no centralized telemetry or ad tracking. Your conversation prompts, responses, and file attachments are sent to the AI providers (e.g., OpenAI, Anthropic, Google) configured on your connected server. Please consult your server administrator's privacy policy for details on upstream provider logging and retention."*

### R3.3: Hosted Documentation
* Publish static, compliant markdown pages under `docs/legal/`:
  * `docs/legal/PRIVACY_POLICY.md` (Served via GitHub Pages).
  * `docs/legal/TERMS_OF_SERVICE.md` (Served via GitHub Pages).

---

## 5. Validation Scorecard

| Check ID | Verification Criteria | Test Method | Pass / Fail |
|:---:|---|---|:---:|
| **VAL-01** | First launch on a clean install presents the EULA consent sheet before chat access is granted. | Clean install test | [ ] |
| **VAL-02** | Accepting EULA persists state so returning users are not redundantly prompted. | App restart test | [ ] |
| **VAL-03** | "Legal & About" section in Settings navigates to working web links for Privacy Policy and Terms. | External link intent test | [ ] |
| **VAL-04** | AI Data Handling dialog clearly explains prompt forwarding to configured backend LLMs. | Dialog inspection | [ ] |
| **VAL-05** | Public privacy policy URL is reachable without authentication or paywalls. | Web browser check | [ ] |
