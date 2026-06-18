# Deliverable 5 — Team assignment plan

## Assignment overview

| Epic | Engineer A (Senior) | Engineer B (Mid) | Engineer C (Junior) | Manager IC |
|------|--------------------|-----------------|--------------------|------------|
| E1 OS & OVA | **Owner** | | Contributor | |Contributor
| E2 Cluster lifecycle | **Owner** | Contributor | | |Contributor
| E3 Platform services | Reviewer | **Owner** | Contributor | |Reviewer 
| E4 Module lifecycle | **Owner** | | | Spec author + co-owner |
| E5 CLI | | **Owner** | **Stretch ★ diagnose** | |Reviewer 
| E6 Storage layer | | **Owner** | Contributor | |
| E7 Build pipeline | | **Owner** | Contributor | |
| E8 Upgrade system | **Owner** | Contributor | | |

★ Engineer C owns `arkctl diagnose` end-to-end as a deliberate stretch assignment within E5.

---

## Engineer A — Senior

A owns E1, E2, E4, and E8 — the four epics that form the critical path and carry the highest architectural risk.

**E1 (OS/OVA):** A has shipped appliance-style products before. openSUSE MicroOS image construction, OVF properties, transactional-update, and Btrfs subvolume layout are familiar territory rather than an exploratory domain. This is not a task to delegate and then review — it is foundational work that determines whether everything else is buildable.

**E2 (Cluster lifecycle):** Deep Kubernetes, RKE2, kube-vip ARP mode, etcd topology, HA failover behaviour under real network conditions. This is squarely A's expertise. B contributes to E2 in the later stages (worker node join process, topology validation tests) once A has the control-plane stable — this keeps B close to the complex system work rather than siloed in the pipeline, building B's system understanding for later quarters.

**E4 (Module lifecycle — core product IP):** The module operator and SupernaModule CRD are the highest-leverage engineering artefact in the project. A makes the API design calls; the manager co-owns the reconciliation loop framework and CRD schema (see manager-IC section). Getting the external-facing API wrong before module teams build on it is the most expensive mistake possible at this scale.

**E8 (Upgrade system):** Requires the Btrfs/transactional-update OS depth A has from E1, plus the Kubernetes and Helm state machine depth from E2 and E4. B implements the Helm upgrade sequencing steps under A's architecture, keeping B engaged on the most operationally consequential work in H2.

---

## Engineer B — Mid

B owns E3, E5 (overall), E6, and E7. These four epics map directly to B's strengths: pipeline and CI/CD work (E7 is B's most natural home — Bitbucket Pipelines, Artifactory, cosign, syft, exactly B's background), solid Kubernetes infrastructure work (E3 and E6 are integration work, not novel controller design), and CLI implementation on top of well-defined APIs (E5 builds on contracts established by E4).

The honest risk with B is the appliance context, which is new to them. The mitigation is sequencing: A's work on E1 and E2 produces reference implementations B can read and pattern-match from before E3 and E6 go deep in Q2. B does not need to invent the appliance idiom — they need to apply it consistently, and the examples will be there when needed.

**B's development goal on this project:** moving from reliable executor to independent problem-identifier. Concretely: by Q3, B should be the person flagging Longhorn edge cases or pipeline regressions in the weekly sync, not waiting for the manager to ask. B's PR reviews should generate observations, not rubber stamps. This shift is the marker of a mid-level engineer maturing toward senior.

---

## Manager-IC plate

**E4 co-ownership:** The manager writes the SupernaModule CRD schema (the field definitions, validation rules, status conditions, and versioning strategy — a real YAML and Go artefact), scaffolds the controller-runtime operator framework, and writes the main reconcile function structure. A handles the most complex internals: the Helm integration, state machine transitions, and dependency resolution. Neither could ship E4 at the required pace alone.

**`module.yaml` spec authorship:** The module contract is the most consequential single document in the project. The manager holds the pen because the manager has done the deepest requirements analysis and has the cross-team relationships to get the spec reviewed and signed off before implementation begins. Delegating spec authorship to save IC time would cost far more in rework than it saves.

**Integration test framework:** The manager builds the test harness that validates first-boot in the air-gapped lab, module lifecycle conformance, and upgrade resume scenarios. This is real engineering requiring full-system understanding — not implementation work B or C should be pulled away from their epic ownership to build.

**E3 security contributions:** Writes the NetworkPolicy framework and cert-manager CA configuration (IP SAN, CA key handling in HA). Maps to the manager's background in cluster security and supply chain integrity. B owns E3 overall; the manager's contribution means B is not alone when the security details get hard.

**Architecture decision records (ADRs):** Written as each epic completes, not at project end. Each ADR requires running validation spikes (footprint benchmarks, failover timing measurements) before the written rationale is credible. These are real engineering work, not documentation.

---

## Engineer C — stretch assignment

C's stretch assignment is owning `arkctl diagnose` end-to-end within the E5 CLI epic. This is a deliberate choice, not a cleanup task.

**Why `arkctl diagnose` is the right stretch for C at this stage:** It is a bounded piece of work with clear acceptance criteria. It touches enough of the full system to force C to understand the complete ARK component model — diagnose queries the API server, Longhorn, each module's health endpoint, and platform pod logs. You cannot implement it without understanding all of those. It is low-blast-radius if C gets stuck (B can unblock without derailing the critical path). And when C ships it, it is something visible and real that C can point to — not test-writing as a euphemism for work nobody else wants.

C also contributes to E1 (cloud-init config validation tests), E7 (cosign and syft SBOM integration stories), and E6 (storage class selection logic tests). Every story C completes has a named reviewer (A or B) who gives genuine technical feedback, not rubber-stamp approvals.

---

## Coaching plan for Engineer C

### Weeks 1–4: Onboarding with directed ownership

C pairs with A on E1 for the first two weeks — not as a shadow. C owns two specific stories from day one: writing the cloud-init configuration for the OVF/GuestInfo datasource, and building the first integration test validating a successful first-boot in the air-gapped lab. A reviews C's design choices before implementation starts and explains decisions C did not know to ask about.

Daily 15-minute check-in between C and manager for weeks 1–4. Not a status update — C brings one technical question they could not find the answer to on their own. Manager either answers it or directs: "ask A, specifically about X." This prevents C from spinning on blockers but also prevents them from asking before genuinely trying.

At week three, C starts reviewing B's pull requests with written technical feedback. Manager reviews the quality of C's code reviews in their next 1:1 — reviewing is a skill that needs deliberate practice, and C's reviews should improve visibly across the project.

### Months 2–3: First solo ownership

C moves to their E7 contribution (cosign and syft integration in the build pipeline). B is the primary technical pair; C drives. 

Weekly 1:1 between C and manager (30 minutes) shifts from unblocking to pattern recognition: "What did you see this week that you did not expect? What assumption turned out to be wrong?" This is not comfortable for most junior engineers but it is how senior thinking develops.

C attends all E4 architecture review sessions as an observer with one rule: they must ask one question per session. Not optional. Asking questions in architecture reviews is a skill that requires deliberate practice, and C needs to build the habit of engaging in technical discussions above their current level.

At month three, manager assigns C to read two or three design documents from related open-source projects (Helm's release lifecycle controller, Longhorn's volume health state machine) and come to a 1:1 prepared to explain one thing they would borrow and one thing they would do differently. The goal is not expertise — it is building the habit of comparative technical analysis.

### Months 4–5: Diagnose ownership

C takes `arkctl diagnose`. Manager pairs with C for the first session only: together they define the acceptance criteria, the bundle data model, and the redaction rules. Then C designs and implements independently.

B is C's primary technical pair for day-to-day questions on the CLI. Manager does a weekly 20-minute architecture review with C on diagnose progress — C presents what they built and why, manager asks one probing question about a decision C made. This is not a status check; it is deliberate coaching on design reasoning.

### Month 6: Documentation and independence

C writes the operator documentation for `arkctl diagnose` and the `module.yaml` developer guide section covering network isolation. Writing documentation forces complete understanding — if C cannot explain the network isolation model clearly enough for a module team engineer to act on it, C does not fully understand it yet.

C presents `arkctl diagnose` in a team demo. Owns the full explanation, takes questions from A without manager filling in gaps.

### What success looks like at the 6-month mark

- C has shipped `arkctl diagnose` including tests and documentation with no stories left open.
- C can explain the `module.yaml` contract to a new module team engineer — not recite it, but explain the reasoning behind each field and what happens if it is wrong.
- C's pull requests need one or two rounds of review, not four or five. The gap between what C writes and what the team accepts has narrowed measurably.
- C is writing code reviews for B's work that B finds genuinely useful — observations that catch something, not approval rubber stamps.
- C has expressed one technical opinion that disagreed with an initial approach, backed it with reasoning, and been taken seriously even if the team did not adopt it.

That last point is the most important signal. Independent technical voice is what separates a junior engineer growing toward mid-level from one who is executing assigned tasks competently but not yet thinking for themselves. If C is not comfortable disagreeing by month six, the coaching approach needs to change — not C's assignment.
