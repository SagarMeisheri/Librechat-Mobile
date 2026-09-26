# Constitution: Mission & Non-Negotiable Boundaries

## 1. Project Vision
**Switchboard** is a native, privacy-respecting, high-performance mobile client for self-hosted [LibreChat](https://www.librechat.ai/) servers on Android and iOS. It delivers a fluid native experience (offline support, adaptive multi-pane layout, home-screen widgets, rich artifact rendering) without requiring any server-side alterations.

## 2. Target Audience
1. **Self-Hosters & Power Users**: Developers, tech-enthusiasts, and privacy-conscious users running LibreChat on home labs, cloud VPS, or local hardware.
2. **Enterprise & Team Members**: Organizations utilizing an internal LibreChat deployment wanting secure, authenticated mobile access for their employees.
3. **App Store Reviewers (Apple & Google)**: Gatekeepers evaluating the app on isolated, strict test environments who demand zero setup friction, absolute safety adherence, and verifiable compliance.

## 3. The Store Readiness Objective
Achieve formal approval and unblocked distribution on:
* **Apple App Store** (iOS / iPadOS)
* **Google Play Store** (Android)

## 4. Non-Negotiable Store Compliance Boundaries

### Boundary 1: Zero Reviewer Friction (Guideline 2.1)
* The app must never leave a reviewer stranded on an empty server URL screen.
* Reviewers must be able to explore the full native UI (chat, streaming, artifacts, code blocks, settings) within 1 tap using a managed demo instance with pre-configured single-tap reviewer login.

### Boundary 2: AI Content Moderation & Abuse Reporting (Apple 1.2 & Google GenAI Policy)
* All AI outputs must be treated as synthetic user content.
* Users must possess a prominent, direct mechanism to report/flag abusive, offensive, or hazardous AI responses directly from the message bubble without leaving the application.

### Boundary 3: Dual-Tier Legal & Privacy Disclosures (Apple 1.2, 5.1.1 & Google User Data)
* An End User License Agreement (EULA) for the Switchboard client with explicit zero tolerance for objectionable content must be acknowledged before active interaction.
* Server-level Terms of Service and Privacy Policies configured in LibreChat must be respected and gated via native acceptance flows.
* Clear disclosures must be provided regarding prompt forwarding to third-party AI models (OpenAI, Anthropic, Google, etc.).
* Accessible links to both Privacy Policy and Terms of Service must exist in-app and on the public web.

### Boundary 4: Strict Trademark Separation (Apple 4.1, 5.2.2 & Google Impersonation)
* The app is **Switchboard**, a client for LibreChat.
* It must never present itself as the official LibreChat app or mislead users regarding sponsorship or endorsement from the upstream open-source project.

### Boundary 5: Age Rating Accuracy (Apple 2.3 & Google IARC)
* Open-ended AI chat clients connecting to general-purpose LLMs must declare a **17+** rating on iOS and **Teen / Mature** on Android. Downgrading to 4+ or 9+ is strictly prohibited.
