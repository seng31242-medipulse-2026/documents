# ADR-01: Adopt a Layered Architecture for the Ward Management System

---

## Context

The system under design is the **Ward Management System (WMS)** for Base Hospital Kiribathgoda. Its scope is the ward floor: admission intake into the ward, bed allocation, doctor observations and treatment, nurse medication tracking, ward rounds, real-time notification, and discharge. **OPD and the Laboratory are external systems** that the WMS integrates with through defined interfaces — OPD calls `IAdmissionIntake` to hand a patient over, the Lab calls `ILabResultIntake` to deliver results, and the WMS calls the Lab's `ILabService` to submit requests. They are out of the WMS build scope.

The forces this architecture must withstand, from the Stakeholder Requirements Survey (n = 13) and the field evidence:

- **Clinical safety is paramount.** 100% of respondents reported delayed diagnosis; the paper medication record was the single highest patient-safety risk. Correctness, auditability, and traceable accountability are non-negotiable.
- **Real-time behaviour is a primary feature.** Real-time lab-result notification with priority escalation, STAT SLAs, and critical-value alerts (FR07, UC07) is a cross-cutting concern.
- **Mandatory auditability.** Every record access, bed allocation, medication tick, and result entry must produce an immutable audit trail (UC02, UC04, UC06, UC09), enforced uniformly.
- **Regulatory interoperability.** Mirror Health 26 Sections A–E, capture ICD-10/11 codes, preserve BHT as a secondary identifier alongside a persistent patient ID, and accept structured analyser output via the Lab (DC-01 to DC-08).
- **Tablet-first, shared devices.** The primary client is a **tablet used at the bedside by ward doctors and nurses**, shared per ward; a ward-desk workstation is an optional secondary client. This implies roaming over hospital WiFi (dead zones), the need for offline tolerance, and fast, safe user switching on a shared device.
- **Loose coupling at the boundary.** Because OPD and Lab are external and may change independently, the integration must be isolated so their changes do not ripple into the ward core.
- **Modest, single-site scale.** One ward context, a small number of concurrent clinical users, on-premise deployment.
- **Delivery constraints.** Built by a student team from Semester 2; the architecture must be learnable, testable, and maintainable by a small team.

Driving quality attributes, in priority order: **maintainability/modifiability, security & auditability, reliability (including offline tolerance), and real-time performance**, with **deployability/simplicity** weighted highly. Scalability is explicitly a lower priority given the single-site scope.

## Decision

Adopt a **Layered (N-tier) architecture** as the primary pattern for the WMS, organised into four logical layers:

1. **Presentation Layer** — a **tablet-first Progressive Web App (PWA)**; the UI is structured internally using **MVC** (MVC operates *within* this layer, not as the whole-system architecture). The PWA uses a service worker for an **offline cache (observations + MAR)** and for push notifications.
2. **Application / Service Layer** — the ward domain modules (Ward, Notification, Auth, Patient Records, Audit) **plus external-integration adapters** that realise the boundary interfaces (`IAdmissionIntake`, `ILabResultIntake` provided; `ILabService` required).
3. **Domain Layer** — the entities and business rules from the class diagram (Patient, Admission, Observation, MedicationOrder, LabResult, …), where invariants such as critical-value checks and allergy alerts live.
4. **Data Access Layer** — repositories abstracting persistence to PostgreSQL; the only layer permitted to talk to the data tier.

Refinements adopted within this baseline:

- A **publish/subscribe (event-driven) overlay** for the **Notification subsystem (UC07)** only — events such as *result-ready*, *critical-value-detected*, and *medication-overdue* are published and consumed asynchronously.
- **Auth, Logging/Audit, and Allergy-alerting** are treated as **cross-cutting concerns** applied uniformly across the application and domain layers.
- **External integration is confined to adapters** at the service layer, keeping the ward core independent of OPD/Lab implementation detail.

The four logical layers map onto the **three physical deployment tiers** (tablet client → web/application server → database server), with **OPD and Lab as external nodes beyond the WMS boundary**.

## Rationale

Layering most directly satisfies the highest-priority attributes at the lowest complexity for this team and scale.

**Maintainability & modifiability.** Strict layering localises change. The Health 26 / ICD / BHT constraints (DC-01–08) are absorbed in the domain and data-access layers. The **tablet PWA presentation is swappable** without touching business rules, and the **external-integration adapters isolate OPD/Lab**, so a change on either side of the boundary does not ripple into the ward core.

**Security & auditability.** A single Auth boundary plus a single Audit cross-cutting concern guarantee uniform authentication and logging of every privileged action — satisfying the immutable-audit-trail requirement by construction. On shared tablets this is reinforced by short auto-lock and fast user switching so the active login (which auto-populates nurse ID on the MAR, UC04) is always correct.

**Reliability & correctness.** Clinical invariants concentrated in the domain layer are testable in isolation. The **PWA offline cache + sync** keeps observation and medication capture working through WiFi dead zones, and the event overlay guarantees critical-alert dispatch/escalation.

**Real-time performance.** The event-driven notification overlay dispatches and escalates P1/P2 alerts asynchronously without blocking the originating transaction.

**Deployability & simplicity.** A layered application deployed across three tiers, with external systems integrated via adapters, is straightforward to build, test, and operate on-premise for one hospital and a student team — and avoids distributed-systems failure modes that would themselves be a clinical-safety risk.

Quality-attribute trade-off across candidates (▲ strong, ● adequate, ▽ weak *for this context*):

| Quality attribute (weight) | Layered (+ event overlay) | Microservices | Pure MVC | Pure Event-Driven | 2-tier Client-Server |
|---|---|---|---|---|---|
| Maintainability (high) | ▲ | ▲ | ● | ● | ▽ |
| Security & auditability (high) | ▲ | ● | ▽ | ▽ | ▽ |
| Reliability/correctness (high) | ▲ | ● | ● | ▽ | ▽ |
| Real-time performance (high) | ● (via overlay) | ▲ | ▽ | ▲ | ▽ |
| Deployability/simplicity (high) | ▲ | ▽ | ▲ | ● | ▲ |
| Scalability (low) | ● | ▲ | ● | ▲ | ▽ |
| Testability (high) | ▲ | ● | ● | ▽ | ▽ |

Layered scores strongly on every high-weight attribute while staying simple to deploy; the event overlay compensates for layering's only relative weakness (real-time push) where it matters.

## Alternatives Considered

| Alternative | Summary | Strengths | Weaknesses for this context | Verdict |
|---|---|---|---|---|
| **Microservices** | Each ward module deployed as an independent service with its own datastore. | Independent deployability/scaling; strong boundaries; fault isolation. | Heavy operational tooling beyond a student team; distributed transactions complicate the cross-module consistency clinical records demand; network partitions become a patient-safety risk; unjustified at single-ward scale. | **Rejected** — complexity and operational cost exceed scale and team capacity. |
| **Pure MVC (whole-system)** | MVC as the top-level structure. | Familiar; clean UI separation; good for the tablet presentation tier. | Not a system architecture; gives no guidance on domain/data separation, cross-cutting audit/auth, the notification subsystem, or external integration. | **Rejected as macro-pattern; adopted within the presentation layer.** |
| **Pure Event-Driven** | All inter-module interaction via an event bus. | Excellent decoupling; natural fit for notifications. | Eventual consistency and choreography make medical-record correctness and step-by-step use-case flows (e.g. discharge open-item checks) hard to reason about, debug, and audit. | **Rejected as macro-pattern; adopted only for the Notification subsystem.** |
| **Two-tier Client-Server** | Thick tablet client querying the database directly. | Simplest to stand up. | No domain layer, so business rules and auditing leak into UI and DB; couples presentation to schema; poor testability — reproduces the fragility of the paper systems being replaced. | **Rejected** — fails maintainability, security, and auditability drivers. |
| **ESB-centred SOA** | Coarse services integrated through an enterprise service bus. | Good interoperability; centralised integration. | ESB infrastructure is disproportionate for one ward; the few integrations (OPD, Lab/HL7, ICD lookup) are handled adequately by service-layer adapters. | **Rejected** — disproportionate integration overhead. |

## Consequences

**Positive**

- Clear separation of concerns; UI, use-case orchestration, business rules, and persistence are independently modifiable and testable, supporting incremental Semester-2 delivery.
- Auth and audit enforced uniformly as cross-cutting concerns, satisfying immutable-trail and access-control requirements by construction.
- The tablet PWA (offline cache + push) operationalises the offline and real-time requirements for roaming bedside use.
- External-integration adapters keep the ward core stable against OPD/Lab changes.
- The four logical layers map cleanly onto the three deployment tiers and onto the ward component diagram's modules, keeping the design artifacts consistent and the decision traceable.
- Low operational complexity suits on-premise hospital deployment and a small team.

**Negative / risks (and mitigations)**

- *Offline sync conflicts* — cached observations/MAR entries may conflict on reconnect. *Mitigation:* per-record sync flag, last-write-wins with audit of overrides; specified in ADR-06.
- *Shared-device session risk* — wrong active login on a shared tablet. *Mitigation:* short auto-lock + fast user switching + re-auth (see NFR-S-08).
- *Layer-traversal overhead.* *Mitigation:* modest single-site load; profile only if a real bottleneck appears.
- *Sinkhole anti-pattern.* *Mitigation:* allow read-only queries to bypass the domain layer via a documented exception.
- *Limited independent scalability.* *Accepted:* scalability is low priority; revisit only if scope expands beyond a single hospital.
- *Two interaction styles* (synchronous layered calls + asynchronous events). *Mitigation:* confine events to the Notification subsystem and document the contracts.
- *Layer discipline must be enforced* (presentation must never reach data access directly). *Mitigation:* module boundaries + code review + an architectural fitness check.

**Follow-up decisions required**

- ADR-02: persistence and repository strategy for the data-access layer.
- ADR-03: notification transport (in-app + push provider) and event-contract schema.
- ADR-04: authentication/authorisation mechanism and audit-log storage model.
- ADR-05: external-integration contracts and adapters for OPD and Laboratory (HL7/REST).
- ADR-06: offline-cache sync and conflict-resolution policy for the tablet PWA.
