# PII Protection — Baseline Definition

> Owner: KPR
> Scope: PII Protection — 6 subcomponents (PII.1–PII.6)
> Module mapping: Personal Identity Protection
> Current completion: ~0% — 0 of 48 capabilities exist, 3 partial, 45 missing
> Source: GTM_SUBCOMPONENT_CAPABILITY_MAP.md

---

## What This Is

The shield. No personal data moves, rests, or transforms without governance. This is a fully greenfield compliance layer covering the core PII lifecycle: encryption & protection → access control → consent → subject rights → audit → data retention.

These 6 subcomponents represent the minimum viable compliance required for GDPR, CCPA, and SOC 2. Each exists because removing it creates an immediate regulatory gap.

**Why this exists before revenue:** Enterprise buyers run procurement checklists. Without PII protection, the answer to "How do you protect personal data?" is "We don't." That ends the conversation. SOC 2 Type II, GDPR, and ISO 27001 all require these controls as prerequisites.

## Current State

| What Exists | What's Missing |
|---|---|
| HTTPS in transit (partial TLS) | Field-level encryption (AES-256 per PII column) |
| Basic role system (partial RBAC) | KMS integration, crypto-shredding, dynamic masking |
| Basic consent collection (not granular) | Granular consent, immutable records, enforcement API |
| | DSAR portal (access, erasure, rectification) |
| | Immutable audit logs, ROPA, compliance reports |
| | Retention rules, automated enforcement, legal holds |

## Baseline Definition

**The line is:** All PII is field-level encrypted with centralized key management, access is controlled by role with MFA, consent is granular and enforceable, data subject rights are fulfilled within SLA, every PII access is audit-logged immutably, and retention is automated with legal hold support. Anything beyond this is refinement.

---

## Baseline Components

### PII.1 Encryption & Data Protection — ~14.5 dev days

**What this is:** Protects every PII field at rest and in transit. Field-level encryption goes beyond disk-level — even a database admin dump is unreadable without the key. Dynamic masking ensures analysts see only what their role permits. Key management centralizes the encryption lifecycle so keys can be rotated and destroyed (crypto-shredding) to render data permanently unrecoverable.

#### Baseline deliverables

**1. Field-level encryption (AES-256)**
Identify all PII columns. Encrypt each with AES-256-GCM using a per-user Data Encryption Key (DEK) wrapped by a Key Encryption Key (KEK) from KMS. Why baseline: disk-level encryption protects against stolen hard drives, not SQL injection or admin access. Field-level means a database dump yields ciphertext.

**2. KMS integration**
Centralized key lifecycle via GCP Cloud KMS or AWS KMS: generation, storage, rotation, revocation, destruction. Every encrypt/decrypt goes through KMS. Why baseline: without it, keys end up in env vars or code. Also the foundation for crypto-shredding.

**3. TLS 1.3 enforcement end-to-end**
Enforce TLS 1.3 (minimum 1.2) on all connections: client→Cloud Run, Cloud Run→Supabase, Cloud Run→Redis, Cloud Run→external APIs. Why baseline: SOC 2 and GDPR both require encryption in transit. Internal connections may not be encrypted today.

**4. Dynamic data masking**
Mask PII fields at query time based on role. Support agents see `j***@example.com`. Admins see full values. Why baseline: the most common breach is an insider seeing data they shouldn't. Masking is the cheapest control.

**5. Crypto-shredding capability**
On right-to-erasure: destroy the user's DEK. All their encrypted data becomes permanently unreadable. Why baseline: true deletion across distributed systems (backups, replicas, caches) is nearly impossible. Crypto-shredding is the only reliable erasure. GDPR Art. 17.

| Refinement (deferred) | Why |
|---|---|
| Tokenization vault | Stripe handles card tokenization; field encryption covers platform PII |
| Static data masking for dev/test | Synthetic data in dev environments instead |
| Key rotation automation | Manual rotation acceptable at baseline |

**Dependencies:** GCP Cloud KMS (infra), PII column inventory (data audit), Auth Gateway roles (Platform 1.5)

**Certifications:** PCI DSS (Req 3, 4), HIPAA (§164.312), ISO 27001 (A.10), SOC 2 (CC6.1, CC6.7), GDPR (Art. 32)

---

### PII.2 Access Control — ~9.5 dev days

**What this is:** Controls who can access what PII and under what conditions. The most common breach isn't a hacker — it's an authorized person accessing data they shouldn't see. RBAC, MFA, least privilege, and database-level security are the basic controls every privacy regulation mandates.

#### Baseline deliverables

**1. Granular RBAC for PII**
Extend basic roles with PII permissions. Roles: `admin` (full PII), `support` (masked PII), `analyst` (anonymized only), `developer` (no PII in non-prod). Permission matrix: role → table → columns → access level. Why baseline: without this, everyone with DB access sees all PII. Fails every audit.

**2. MFA for PII access**
Any PII read/write requires MFA. Middleware check: if request touches PII-flagged table and session lacks recent MFA → 403 + challenge redirect. Why baseline: HIPAA, SOC 2, ISO 27001 all require MFA for sensitive data. A stolen JWT without MFA = full exfiltration.

**3. Row-level and column-level database security**
Extend existing RLS with column-level views per role. `support_view`: `name, masked_email, masked_phone`. `analyst_view`: `anonymized_id, region, plan_type`. Why baseline: application-level masking can be bypassed with direct DB access. Database-level cannot.

**4. Session timeout for PII access**
Auto-expire after 15 minutes of inactivity. Re-auth required. Why baseline: unattended session = open door to PII.

| Refinement (deferred) | Why |
|---|---|
| Access request/approval workflows | Manual granting acceptable with small team |
| Periodic access reviews (quarterly) | Document as manual process initially |
| Separation of duties | Relevant at larger team size |

**Dependencies:** Auth Gateway (Platform 1.5), PII column inventory (PII.1), MFA provider (infra)

**Certifications:** SOC 2 (CC6.1–6.3), ISO 27001 (A.9), HIPAA (§164.312), GDPR (Art. 25, 32), PCI DSS (Req 7, 8)

---

### PII.3 Consent Management — ~7.5 dev days

**What this is:** Processing personal data without valid consent is illegal under GDPR and CCPA. Consent must be granular (per purpose), recorded immutably, and enforced in real-time. When a user withdraws consent, processing must stop within minutes. This isn't a preference setting — it's a legal obligation.

#### Baseline deliverables

**1. Granular consent per purpose**
Signup flow with per-purpose toggles: `[Essential ✓] [Analytics ○] [Marketing ○] [Personalization ○] [Third-party ○]`. Each is an independent record. Why baseline: GDPR Art. 6-7 requires specific, informed, per-purpose consent. A single checkbox is not legally valid.

**2. Immutable consent records**
`consent_records` table: `{user_id, purpose, granted, version, timestamp, ip, user_agent}`. Write-once with hash-chain integrity. Why baseline: during audit, must prove when consent was granted, for what, under which policy version.

**3. Consent enforcement API**
`consent_api.check(user_id, purpose)` → true/false. Every downstream system calls before processing. On withdrawal → Event Bus `consent.withdrawn:{purpose}` → subscribers stop within 5 min. Why baseline: collecting consent without enforcing it is provable negligence.

**4. One-click withdrawal**
Toggle any non-essential purpose OFF → record written → event published → enforcement within 5 min. Why baseline: GDPR Art. 7(3): "as easy to withdraw as to give."

| Refinement (deferred) | Why |
|---|---|
| Full preference center (UI polish) | Minimal toggles sufficient |
| Consent versioning with re-consent | Build when first policy change happens |
| Cookie consent | Relevant for web; mobile doesn't use cookies the same way |

**Dependencies:** Event Bus (Platform 1.2), Identity Store (Platform 8.1), Data Store (Platform 1.8)

**Certifications:** GDPR (Art. 6, 7, 8), CCPA/CPRA (Right to Opt-Out), ISO 27701 (A.7.2.3–4)

---

### PII.4 Data Subject Rights (DSAR) — ~15 dev days

**What this is:** GDPR mandates response within 30 days, CCPA within 45. Data subjects can request access, correction, or deletion of their data. The platform must find all PII across every system, verify identity, execute the action, and document the response. Missed deadlines are regulatory fines: up to 4% of global revenue.

#### Baseline deliverables

**1. DSAR intake portal** — Web form + in-app submission. Auto-creates case with SLA clock.

**2. Identity verification** — Email + SMS OTP before any data release. Releasing PII to the wrong person is itself a breach.

**3. Right to Access (data export)** — Compile all PII from every system → JSON + PDF → secure download link.

**4. Right to Erasure (deletion orchestration)** — Delete/crypto-shred across primary DB, analytics, logs, caches, backups, third-party systems. Verification scan confirms nothing missed.

**5. Right to Rectification** — Correction workflow with before/after audit log.

**6. SLA tracking with escalation** — Day 20 yellow → Day 25 orange → Day 28 red → Day 30 legal.

| Refinement (deferred) | Why |
|---|---|
| Right to Restrict Processing | Rare; handle manually (flag account) |
| Automated evidence packages | Manual assembly acceptable at low volume |
| API-based intake | Web form covers all use cases |

**Dependencies:** PII.1 (column inventory, crypto-shredding), PII.5 (audit logging), Data Store (Platform 1.8), Notification Service (Platform 1.10)

**Certifications:** GDPR (Art. 15–22), CCPA/CPRA (§1798.100–125), ISO 27701 (A.7.3), LGPD (Art. 18)

---

### PII.5 Audit Logging & Compliance — ~11.5 dev days

**What this is:** Auditors trust evidence, not promises. Every PII access must be logged immutably — who, what, when, where, which data, why. Without audit logging, you can build every other PII control perfectly and still fail the audit. SOC 2, GDPR, ISO 27001, and HIPAA all require tamper-evident logs.

#### Baseline deliverables

**1. Immutable audit log repository** — Write-once `pii_audit_log` with SHA-256 hash chains. Tampering breaks the chain. SOC 2 (CC7.2) specifically requires "tamper-evident."

**2. Structured 6W logging** — `{who, what, when, where, which_data, why}`. The "why" field ties access to business justification. Auditors want to know access was justified.

**3. Records of Processing Activities (ROPA)** — GDPR Art. 30 register: processing type, purpose, legal basis, retention, recipients. Mandatory artifact. Not having it = automatic finding.

**4. Pre-built compliance reports** — SOC 2 (access patterns, controls) + GDPR (consent, DSAR, ROPA). Generated on demand as PDF. Auditor needs evidence packages, not raw queries.

**5. Configurable log retention** — Per regulation: GDPR (3yr), SOC 2 (1yr), HIPAA (6yr), financial (7yr). Both too-long (minimization violation) and too-short (compliance violation) are problems.

| Refinement (deferred) | Why |
|---|---|
| Real-time compliance dashboard | On-demand reports sufficient |
| Evidence collection automation | Manual assembly for first audit |
| Breach notification logging | Build when incident response process defined |

**Dependencies:** RBAC roles (PII.2), Data Store (Platform 1.8), PII column inventory (PII.1)

**Certifications:** SOC 2 (all TSC), ISO 27001 (A.12.4), GDPR (Art. 30, 33, 35), HIPAA (§164.312(b)), PCI DSS (Req 10)

---

### PII.6 Data Retention & Deletion — ~9.5 dev days

**What this is:** GDPR requires data be kept only as long as necessary. Without automated retention enforcement, data accumulates forever — increasing breach blast radius. This module defines retention rules, auto-enforces expiration, supports legal holds for litigation, and verifies deletion across all systems including backups.

#### Baseline deliverables

**1. Retention rule configuration** — `retention_rules` table: `{data_type, jurisdiction, retention_period, action, legal_basis}`. Pre-loaded: profiles (lifetime + 30d), transcripts (2yr), chat (2yr), payments (7yr), analytics (1yr). GDPR Art. 5(1)(e).

**2. Automated enforcement** — Nightly job: scan → identify expired → if no legal hold → archive/delete → audit log. Rules without enforcement are just documentation.

**3. Legal hold management** — `legal_holds` table with mandatory review dates. Retention enforcement skips held records. Deleting data under litigation = spoliation of evidence.

**4. Deletion cascading + verification** — Cascade to: derived data, caches (Redis), backups (crypto-shredding). Verification scan for orphaned references. Deleting from primary DB but leaving copies everywhere is not deletion.

| Refinement (deferred) | Why |
|---|---|
| Retention policy templates | Manual rule config covers the same ground |
| Retention reporting dashboard | PII.5 compliance reports cover status |

**Dependencies:** Crypto-shredding (PII.1), Audit logging (PII.5), Job Queue (Platform 1.9), Data Store (Platform 1.8)

**Certifications:** GDPR (Art. 5, 17), CCPA (Right to Delete), ISO 27001 (A.8.3), SOC 2 (CC6.5), HIPAA (§164.530(j))

---

## Build Order

```
Phase 1 — Foundations (enable everything else)
├── PII.1  Encryption & Data Protection (field encryption, KMS, masking, crypto-shred)
└── PII.2  Access Control (RBAC, MFA, row/column security)

Phase 2 — Compliance Core (required for GDPR/SOC 2)
├── PII.5  Audit Logging & Compliance (immutable logs, ROPA, reports)
└── PII.3  Consent Management (collection, enforcement, withdrawal)

Phase 3 — Subject Rights + Retention (required before enterprise sales)
├── PII.4  Data Subject Rights (DSAR portal, access/erasure/rectification)
└── PII.6  Data Retention & Deletion (rules, enforcement, legal holds)
```

## What Counts as Refinement (explicitly deferred)

| Category | Examples | Why deferred |
|---|---|---|
| Encryption extras | Tokenization vault, static masking, key rotation automation | Field encryption + manual rotation sufficient |
| Access maturity | Approval workflows, quarterly reviews, separation of duties | Manual processes acceptable at small team |
| Consent polish | Preference center, versioning, cookie consent | Minimal toggles + enforcement API sufficient |
| DSAR automation | Restrict processing, automated evidence, API intake | Manual handling at low request volume |
| Audit dashboard | Real-time posture, evidence automation, breach tracking | On-demand reports sufficient for first audit |
| Retention polish | Templates, reporting dashboard | Manual rule config covers it |

## Dependencies

| Needs | From | Why |
|---|---|---|
| GCP Cloud KMS | Infrastructure | Encryption key management |
| MFA provider | Infrastructure | PII access authentication |
| Auth Gateway | Platform 1.5 | User identity and role resolution |
| Event Bus | Platform 1.2 | Consent withdrawal propagation |
| Data Store | Platform 1.8 | All PII, audit, consent, retention tables |
| Job Queue | Platform 1.9 | Retention enforcement jobs |
| Notification Service | Platform 1.10 | DSAR status updates |

## Effort Estimate

| Module | Effort |
|---|---|
| PII.1 Encryption & Data Protection | 14.5 days |
| PII.2 Access Control | 9.5 days |
| PII.3 Consent Management | 7.5 days |
| PII.4 Data Subject Rights (DSAR) | 15 days |
| PII.5 Audit Logging & Compliance | 11.5 days |
| PII.6 Data Retention & Deletion | 9.5 days |
| **Total** | **~68 dev days** |

## Key Concepts Explained

### What is field-level encryption?
Disk-level encryption protects data if someone steals the physical hard drive. But anyone with database access sees plaintext. Field-level encrypts individual columns with separate keys. Even a full dump yields ciphertext. You need both DB access AND the key from KMS.

### What is crypto-shredding?
Instead of finding and deleting every copy across databases, backups, caches, and logs, destroy the encryption key. All encrypted data becomes permanently unreadable — effectively deleted. The only reliable "right to be forgotten" in distributed systems.

### What is ROPA?
Records of Processing Activities — mandatory GDPR Art. 30 artifact. A living register of every type of personal data processing: what data, why, legal basis, how long kept, who has access, where it goes. The DPA can request it at any time.
