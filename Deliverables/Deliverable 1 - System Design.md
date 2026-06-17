# Deliverable 1 — System Design

## Problem decomposition

Based on the provided Functional Requiremnts I have created groups to disect the problem into smaller chunks.

**Functional Requirements:**

1. Customers must be able to install the system by deploying a virtual appliance onto their hypervisor and providing a small set of configuration values (network, hostname, credentials) at deploy time.
2. The system must come up to a working state on first boot without an operator logging in.
3. The system must be able to host one or more Superna products (“modules”) and must support adding, removing,and upgrading those products independently of the base platform.
4. Customers must be able to install and upgrade the system and its modules with no outbound internet access.
5. The system must support in-place upgrades that match the operational shape of Superna’s legacy product upgrades— what customers do today. Upgrades must be idempotent and must not cause data loss.
6. The system must support deployments that remain available when a node is lost.
7. The system must support deployments where clients can reach it only by IP address (no DNS).
8. Operators must be able to perform day-2 tasks (inspect health, restart components, collect diagnostics, manage modules) via a single CLI without needing deep knowledge of the underlying runtime.
9. The system must accommodate modules with varying storage, network, and dependency requirements without each module re-solving these concerns.
10. Multiple Superna products installed on the same appliance must be able to share common services (ingress, TLS,identity, container registry) rather than each shipping their own.

**Grouping:**

1. **Unattended bootstrap and idempotent lifecycle** (FR1, FR2, FR5) — the platform brings itself from an empty VM to a serving state with zero human input, and moves itself from version N to N+1 the same way, safely and repeatably.

2. **Module hosting and lifecycle contract** (FR3, FR9) - a stable boundary between platform and product so each module team builds against an API, not against RKE2(Platform) primitives directly.

3. **Shared platform services** (FR10) — a small set of services every module gets for free rather than re-implementing.

4. **Infrastructure-optional connectivity** (FR7, plus the no-DNS/no-LB/no-CA/no-IdP constraints) — the system supplies its own substitutes for things the environment is never guaranteed to have.

5. **Resilience** (FR6, NFR availability) — survive node loss with minimised client impact in the multi-node shape; explicitly no HA target in the single-node shape. Both are first-class.

6. **Air-gapped operation by construction** (FR4, "hard requirement" NFR) — every lifecycle action works at zero network egress by default, not as an exception path.

7. **Day-2 operability surface** (FR8) — one CLI abstracting the runtime so field engineers and customer operators don't need Kubernetes literacy.

8. **External system coexistence** (storage, hypervisor, optional identity/DNS constraints) — ARK coexists with what's already there and exposes generic extension points, never vendor-specific code.

---

## Component model

Components are organised into four layers. Each component is listed with what it owns and what state it holds.

### Networking and access layer

**HA networking with no DNS, no LB:** 
kube-vip(ARP/Layer-2 mode, static pod on control-plane nodes) — owns the control-plane API VIP and the ingress Service VIP, plus leader election via Kubernetes lease and gratuitous ARP on failover. Active-passive: one node owns the VIP at any moment. 

**Ingress controller** (Traefik vs ingress-nginx) 

| Feature | Traefik | Ingress-NGINX |
| :--- | :--- | :--- |
| **Language & Security** | Go (memory-safe, static linking) | C & Lua (susceptible to configuration injections) |
| **Configuration Model** | Dynamic, API-driven (Zero-downtime hot reloads) | Static file generation (Requires NGINX reloads) |
| **Customizations** | Elegant Middleware CRDs and Plugins | Bulky annotations and raw NGINX config snippets |
| **TLS/Certificates** | Built-in Let's Encrypt / ACME integration | Requires external tools (like cert-manager) |
| **Observability** | Native web UI dashboard & rich metrics | Relies entirely on external Prometheus exporters |
| **Project Status** | Highly active, fully backed by Traefik Labs | Archived / Retired (Community version ended) |

Based on this I choose Traefik.

### Platform services layer

**registry** — Three Options: Zot Registry and Harbor registry and bare registry:2

Zot Registry owns OCI image and Helm chart storage plus garbage collection. 

Zot is lightweight more suitable for edge appliance, Harbor has more security features but heavier in size. Bare registry is very basic and does not have garbage collection. 

Therefore choosing Zot. 

**cert-manager + internal self-signed CA** — owns certificate issuance and rotation for every client-facing endpoint. Holds the CA private key, which is the single most sensitive piece of state in the whole system. In HA deployments, the CA key lives as a Kubernetes Secret replicated across control-plane nodes via etcd — no manual distribution step required. IP only. 

**Identity service** — owns local admin and operator accounts plus optional federation configuration to customer LDAP, AD, or OIDC. 

**Module operator** (custom controller + SupernaModule CRD) — owns the install, upgrade, remove, and health lifecycle for every module. Translates "install module X at version Y" into a namespace, ResourceQuota, LimitRange, NetworkPolicy, PVC provisioning, and Helm release. 

### Modules layer

Each module owns its own functionality, its own data schema, and its own RPO/RTO. State lives on PVCs provisioned by the storage layer in the module's own namespace. ARK's guarantee stops at "the volume survives a platform upgrade or node loss." What is inside that volume being internally consistent is the module's problem, not ARK's.

### Orchestration and storage layer

**RKE2** or K3s 

(control plane: API server, scheduler, controller-manager, etcd) — owns cluster state, scheduling, RBAC, and encrypted Secrets at rest. 

| Factor | K3s ✗ | RKE2 ✓ |
|--------|--------|---------|
| **Default datastore** | SQLite (single-node); etcd is a manual opt-in | etcd always, regardless of node count |
| **Single → HA scaling** | Requires SQLite → etcd migration; two support runbooks | Cluster expansion only — one operational model |
| **Memory footprint** | ~1.2 GB — lighter ✓ | ~2–3 GB — fixed overhead ✗ |
| **CIS hardening** | Manual, add-on | Ships by default |
| **FIPS compliance** | Not available | FIPS-capable builds included |
| **Air-gap support** | Yes | Yes |
| **openSUSE compatibility** | Yes | Yes |
| **Regulated industry fit** | Needs additional hardening work | Ready out of the box |
| **Reversal condition** | Switch to K3s if RKE2 baseline exceeds 20% of minimum VM RAM on Q1 spike test | — |

**Control-plane topology rule**: exactly 3 control-plane nodes for any HA deployment.Every node beyond three joins strictly as a worker — module capacity scales by adding workers, not by growing the etcd membership. A 2-node topology must be actively blocked: losing either node of a 2-node cluster loses quorum and takes the cluster read-only. 

**Longhorn** (multi-node) / **local-path** (single-node) — owns PV provisioning, replication where applicable, and snapshots. State: the actual block data behind every module's PVC — customer data. Longhorn provides synchronous replication to a configurable replica count before acknowledging writes. Honest cost: a customer scaling from single-node to HA needs an explicit storage migration step (provision Longhorn, migrate volumes off local-path), which is why this single-node→HA path must be designed as one journey rather than two separate products.

**Host OS** (openSUSE MicroOS, assumed) — owns the node's base OS and atomic update/rollback via Btrfs snapshots (transactional-update). State: a read-only root filesystem plus explicitly separate writable subvolumes for persistent data — kept outside the snapshot boundary on purpose. A platform rollback (booting the previous Btrfs snapshot) reverts the OS and platform binaries but never touches PVC mount paths.


### Standing components (outside the layer model)

**Bootstrap agent** — a one-shot, idempotent systemd service that runs exactly once on first boot. Reads OVF properties, writes the RKE2 config, starts `rke2-server`. No persistent state. If RKE2 is already running when the bootstrap agent starts, it exits immediately.

**ARK CLI (`arkctl`)** — a stateless client that talks to the API server and the module operator. The single surface that FR8 asks for. No local config file required beyond the kubeconfig written at bootstrap.

**Self-extracting upgrade bundle** — not a running component, but the boundary artifact between Superna's build pipeline (which has internet access) and the appliance (which never does). The pipeline bakes everything the appliance needs into the bundle at build time.

Diagram: SystemDesign1.png

---


## Data flow

### Flow 1 — First-boot bootstrap

The hypervisor supplies all configuration as OVF vApp properties at power-on. On VMware, cloud-init reads these via the GuestInfo interface. On other OVF-capable hypervisors, cloud-init reads from an `ovf-env.xml` file on a mounted virtual CD-ROM. Both paths produce identical bootstrap agent inputs.

The bootstrap agent writes the RKE2 server configuration and starts the `rke2-server` systemd service. RKE2 bootstraps etcd, starts the API server, and issues its own internal cluster PKI for node-to-node trust. Once the API server is serving, the bootstrap agent installs the platform Helm charts from the Zot registry pre-loaded in the OVA — in dependency order: kube-vip first (static pod, no Helm dependencies), then cert-manager, Zot (PVC provisioned now), ingress controller, module operator, identity service. cert-manager generates the self-signed root CA and issues the first ingress TLS certificate, with the VIP's IP address in the SAN.

The system reaches working state — all platform pods Running, `arkctl cluster status` healthy — without an operator logging in. End-to-end time: under 10 minutes from power-on.

![Bootstrap flow] BootStrapflow.png

### Flow 2 — Module install and lifecycle

`arkctl module install <bundle.tar.gz>` extracts and validates the `module.yaml` manifest before touching the cluster: platform version compatibility checked, every declared dependency verified as installed at or above the required version. On success, the CLI submits a SupernaModule CR to the API server.

The module operator reconciles: creates namespace → applies ResourceQuota and LimitRange from resource declarations → applies default-deny NetworkPolicy with allow rules for platform services and declared inter-module dependencies → provisions PVCs from storage declarations → runs `helm install` pulling from Zot. CR status transitions: `Pending → Provisioning → Installing → Running`.

Remove is the reverse: Helm uninstall → PVC deletion (if `retention: delete`) or preservation (if `retention: retain`) → CR deletion.

### Flow 3 — Runtime request path

Client at IP address → TCP 443 → kube-vip VIP (leader holds VIP via ARP) → ingress controller (TLS terminated using cert-manager cert with IP SAN) → module Service (ClusterIP) → kube-proxy load-balances across module pod endpoints → module pod → PVC (Longhorn for multi-node, local-path for single-node). No cluster state is created on a normal request; all routing config already lives in etcd.

### Flow 4 — Node failure and HA failover

Control-plane node goes unreachable. etcd retains quorum (2 of 3 members). If the failed node held the kube-vip leader lease, the lease expires, a surviving node wins election, and emits a gratuitous ARP. Network switches update their ARP cache — seconds, not zero, and some hardware caches longer than the broadcast suggests. In-flight connections to the old VIP owner die; clients retry. kube-scheduler marks the node NotReady and evicts pods; Longhorn reconstructs replicas on surviving nodes.

### Flow 5 — Platform upgrade (checkpointed)

`arkctl upgrade apply <bundle.tar.gz>` validates the bundle signature, disk space, and cluster quorum health. Then executes four checkpointed stages, writing a state file before each:

- Checkpoint 1: Stage new OS snapshot via `transactional-update`
- Checkpoint 2: Helm upgrade for each platform chart in dependency order
- Checkpoint 3: Module upgrades in declared dependency order
- Checkpoint 4: Reboot into new OS snapshot

On any failure: re-running `arkctl upgrade apply` resumes from the last incomplete checkpoint, not from the beginning. PVC mount paths are excluded from the Btrfs snapshot boundary — a platform rollback (booting the previous snapshot) never touches module data.

![Upgrade checkpoint flow]Upgradeflow.png


### Consistency model

**etcd** (strong consistency): all Kubernetes objects — pods, services, secrets, CRD instances including every SupernaModule CR, Helm release secrets. Source of truth for everything the orchestrator knows. RKE2 encrypts Secrets at rest by default.

**PVC storage** (module-owned, Longhorn-synchronous within a volume): cross-volume consistency is the module's responsibility — this is the precise boundary the brief draws when it says "RPO/RTO targets are owned by individual modules, not by ARK."

**Zot registry** (content-addressed, immutable): a given OCI digest is the hash of its content. No eventual consistency risk. Garbage collection removes only unreferenced content.

---

## External interfaces

**Hypervisor** (deploy-time, read-once): cloud-init reads OVF vApp properties at first boot and never again. VMware: GuestInfo API. Other OVF: `ovf-env.xml` on virtual CD-ROM. Parameters: IP/mask/gateway, hostname (display only), admin credential, optional VIP, optional NTP server.

**Customer network** (runtime, inbound only by default): HTTPS to VIP:443 (client traffic), SSH to node IPs (key-based, support access). ARK never initiates outbound connections to Superna infrastructure. NTP to customer-provided server if configured.

**Customer storage** (optional, runtime, bidirectional): NFS CSI driver. Modules declare NFS mounts in `module.yaml`; ARK provisions the PVC. No vendor-specific code for any NAS product.

**Customer identity** (optional, additive): local identity service handles auth by default. Optional LDAP/AD/OIDC federation added via `arkctl identity configure`. Federation failure falls back to local accounts; it never breaks the platform.

**Superna build pipeline** (pre-deploy, one-way from pipeline): produces OVA and upgrade bundles, pushes to Artifactory. The appliance never calls back to Superna's infrastructure. This is the build-time-vs-runtime distinction that makes air-gap a structural property rather than a mode.

**Humans**: `arkctl` (all day-2 operations, stateless client) and SSH (break-glass, key-based only).

---

## Key design decisions

### Decision 1 — Orchestrator: RKE2 over K3s

**Context**: ARK needs a Kubernetes distribution that works on openSUSE-based hosts, handles both single-node and HA multi-node without a different code path for each, supports air-gapped install natively, and fits on modestly-sized customer VMs.

**Options considered**:

*K3s* — designed for edge and resource-constrained environments. ~1.2 GB RAM for a server node. Defaults to SQLite for single-node; embedded etcd available as a separate opt-in. The hidden cost: a customer adding nodes for HA needs a SQLite → etcd migration — a non-trivial operational event that produces two different support runbooks depending on which datastore that appliance is running. Not FIPS-capable or CIS-hardened by default.

*RKE2* — designed for regulated environments. Higher baseline footprint (~150% higher memory than a kubeadm baseline). Always uses etcd regardless of node count. Ships CIS Benchmark hardening and FIPS-capable builds as defaults. Single-node and multi-node run the same code path at different scale.

*Talos Linux* — philosophically the most aligned option (immutable OS, API-driven, purpose-built for Kubernetes). Disqualified by one constraint: it is its own OS, not openSUSE-based. Named and ruled out explicitly rather than ignored.

**Chosen**: RKE2.

**Rationale**: The footprint tax is roughly fixed and does not grow with modules. The bigger cost of K3s is not the RAM — it is the two-datastore problem. "Single-node and multi-node as two shapes of one product" is a structural design principle, and RKE2 gives one operational model from node one to node twelve. The compliance posture (CIS hardening, FIPS) maps to the regulated-industry customer base without additional configuration.

**Reversal condition**: If a Q1 performance spike shows RKE2's baseline consuming more than 20% of total RAM on the minimum supported VM config before a single module pod runs, switch to K3s and accept the migration cost as the lesser evil. This is a concrete, measurable test, not a hedge.

---

### Decision 2 — HA networking: kube-vip (ARP/Layer-2) over alternatives

**Context**: FR7 requires IP-only client access. The brief prohibits requiring the customer to stand up a load balancer. Something on the appliance itself must own the VIP.

**Options considered**:

*keepalived + HAProxy* — the traditional Linux HA stack. Not natively Kubernetes-aware; VIP assignment is configuration-managed separately from the Kubernetes objects it fronts.

*MetalLB* — a Kubernetes-native Service LoadBalancer controller. Adds meaningful capability if modules need many independently-routable IPs. Overkill for the present requirement: ARK needs exactly two externally-routable VIPs (control-plane API and ingress). MetalLB's IP pool management solves a problem we do not have.

*kube-vip in ARP/Layer-2 mode* — a static pod on control-plane nodes that elects a leader via a Kubernetes lease and emits gratuitous ARP for the VIP. Natively Kubernetes-aware. Handles both the control-plane VIP and Service-type LoadBalancer VIPs without a second component. Zero external dependencies.

*BGP mode (either tool)* — requires customer switch/router to peer over BGP with appliance nodes. Functionally equivalent to requiring a load balancer, just one layer down the stack. Ruled out by the same constraint.

**Chosen**: kube-vip in ARP/Layer-2 mode.

**Rationale**: The only option that survives the customer-environment constraints intact. Failover takes seconds, in-flight connections die and require client retry, some network gear caches ARP entries longer than expected — this is documented kube-vip behaviour, and the reason the NFR says "client impact minimised" rather than "zero impact."

**Reversal condition**: If a future module requirement emerges for a pool of independently-routable IPs rather than a single shared ingress VIP, add MetalLB alongside kube-vip. The two coexist; kube-vip holds the control-plane VIP, MetalLB manages the pool. Forward-compatible, not a migration.

---

### Decision 3 — Borrow SUSE Edge component choices; do not adopt SUSE Edge as a platform

**Context**: SUSE ships a production reference stack (SUSE Edge) that targets exactly this problem — air-gapped, openSUSE-based, appliance-shaped Kubernetes. Its component choices (RKE2, kube-vip, Longhorn, Btrfs/transactional-update, Zot) are the same components this design arrives at independently. The question is whether to adopt SUSE Edge wholesale or borrow the individual components.

**Options considered**:

*Adopt wholesale* — take Rancher Manager, Fleet, Elemental. Faster time-to-market; inherit SUSE's CVE cadence and air-gap tooling. The cost: Rancher's entire design point is "one control plane manages many downstream clusters." Installing it on every appliance to manage one local cluster deploys full fleet-management machinery that serves no function here. Every support call teaches customers Rancher concepts instead of Superna concepts. Product roadmap coupled to SUSE's release cadence for the substrate of our own product.

*Borrow components, build thin Superna layer* — take RKE2, kube-vip, Longhorn, Btrfs/transactional-update, Zot, cert-manager as standalone components; write a Superna-owned module operator, CLI, and upgrade orchestrator on top. More upfront engineering. The product surface is entirely Superna's.

**Chosen**: Borrow components; build the thin Superna layer.

**Rationale**: "No Superna-side control plane, each appliance managed independently" is a structural requirement in the brief. Rancher's fleet-management model fights this at its core design, not at the edges — Fleet exists to coordinate many clusters from one control plane, which is exactly what the brief rules out. This is not a stylistic preference; it is a direct conflict between Rancher's architecture and the explicit out-of-scope list.

**Reversal condition**: If Superna's roadmap later includes a fleet-visibility product above ARK — aggregate health across customer appliances — then Rancher's multi-cluster model becomes a genuine asset rather than a mismatch. At that point the calculus tilts toward Rancher for the fleet layer with ARK as a managed downstream cluster. That is the condition to revisit this decision; it should be stated in the design record so future engineers know when the reasoning changes.

---

## Stated assumptions

- Host OS is openSUSE MicroOS (immutable, transactional-update). If Superna's existing product line uses openSUSE Leap, the upgrade architecture reverts to a scripted installer pattern. Flagged as open question OQ-1 in the risk register.
- OVA deploys with an empty platform (no default module baked in). "Working state on first boot" means the platform is healthy and awaiting a module install command, not that a specific module is pre-installed.
- Minimum supported VM config is sufficient for RKE2's baseline footprint. Flagged for a Q1 validation spike; reversal condition documented in Decision 1.
- Customer NAS integration is generic NFS. No vendor-specific PowerScale or NetApp code is in scope for ARK.
- "No Superna-side control plane" means no central service the appliance phones home to, no fleet management, no push-based upgrades. Customer-initiated, customer-driven, forever.
