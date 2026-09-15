# Accessible MFA System — Implementation Plan & Work Division

A multifactor authentication system for visually impaired users, combining:

1. **Fingerprint authentication** (primary factor — "something you are") — replaces
   passwords entirely.
2. **Mapped-Index Voice OTP ("MIVO")** (secondary factor — "something you know") —
   a phone-call-based challenge-response scheme, fully audio, requiring no screen.

This document is a **planning document only**. It defines the architecture, the
MIVO mechanism, and how the work is split across 5 team members so each can
implement their part independently against agreed interfaces. No implementation
code is included here.

---

## 1. Authentication Model

| Factor | Type | Channel | Replaces |
|---|---|---|---|
| Fingerprint | Inherence ("something you are") | Device biometric sensor (WebAuthn / platform authenticator) | Password |
| MIVO Voice Challenge | Knowledge ("something you know") | Outbound phone call + keypad (DTMF) response | Traditional numeric OTP / SMS |

**Login requires both factors**, in sequence:
`Fingerprint verified → MIVO voice challenge triggered → MIVO response verified → session issued`

This ordering is deliberate: fingerprint identifies *which account* is
attempting login (and is always accessible without sight), and MIVO confirms
possession of the registered phone line plus recall of a private secret that
is never spoken or typed in the clear.

---

## 2. The MIVO Scheme — How It Works

This section is the shared spec everyone builds against, so it's described in
full even though no code is written here.

### 2.1 Enrollment (one-time, during account setup)

- The user selects, or is assigned, a personal **secret index set** of **5
  indices**, each a number in a fixed public range (e.g., 1–25).
  - Example secret index set: `{3, 8, 12, 19, 24}`
  - These 5 numbers are what the user memorizes — nothing else. They are
    never written down, displayed again, or transmitted in plaintext after
    enrollment.
- The server stores the index set encrypted/hashed, tied to the user's
  account and registered phone number.

### 2.2 Challenge generation (every login attempt)

1. The server generates a **mapping table** covering a fixed number of public
   indices (e.g., indices 1–10, drawn from the same 1–25 range), each
   randomly assigned a fresh 2-digit value:

   ```
   Index 1 → 23
   Index 2 → 43
   Index 5 → 32
   Index 7 → 91
   ...
   ```

   This mapping is generated fresh for every single call and discarded after
   use. The full table is read aloud — not just the indices relevant to the
   user — so a listener cannot tell which indices matter to this particular
   user.

2. The server randomly selects **3 of the user's 5 memorized indices** for
   this session (a different 3 each time). Only these 3 need to be answered.

### 2.3 The call

- An automated voice call is placed to the user's registered number.
- Text-to-speech reads the full mapping table aloud, at a deliberately
  measured pace, with the option to repeat (press any key).
- The user identifies, from memory, which 3 of the spoken indices are theirs,
  notes the value each maps to, and enters those values on the keypad
  (DTMF tones) in response — either as one concatenated code or one at a
  time, prompted individually.
- The server checks the entered values against the values it assigned to the
  correct 3 indices for that call.

### 2.4 Why this is secure and accessible

- **Nothing secret is ever spoken or typed.** The user's actual secret (which
  indices belong to them) is never transmitted — only a random mapped value
  that changes every call.
- **Shoulder-surfing / call interception is not useful to an attacker.** Even
  someone who hears the whole call and every keypress cannot reuse it, since
  next time the mapping is different and a different 3-of-5 subset is asked.
- **Fully audio and keypad-based**, so it requires no screen, no visual
  reading, and works on any phone.
- **Partial subset (3 of 5) per session** keeps each call short (accessible
  for cognitive load and call duration) while still requiring genuine recall
  of the full secret set over time, since which 3 are asked varies.

---

## 3. System Architecture (high level)

```
                 ┌────────────────────┐
                 │   Client (Web/App)  │
                 │  Accessible UI      │
                 └─────────┬──────────┘
                           │
                 ┌─────────▼──────────┐
                 │  Auth Orchestrator  │   (Member D)
                 │  (MFA flow control) │
                 └───┬─────────────┬──┘
                     │             │
          ┌──────────▼───┐   ┌─────▼─────────────┐
          │ Fingerprint   │   │ MIVO Voice OTP     │
          │ Module        │   │ Module             │
          │ (Member A)    │   │ (Member B)         │
          └───────────────┘   └───────┬────────────┘
                                       │
                              ┌────────▼─────────┐
                              │ Telephony/IVR      │
                              │ Provider           │
                              └────────────────────┘

          Cross-cutting: Accessible UX (Member C)
                         QA / Accessibility / DevOps (Member E)
```

---

## 4. Work Division — 5 Members

Each member owns one module end-to-end: design, build, unit-test, and deliver
an agreed interface contract that the others can integrate against. Interface
contracts should be finalized in Phase 1 so all members can work in parallel
from Phase 2 onward.

### Member A — Fingerprint (Biometric) Authentication

**Owns:** everything related to the primary "something you are" factor.

Responsibilities:
- Evaluate and choose the biometric integration path (WebAuthn/FIDO2
  platform authenticator for web; native BiometricPrompt on Android;
  LocalAuthentication/Touch ID/Face ID exclusions on iOS — fingerprint only,
  per project scope).
- Design the fingerprint **enrollment** flow (credential registration,
  public key storage handoff to Member D).
- Design the fingerprint **verification** flow (challenge issuance, signature
  verification handoff to Member D).
- Define fallback behavior if a device has no fingerprint sensor (e.g.,
  registered backup device, admin-assisted enrollment — document policy,
  do not design around passwords).
- Document anti-spoofing / liveness considerations and attestation level
  required (e.g., "none" vs "direct" attestation trade-offs).

Deliverables:
- Fingerprint enrollment interface contract (inputs/outputs, error states)
- Fingerprint verification interface contract
- Device/platform support matrix
- Security notes: attestation policy, key storage expectations

### Member B — MIVO Voice Challenge-Response OTP

**Owns:** the full MIVO mechanism described in Section 2.

Responsibilities:
- Finalize the index range, table size, subset size, and value length
  (defaults proposed: index range 1–25, table of 10 indices per call, 3-of-5
  subset, 2-digit mapped values — subject to entropy review).
- Design the random mapping generation algorithm and its entropy/security
  analysis (how many attempts an attacker would need, replay window, etc.).
- Design secret index-set enrollment and secure storage requirements
  (hand off actual storage/encryption to Member D).
- Select and integrate a telephony/IVR provider (e.g., Twilio Voice or
  equivalent) for outbound calls, TTS reading, and DTMF capture.
- Define retry, timeout, "repeat the table," and lockout policies.
- Define OTP/session expiry and replay protection rules.

Deliverables:
- MIVO algorithm specification (mapping generation, subset selection)
- IVR call script / TTS prompt specification
- Challenge-generation and response-verification interface contracts
- Entropy/security analysis document
- Telephony provider integration plan

### Member C — Accessible UX & Frontend Design

**Owns:** every user-facing interaction, designed audio-first.

Responsibilities:
- Design enrollment flows for both factors with screen-reader-first
  thinking (fingerprint capture prompts, secret index-set selection/
  memorization guidance).
- Design the login flow's on-screen and audio state announcements
  (aria-live equivalents, status text for each MFA step).
- Write the exact wording for all TTS/voice prompts used in the MIVO call,
  in collaboration with Member B.
- Define visual design for sighted users too (high contrast, large touch
  targets) without letting it compromise the audio-first flows.
- Produce an accessibility requirements checklist (target: WCAG 2.2 AA
  minimum) for the other members to build against.

Deliverables:
- Enrollment & login flow diagrams (state-by-state, not code)
- Full audio/voice script text for every screen and call prompt
- Accessibility requirements checklist (WCAG mapping)
- Annotated wireframes noting ARIA roles/labels required at each step

### Member D — Backend, Identity & Security Architecture

**Owns:** the orchestration layer that ties both factors together.

Responsibilities:
- Design the MFA orchestration flow (sequencing, session state between
  factor 1 success and factor 2 challenge, final session/token issuance).
- Design the account/identity data model (user record, registered phone
  number, fingerprint public key reference, encrypted index-set reference).
- Define encryption-at-rest and in-transit requirements for all stored
  secrets (fingerprint public keys, index sets).
- Define rate limiting, lockout thresholds, and anomaly detection rules
  (e.g., repeated failed MIVO attempts, geo-velocity on phone calls).
- Own the overall API gateway contract that Members A, B, and C integrate
  against.

Deliverables:
- System architecture diagram (data flow between all modules)
- Data model / schema design document
- MFA orchestration state-machine specification
- Security policy document (encryption, key management, lockout rules)
- Consolidated API contract reference for the whole system

### Member E — QA, Accessibility Testing & DevOps

**Owns:** verification that the system actually works for the target users,
and that it ships reliably.

Responsibilities:
- Write the test plan covering: fingerprint enrollment/verification,
  MIVO challenge/response (including edge cases like wrong subset answers,
  expired calls, repeated table requests), and full end-to-end MFA flow.
- Plan accessibility testing sessions with real screen readers (NVDA, JAWS,
  VoiceOver) and, ideally, with visually impaired test users.
- Define CI/CD pipeline stages, staging/production environment plan, and
  rollback strategy.
- Define monitoring/alerting for call success rate, fingerprint failure
  rate, OTP failure rate, and lockout incidents.
- Own the Definition of Done checklist that every module must satisfy before
  integration sign-off.

Deliverables:
- Test plan (unit/integration/security/accessibility scope)
- Accessibility test script and target device/screen-reader matrix
- CI/CD pipeline design
- Monitoring & alerting plan
- Definition of Done checklist per module

---

## 5. Timeline & Milestones (proposed)

| Phase | Weeks | Focus | Owners |
|---|---|---|---|
| 1. Design & interface contracts | 1–2 | Finalize architecture, MIVO parameters, API contracts, UX flows | All (joint) |
| 2. Independent module build | 3–6 | Each member builds their module against the agreed contracts | A, B, C, D (parallel) |
| 3. Integration | 7 | Wire fingerprint + MIVO + orchestration + UI together | A, B, D, C |
| 4. Accessibility & security testing | 8 | Screen reader testing, security review, load testing | E (leads), all support |
| 5. Deployment | 9 | Staged rollout, monitoring live | E (leads), D supports |

Weekly sync recommended throughout Phase 1 to lock interface contracts before
parallel work begins in Phase 2 — this is the main dependency risk point.

---

## 6. Cross-Cutting Requirements (apply to every member's work)

- No factor may require reading a screen or an image (WCAG 1.1.1 / 1.4.x).
- No secret is ever transmitted, displayed, or spoken in the clear after
  enrollment.
- Every status change during login must be announced via an accessible
  channel (screen reader live region on web/app; TTS on the call).
- All stored secrets (fingerprint public keys, index sets) are encrypted at
  rest; index sets are never stored or logged in plaintext.
- Every module exposes clear error states distinguishable by the
  orchestrator (e.g., "sensor unavailable" vs. "no match" vs. "timeout").

---

## 7. Glossary

- **MIVO** — Mapped-Index Voice OTP, the custom challenge-response scheme
  described in Section 2.
- **DTMF** — Dual-Tone Multi-Frequency, the keypad tones used to send digits
  during a phone call.
- **IVR** — Interactive Voice Response, the automated call system reading
  prompts and capturing keypad input.
- **WebAuthn/FIDO2** — Web standard for passwordless/biometric
  authentication using public-key cryptography.
- **Secret index set** — the 5 numbers a user memorizes at enrollment; their
  actual long-term secret in the MIVO scheme.
