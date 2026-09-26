# Spec 02: AI Content Flagging & Abuse Reporting

## 1. Context & Problem Statement
* **Google Play Store (Generative AI Policy)**: Apps featuring generative AI features *must* include an in-app reporting or flagging system for offensive or inappropriate AI-generated content directly in the chat interface.
* **Apple App Store (Guideline 1.2 User-Generated Content)**: AI outputs are treated as synthetic user content. Apps must provide a clear mechanism to report offensive responses and a process to address reported concerns.

### Backend Parity & Mirror Integrity Constraint:
* In `Librechat-Mobile`, `FeedbackTag.kt` is mirrored directly from LibreChat's TypeScript package (`packages/data-provider/src/feedback.ts`) and guarded by `scripts/mirrors.json` and `scripts/check-mirrors.py`.
* LibreChat backend validates feedback strictly via `feedbackSchema.safeParse()`:
  ```typescript
  export const feedbackSchema = z.object({
    rating: feedbackRatingSchema, // 'thumbsUp' | 'thumbsDown'
    tag: feedbackTagKeySchema,    // FEEDBACK_REASON_KEYS enum
    text: z.string().max(1024).optional(),
  });
  ```
* **Critical Requirement**: Mutating `FeedbackTag.kt` with custom safety enum keys will break CI mirror validation and trigger `HTTP 400 Bad Request: Invalid feedback` from the LibreChat backend. All abuse reporting must be modeled as a client-side domain layer that safely encodes into the upstream feedback wire contract.

## 2. Objective
Implement a compliant, user-facing reporting dialog for offensive AI responses that communicates over LibreChat's standard `POST /api/messages/{conversationId}/{messageId}/feedback` endpoint without breaking upstream schema parity.

---

## 3. Architecture & File Modifications

### New Files:
* `feature/chat/src/commonMain/kotlin/com/garfiec/librechat/feature/chat/model/ReportReason.kt`
  * UI enum declaring store-compliant abuse categories: `HATE_SPEECH`, `HARMFUL_DANGEROUS`, `SEXUAL_CONTENT`, `HARASSMENT`, `OTHER`.
* `feature/chat/src/commonMain/kotlin/com/garfiec/librechat/feature/chat/components/ReportMessageDialog.kt`
  * Modal dialog offering category selection, an optional 500-character detail field, and submit/cancel buttons.

### Modified Files:
* `feature/chat/src/commonMain/kotlin/com/garfiec/librechat/feature/chat/components/MessageContentAndActions.kt`
  * Add a dedicated "Report Inappropriate Content" (flag icon) item to the assistant message action menu.
* `feature/chat/src/commonMain/kotlin/com/garfiec/librechat/feature/chat/viewmodel/ChatViewModel.kt`
  * Add `submitAbuseReport(messageId: String, reason: ReportReason, details: String?)` method.
* `core/data/src/commonMain/kotlin/com/garfiec/librechat/core/data/repository/MessageRepository.kt`
  * Add `reportMessage(conversationId: String, messageId: String, reason: ReportReason, details: String?)`.
* `feature/chat/src/commonMain/composeResources/values/strings.xml`
  * Add localization strings: `action_report_content`, `report_dialog_title`, `report_reason_hate`, `report_reason_harmful`, `report_reason_sexual`, `report_reason_harassment`, `report_reason_other`, `report_submitted_toast`.

---

## 4. Requirements & Implementation Details

### R2.1: Report Categories (Taxonomy)
Aligned with Google Play Generative AI & Apple Safety guidelines:
1. **Hate Speech & Discrimination**: Discrimination or slurs targeting protected groups.
2. **Harmful, Dangerous or Illegal**: Self-harm, violence, weapon instructions, or cyberattacks.
3. **Sexually Explicit**: Unsolicited graphic nudity or sexual prose.
4. **Harassment & Defamation**: Abusive or threatening output.
5. **Other Policy Violation**: Misinformation or general objectionable content.

### R2.2: Backend Wire Protocol & Payload Encoding
When a user flags a message, the client maps the report to LibreChat's native `MinimalFeedback` structure:
* **Route**: `POST /api/messages/{conversationId}/{messageId}/feedback`
* **Wire Body**:
  ```json
  {
    "feedback": {
      "rating": "thumbsDown",
      "tag": "other",
      "text": "[SAFETY_REPORT: <REPORT_REASON_NAME>] <Optional user details truncated to 900 chars>"
    }
  }
  ```
* **Backend Processing**:
  1. LibreChat's `feedbackSchema` validates `rating` ('thumbsDown') and `tag` ('other') successfully.
  2. Updates the message's `feedback` field in MongoDB.
  3. If Langfuse is configured on the backend, emits `createFeedbackScore`, capturing the safety incident in telemetry.

### R2.3: Safe Fallback & Non-Blocking Resilience
* Even if the server returns an error (or a self-hosted instance is offline), the client:
  1. Logs the incident to the local diagnostic logger (`AppLogger.w("Abuse report failed to reach server: ...")`).
  2. Displays the confirmation snackbar:
     *"Thank you for reporting. This response has been submitted for moderation review."*
  3. Never crashes or interrupts the active chat thread.

---

## 5. Validation Scorecard

| Check ID | Verification Criteria | Test Method | Pass / Fail |
|:---:|---|---|:---:|
| **VAL-01** | `python3 scripts/check-mirrors.py` exits `0` with zero schema drift. | CI / CLI execution | [ ] |
| **VAL-02** | "Report Inappropriate Content" menu item appears exclusively on assistant messages. | Visual UI audit | [ ] |
| **VAL-03** | Submitting report sends `tag: "other"` and `text: "[SAFETY_REPORT: ...]"` to backend. | HTTP wire mock / Network trace | [ ] |
| **VAL-04** | Backend responds with `200 OK` and updates message feedback without schema errors. | Integration test against LibreChat | [ ] |
| **VAL-05** | Simulating `500 Internal Error` from backend still closes dialog gracefully with toast. | Failure injection test | [ ] |
| **VAL-06** | Text input enforces 500 character maximum and handles special Unicode characters. | Unit test | [ ] |

---

## 6. Backend Contract & Wire Schema Details

### 6.1 Route Definition & Middleware Chain
* **Endpoint**: `POST /api/messages/:conversationId/:messageId/feedback`
* **File**: `api/server/routes/messages.js` (lines 694–748)
* **Middleware Chain**:
  1. `requireJwtAuth`: Validates caller JWT token and sets `req.user`.
  2. Route handler: Checks user ownership of `conversationId` and validates payload using Zod.

### 6.2 Zod Validation Contract (`feedbackSchema`)
Defined in `packages/data-provider/src/feedback.ts`:
```typescript
export const feedbackSchema = z
  .object({
    rating: z.enum(['thumbsUp', 'thumbsDown']),
    tag: z.enum([
      'not_matched', 'inaccurate', 'bad_style', 'missing_image',
      'unjustified_refusal', 'not_helpful', 'other',
      'accurate_reliable', 'creative_solution', 'clear_well_written', 'attention_to_detail'
    ]),
    text: z.string().max(1024).optional(),
  })
  .refine(
    ({ rating, tag }) =>
      FEEDBACK_TAGS.some(
        (feedbackTag) => feedbackTag.key === tag && feedbackTag.direction === rating,
      ),
    { message: 'Feedback tag does not match rating', path: ['tag'] },
  );
```
* **Failure Response (`400 Bad Request`)**:
  ```json
  { "error": "Invalid feedback" }
  ```
  *Occurs if the mobile app attempts to submit an unmapped safety tag directly.*
* **Success Response (`200 OK`)**:
  ```json
  {
    "messageId": "msg-uuid-1234",
    "conversationId": "convo-uuid-5678",
    "feedback": {
      "rating": "thumbsDown",
      "tag": "other",
      "text": "[SAFETY_REPORT: HARMFUL_DANGEROUS] User description"
    }
  }
  ```

### 6.3 MongoDB Storage & Downstream Telemetry
* **Mongoose Schema (`packages/data-schemas/src/schema/message.ts`)**:
  ```typescript
  feedback: {
    type: {
      rating: { type: String, enum: ['thumbsUp', 'thumbsDown'], required: true },
      tag: { type: mongoose.Schema.Types.Mixed, required: false },
      text: { type: String, required: false },
    },
    default: undefined,
  }
  ```
* **Langfuse Telemetry Hook**: In `api/server/routes/messages.js`, the route automatically invokes `createFeedbackScore()` asynchronously:
  ```javascript
  createFeedbackScore({
    conversationId,
    user: req.user.id,
    messageId,
    feedback: updatedMessage.feedback,
  }).catch((err) => logger.error('[langfuse] feedback score failed:', err));
  ```
  Safety reports formatted with `[SAFETY_REPORT: ...]` are recorded in Langfuse trace scoring for audit and administrator review.


