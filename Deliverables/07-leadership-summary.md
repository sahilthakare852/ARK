# Deliverable 7 — Leadership summary

*For the Chief Product & Engineering Officer*

---

## What we are delivering

ARK is the shared platform that hosts Superna's next-generation containerised products in customer environments — delivered as a familiar virtual appliance that runs fully offline on customers' own infrastructure, with no Superna-managed cloud or control plane involved.

Without ARK, every new containerised product builds and maintains its own incompatible platform stack: redundant cost, fragmented customer experience, and a support problem that compounds with each product we ship. With ARK, the platform is built once. Every product team writes product logic instead of infrastructure plumbing. Customers run one platform instead of one per product. The delivery model is unchanged from what customers already know and field operations already supports — a downloadable appliance image and offline upgrade bundles.

---

## Delivery timeline

| Milestone | Target | What it unlocks |
|-----------|--------|----------------|
| Walking skeleton | End of Q1 | Internal validation; no external dependency |
| Module contract published | End of Q1 | Product teams can start designing against ARK |
| **Platform complete ★** | **End of Q2** | **Product teams can start building on ARK** |
| Upgrade system + full operator CLI | End of Q3 | Platform is customer-operable day-to-day |
| Production ready — first customer trial | End of Q4 | First external deployment |

★ End of Q2 is the milestone that matters most to the rest of the organisation. From that point forward, product teams are unblocked to build on ARK without waiting for the platform team.

---

## What it costs

Three engineers full-time plus manager at roughly half IC capacity for 12 months. No new headcount required beyond the current team. No additional infrastructure spend beyond the Bitbucket Pipelines and JFrog Artifactory licences already in place.

The investment is front-loaded: the foundational work that everything else builds on lands in Q1 and Q2. H2 is delivery tooling, hardening, and the first customer deployment — lower-risk work building on a proven base.

---

## What is at risk and what we are doing about it

**Module contract — action required this month.** The technical interface that every product team builds against must be validated by a real product team before engineering locks the design at end of Q1. A wrong spec found in Q2 blocks every product team simultaneously. It is manageable now; expensive later. We need a named product team lead committed to a review before we proceed.

**Security patching — SLA needed before Q4.** A critical CVE in a core component after an OVA ships requires producing and distributing a patch quickly. The pipeline is being built to handle this automatically. It needs a customer-facing patch commitment as its design target — without that number, the capability exists but the promise cannot be made.

**Two technical assumptions that need confirmation before Q1 week three.** The upgrade architecture depends on which variant of openSUSE the appliance uses, and the infrastructure choice depends on the minimum VM spec Superna commits to supporting. Both have been designed under stated assumptions; both need confirmation before engineering proceeds past the prototype stage.

---

## What we need from leadership

**1 — Minimum VM spec.** What is the smallest server configuration Superna commits to supporting in customer deployments? Sales and product have this data. Engineering needs the number before end of Q1 week two.

**2 — Host OS variant confirmation.** Does Superna's existing product line use openSUSE MicroOS or openSUSE Leap? The upgrade architecture differs materially between the two. Needed before Q1 week three.

**3 — Named product team for spec review.** Which product team lead will review the module contract before end of Q1? A name and a commitment, not an agreement in principle. This is the single highest-leverage action available to unblock the project at this stage.

**4 — CVE patch SLA.** What is the customer-facing commitment for time-to-patch on a critical security vulnerability? Legal and customer success own this number. Engineering builds the pipeline to meet it.
