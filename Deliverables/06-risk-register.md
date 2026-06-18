# Deliverable 6 — Risk register and open questions

## Risk register

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|-----------|
| R1 | **module.yaml spec invalidated by first module team.** The module contract is designed without a real consumer. If the first Superna module team tries to build against it in Q2 and finds the spec unworkable, fixing it mid-stream blocks every product team simultaneously. | Med | High | Gate E4 implementation start on a formal spec review by at least one module team lead — target end of Q1. Manager owns scheduling the review. A wrong-spec finding at this stage is cheap; the same finding at Q3 is a multi-month setback across the organisation. |
| R2 | **Host OS variant ambiguity — MicroOS vs Leap.** "openSUSE-based" is ambiguous between MicroOS (immutable, transactional-update) and Leap (mutable, zypper-managed). The entire upgrade architecture assumes MicroOS. If Superna's existing tooling is built around Leap, the upgrade story changes materially. | Med | High | Surface as OQ-1 before E1 development begins. Design proceeds on MicroOS assumption with this risk explicitly flagged. If Leap is confirmed, the upgrade architecture reverts to a scripted installer pattern rather than transactional-update snapshots — a real design change, not just a configuration difference. |
| R3 | **Q2 module operator slip gates external module teams.** E4 (module operator) is the dependency that unblocks other Superna product teams. A slip in E4 does not just delay ARK — it delays every team building on top of it, with compounding impact across the organisation's roadmap. | Med | High | Manager and A co-own E4 to prevent single-point-of-failure on the critical path. Publish a stable draft `module.yaml` spec and stub SupernaModule CRD by end of Q1 so module teams can begin designing against the contract before the operator is production-grade. |
| R4 | **Minimum VM sizing undefined — RKE2 footprint may not fit.** RKE2 idles at roughly 2–3 GB RAM before any module pod runs. "Modestly-sized customer VMs" is the NFR but no floor is specified. If the smallest supported config is 4 GB, there may not be enough headroom. | Med | Med | Surface as OQ-2 before orchestrator choice is locked. Run a Q1 spike: deploy RKE2 plus the full platform stack on the smallest VM the team can agree on and measure idle footprint. If RKE2 exceeds 20% of total RAM before any module pod runs, switch to K3s and accept the single-node→HA datastore migration cost. |
| R5 | **Critical CVE in base image shortly after OVA release.** JFrog Xray monitors stored artifacts continuously. A critical CVE in the openSUSE base image or RKE2 after an OVA ships requires producing and distributing a patch quickly. Without pipeline automation, customers sit exposed while the team manually rebuilds. | High | Med | Automate the CVE-triggered rebuild loop in E7 (Xray alert → pipeline trigger → new artifact in Artifactory Staging) before first OVA ships. Agree a target patch SLA with leadership before Q4 (OQ-5). Target: signed patch bundle available within 24 hours of a critical CVE disclosure. |
| R6 | **kube-vip ARP failover slower than expected on customer network gear.** Some enterprise switches cache ARP entries longer than gratuitous ARP broadcasts suggest, making VIP failover appear to take 30–60 seconds on specific hardware. This varies by customer environment and cannot be fully tested in the lab. | Med | Med | Document the ARP-dependent failover behaviour explicitly in operator documentation and the field runbook. Add ARP cache flush to the kube-vip leader handoff where the OS supports it. In the Q4 pre-production trial, measure actual failover time on the customer's network before go-live and set expectations accordingly. Do not promise sub-second failover. |
| R7 | **etcd and Longhorn I/O contention on resource-constrained VMs.** etcd is latency-sensitive and can trigger leader elections under slow disk I/O. Longhorn also does significant I/O for volume replication. On a VM with a single shared disk, these workloads competing can cause intermittent etcd timeouts that look like cluster instability. | Med | Med | In Q2, run a sustained I/O load test with a representative module workload active and Longhorn replicating simultaneously on the minimum VM spec. If etcd heartbeat latency degrades, configure etcd to use a dedicated disk path separate from Longhorn volumes. Add etcd heartbeat latency to `arkctl diagnose` output as a baseline health metric. |
| R8 | **NTP unavailable in extreme air-gap multi-node deployments.** etcd consensus and TLS certificate validation both fail under clock skew between nodes. Some customers air-gap so completely that no NTP source is reachable. Multi-node clusters on independently-drifting VMs will eventually go unhealthy in ways that are hard to diagnose without knowing the root cause. | Low | High | Accept an optional NTP server address as an OVF property and configure chrony on each node to use it. If the field is empty, configure nodes to sync to the hypervisor clock via VMware Tools (available in most vSphere deployments). Add node clock drift to `arkctl diagnose` output. Flag extreme air-gap NTP scenarios in field engineer documentation. |
| R9 | **Upgrade bundle size becomes impractical for sneakernet delivery.** Shipping complete OCI tarballs per bundle keeps the format simple but grows with each additional component. If bundles reach 10–15 GB after several releases, customers transferring via USB or limited internal networks hit a practical ceiling. | Low | Med | Add bundle size as a measured pipeline gate from day one. Set an initial ceiling (OQ-3) and fail the build if exceeded. Review actual size trends at Q3 retrospective. Layer-delta bundles are the Q4 or post-Q4 investment if the growth trajectory becomes a problem — not something to build speculatively now. |
| R10 | **Engineer A is a single point of failure on the critical path.** A owns E1, E2, E4, and E8 — every epic on the critical path — and is the only team member with prior appliance experience. An extended absence (illness, departure, pull to another initiative) has no direct backup for the highest-risk work. | Low | High | Manager co-owns E4 as a deliberate backup. A writes ADRs as each epic completes — not at project end — so knowledge is continuously transferred. Manager pairs with A on E1 and E2 design sessions in Q1 to build familiarity with the appliance decisions. If A's departure becomes likely, escalate to leadership for backfill before the gap is critical. |

---

## Open questions

These are questions that cannot be answered by the engineering team alone and that would materially change the design or plan if answered differently.

**OQ-1 — Host OS variant.** Which openSUSE variant does Superna intend: MicroOS (immutable, transactional-update) or Leap (mutable, zypper-managed)? The upgrade architecture in Deliverable 1 depends on the answer. Design proceeds on MicroOS; if Leap is confirmed, the upgrade strategy changes materially. *Needed before E1 development begins.*

**OQ-2 — Minimum supported VM spec.** What is the smallest VM configuration (vCPU count, RAM, disk) Superna commits to supporting in customer deployments? This determines whether RKE2's baseline footprint is acceptable or whether the team must switch to K3s. *Needed before orchestrator choice is locked in Q1.*

**OQ-3 — Upgrade bundle size ceiling.** What is the maximum upgrade bundle size customers can practically receive and transfer given their typical distribution paths (customer portal download, USB transfer, internal file share)? This sets the pipeline gate threshold and determines whether complete OCI tarballs per bundle are feasible long-term. *Sales and field operations can answer this; engineering cannot.*

**OQ-4 — First module team commitment.** Which Superna product or module team will be the first to build against the `module.yaml` contract, and can their lead commit to reviewing the spec draft before end of Q1? We need a name and a commitment, not an agreement in principle. Without a real consumer reviewing the spec, R1 (spec invalidated by first consumer) has no effective mitigation.

**OQ-5 — CVE patch SLA.** What is the customer-facing commitment for time-to-patch on a critical security vulnerability once disclosed? Legal and customer success define this number; engineering builds the pipeline to meet it. Without a target, the pipeline automation has no design goal. *Needed before Q2 pipeline work is finalised.*

**OQ-6 — Regulatory certification targets.** Do any current or anticipated customers require specific compliance posture — FedRAMP, SOC 2, ISO 27001, FIPS 140-2, or specific approved cipher suites? FIPS-capable RKE2 builds exist but must be selected at build time. The answer changes what is baked into the OVA and asserted in the SBOM. *Needed before Q2 pipeline work begins.*

**OQ-7 — Fleet visibility on the 18-month roadmap.** Does Superna plan to build a product providing aggregate visibility across customer ARK appliances within the next 18 months? If yes, designing ARK's internal APIs to be queryable by a future fleet layer is worth doing now at low incremental cost. If no, it is premature optimisation. *A product strategy decision, not an engineering one.*

**OQ-8 — Known target customer environments.** Are there specific VMware versions, network hardware configurations, or storage array firmware versions in known target customers' environments that should be included in Q4 pre-production testing? Getting this list before Q3 allows the team to replicate the environment in the lab. *Needed before Q3 to inform lab setup.*

---

## Decisions the manager owns vs. decisions that need leadership

### Manager owns — not seeking approval, informing stakeholders

The following are engineering decisions with documented trade-offs. These are the manager's calls to make and defend. If a stakeholder disagrees, the manager explains the reasoning and either updates the decision record or holds the position.

- Orchestrator choice (RKE2 vs K3s) and all conditions under which that choice reverses
- HA networking approach (kube-vip in ARP/Layer-2 mode)
- Registry selection (Zot vs Harbor vs registry:2)
- Storage split (Longhorn multi-node, local-path single-node) and the migration path between them
- Build-vs-adopt-SUSE-Edge decision and the components borrowed from SUSE Edge
- `module.yaml` spec design, SupernaModule CRD API, and upgrade bundle format
- Test coverage targets, CI quality gates, code review standards, and sprint structure
- Engineering resourcing decisions within the allocated team (who owns which epic, coaching approach for C)

### Need from leadership — decisions the manager is not equipped to make alone

- **Minimum VM spec** (OQ-2): a product and sales decision. Depends on what Superna has committed to customers and the hardware floor in target data centres. Engineering measures RKE2's footprint; it does not set the supported floor.
- **CVE patch SLA** (OQ-5): a customer commitment that legal and customer success must define. Engineering builds the capability to meet whatever SLA is committed.
- **Host OS variant** (OQ-1): depends on Superna's existing OS tooling and product line standardisation decisions made before ARK. Engineering can argue for MicroOS; the decision about whether to break from or align with the existing OS pattern is a product architecture call.
- **Fleet visibility on the roadmap** (OQ-7): a product strategy decision with a 12–18 month time horizon. Affects API design choices in E4 if the answer is yes; has no impact if no.
- **Regulatory certification targets** (OQ-6): driven by which markets Superna is targeting. Legal and sales define it; engineering implements to it.

**The explicit boundary:** if the answer to any open question changes the system design materially, the manager redesigns the affected component and restarts that section of the roadmap — but does not restart the whole project. Staged discovery of constraints is normal engineering. The risk register exists so those discoveries are managed and communicated rather than absorbed silently.
