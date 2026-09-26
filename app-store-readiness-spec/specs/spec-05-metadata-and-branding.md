# Spec 05: Store Metadata, Trademarks & Content Ratings

## 1. Context & Problem Statement
* **Apple Guideline 5.2.2 (Third-Party Intellectual Property)** & **Google Impersonation Policy**: Submitting an app named "LibreChat Mobile" or using official LibreChat branding without documented authorization from the trademark owners will lead to immediate rejection or takedown for trademark infringement.
* **Apple Guideline 2.3 (Accurate Metadata & Age Ratings)**: Classifying an open-ended AI assistant app that connects to general-purpose LLMs as 4+ or 9+ will trigger an automatic rejection. Reviewers require a **17+** (Apple) and **Teen / Mature** (Google Play) rating due to unpredictable AI output and web browsing capabilities.

## 2. Objective
Establish compliant naming, store descriptions, disclaimers, and questionnaire answers to ensure zero pushback regarding intellectual property, branding, or age suitability.

---

## 3. Metadata Specifications

### 3.1: App Naming & Subtitle Matrix

| Platform | Field | Compliant Copy | Notes |
|---|---|---|---|
| **Apple App Store** | App Name (30 chars) | `Switchboard: AI Chat Client` | Avoids trademark in primary name |
| **Apple App Store** | Subtitle (30 chars) | `Client for LibreChat Servers` | Permissible descriptive reference |
| **Google Play** | App Title (30 chars) | `Switchboard for LibreChat` | Permissible descriptive reference |
| **Google Play** | Short Description (80 chars) | `Native, private mobile client for your self-hosted LibreChat server.` | Clear, direct value prop |

### 3.2: Mandatory Store Description Disclaimer
The **first two lines** of the full description on both the App Store and Google Play must contain the following disclaimer:

```text
IMPORTANT DISCLAIMER:
Switchboard is an independent, third-party native client designed for user-configured, self-hosted LibreChat instances. Switchboard is NOT affiliated with, sponsored by, or endorsed by the official LibreChat open-source project or its maintainers.
```

### 3.3: Content Rating / Age Rating Questionnaire Answers

#### Apple App Store Connect Questionnaire:
* **Unrestricted Web Access**: **Yes** (Due to MCP web search tools and agent browsing capabilities).
* **Infrequent/Mild Profanity or Crude Humor**: **Yes** (Generative AI capability).
* **Infrequent/Mild Mature/Suggestive Themes**: **Yes** (Generative AI capability).
* **Generative AI Features**: **Yes**.
* **Resulting Age Rating**: **17+** (Required and standard for AI chatbot clients).

#### Google Play Console (IARC Questionnaire):
* **Category**: Utility / Productivity / Communication.
* **Can users interact or exchange content?**: **Yes**.
* **Does the app contain Generative AI features?**: **Yes**.
* **Are user reporting mechanisms present for AI output?**: **Yes** (Refer to Spec 02).
* **Does the app share location?**: **No**.
* **Resulting Rating**: **Teen (US) / PEGI 12 / 16+**.

---

## 4. App Store Review Information (The "Review Notes" Template)

Copy and paste the following into the **Review Notes** field in App Store Connect & Google Play Console:

```text
Dear App Review Team,

Thank you for reviewing Switchboard.

WHAT IS SWITCHBOARD:
Switchboard is a third-party native client (Android/iOS) for users who operate their own self-hosted LibreChat servers (an open-source AI chat platform). 

HOW TO TEST IMMEDIATELY (DEMO SERVER):
To ensure you can test all features without setting up a backend:
1. On the opening screen, tap "Try with Demo Server" (or enter: https://demo.switchboard.chat).
2. On the login screen, enter our test credentials:
   - Email: appreview@switchboard.chat
   - Password: [SECURE_PASSWORD]
3. The account is pre-populated with sample conversations showcasing:
   - Live AI streaming
   - Interactive Mermaid diagrams and code blocks
   - LaTeX mathematical typesetting
   - Offline reading mode

AI SAFETY & REPORTING:
- In accordance with guidelines, users can report any offensive AI response directly by tapping the three dots on any assistant message and choosing "Report Inappropriate Content".
- A comprehensive End User License Agreement (EULA) with zero tolerance for objectionable content is accepted by users before accessing chats.
- Terms of Service & Privacy Policy are accessible in-app via Settings -> Legal & About.

If you have any questions or require custom test scenarios, please contact us immediately via the developer email.
```

---

## 5. Validation Scorecard

| Check ID | Verification Criteria | Test Method | Pass / Fail |
|:---:|---|---|:---:|
| **VAL-01** | App title on store listings does not impersonate official project ("Switchboard for LibreChat"). | Text audit | [ ] |
| **VAL-02** | Description begins with the non-affiliation disclaimer. | Store listing preview | [ ] |
| **VAL-03** | Age rating questionnaire outputs 17+ on App Store Connect. | Questionnaire dry-run | [ ] |
| **VAL-04** | Reviewer credentials verified active against the live demo server. | Pre-submission login test | [ ] |
| **VAL-05** | Screenshots depict actual app running without promotional mockups hiding core UI. | Screenshot review | [ ] |
