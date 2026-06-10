# Non-Functional Requirements (NFRs) — Ward Management System

## Purpose & scope

This document specifies the quality attributes for the **Ward Management System (WMS)** at Base Hospital Kiribathgoda. Each NFR is **measurable** (target + verification method) and **traced to a Sprint 3 SDS design decision**. NFRs are grouped into the six mandated categories: performance, security, usability, reliability, scalability, and maintainability.

Operating context: a single ward setting, on-premise deployment, peak of roughly **50 concurrent clinical users**. The **primary client is a shared, tablet-first PWA** used by ward doctors and nurses at the bedside; a ward-desk workstation is an optional secondary client. **OPD and Laboratory are external systems** integrated via interfaces. Figures are p95 unless noted.

### Sprint 3 SDS design-decision key

| Ref | Sprint 3 SDS design decision |
|---|---|
| **SDS-A1** | Layered (N-tier) architecture (ADR-01) |
| **SDS-A2** | Event-driven publish/subscribe overlay for the Notification subsystem (ADR-01) |
| **SDS-PWA** | Tablet-first Progressive Web App: service worker offline cache (observations + MAR) + push |
| **SDS-CMP** | Ward modules: Ward, Notification, Auth, Patient Records, Audit; external-integration adapters |
| **SDS-DEP** | 3-tier deployment (tablet client / app server / DB) + external OPD & Lab nodes; TLS paths |
| **SDS-CLS** | Domain class model (`AuditLog`, `Observation` offline-sync flag, `MedicationAdministration` nurse-ID + timestamp, auto-BMI) |

Priority scale: **M** = Must, **S** = Should, **C** = Could.

---

## 1. Performance

| ID | Requirement (source) | Measurable acceptance criterion | Verification | Pri. | Linked SDS |
|---|---|---|---|---|---|
| NFR-P-01 | Interactive screens load quickly | Any screen renders interactive content in **< 2 s (p95)**, < 4 s (p99) under normal load | Load test | M | SDS-A1, SDS-DEP |
| NFR-P-02 | Real-time critical alerting (UC07) | A P1 critical notification is dispatched to the recipient device **within 5 s** of the triggering event | Timing test on event→dispatch | M | SDS-A2 |
| NFR-P-03 | Lab worklist / result exchange (UC05/UC06) | A submitted lab request reaches the external Lab in **< 3 s**; an inbound result is visible on the ward in **< 3 s** of receipt; STAT SLA timer accurate to ≤ 1 s | Integration test | M | SDS-CMP (adapters) |
| NFR-P-04 | Single-tap medication tick (UC04, DC-05) | Recording an administration completes and turns the row green in **≤ 1 s** on a mid-range tablet | On-device UI test | M | SDS-PWA, SDS-CLS |
| NFR-P-05 | Bed dashboard freshness (UC02) | A bed status change is reflected on all viewers in **< 2 s** | Integration test | S | SDS-A2, SDS-CMP (Ward) |
| NFR-P-06 | Patient history retrieval (UC09) | The longitudinal timeline returns in **< 3 s (p95)** for a patient with ≤ 50 admissions | Load test | M | SDS-CMP (Patient Records) |
| NFR-P-07 | Sustained throughput | Sustains **50 concurrent users** with all latency targets held and error rate < 0.1% | Soak/load test | M | SDS-A1, SDS-DEP |
| NFR-P-08 | Tablet device performance (tablet-first) | Cold PWA launch to usable in **< 4 s**; smooth scroll/interaction (no frame stalls > 100 ms) on a specified mid-range reference tablet | On-device performance test | S | SDS-PWA |

## 2. Security

| ID | Requirement (source) | Measurable acceptance criterion | Verification | Pri. | Linked SDS |
|---|---|---|---|---|---|
| NFR-S-01 | Authenticated access only | **100%** of protected endpoints reject unauthenticated requests; credentials stored only as salted hashes (Argon2/bcrypt) — **0** plaintext secrets | Security test + secret scan | M | SDS-CMP (Auth) |
| NFR-S-02 | Role-based access control | Server-side checks for every role-restricted action; **0** authorization bypasses in test suite | RBAC test matrix | M | SDS-A1, SDS-CMP (Auth) |
| NFR-S-03 | Encryption in transit & at rest | All client↔server, server↔DB, and server↔external (OPD/Lab) traffic over **TLS 1.2+**; data at rest **AES-256**; **0** cleartext channels | TLS scan + config audit | M | SDS-DEP |
| NFR-S-04 | Immutable audit trail (UC02/04/06/09; DC-05) | **100%** of access/modification events produce a tamper-evident, append-only audit entry (user ID, role, action, timestamp); retained **≥ 7 years** | Audit-coverage + tamper test | M | SDS-CLS (`AuditLog`) |
| NFR-S-05 | Confidential-record justification (UC09 AF09B) | Accessing a restricted record without a logged "clinical need to access" justification is **blocked in 100%** of attempts | Functional + audit test | S | SDS-CMP (Patient Records, Auth) |
| NFR-S-06 | Session management | Idle sessions terminate after **15 min** (workstation); re-authentication required thereafter | Functional test | S | SDS-CMP (Auth) |
| NFR-S-07 | Vulnerability posture | **0 critical/high** findings (OWASP Top 10) outstanding at go-live | Pre-release penetration test | M | SDS-A1, SDS-DEP |
| NFR-S-08 | Shared-tablet auto-lock & fast switching (tablet-first) | A shared ward tablet auto-locks after **≤ 90 s** idle; switching users takes **≤ 5 s**; the MAR nurse-ID always reflects the active login (**0** mismatches in test) | Functional + usability test | M | SDS-PWA, SDS-CMP (Auth) |
| NFR-S-09 | Offline data protection (tablet-first) | The PWA offline cache is **encrypted on device** and **purged on logout/lock**; **0** readable clinical data at rest after logout | Device security test | M | SDS-PWA |

## 3. Usability

| ID | Requirement (source) | Measurable acceptance criterion | Verification | Pri. | Linked SDS |
|---|---|---|---|---|---|
| NFR-U-01 | Single-tap administration parity (UC04, DC-05) | Nurses record a dose in **1 tap**, no intermediate screens; task success **≥ 95%** | Usability test (≥ 8 nurses) | M | SDS-CLS, SDS-CMP (Ward) |
| NFR-U-02 | Learnability | After **≤ 30 min** orientation, a new user completes admit-intake, record-observation, view-results with **≥ 90%** task completion | Moderated usability test | S | SDS-A1 |
| NFR-U-03 | Bilingual interface (Health 26 is Sinhala/English) | All clinical labels available in **Sinhala and English**; language switch with **0** loss of entered data | i18n test | S | SDS-CMP (UI) |
| NFR-U-04 | Priority colour coding (UC04/UC10) | Green/amber/red scheme consistent with the paper tick; meets **WCAG 2.1 AA** contrast | Accessibility audit | S | SDS-CMP (Ward, Notification) |
| NFR-U-05 | Overall satisfaction | **SUS ≥ 75** across the four role groups | Post-pilot SUS survey | C | — |
| NFR-U-06 | Touch ergonomics (tablet-first) | All interactive targets **≥ 48 dp (≈ 44 pt)** with adequate spacing; operable with a gloved hand; primary actions reachable one-handed | Touch usability test | M | SDS-PWA |
| NFR-U-07 | Responsive tablet layout (tablet-first) | Renders without horizontal scroll or clipping on a **10″ tablet** in both orientations at common resolutions (e.g. 1280×800, 2360×1640) | Responsive/layout test | M | SDS-PWA |

## 4. Reliability

| ID | Requirement (source) | Measurable acceptance criterion | Verification | Pri. | Linked SDS |
|---|---|---|---|---|---|
| NFR-R-01 | Availability | **≥ 99.5%** uptime measured monthly | Uptime monitoring | M | SDS-DEP |
| NFR-R-02 | No clinical data loss | Backup **RPO ≤ 5 min**, restore **RTO ≤ 1 h**; **0** committed records lost in DR drill | DR drill | M | SDS-DEP, SDS-CLS |
| NFR-R-03 | Offline capture & sync (UC03 AF03C; tablet-first) | Observations **and** medication ticks entered during a WiFi outage are cached and **100%** synced within **30 s** of reconnection, with no duplicates and conflicts resolved per policy | Fault-injection test | M | SDS-PWA, SDS-CLS |
| NFR-R-04 | Guaranteed critical-alert escalation (UC07 AF07A/B) | **100%** of undelivered/un-acknowledged P1 alerts escalate within the defined window (5 / 10 / 15 min chain) | Notification reliability test | M | SDS-A2, SDS-CMP (Notification) |
| NFR-R-05 | Medication-record integrity (DC-05) | **100%** of administration records carry a nurse ID and timestamp; none can be saved without them | Data-integrity test | M | SDS-CLS (`MedicationAdministration`) |
| NFR-R-06 | Graceful external-integration degradation | If OPD or Lab is unreachable, the ward core stays available; inbound/outbound integration messages are **queued and retried**, with **0** lost on recovery | Integration fault test | S | SDS-CMP (adapters) |

## 5. Scalability

| ID | Requirement (source) | Measurable acceptance criterion | Verification | Pri. | Linked SDS |
|---|---|---|---|---|---|
| NFR-SC-01 | Concurrent-user headroom | Sustains **2× peak (100 users)** with ≤ 20% latency degradation and no architecture change | Scaled load test | S | SDS-A1, SDS-DEP |
| NFR-SC-02 | Data-volume growth | Maintains performance NFRs at **≥ 100,000 patient records** and **≥ 10,000 admissions/year** | Volume test | S | SDS-DEP, SDS-CMP (Patient Records) |
| NFR-SC-03 | Configuration-driven expansion | Adding a ward, bed, or ICD code set requires **0 code changes / 0 redeploys** (data/config only) | Functional test | M | SDS-CMP (Ward) |
| NFR-SC-04 | Tablet fleet scaling (tablet-first) | Adding tablets to a ward needs **no server change**; the system supports **≥ 20 active tablets per ward** within latency targets | Device-scale test | S | SDS-PWA, SDS-DEP |

## 6. Maintainability

| ID | Requirement (source) | Measurable acceptance criterion | Verification | Pri. | Linked SDS |
|---|---|---|---|---|---|
| NFR-M-01 | Modular separation (ADR-01) | Each module independently buildable/unit-testable; **0** disallowed cross-layer calls (presentation → data access) | Architecture fitness test in CI | M | SDS-A1, SDS-CMP |
| NFR-M-02 | Test coverage | **≥ 80%** unit-test line coverage on the domain layer; CI fails below threshold | Coverage gate | M | SDS-CLS |
| NFR-M-03 | Code complexity | Average cyclomatic complexity **≤ 10** per method; none **> 20** without justification | Static analysis | S | SDS-A1 |
| NFR-M-04 | Modifiable clinical reference data | A new lab test in the catalogue is live with **0** code change within **1 working day** | Change-effort check | S | SDS-CMP (adapters) |
| NFR-M-05 | Documented interfaces | **100%** of public service and integration interfaces have current documentation | Doc-coverage review | S | SDS-CMP |
| NFR-M-06 | Event-contract clarity (ADR-01) | All notification events have a versioned schema; **0** undocumented event types in production | Contract review | S | SDS-A2 |
| NFR-M-07 | External-integration isolation | OPD/Lab integration is confined to adapters; a change to an external contract touches **only** its adapter (no domain/UI change) | Change-impact review | S | SDS-CMP (adapters) |

---

## Traceability summary

| Category | NFR IDs | Primary Sprint 3 SDS links |
|---|---|---|
| Performance | NFR-P-01..08 | SDS-A1, SDS-A2, SDS-PWA, SDS-CMP, SDS-DEP, SDS-CLS |
| Security | NFR-S-01..09 | SDS-CMP (Auth), SDS-DEP, SDS-PWA, SDS-CLS (`AuditLog`) |
| Usability | NFR-U-01..07 | SDS-PWA, SDS-CLS, SDS-CMP (UI) |
| Reliability | NFR-R-01..06 | SDS-PWA, SDS-A2, SDS-DEP, SDS-CLS, SDS-CMP (adapters) |
| Scalability | NFR-SC-01..04 | SDS-A1, SDS-PWA, SDS-DEP, SDS-CMP |
| Maintainability | NFR-M-01..07 | SDS-A1, SDS-CMP (adapters), SDS-CLS, SDS-A2 |

**Open points for review:** confirm the availability target (99.5% vs 24/7 99.9%), the audit-retention period against local MOH policy, the shared-tablet auto-lock interval (90 s assumed), the reference tablet model for NFR-P-08, and whether bilingual support is Must or Should for the first release.
