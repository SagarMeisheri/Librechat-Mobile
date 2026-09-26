# Spec 04: Authentication & Social Login Strategy

## 1. Context & Problem Statement
* **Apple App Store Guideline 4.8 (Sign in with Apple)**: If an app offers third-party social logins (such as Google, Facebook, GitHub, or Discord) to authenticate users, it *must* also offer **Sign in with Apple** as an equivalent option.
* **Apple Guideline 5.1.1(v) & Google Play Account Deletion**: If an app allows account creation, it must allow users to initiate account deletion from within the app.

In LibreChat, authentication capabilities are dynamically served by the connected backend via `GET /api/config`:
* Some servers configure standard email/password only.
* Other servers configure Google, GitHub, Discord, or Apple OAuth.
* Account deletion (`DELETE /api/user/delete`) is guarded by `canDeleteAccount` middleware, which checks `process.env.ALLOW_ACCOUNT_DELETION`.

## 2. Objective
Guarantee full compliance with Apple Guideline 4.8 (Sign in with Apple) and Guideline 5.1.1(v) (In-App Account Deletion), protecting the app against review rejections while honoring LibreChat's backend middleware constraints.

---

## 3. Architecture & Strategic Safeguards

### Strategy A: Reviewer Demo Instance Configuration (Guideline 4.8 Exemption)
* The official review demo instance (`https://demo.switchboard.chat`) must strictly enforce:
  ```env
  ALLOW_SOCIAL_LOGIN=false
  ALLOW_SOCIAL_REGISTRATION=false
  ALLOW_EMAIL_LOGIN=true
  ALLOW_ACCOUNT_DELETION=true
  ```
* **Why**: Apple Guideline 4.8 explicitly states:
  > *"Sign in with Apple is not required if your app exclusively uses your own account setup and sign-in systems."*
  Because the review demo server presents *only* email/password fields, Guideline 4.8 is not triggered during store review.

### Strategy B: Client-Side Apple-First Ordering & HIG Styling
* On any self-hosted LibreChat server that enables social logins (`uiState.socialLoginEnabled == true`):
  * In `feature/auth/src/commonMain/kotlin/com/garfiec/librechat/feature/auth/screen/LoginScreen.kt`:
  * If `"apple"` is present in `uiState.socialLogins`, sort it to **index 0** (top of the social login list).
  * Render the Apple login button using standard Apple Human Interface Guidelines:
    * Solid black background (dark mode inverse white).
    * Official Apple monochrome logo.
    * Exact copy: *"Sign in with Apple"* (or *"Continue with Apple"*).
  * Render all other third-party social logins below the Apple button.

### Strategy C: Verified Native Account Deletion (Apple 5.1.1(v))
* LibreChat backend handles account deletion via:
  * Route: `DELETE /api/user/delete` (Express router in `api/server/routes/user.js`).
  * Middleware: `canDeleteAccount.js` checks `process.env.ALLOW_ACCOUNT_DELETION !== 'false'`. If disabled, returns `403 Forbidden`.
* Mobile flow in `AccountSettingsScreen.kt`:
  1. User navigates to `Settings` -> `Account Settings`.
  2. Taps "Delete Account".
  3. Dialog alerts: *"Are you sure you want to delete your account? This action is permanent and deletes all chats, presets, and stored files on this server."*
  4. If 2FA is active on the account, prompts for current 2FA code / backup code.
  5. Calls `SettingsViewModel.deleteAccount()`, which calls `UserApi.deleteUser()`.
  6. Backend deletes user record, files, and conversations.
  7. Client wipes local Room database, clears DataStores, and returns to `ServerUrlScreen`.

---

## 4. Requirements & Implementation Details

### R4.1: Social Button Sorting & Styling in `LoginScreen.kt`
```kotlin
val sortedSocialLogins = remember(uiState.socialLogins) {
    uiState.socialLogins.sortedWith { a, b ->
        when {
            a.equals("apple", ignoreCase = true) -> -1
            b.equals("apple", ignoreCase = true) -> 1
            else -> 0
        }
    }
}
```
* When `provider.equals("apple", ignoreCase = true)`, render a dedicated `AppleSignInButton` composable adhering to Apple branding guidelines.

### R4.2: Demo Account Setup Constraints
* The test account `appreview@switchboard.chat` on the demo server:
  * Must have **2FA disabled** so that reviewer account deletion test cases can execute cleanly without requesting an authenticator seed.
  * The demo server must have `ALLOW_ACCOUNT_DELETION=true` active in its environment.

---

## 5. Validation Scorecard

| Check ID | Verification Criteria | Test Method | Pass / Fail |
|:---:|---|---|:---:|
| **VAL-01** | Demo server presents zero third-party social buttons to reviewer. | Reviewer flow check | [ ] |
| **VAL-02** | Servers providing Apple OAuth render Apple button at top with HIG black/white styling. | Mock server config test | [ ] |
| **VAL-03** | "Delete Account" button is accessible in Account Settings. | UI inspection | [ ] |
| **VAL-04** | Confirming deletion hits `DELETE /api/user/delete` and receives `200 OK`. | Backend integration test | [ ] |
| **VAL-05** | Deletion wipes local Room account database and secure storage tokens completely. | Local storage audit | [ ] |
| **VAL-06** | Servers with `ALLOW_ACCOUNT_DELETION=false` gracefully display an error dialog instead of crashing. | Error handling test | [ ] |

---

## 6. Backend Contract & Middleware Details

### 6.1 Account Deletion Contract: `DELETE /api/user/delete`
* **Route Definition**: `api/server/routes/user.js`:
  ```javascript
  router.delete('/delete', requireJwtAuth, canDeleteAccount, configMiddleware, deleteUserController);
  ```
* **Middleware Logic (`api/server/middleware/canDeleteAccount.js`)**:
  ```javascript
  const canDeleteAccount = async (req, res, next) => {
    const { user } = req;
    const { ALLOW_ACCOUNT_DELETION = true } = process.env;
    if (isEnabled(ALLOW_ACCOUNT_DELETION)) {
      return next();
    }
    // Only users with SystemCapabilities.ACCESS_ADMIN can bypass
    if (hasAdminAccess) {
      return next();
    }
    return res.status(403).send({ message: 'You do not have permission to delete this account' });
  };
  ```
* **Wire Request**:
  * Headers: `Authorization: Bearer <JWT>`
  * Body *(Only if 2FA is active on user account)*:
    ```json
    {
      "token": "123456",
      "backupCode": null
    }
    ```
* **Wire Response**:
  * Success: HTTP `200 OK` `{ "message": "Account deleted successfully" }`
  * Blocked by Policy: HTTP `403 Forbidden` `{ "message": "You do not have permission to delete this account" }`
  * Invalid 2FA: HTTP `400 Bad Request` `{ "message": "Invalid 2FA code" }`

### 6.2 Apple Sign-In OAuth Contract
* **Route Definition**: `api/server/routes/oauth.js` and `api/server/socialLogins.js`:
  * Initiation: `GET /oauth/apple`
  * Callback: `POST /oauth/apple/callback`
  * Strategy: `passport-apple`
* **Discovery in `GET /api/config`**:
  * If `APPLE_CLIENT_ID` is set on the server, `appleLoginEnabled: true` is returned and `"apple"` is added to `payload.socialLogins`.
* **Mobile Custom Tab / WebAuth Contract**:
  * Mobile client launches `https://<server>/oauth/apple` via Chrome Custom Tab (Android) or `ASWebAuthenticationSession` (iOS).
  * Upon completion, server redirects with auth token cookies, which the native client extracts and persists.


