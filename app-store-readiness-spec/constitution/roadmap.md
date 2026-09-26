# Constitution: Store Readiness Roadmap

This living roadmap organizes the required tasks into sequential phases following the Spec-Driven Development lifecycle.

```mermaid
gantt
    title Switchboard Store Readiness Roadmap
    dateFormat  YYYY-MM-DD
    section Phase 1: Reviewer Access
    Public Demo Server Setup           :p1_1, 2026-09-27, 2d
    Quick-Connect Demo UI              :p1_2, after p1_1, 2d
    section Phase 2: Content Safety
    Abuse Reporting Dialog & API       :p2_1, after p1_2, 3d
    Model Safety & System Prompts      :p2_2, after p1_2, 2d
    section Phase 3: Legal & Privacy
    First-Launch EULA Acceptance       :p3_1, after p2_1, 2d
    In-App Legal & AI Disclosures      :p3_2, after p3_1, 2d
    Hosted Privacy & EULA Pages        :p3_3, after p3_1, 1d
    section Phase 4: Submission
    Metadata, Screenshots & Age Rating :p4_1, after p3_2, 2d
    Reviewer Account & IPv6 Audit      :p4_2, after p4_1, 1d
    Submission & Monitoring            :p4_3, after p4_2, 3d
```

---

## Phase 0: Baseline Audit & Gap Resolution (Completed)
- [x] Comprehensive review of Apple Review Guidelines & Google Play GenAI policies (September 2026).
- [x] Full codebase audit of `Librechat-Mobile` (`Switchboard`).
- [x] Discovery of blocking failure points (App Completeness, AI Flagging, Missing EULA, AI Data Disclosures).

## Phase 1: Reviewer Access & Zero-Friction Demo Mode
*Target Spec*: [`spec-01-reviewer-access-and-demo-mode.md`](../specs/spec-01-reviewer-access-and-demo-mode.md)
- [ ] Deploy stable public HTTPS LibreChat instance with pre-seeded sample conversations.
- [ ] Add "Explore with Demo Server" 1-tap connection option on `ServerUrlScreen`.
- [ ] Seed reviewer test account credentials and prepare Reviewer Instructions document.

## Phase 2: AI Content Flagging & In-App Reporting
*Target Spec*: [`spec-02-ai-content-flagging-reporting.md`](../specs/spec-02-ai-content-flagging-reporting.md)
- [ ] Expand `FeedbackTag.kt` with safety-specific tags (`offensive_content`, `harmful_unsafe`).
- [ ] Add explicit "Report AI Response" action item in `MessageContentAndActions.kt`.
- [ ] Implement `ReportContentDialog` with abuse categories and confirmation feedback.
- [ ] Configure server-side endpoint or reporting webhook sink.

## Phase 3: Legal Disclosures, EULA & Privacy Flows
*Target Spec*: [`spec-03-eula-terms-privacy-compliance.md`](../specs/spec-03-eula-terms-privacy-compliance.md)
- [ ] Implement initial onboarding EULA Acceptance BottomSheet with zero-tolerance policy.
- [ ] Add "Legal & Privacy" section in `SettingsScreen` / `AccountSettingsScreen`.
- [ ] Add third-party AI model disclosure banner explaining data routing.
- [ ] Host public, accessible Privacy Policy and Terms of Use web pages.

## Phase 4: Authentication Hardening & Apple Sign-In Guard
*Target Spec*: [`spec-04-auth-and-social-login-guard.md`](../specs/spec-04-auth-and-social-login-guard.md)
- [ ] Configure demo server with email/password authentication only (bypassing Guideline 4.8).
- [ ] Verify native account deletion flow in `AccountSettingsScreen` end-to-end.
- [ ] Document architecture explanation for review notes regarding self-hosted OAuth variability.

## Phase 5: Store Metadata, Trademarks & Submission
*Target Spec*: [`spec-05-metadata-and-branding.md`](../specs/spec-05-metadata-and-branding.md)
- [ ] Finalize App Store Connect & Google Play Console listings as "Switchboard - Client for LibreChat".
- [ ] Add mandatory disclaimer prominently at top of store descriptions.
- [ ] Complete Content Rating / IARC questionnaires targeting 17+ (Apple) and Teen / Mature (Google).
- [ ] Verify IPv6 compatibility on review endpoints.
