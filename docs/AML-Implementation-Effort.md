# Implementation Effort — v1 (Managed Compute) vs. v2 (AML on AKS)

> **Status:** Draft for review
> **Owner:** ML Platform Team
> **Last updated:** May 2026
> **Audience:** ML Platform Engineers, Programme Managers, Cloud Architects, FinOps, Engineering Managers
> **Companions:** [`AML-Scale-Out-Architecture.md`](AML-Scale-Out-Architecture.md), [`AML-AKS-ScaleOut-Architecture.md`](AML-AKS-ScaleOut-Architecture.md)

---

## 0. Starting state

- **One subscription** today, hosting all **300 workspaces** (100 products × 3 environments).
- Every workspace has its own 14-node `AmlCompute` cluster.
- ALZ + subscription-stamping pipeline already exist.
- Maintenance is easy (one sub, one team, one quota table) **but scale is the wall** — endpoints, compute targets, and core quotas are all hit.

Both target architectures require **re-stamping every workspace** into a new subscription topology because AML workspaces cannot be cleanly moved between subscriptions with all their artefacts (models, runs, datastores, endpoints).

## 1. How to read the tables

Effort columns:

- **Effort (PD)** — person-days of focused work for the named persona/team. PD = 8 hours of one engineer.
- **Elapsed** — wall-clock time including waits (procurement, RBAC approvals, support tickets, change windows, batches of 25 product migrations, etc.).
- **Persona / team** — primary owner. R = Responsible, A = Accountable, C = Consulted, I = Informed.
- **Risk** — L / M / H, with the dominant risk reason.

Effort estimates assume a **3 – 5 person platform team** with one tech lead, plus access to existing ALZ / network / identity teams as consulted parties. They are planning estimates; treat as ranges, not commitments.

---

## 2. v1 — Managed compute (per-product `AmlCompute`) — implementation effort

Final state: **12 AML-dedicated subscriptions (4 PG × 3 env)**, 25 products per sub, two new pipeline modules, every workspace re-stamped into its destination sub.

### 2.1 Foundation — pipeline & shared modules (one-time, before any product migration)

| # | Task | What changes | Why | Persona / team | Effort (PD) | Elapsed | Risk |
|---|---|---|---|---|---|---|---|
| F1 | Design review with ALZ + Networking + Identity | RACI, MG placement, subscription naming, RBAC bootstrap pattern agreed | Cannot extend the stamping pipeline without alignment from owners | Platform tech lead (R), ALZ/Net/Id (C) | 5 | 2 weeks | M — discovery may surface ALZ gaps |
| F2 | Build `aml-sub-shared` module (Bicep/TF + AVM) | New module: Premium ACR, central-DNS zone links, diagnostic settings, Quota Group bootstrap | Must run once per new subscription before any workspace lands | Platform engineer (R), ALZ pipeline owner (C) | 10 | 2 weeks | L |
| F3 | Build `aml-product-stamp` module | New module: workspace, ADLS, KV, AI, UAMI, 14-node `AmlCompute`, batch endpoint, PEs, tags, CMK (prd) | Authoritative AML stamp per product per env | Platform engineer (R) | 15 | 3 weeks | M — PE/DNS wiring tends to need iterations |
| F4 | Wire allocator (`pg_index = ceil(product_id / 25)`) | Determines target sub from `product_id`; drives stamping pipeline | Without it, onboarding is manual | Platform engineer (R) | 4 | 1 week | L |
| F5 | Quota raise automation via Microsoft.Quota REST API | Pipeline fires VM-core, endpoint, compute-target raises after sub vending | Default quotas insufficient for 25 × 14-node clusters | Platform engineer (R), Subscription owner (A) | 6 | 2 weeks (incl. first ticket cycle) | M — Microsoft response time variable |
| F6 | Naming, tagging, RBAC standards (AML-specific) | New conventions documented and enforced via Azure Policy | Operability and chargeback at scale | Platform tech lead (R), Cloud Governance (C) | 4 | 1 week | L |
| F7 | Vend the 12 destination subscriptions | Existing pipeline creates `aml-{env}-pg01..04` × 3 envs | Targets must exist before re-stamping starts | ALZ pipeline owner (R), Platform (A) | 2 | 2 weeks (incl. ticketing) | L |
| F8 | Run `aml-sub-shared` against the 12 subs | ACR + DNS links + diagnostics established | Pre-requisite for any product stamp | Platform engineer (R) | 3 | 1 week | L |
| F9 | Pilot end-to-end with 2 throwaway products | Validate stamp + endpoint deploy + batch job | De-risks bulk migration | Platform + 1 product team (R), DS lead (C) | 8 | 2 weeks | M |
| **Foundation subtotal** | | | | | **57 PD** | **~6 weeks elapsed** | |

### 2.2 Migration of the 100 existing products (× 3 envs each = 300 workspaces)

Workspaces are re-provisioned, not moved. Migration is per-product, in waves.

| # | Task | What changes | Why | Persona / team | Effort (PD per product) | Elapsed |
|---|---|---|---|---|---|---|
| M1 | Allocator computes target sub per product | Pinned membership recorded in CMDB | Determinism; never reshuffle | Platform pipeline (auto) | 0.05 (auto) | seconds |
| M2 | `aml-product-stamp` runs in dev / tst / prd | New workspace + cluster + endpoint + PEs created | Target shells ready | Platform pipeline (auto) | 0.5 (oversight only) | ~15 min wall, batched |
| M3 | Re-register models from old workspace to new | Model artefacts copied; lineage rebuilt | Workspaces can't be moved | Product DS team (R), Platform (C) | 1 – 3 | 2 – 5 days per product |
| M4 | Re-point datastores to project data lake | Datastore registrations recreated; SAS/MI swapped | Old workspace credentials become invalid | Product DS team (R) | 0.5 – 1 | 1 – 2 days |
| M5 | Re-create batch endpoints + deployments | Endpoint + deployment YAMLs re-applied to new workspace | New workspace has none | Product DS / ML eng (R) | 1 – 2 | 2 – 4 days |
| M6 | Re-run smoke batch + result diff against legacy | Confirms prediction parity | Catches silent regressions | Product DS (R), Model tester (C) | 1 | 2 days |
| M7 | Cutover scheduler/upstream callers | Triggers and consumers point at new endpoint URL | End-to-end live | Product app team (R), Platform (C) | 0.5 – 1 | 1 – 2 days |
| M8 | Decommission old workspace | Old workspace + cluster + endpoint deleted; cost stops | Avoid dual-running | Platform (R), Product (A) | 0.25 | 1 day (after stable on new) |
| **Per product (3 envs) subtotal** | | | | | **5 – 9 PD per product** | **2 – 3 weeks per product per wave** |

**Bulk migration plan (100 products):** 10 waves × 10 products in parallel, each wave runs ~3 weeks end-to-end → **~7 months elapsed** with 2 platform engineers riding shotgun and 0.5 FTE per product team during their wave.

| Aggregate | Value |
|---|---|
| Total per-product effort | 100 × ~7 PD avg = **~700 PD** distributed across product teams + platform |
| Platform-team carry during migration | ~80 PD over 7 months |

### 2.3 Process / people changes (v1)

| # | Change | What changes | Why | Persona / team | Effort (PD) | Elapsed |
|---|---|---|---|---|---|---|
| P1 | New onboarding intake form (no `env` field) | Single intake creates dev/tst/prd in parallel | Eliminates ticket-per-env pattern | Platform PM (R), Product leads (C) | 3 | 2 weeks |
| P2 | Quota operating model + Quota Groups setup | Pool VM-family quotas across same-env subs | Smooths peaks across 4 subs/env | Platform + FinOps (R), MS support (C) | 6 | 3 weeks |
| P3 | RBAC redesign per workspace × per env | Entra groups by product × env; PIM for prd | Sub fan-out multiplies group count 12× | Identity team (R), Platform (C) | 8 | 3 weeks |
| P4 | FinOps chargeback wiring | Tags + per-sub budgets + alerts at 75 % | 12 subs need rolled-up reporting | FinOps (R), Platform (C) | 5 | 2 weeks |
| P5 | Runbooks: vend new PG sub, raise quotas, decommission | Documented for support team | Operability when platform team is off-shift | Platform (R), Ops (C) | 4 | 1 week |
| P6 | Product-team training + comms | Each product team learns the new workspace URL, endpoint pattern, CI/CD changes | Adoption blocker if skipped | Platform (R), All product teams (I) | 6 | spread over migration |
| P7 | Day-2 ops re-org | Platform team needs on-call for 12 subs vs. 1 | Operational scope grows | Engineering manager (A), Platform (R) | 4 | 4 weeks (HR + rota) |
| **Process/people subtotal** | | | | | **36 PD** | **~6 weeks elapsed (overlaps migration)** |

### 2.4 v1 totals

| Bucket | Effort (PD) | Elapsed |
|---|---|---|
| Foundation (technical) | 57 | ~6 weeks |
| Per-product migration (100 products) | ~700 (across product teams) + ~80 platform | ~7 months |
| Process / people | 36 | ~6 weeks (parallel) |
| **Total** | **~870 PD** | **~9 – 10 months end-to-end** |

---

## 3. v2 — AML on AKS (`KubernetesCompute`) — implementation effort

Final state: **9 AML-dedicated subscriptions (3 PG × 3 env)** at 1,000-product capacity (3 needed for the 100-product baseline; design is ready for growth without further sub fan-out). Compute pooled into one shared AKS cluster pair per env per sub. Three new pipeline modules.

### 3.1 Foundation — pipeline, AKS platform, shared modules (one-time)

| # | Task | What changes | Why | Persona / team | Effort (PD) | Elapsed | Risk |
|---|---|---|---|---|---|---|---|
| F1 | Design review with ALZ + Networking + Identity + AKS Platform team | Same as v1 plus AKS platform-team RACI, GitOps tooling choice (Argo CD/Flux), namespace isolation pattern | More moving parts and more teams in scope | Platform tech lead (R), ALZ/Net/Id/AKS (C) | 8 | 3 weeks | M |
| F2 | Build `aml-sub-shared` module | Premium ACR (geo-replicated for prd), DNS zone links, diagnostics, AML Registry per env | Same shape as v1; ACR sized for cross-product image density | Platform engineer (R) | 10 | 2 weeks | L |
| F3 | Build `aks-platform-cluster` module | **New**: private AKS, API server VNet integration, Azure CNI Overlay, Workload Identity, OIDC issuer, system + cpu-od + cpu-spot + gpu-od + gpu-spot pools, AML extension (`enableTraining=true enableInference=true`), instance-type CRDs, baseline NetworkPolicy + ResourceQuota templates, Defender for Containers, Cost Analysis add-on, GitOps bootstrap | The biggest new asset; runs once per env per sub | Platform engineer + AKS engineer (R), Net/Sec (C) | 25 | 4 – 5 weeks | **H** — first-time-right hardening on private cluster + extension |
| F4 | Build `aml-product-stamp` module | Same as v1 **minus** `AmlCompute`, **plus** namespace creation, ResourceQuota/NetworkPolicy, `az ml compute attach --type Kubernetes`, default instance-type binding | Per-product stamp pivots from compute-owning to compute-attaching | Platform engineer (R) | 12 | 3 weeks | M |
| F5 | Curated instance-type catalogue (CRDs) | `cpu-small/medium/large`, `gpu-small-spot`, `gpu-large` | Data-scientist-facing contract; abstracts SKUs | Platform engineer + DS lead (R) | 4 | 1 week | L |
| F6 | GitOps repo + bootstrap (Argo CD or Flux) | Cluster state (instance types, RBAC, taints, quotas) declared in Git, reconciled per cluster | Replayable across cluster pairs and regions | Platform engineer (R), AKS team (C) | 8 | 2 weeks | L |
| F7 | Allocator (`pg_index = ceil(product_id / 334)` at 1k scale, currently `/ 100`) | Drives stamping pipeline | Determinism | Platform engineer (R) | 4 | 1 week | L |
| F8 | Quota raise automation (VM cores **per AKS pool**, endpoint cap raise) | One quota raise per cluster pool, not per product cluster | Pooled quotas — far fewer raises | Platform engineer (R) | 5 | 2 weeks | M — endpoint raise approval cycle |
| F9 | Naming, tagging, RBAC standards (AML + AKS namespaces) | Workspace = namespace = `ws-p{nnn}` mapping codified | Operability + chargeback | Platform tech lead (R), Cloud Governance (C) | 4 | 1 week | L |
| F10 | Vend the 3 destination subscriptions | Existing pipeline creates `aml-aks-{env}-pg01` × 3 envs | 3 instead of 12 | ALZ pipeline owner (R) | 1 | 2 weeks (ticketing) | L |
| F11 | Run `aml-sub-shared` + `aks-platform-cluster` | 3 AKS clusters provisioned and hardened | Compute substrate ready | Platform + AKS engineer (R) | 6 | 2 weeks (incl. validation) | M |
| F12 | Pilot end-to-end: 2 throwaway products on the cluster | Validate namespace isolation, instance-type selection, batch job, batch endpoint | Multi-tenant cluster correctness is the new risk | Platform + AKS + 1 product team (R) | 12 | 3 weeks | **H** — noisy-neighbour, taint, GPU-share validation |
| F13 | Multi-region prep (paired region for prd, AML Registry replication) | Standby AKS cluster + registry replication | DR + endpoint-cap relief | Platform engineer (R) | 8 | 2 weeks | M |
| **Foundation subtotal** | | | | | **107 PD** | **~10 weeks elapsed** | |

### 3.2 Migration of the 100 existing products (× 3 envs each = 300 workspaces)

Same migration shape as v1 (workspaces are re-provisioned, not moved), with two differences: **no per-product cluster to provision**, but **per-product namespace + quota + attachment** to validate.

| # | Task | What changes | Why | Persona / team | Effort (PD per product) | Elapsed |
|---|---|---|---|---|---|---|
| M1 | Allocator computes target sub | Pinned in CMDB | Determinism | Platform pipeline (auto) | 0.05 | seconds |
| M2 | `aml-product-stamp` runs in dev / tst / prd | New workspace + namespace + ResourceQuota + KubernetesCompute attachment + endpoint shell | Target shells ready, no compute provisioning | Platform pipeline (auto) | 0.4 (oversight) | ~12 min wall, batched |
| M3 | Re-register models | Same as v1 | Workspaces can't be moved | Product DS (R) | 1 – 3 | 2 – 5 days |
| M4 | Re-point datastores | Same as v1 | Same | Product DS (R) | 0.5 – 1 | 1 – 2 days |
| M5 | Re-create batch endpoints + deployments **on `KubernetesCompute`** | Endpoint YAML targets `aks-shared-{env}` instead of `cpu-cluster`; instance type chosen | New target + new abstraction | Product DS / ML eng (R), Platform (C first 5 products) | 1 – 2.5 (slightly higher 1st time per team) | 2 – 5 days |
| M6 | Smoke batch + parity diff | Same as v1 | Same | Product DS, Model tester (R) | 1 | 2 days |
| M7 | Cutover upstream callers | Same as v1 | Same | Product app team (R) | 0.5 – 1 | 1 – 2 days |
| M8 | Decommission old workspace + old `AmlCompute` cluster | Old per-product cluster deleted — **immediate large cost saving** | Avoid dual-running; v2 cost benefit starts here | Platform (R) | 0.25 | 1 day |
| **Per product (3 envs) subtotal** | | | | | **5 – 9 PD per product** | **2 – 3 weeks per product per wave** |

**Bulk migration plan (100 products):** 10 waves × 10 products in parallel → **~7 months elapsed**, same shape as v1. **Effort per product is essentially the same** as v1 (the bulk of migration work is data/model/endpoint re-registration, which both designs share). The platform-team carry is slightly higher in the **first** few waves due to the new `KubernetesCompute` pattern, then drops below v1 because there is no per-product cluster to baby-sit.

| Aggregate | Value |
|---|---|
| Total per-product effort | 100 × ~7 PD avg = **~700 PD** distributed across product teams + platform |
| Platform-team carry during migration | ~70 PD over 7 months (lower than v1 because no per-product compute provisioning) |

### 3.3 Process / people changes (v2)

v2 adds genuinely new capabilities (a multi-tenant compute platform), so process/people effort is materially higher than v1.

| # | Change | What changes | Why | Persona / team | Effort (PD) | Elapsed |
|---|---|---|---|---|---|---|
| P1 | New onboarding intake form (no `env` field) | Same as v1 | Same | Platform PM (R) | 3 | 2 weeks |
| P2 | **Stand up an AKS Platform team / capability** (or assign owners within Platform) | Dedicated owners for cluster lifecycle, upgrades, GitOps, cost, on-call | Multi-tenant cluster is now business-critical infra | Engineering manager (A), Platform (R), HR (C) | 10 (planning) + ramp | 4 – 8 weeks (recruit/up-skill) |
| P3 | Up-skill Platform team on Kubernetes + AML extension | Training, hands-on labs, on-call certification | Existing AML-only team likely lacks Kubernetes depth | Engineering manager (A), Platform (R) | 15 (training time) | 4 – 6 weeks |
| P4 | Up-skill product DS / ML-eng teams on `KubernetesCompute` semantics | Workshops on instance types, namespaces, ResourceQuota visibility, spot evictions | New abstraction in their job YAML | Platform DevRel (R), Product teams (I) | 8 | 4 weeks (rolling) |
| P5 | Quota operating model | One raise per cluster pool, not per product family | Different Microsoft-support workflow | Platform + FinOps (R) | 4 | 2 weeks |
| P6 | RBAC redesign per workspace × per env **+ Kubernetes namespace RBAC** | Entra groups bound to namespace `Role`s via Workload Identity; PIM for cluster-admin | Two RBAC layers now (Azure + Kubernetes) | Identity team (R), Platform (C) | 12 | 4 weeks |
| P7 | FinOps chargeback wiring **including AKS Cost Analysis add-on per namespace** | Per-product-namespace cost attribution; budgets per workspace + per cluster | Pooled compute requires sharper allocation | FinOps (R), Platform (C) | 8 | 3 weeks |
| P8 | Runbooks: cluster upgrade (blue/green), pool drain, namespace eviction, endpoint-raise, Spot eviction triage | Documented and rehearsed | Multi-tenant cluster blast radius needs pre-planned response | Platform + AKS (R), Ops (C) | 10 | 3 weeks |
| P9 | Change Advisory Board involvement for cluster upgrades | New CAB workflow: AKS minor version bumps every ~3 months | Multi-tenant blast radius requires governance | Engineering manager (A), Change mgmt (R) | 3 | 2 weeks |
| P10 | Day-2 on-call rota (24×7 if prod) | Shared between Platform and AKS owners | Production AKS uptime expectation | Engineering manager (A), Platform (R) | 5 | 4 weeks |
| P11 | Disaster-recovery runbook + game-day | Test failover from primary cluster to standby (multi-region) | DR is a real capability now | Platform + AKS (R), Product owners (I) | 8 | 3 weeks |
| P12 | Comms plan for product teams (workspace URL stable, endpoint contract changed slightly) | Migration comms + office hours | Adoption blocker if skipped | Platform PM (R), All product teams (I) | 6 | spread over migration |
| **Process/people subtotal** | | | | | **92 PD** | **~10 weeks elapsed (some parallel)** |

### 3.4 v2 totals

| Bucket | Effort (PD) | Elapsed |
|---|---|---|
| Foundation (technical) | 107 | ~10 weeks |
| Per-product migration (100 products) | ~700 (across product teams) + ~70 platform | ~7 months |
| Process / people | 92 | ~10 weeks (parallel) |
| **Total** | **~970 PD** | **~10 – 12 months end-to-end** |

---

## 4. Side-by-side comparison

### 4.1 Effort summary

| Bucket | v1 (Managed) | v2 (AKS) | Δ |
|---|---|---|---|
| Foundation technical PD | 57 | 107 | +50 PD (AKS cluster + extension + GitOps + multi-region) |
| Foundation elapsed | ~6 weeks | ~10 weeks | +4 weeks |
| Per-product migration (100 products) | ~780 PD | ~770 PD | ~equal — bulk is data/model re-registration shared by both |
| Per-product elapsed (parallel waves) | ~7 months | ~7 months | equal |
| Process/people PD | 36 | 92 | +56 PD (Kubernetes up-skilling, two-layer RBAC, GitOps, on-call, DR) |
| Process/people elapsed | ~6 weeks | ~10 weeks | +4 weeks |
| **Total effort** | **~870 PD** | **~970 PD** | **+~100 PD (~12 % more)** |
| **Total elapsed** | **~9 – 10 months** | **~10 – 12 months** | **+~1 – 2 months** |

### 4.2 What that ~12 % extra effort buys you

| Dimension | v1 | v2 |
|---|---|---|
| Subscriptions @ 100 products | 12 | 3 |
| Subscriptions @ 1,000 products | **120** | **9** |
| Spoke-hub VNet peerings @ 1,000 products | 120 | 9 (or 18 with multi-region prod) |
| Per-product compute baseline cost | 14-node cluster (idle small but non-zero) | **0** when idle, bursts into shared pool |
| Spot / Reserved Instance uplift | Hard to apply | Native (cluster-wide spot pools, RIs on baseline) |
| GPU sharing across products | Not possible | Yes (MIG / time-slicing) |
| Multi-region failover | Per-product re-stamp | Cluster-pair + AML Registry — runbook-driven |
| Operational complexity | Low (familiar `AmlCompute`) | Higher (multi-tenant Kubernetes) |
| Day-2 quota raises per year | Per VM family per sub × 12 | Per pool per cluster × 3 |

### 4.3 When to choose which

| Pick v1 if... | Pick v2 if... |
|---|---|
| Estate stays under ~300 products | Estate is heading to 500+ products |
| Your team has zero Kubernetes operational experience and no appetite to acquire it | You have or are building an AKS platform capability |
| Cost of per-product idle compute is acceptable | Cost optimisation (spot, RIs, GPU sharing) is a stated goal |
| Multi-region DR is not a hard requirement | Multi-region is required (regulatory or SLA) |
| Endpoint-cap raises are unlikely to be approved at scale | Endpoint-cap raise + paired-region split is feasible |

### 4.4 Migration path between the two

If you start with **v1** and later need **v2**, the additional incremental effort is roughly **F3 + F5 + F6 + F11 (AKS cluster + instance types + GitOps + provisioning) + a re-do of M5 and M8 per product** to swap the compute target. That is ~50 PD foundation + ~1 PD per product (only the endpoint re-target step), not a full re-migration. The reverse direction (v2 → v1) is similarly cheap because the workspaces and stamp shape are the same; only the compute target changes.

This is the strongest argument for **starting with v2 if growth beyond 300 products is plausible** — the foundation premium is paid once, and product teams never need to re-learn the contract.

---

## 5. Assumptions and caveats

- Estimates assume the **existing ALZ and stamping pipeline are healthy** and that adding two/three modules is the change scope. If the pipeline itself needs work, add a separate stream.
- Per-product migration effort assumes **product teams own their own model code and endpoint configuration**. If platform must do this on their behalf, multiply per-product effort by 2 – 3×.
- Effort excludes **org-wide policy work** (Defender, Sentinel, Purview onboarding) which is assumed already in place per ALZ.
- Effort excludes **non-AML data-platform work** (project data lake, Fabric/Synapse integration) — owned by the data team.
- v2 **AKS up-skilling estimate** assumes mid-level engineers with prior container experience. If the team is starting from zero, double P3.
- All time estimates are **planning ranges**, not commitments. Add 20 % contingency for the first wave of any new pattern.
