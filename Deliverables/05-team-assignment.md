# Deliverable 5 — Team assignment plan

## Assignment overview

| Epic | Eng A — Senior | Eng B — Mid | Eng C — Junior | Manager IC (Sahil) |
|------|---------------|------------|----------------|-------------------|
| **E1** OS & OVA | **Owner** | | Contributor | Reviewer *(MicroOS/Btrfs/OVA not in background)* |
| **E2** Cluster lifecycle | **Owner** | Contributor | | **Co-owner** *(1,500+ prod nodes at OpenText; HA validation, kube-vip, etcd)* |
| **E3** Platform services | Reviewer | **Owner** | Contributor | **Co-owner** *(NetworkPolicy, RBAC, TLS — zero-trust at OpenText & Red Hat Canada)* |
| **E4** Module lifecycle | **Owner** *(Go controller, state machine)* | | | **Spec + Helm** *(module.yaml spec, CRD schema, Helm integration — ArgoCD/GitOps at OpenText)* |
| **E5** ARK CLI | | **Owner** | **Stretch ★** diagnose | Reviewer *(validates CLI against OpenText ops patterns)* |
| **E6** Storage layer | | **Owner** | Contributor | Contributor *(NetApp background — NFS CSI, storage integration)* |
| **E7** Build pipeline | | **Owner** | Contributor | **Co-owner** *(DevSecOps at Red Hat Canada — Snyk, Xray, cosign, SBOM, Artifactory)* |
| **E8** Upgrade system | **Owner** | Contributor | | Contributor *(non-disruptive OpenShift upgrades at Red Hat India; Helm upgrade sequencing)* |

★ Engineer C owns `arkctl diagnose` end-to-end as a deliberate stretch assignment within E5.

---

## Engineer A — Senior

A owns E1, E2 (primary), E4 (controller), and E8 — the epics requiring the deepest appliance-specific knowledge and the highest architectural risk.

**E1 (OS/OVA):** A has shipped appliance-style products before. openSUSE MicroOS image construction, OVF properties, transactional-update, and Btrfs subvolume layout are familiar territory. This is foundational work that determines whether everything else is buildable — not something to delegate and review after the fact.

**E4 (Module lifecycle — Go controller):** The SupernaModule CRD reconciliation loop, state machine transitions, and controller-runtime internals go to A. This requires Go controller development depth the manager does not have, and is the most complex custom engineering in the project. The manager owns the spec and Helm integration layer; A owns the controller implementation.

**E2 and E8:** A leads on the appliance-specific decisions in cluster lifecycle and owns the upgrade system architecture (Btrfs integration, checkpoint orchestration). The manager co-contributes based on production Kubernetes depth; A makes the final calls.

---

## Engineer B — Mid

B owns E3 (overall), E5 (overall), E6, and E7 (overall). These epics map directly to B's solid infrastructure and pipeline background.

**E7 (Build pipeline):** B drives overall pipeline delivery. The manager co-owns the DevSecOps-specific components (Snyk, Xray, cosign, SBOM), so B is not alone on the security toolchain — but B owns the OVA build automation, runner setup, Artifactory workflow, and overall pipeline delivery.

**E3 (Platform services):** B owns ingress, cert-manager, and the identity service. The manager co-owns the NetworkPolicy framework and RBAC design. B integrates both into a coherent platform services layer.

**B's development goal:** Moving from reliable executor to independent problem-identifier. By Q3, B should be the person surfacing Longhorn edge cases or pipeline regressions in the weekly sync without being asked — and writing code reviews that catch real issues, not rubber stamps.

---

## Manager-IC plate

Realistic IC capacity on a three-person team: 45% of working time. The management overhead is real but does not consume the majority of the week. The manager ships code and owns technical deliverables — four specific areas mapped directly to actual background.

**E2 co-ownership — cluster lifecycle.** Having managed 1,500+ production Kubernetes nodes at OpenText and administered large-scale OpenShift clusters through non-disruptive upgrades at Red Hat India, the manager co-owns the HA topology validation, kube-vip failover testing, and etcd configuration. A owns the initial RKE2 bootstrap and appliance-specific decisions; the manager brings production-scale Kubernetes depth to the HA validation work A should not be doing alone.

**E3 co-ownership — platform services security layer.** The NetworkPolicy framework, RBAC model, and cert-manager CA configuration are directly in the manager's wheelhouse. Enforcing zero-trust compliance for highly regulated data workloads using Kubernetes Network Policies and RBAC is cited specifically from the OpenText role — this is not new territory. B owns E3 overall; the manager owns the security-specific implementation so B is not alone when those decisions get hard.

**E4 — module.yaml spec and Helm integration.** The manager writes the SupernaModule CRD schema and the `module.yaml` manifest specification, and owns the Helm release management layer within the operator. This is grounded in deep ArgoCD, Helm, and GitOps experience at OpenText (70% reduction in manual deployments, environment provisioning from 4 days to 12 minutes). The Go controller internals — state machine, controller-runtime reconciliation loop — go to A. The contract design and the Helm plumbing are the manager's.

**E7 co-ownership — build pipeline security.** The DevSecOps pipeline work at Red Hat Canada (integrating SAST/DAST, identifying critical vulnerabilities prior to production release) translates directly to the cosign signing, Snyk/Xray integration, SBOM generation, and Artifactory promotion workflow in E7. B owns the overall pipeline; the manager co-owns the security and supply chain components — the part that requires knowing what shift-left security actually looks like in a regulated production environment.

**E6 contributor — storage expertise.** Having worked at NetApp and optimised Azure NetApp Files for Kubernetes workloads (25% boost in IOPS throughput), the manager contributes to the NFS CSI driver setup and storage integration decisions without owning the epic outright.

**E1 — reviewer only.** openSUSE MicroOS, Btrfs/transactional-update, and OVA packaging are not in the manager's background. Claiming ownership here would slow A down rather than help. The manager reviews E1 design decisions and integration tests but does not hold the implementation pen.

---

## Engineer C — stretch assignment

C's stretch assignment is owning `arkctl diagnose` end-to-end within the E5 CLI epic. This is a deliberate choice, not a cleanup task.

**Why `arkctl diagnose` is the right stretch for C:** It is bounded with clear acceptance criteria. It touches enough of the full system — the API server, Longhorn, each module's health endpoint, platform pod logs — to force C to understand the complete ARK component model. It is low-blast-radius if C gets stuck; B can unblock without derailing the critical path. And when C ships it, it is something visible and real they can point to — not test-writing as a euphemism for work nobody else wants.

C also contributes to E1 (cloud-init config validation tests), E7 (cosign and syft SBOM integration stories), and E6 (storage class selection logic tests). Every story C completes has a named reviewer (A or B) giving genuine technical feedback.

---

## Coaching plan for Engineer C

### Weeks 1–4: Onboarding with directed ownership

C pairs with A on E1 for the first two weeks — not as a shadow. C owns two specific stories from day one: writing the cloud-init configuration for the OVF/GuestInfo datasource, and building the first integration test validating a successful first-boot in the air-gapped lab. A reviews C's design choices before implementation starts.

Daily 15-minute check-in between C and manager for weeks 1–4. C brings one technical question they could not find the answer to on their own. Manager either answers it or directs: "ask A, specifically about X." At week three, C starts reviewing B's pull requests with written technical feedback — manager reviews the quality of C's code reviews in their next 1:1.

### Months 2–3: First solo ownership

C moves to their E7 contribution (cosign and syft integration in the build pipeline). B is the primary technical pair; C drives.

Weekly 1:1 between C and manager shifts from unblocking to pattern recognition: "What did you see this week that you did not expect? What assumption turned out to be wrong?"

C attends all E4 architecture review sessions as an observer with one rule: they must ask one question per session. Not optional. Manager assigns C to read two or three design documents from related open-source projects and come to a 1:1 prepared to explain one thing they would borrow and one thing they would do differently.

### Months 4–5: Diagnose ownership

C takes `arkctl diagnose`. Manager pairs with C for the first session only: together they define the acceptance criteria, the bundle data model, and the redaction rules. Then C designs and implements independently.

B is C's primary pair for day-to-day questions. Manager does a weekly 20-minute architecture review with C on diagnose progress — C presents what they built and why, manager asks one probing question about a decision C made.

### Month 6: Documentation and independence

C writes the operator documentation for `arkctl diagnose` and the `module.yaml` developer guide section covering network isolation. Writing documentation forces complete understanding.

C presents `arkctl diagnose` in a team demo. Owns the full explanation, takes questions from A without manager filling in gaps.

### What success looks like at the 6-month mark

- C has shipped `arkctl diagnose` including tests and documentation with no stories left open.
- C can explain the `module.yaml` contract to a new module team engineer — not recite it, but explain the reasoning behind each field and what happens if it is wrong.
- C's pull requests need one or two rounds of review, not four or five.
- C is writing code reviews for B's work that B finds genuinely useful.
- C has expressed one technical opinion that disagreed with an initial approach, backed it with reasoning, and been taken seriously even if the team did not adopt it.

That last point is the most important signal. Independent technical voice is what separates a junior engineer growing toward mid-level from one who is executing assigned tasks competently but not yet thinking for themselves.