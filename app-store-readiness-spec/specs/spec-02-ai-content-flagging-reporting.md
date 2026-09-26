# Spec 02: AI Content Flagging & Abuse Reporting

## 1. Context & Problem Statement
* **Google Play Store (Generative AI Policy)**: Apps featuring generative AI features *must* include an in-app reporting or flagging system for offensive or inappropriate AI-generated content directly in the chat interface. Failure to provide this results in immediate rejection.
* **Apple App Store (Guideline 1.2 User-Generated Content)**: AI outputs are treated as synthetic user content. Apps must provide a clear mechanism to report offensive responses and a process to address reported concerns.

Currently, `FeedbackTag.kt` only contains qualitative assessment tags (`inaccurate`, `not_helpful`, `bad_style`). There is no mechanism for reporting policy-violating, abusive, or dangerous AI outputs.

## 2. Objective
Implement a clear in-app reporting flow allowing users to flag offensive or harmful AI responses directly from the message bubble action menu, satisfying Google Play and Apple App Store mandates.

---

## 3. Architecture & File Modifications

### Modified Files:
* `core/model/src/commonMain/kotlin/com/garfiec/librechat/core/model/FeedbackTag.kt`
  * Add safety tags: `OFFENSIVE_CONTENT`, `HARMFUL_DANGEROUS`, `SEXUAL_CONTENT`, `HATE_SPEECH`.
* `feature/chat/src/commonMain/kotlin/com/garfiec/librechat/feature/chat/components/MessageContentAndActions.kt`
  * Add a dedicated "Report Response" action item in the message overflow menu.
* `feature/chat/src/commonMain/kotlin/com/garfiec/librechat/feature/chat/viewmodel/ChatViewModel.kt`
  * Add `reportMessage(messageId: String, reason: ReportReason, details: String?)` handler.
* `feature/chat/src/commonMain/composeResources/values/strings.xml`
  * Add localization strings: `action_report_response`, `report_dialog_title`, `report_reason_offensive`, `report_reason_harmful`, `report_reason_sexual`, `report_reason_hate`, `report_submitted_toast`.

### New Files:
* `feature/chat/src/commonMain/kotlin/com/garfiec/librechat/feature/chat/components/ReportMessageDialog.kt`
  * Modal dialog displaying report categories, optional explanation field, and submit/cancel buttons.

---

## 4. Requirements & Implementation Details

### R2.1: Report Categories
The reporting dialog must offer predefined categories aligned with Google and Apple safety taxonomies:
1. **Hate Speech & Discrimination**: Attacks based on race, religion, gender, or sexual orientation.
2. **Harmful, Dangerous or Illegal**: Encouragement of violence, self-harm, cyberattacks, or weapon creation.
3. **Sexually Explicit**: Unsolicited pornographic or intimate imagery/prose.
4. **Harassment & Defamation**: Abusive or threatening output.
5. **Other / Inappropriate**: General objectionable material.

### R2.2: Reporting Flow
1. User taps the overflow menu (three dots) on any assistant response bubble.
2. User selects **"Report Inappropriate Content"** (accompanied by a flag icon).
3. The `ReportMessageDialog` appears.
4. User selects a category and may optionally supply up to 500 characters of additional context.
5. User taps **"Submit Report"**.
6. The client sends a payload containing `messageId`, `conversationId`, `reason`, and `details` to the backend feedback route (or a dedicated reporting sink).
7. The dialog closes and displays a confirmation notification:
   *"Thank you for reporting. This response has been submitted for moderation review."*

### R2.3: Safe Fallback for Self-Hosted Servers
* On standard LibreChat servers, the feedback route `/api/messages/feedback` or the app's diagnostic logger records the flagged message.
* If the self-hosted server returns an error (or lacks an explicit moderation route), the app logs the report locally and safely informs the user without throwing a crash or jarring network error.

---

## 5. Validation Scorecard

| Check ID | Verification Criteria | Test Method | Pass / Fail |
|:---:|---|---|:---:|
| **VAL-01** | "Report Inappropriate Content" appears in the action menu for assistant messages. | Menu inspection | [ ] |
| **VAL-02** | Report dialog renders all mandatory categories with accessible radio selections. | UI inspection | [ ] |
| **VAL-03** | Submitting report emits payload to repository without blocking the active chat thread. | Unit / Integration Test | [ ] |
| **VAL-04** | Confirmation message appears upon submission. | UI verification | [ ] |
| **VAL-05** | Dialogue handles network drops gracefully without crashing or losing chat state. | Network simulator | [ ] |
