# Spec 04: Authentication & Social Login Strategy

## 1. Context & Problem Statement
* **Apple App Store Guideline 4.8 (Sign in with Apple)**: If an app offers third-party social logins (such as Google, Facebook, GitHub, or Discord) to authenticate users, it *must* also offer **Sign in with Apple** as an equivalent option.
* **Apple Guideline 5.1.1(v) & Google Play Account Deletion**: If an app allows account creation, it must allow users to initiate account deletion from within the app.

In LibreChat, authentication capabilities are dynamically served by the connected backend via `/api/config`:
* Some servers configure standard username/password.
* Other servers configure Google, GitHub, or Discord OAuth.
If an Apple reviewer connects to a server that shows "Continue with Google" without "Continue with Apple", the app faces an immediate Guideline 4.8 rejection.

## 2. Objective
Ensure the authentication flow completely complies with Apple's Sign in with Apple requirements, safeguard the app during App Store Review, and verify native in-app account deletion.

---

## 3. Architecture & Strategic Plan

### Strategy A: Reviewer Demo Instance Isolation (Primary Safeguard)
* When configuring the public demonstration instance (`https://demo.switchboard.chat`) for App Store Review:
  * **Disable all social login providers** (`GOOGLE_CLIENT_ID`, `GITHUB_CLIENT_ID`, etc.) in the demo server's environment.
  * **Enable only Email & Password authentication**.
  * **Why**: Apple Guideline 4.8 explicitly states:
    > *"Sign in with Apple is not required if your app exclusively uses your own account setup and sign-in systems."*
  * Since the reviewer will exclusively interact with standard email/password fields on the demo server, Guideline 4.8 is not triggered during review.

### Strategy B: Self-Hosted Client Reviewer Defense Documentation
* In the App Review Notes, document the multi-tenant, self-hosted nature of the client:
  ```text
  AUTHENTICATION ARCHITECTURE DISCLOSURE (Guideline 4.8):
  Switchboard is an open client for connecting to user-owned, self-hosted LibreChat instances.
  The client does not operate a centralized authentication service.
  Login options shown on screen are dynamically configured by whichever server the user connects to.
  Our provided review demo server exclusively uses standard email/password authentication.
  The client codebase already includes full native support for "Sign in with Apple" whenever an individual server administrator configures Apple OAuth credentials on their backend.
  ```

### Strategy C: Verification of Account Deletion (Apple 5.1.1(v))
* Verify that `AccountSettingsScreen.kt` provides clear, unblocked access to delete the user account:
  * Navigating to `Settings` -> `Account` -> `Delete Account`.
  * Prompts confirmation and OTP (if 2FA is active).
  * Calls `UserApi.deleteUser()`, wipes local credentials from secure storage, and returns to `ServerUrlScreen`.

---

## 4. Requirements & Implementation Details

### R4.1: Social Button Ordering & Apple Presentation
* In `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/screen/LoginScreen.kt`:
  * Ensure that if the server provides `apple` in `uiState.oauthProviders`:
    * Render `Continue with Apple` at the top of the social login list with official Apple button styling (black background with white Apple logo on iOS).
    * Use the standard Apple button branding guidelines.

### R4.2: End-to-End Account Deletion Testing
* Validate that account deletion works seamlessly against the demo server:
  * When initiated, a confirmation dialog appears:
    *"Are you sure you want to delete your account? This action is permanent and deletes all chats, presets, and stored files on the server."*
  * Once confirmed, local caches (Room database, data store tokens) are wiped clean.

---

## 5. Validation Scorecard

| Check ID | Verification Criteria | Test Method | Pass / Fail |
|:---:|---|---|:---:|
| **VAL-01** | Demo server presents only Email & Password fields; zero social buttons visible to reviewer. | Reviewer flow check | [ ] |
| **VAL-02** | "Delete Account" button is visible and active under Account Settings. | Settings navigation | [ ] |
| **VAL-03** | Initiating account deletion shows unambiguous warning dialog. | UI test | [ ] |
| **VAL-04** | Confirming deletion wipes local tenant storage and routes user back to server setup. | End-to-end integration test | [ ] |
| **VAL-05** | If a server provides Apple OAuth, the button adheres to Human Interface Guidelines. | Mocked server config test | [ ] |
