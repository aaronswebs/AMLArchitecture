# Implementation Effort — v1 (Managed Compute) vs. v2 (AML on AKS)

> **Status:** Draft for review
> **Owner:** ML Platform Team
> **Last updated:** May 2026
> **Audience:** ML Platform Engineers, Programme Managers, Cloud Architects, FinOps, Engineering Managers
> **Primary region:** Australia East (`australiaeast` / `aue`) · **Secondary region:** Australia Southeast (`australiasoutheast` / `ase`)
> **Companions:** [`AML-Scale-Out-Architecture.md`](AML-Scale-Out-Architecture.md), [`AML-AKS-ScaleOut-Architecture.md`](AML-AKS-ScaleOut-Architecture.md)

---

## 0. Starting state and target

**Starting state**

- **One subscription** today in Australia East, hosting all **300 workspaces** (100 products × 3 environments).
- Every workspace has its own 14-node `AmlCompute` cluster.
- ALZ + subscription-stamping pipeline already exist, with hubs in Australia East (primary) and Australia Southeast (paired).
- Maintenance is easy (one sub, one team, one quota table) **but scale is the wall** — endpoints, compute targets, and core quotas are all hit even at 100 products.

**Target capacity for both designs: 1,000 products × 3 environments = 3,000 workspaces.** The 100 existing products migrate first; the next 900 onboard incrementally over time. The doc separates **one-time migration** from **steady-state growth** so you can see where each design pays off.

Both target architectures require **re-stamping every existing workspace** into a new subscription topology because AML workspaces cannot be cleanly moved between subscriptions with all their artefacts (models, runs, datastores, endpoints).

## 1. How to read the tables

Effort columns:

- **Effort (PD)** — person-days of focused work for the named persona/team. PD = 8 hours of one engineer.
- **Elapsed** — wall-clock time including waits (procurement, RBAC approvals, support tickets, change windows, batches of 25 product migrations, etc.).
- **Persona / team** — primary owner. R = Responsible, A = Accountable, C = Consulted, I = Informed.
- **Risk** — L / M / H, with the dominant risk reason.

Effort estimates assume a **3 – 5 person platform team** with one tech lead, plus access to existing ALZ / network / identity teams as consulted parties. They are planning estimates; treat as ranges, not commitments.

---

## 2. v1 — Managed compute (per-product `AmlCompute`) — implementation effort

Final state at the **1,000-product capacity target**: **120 AML-dedicated subscriptions (40 PG × 3 env)** in Australia East, 25 products per sub, two new pipeline modules, every workspace re-stamped into its destination sub. At the migration cutover only the **first 12 subs (4 PG × 3 env)** for the existing 100 products are vended; the other 108 subs are vended incrementally as products 101–1,000 onboard.

### 2.1 Foundation — pipeline & shared modules (one-time, before any product migration)

| # | Task | What changes | Why | Persona / team | Effort (PD) | Elapsed | Risk |
|---|---|---|---|---|---|---|---|
| F1 | Design review with ALZ + Networking + Identity | RACI, MG placement, subscription naming, RBAC bootstrap pattern agreed | Cannot extend the stamping pipeline without alignment from owners | Platform tech lead (R), ALZ/Net/Id (C) | 5 | 2 weeks | M — discovery may surface ALZ gaps |
| F2 | Build `aml-sub-shared` module (Bicep/TF + AVM) | New module: Premium ACR, central-DNS zone links, diagnostic settings, Quota Group bootstrap | Must run once per new subscription before any workspace lands | Platform engineer (R), ALZ pipeline owner (C) | 10 | 2 weeks | L |
| F3 | Build `aml-product-stamp` module | New module: workspace, ADLS, KV, AI, UAMI, 14-node `AmlCompute`, batch endpoint, PEs, tags, CMK (prd) — all in Australia East | Authoritative AML stamp per product per env | Platform engineer (R) | 15 | 3 weeks | M — PE/DNS wiring tends to need iterations |
| F4 | Wire allocator (`pg_index = ceil(product_id / 25)`) | Determines target sub from `product_id`; drives stamping pipeline | Without it, onboarding is manual | Platform engineer (R) | 4 | 1 week | L |
| F5 | **Quota raise automation via `Microsoft.Quota` REST API** | Pipeline fires VM-core, endpoint, compute-target raises after sub vending. Designed to fire **120 raises** unattended over the life of the design | Default Australia East quotas insufficient for 25 × 14-node clusters; manual ticketing not viable at growth scale | Platform engineer (R), Subscription owner (A) | 8 | 2–3 weeks (incl. first ticket cycle with Microsoft) | **H** — Microsoft response time variable; GPU quotas in Australia East often start at 0 |
| F6 | Naming, tagging, RBAC standards (AML-specific) | New conventions documented and enforced via Azure Policy; tag schema mandatory for tag-based subscription lookup at 120-sub scale | Operability and chargeback at scale | Platform tech lead (R), Cloud Governance (C) | 4 | 1 week | L |
| F7 | Vend the **first 12** destination subscriptions (for migration of existing 100 products) | Existing pipeline creates `aml-{env}-pg01..04` × 3 envs in Australia East | Targets must exist before re-stamping starts; remaining 108 subs vend on-demand later | ALZ pipeline owner (R), Platform (A) | 2 | 2 weeks (incl. ticketing) | L |
| F8 | Run `aml-sub-shared` against the 12 subs | ACR + DNS links + diagnostics established | Pre-requisite for any product stamp | Platform engineer (R) | 3 | 1 week | L |
| F9 | Pilot end-to-end with 2 throwaway products | Validate stamp + endpoint deploy + batch job in Australia East | De-risks bulk migration | Platform + 1 product team (R), DS lead (C) | 8 | 2 weeks | M |
| **Foundation subtotal** | | | | | **59 PD** | **~6 weeks elapsed** | |

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
| P2 | Quota operating model + Quota Groups setup | Pool VM-family quotas across same-env subs in Australia East | Smooths peaks across 4–40 subs/env over the design's lifetime | Platform + FinOps (R), MS support (C) | 8 | 3 weeks |
| P3 | RBAC redesign per workspace × per env (sized for 120-sub end-state) | Entra groups by product × env; PIM for prd | Sub fan-out multiplies group count up to 120× | Identity team (R), Platform (C) | 12 | 4 weeks |
| P4 | FinOps chargeback wiring (rolled-up across up to 120 subs) | Tags + per-sub budgets + alerts at 75 %; cross-sub roll-up dashboards | 12 → 120 subs need automated rolled-up reporting | FinOps (R), Platform (C) | 8 | 3 weeks |
| P5 | Runbooks: vend new PG sub, raise quotas, decommission, Australia Southeast overflow vending | Documented for support team; covers the on-demand vending of subs 13–120 | Operability when platform team is off-shift; growth is unattended | Platform (R), Ops (C) | 6 | 2 weeks |
| P6 | Product-team training + comms | Each product team learns the new workspace URL, endpoint pattern, CI/CD changes | Adoption blocker if skipped | Platform (R), All product teams (I) | 6 | spread over migration |
| P7 | Day-2 ops re-org sized for 120 subs | Platform team needs on-call across an estate that grows 1 → 12 → 120 subs | Operational scope grows 120× over the design lifetime | Engineering manager (A), Platform (R) | 6 | 6 weeks (HR + rota + training) |
| **Process/people subtotal** | | | | | **49 PD** | **~7 weeks elapsed (overlaps migration)** |

### 2.4 Steady-state growth from 100 → 1,000 products (v1)

Growth from 100 → 1,000 products vends **108 additional subscriptions** (`pg05 … pg40` × 3 envs) and onboards **900 new products**. The pipeline does the heavy lifting, but each subscription incurs unattended-but-non-zero cost in vending, quota raises, and ongoing operations.

| # | Activity | Per unit | Units (100 → 1,000) | Total PD |
|---|---|---|---|---|
| G1 | Vend new product-group subscription (auto via pipeline) | 0.1 PD oversight | 108 subs | ~11 |
| G2 | Run `aml-sub-shared` (auto) | 0.1 PD oversight | 108 subs | ~11 |
| G3 | Automated quota raises per new sub (§F5) | 0.1 PD oversight + Microsoft cycle time | 108 subs | ~11 |
| G4 | Onboard each new product (`aml-product-stamp` only — no compute provisioning code change) | 0.5 PD platform + 5–9 PD product team | 900 products | ~450 platform + ~6,300 product-team |
| G5 | Endpoint-cap raise + Australia Southeast overflow planning per `prd` PG that crosses 70 EPs | 1 PD per crossing | ~10 PGs over time | ~10 |
| G6 | Day-2 ops scaling (on-call expanded as estate grows) | continuous | 1× over 1–2 yrs | ~30 |
| **v1 growth subtotal** | | | | **~520 platform + ~6,300 product-team PD** |

### 2.5 v1 totals

| Bucket | Effort (PD) | Elapsed |
|---|---|---|
| Foundation (technical) | 59 | ~6 weeks |
| Per-product migration of existing 100 products | ~700 product-team + ~80 platform = ~780 | ~7 months |
| Process / people | 49 | ~7 weeks (parallel) |
| Steady-state growth 100 → 1,000 products | ~520 platform + ~6,300 product-team = ~6,820 | spread over 1–2 years post-migration |
| **Total to migration cutover** | **~890 PD** | **~9 – 10 months end-to-end** |
| **Total to 1,000 products in steady state** | **~7,700 PD** (mostly distributed product-team effort over 1–2 yrs) | **~2.5 – 3 years from project start** |

---

## 3. v2 — AML on AKS (`KubernetesCompute`) — implementation effort

Final state at the **1,000-product capacity target**: **9 AML-dedicated subscriptions (3 PG × 3 env)** in Australia East, with on-demand Australia Southeast pairs for prd DR / endpoint-cap relief. Compute pooled into one shared AKS cluster pair per env per sub. Three new pipeline modules. At the migration cutover only the **first 3 subs (1 PG × 3 env)** are vended for the existing 100 products; the remaining 6 vend incrementally as products 335 and 668 onboard.

### 3.1 Foundation — pipeline, AKS platform, shared modules (one-time)

| # | Task | What changes | Why | Persona / team | Effort (PD) | Elapsed | Risk |
|---|---|---|---|---|---|---|---|
| F1 | Design review with ALZ + Networking + Identity + AKS Platform team | Same as v1 plus AKS platform-team RACI, GitOps tooling choice (Argo CD/Flux), namespace isolation pattern | More moving parts and more teams in scope | Platform tech lead (R), ALZ/Net/Id/AKS (C) | 8 | 3 weeks | M |
| F2 | Build `aml-sub-shared` module | Premium ACR (geo-replicated AUE↔ASE for prd), DNS zone links, diagnostics, AML Registry per env | Same shape as v1; ACR sized for cross-product image density | Platform engineer (R) | 10 | 2 weeks | L |
| F3 | Build `aks-platform-cluster` module | **New**: private AKS in Australia East, API server VNet integration, Azure CNI Overlay, Workload Identity, OIDC issuer, system + cpu-od + cpu-spot + gpu-od + gpu-spot pools, AML extension (`enableTraining=true enableInference=true`), instance-type CRDs, baseline NetworkPolicy + ResourceQuota templates, Defender for Containers, Cost Analysis add-on, GitOps bootstrap | The biggest new asset; runs once per env per sub | Platform engineer + AKS engineer (R), Net/Sec (C) | 25 | 4 – 5 weeks | **H** — first-time-right hardening on private cluster + extension |
| F4 | Build `aml-product-stamp` module | Same as v1 **minus** `AmlCompute`, **plus** namespace creation, ResourceQuota/NetworkPolicy, `az ml compute attach --type Kubernetes`, default instance-type binding | Per-product stamp pivots from compute-owning to compute-attaching | Platform engineer (R) | 12 | 3 weeks | M |
| F5 | Curated instance-type catalogue (CRDs) | `cpu-small/medium/large`, `gpu-small-spot`, `gpu-large` | Data-scientist-facing contract; abstracts SKUs | Platform engineer + DS lead (R) | 4 | 1 week | L |
| F6 | GitOps repo + bootstrap (Argo CD or Flux) | Cluster state (instance types, RBAC, taints, quotas) declared in Git, reconciled per cluster | Replayable across cluster pairs and Australia East / Southeast regions | Platform engineer (R), AKS team (C) | 8 | 2 weeks | L |
| F7 | Allocator (`pg_index = ceil(product_id / 334)` for the 1,000-product target) | Drives stamping pipeline | Determinism | Platform engineer (R) | 4 | 1 week | L |
| F8 | Quota raise automation (VM cores **per AKS pool**, endpoint cap raise) | One quota raise per cluster pool, not per product cluster | Pooled quotas — ~9 raises over the design's life vs. ~120 in v1 | Platform engineer (R) | 5 | 2 weeks | M — endpoint raise approval cycle |
| F9 | Naming, tagging, RBAC standards (AML + AKS namespaces) | Workspace = namespace = `ws-p{nnn}` mapping codified | Operability + chargeback | Platform tech lead (R), Cloud Governance (C) | 4 | 1 week | L |
| F10 | Vend the **first 3** destination subscriptions (for migration of existing 100 products) | Existing pipeline creates `aml-aks-{env}-pg01` × 3 envs in Australia East | 3 instead of 12 at cutover; 6 more on-demand later | ALZ pipeline owner (R) | 1 | 2 weeks (ticketing) | L |
| F11 | Run `aml-sub-shared` + `aks-platform-cluster` in Australia East | 3 AKS clusters provisioned and hardened in Australia East | Compute substrate ready | Platform + AKS engineer (R) | 6 | 2 weeks (incl. validation) | M |
| F12 | Pilot end-to-end: 2 throwaway products on the cluster | Validate namespace isolation, instance-type selection, batch job, batch endpoint | Multi-tenant cluster correctness is the new risk | Platform + AKS + 1 product team (R) | 12 | 3 weeks | **H** — noisy-neighbour, taint, GPU-share validation |
| F13 | Multi-region prep — build the Australia Southeast standby cluster module + AML Registry replication | Standby AKS cluster blueprint + registry replication | DR + endpoint-cap relief on demand; not deployed at cutover | Platform engineer (R) | 8 | 2 weeks | M |
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

### 3.4 Steady-state growth from 100 → 1,000 products (v2)

Growth from 100 → 1,000 products vends only **6 additional subscriptions** (`pg02` and `pg03` × 3 envs) and onboards 900 new products into existing AKS clusters. Each new subscription brings up its own `aks-platform-cluster` (~25 PD per cluster amortised, but using the same module) and instance-type catalogue.

| # | Activity | Per unit | Units (100 → 1,000) | Total PD |
|---|---|---|---|---|
| G1 | Vend new product-group subscription (auto via pipeline) | 0.1 PD oversight | 6 subs | ~1 |
| G2 | Run `aml-sub-shared` + `aks-platform-cluster` per new sub | 4 PD oversight + validation | 6 subs | ~24 |
| G3 | Quota raises per new AKS cluster pool | 0.5 PD oversight | 6 subs | ~3 |
| G4 | Onboard each new product (`aml-product-stamp` only — namespace + attach) | 0.4 PD platform + 5–9 PD product team | 900 products | ~360 platform + ~6,300 product-team |
| G5 | Optional Australia Southeast pair vend for selected `prd` PGs (DR / endpoint relief) | 8 PD per pair | 0–3 pairs | 0 – 24 |
| G6 | AKS cluster upgrades (blue/green, ~3 per year per cluster) | 3 PD per upgrade per cluster | ~30 cluster-upgrades over 2–3 yrs | ~90 |
| G7 | Day-2 ops scaling | continuous | 1× over 1–2 yrs | ~20 |
| **v2 growth subtotal** | | | | **~500 platform + ~6,300 product-team PD** |

### 3.5 v2 totals

| Bucket | Effort (PD) | Elapsed |
|---|---|---|
| Foundation (technical) | 107 | ~10 weeks |
| Per-product migration of existing 100 products | ~700 product-team + ~70 platform = ~770 | ~7 months |
| Process / people | 92 | ~10 weeks (parallel) |
| Steady-state growth 100 → 1,000 products | ~500 platform + ~6,300 product-team = ~6,800 | spread over 1–2 years post-migration |
| **Total to migration cutover** | **~970 PD** | **~10 – 12 months end-to-end** |
| **Total to 1,000 products in steady state** | **~7,770 PD** (mostly distributed product-team effort over 1–2 yrs) | **~2.5 – 3 years from project start** |

---

## 4. Side-by-side comparison

### 4.1 Effort summary

| Bucket | v1 (Managed) | v2 (AKS) | Δ |
|---|---|---|---|
| Foundation technical PD | 59 | 107 | +48 PD (AKS cluster + extension + GitOps + Australia Southeast prep) |
| Foundation elapsed | ~6 weeks | ~10 weeks | +4 weeks |
| Per-product migration of existing 100 products | ~780 PD | ~770 PD | ~equal — bulk is data/model re-registration shared by both |
| Per-product migration elapsed (parallel waves) | ~7 months | ~7 months | equal |
| Process/people PD | 49 | 92 | +43 PD (Kubernetes up-skilling, two-layer RBAC, GitOps, on-call, DR) |
| Process/people elapsed | ~7 weeks | ~10 weeks | +3 weeks |
| **Total to migration cutover** | **~890 PD** | **~970 PD** | **+~80 PD (~9 % more)** |
| **Total elapsed to cutover** | **~9 – 10 months** | **~10 – 12 months** | **+~1 – 2 months** |
| Steady-state growth 100 → 1,000 products | ~520 platform + ~6,300 product-team | ~500 platform + ~6,300 product-team | **~equal at the product-team layer; v1 is +20 PD platform overhead, but the real v1 penalty is operating 120 subs vs. 9** |
| **Total to 1,000-product steady state** | **~7,700 PD** | **~7,770 PD** | **~equal in headline PD, very different operationally** |

### 4.2 What the v2 design buys you for ~1–2 months extra at cutover

| Dimension | v1 | v2 |
|---|---|---|
| Subscriptions @ 100 products | 12 (Australia East) | 3 (Australia East) |
| Subscriptions @ 1,000 products | **120 (Australia East)** | **9 (Australia East)** — plus on-demand ASE pairs only when needed |
| Spoke-hub VNet peerings @ 1,000 products | 120 | 9 (or 18 if Australia Southeast prod pairs are deployed) |
| Quota raises over the design's life | ~120 (per VM family per sub) | ~9 (per cluster pool) |
| Per-product compute baseline cost | 14-node cluster (idle small but non-zero) | **0** when idle, bursts into shared pool |
| Spot / Reserved Instance uplift | Hard to apply | Native (cluster-wide spot pools, RIs on baseline) |
| GPU sharing across products in Australia East | Not possible | Yes (MIG / time-slicing) — important given AUE GPU scarcity |
| Multi-region failover (AUE ↔ ASE) | Per-product re-stamp | Cluster-pair + AML Registry — runbook-driven |
| Operational complexity | Low (familiar `AmlCompute`) | Higher (multi-tenant Kubernetes) |
| Day-2 on-call surface | 120 subs by year 2–3 | 9 subs by year 2–3 |

### 4.3 When to choose which

| Pick v1 if... | Pick v2 if... |
|---|---|
| Estate is firmly capped at ≤ 300 products | Estate is heading to 500+ products (which the user scenario is) |
| Your team has zero Kubernetes operational experience and no appetite to acquire it | You have or are building an AKS platform capability |
| Cost of per-product idle compute is acceptable | Cost optimisation (spot, RIs, GPU sharing) is a stated goal |
| Multi-region DR / data-residency-pair is not a hard requirement | Australia East + Australia Southeast pair is required (regulatory or SLA) |
| Operating 120 subscriptions long-term is acceptable | Operating 9 subscriptions long-term is strongly preferred |

### 4.4 Migration path between the two

If you start with **v1** and later need **v2**, the additional incremental effort is roughly **F3 + F5 + F6 + F11 (AKS cluster + instance types + GitOps + provisioning) + a re-do of M5 and M8 per product** to swap the compute target. That is ~50 PD foundation + ~1 PD per product (only the endpoint re-target step), not a full re-migration. The reverse direction (v2 → v1) is similarly cheap because the workspaces and stamp shape are the same; only the compute target changes.

Given the user scenario explicitly targets **1,000+ products**, the strongest argument is to **start with v2 directly** — the foundation premium (~80 PD) is paid once, product teams never need to re-learn the contract, and the 120-sub operating burden of v1 is avoided entirely.

---

## 5. Assumptions and caveats

- Estimates assume the **existing ALZ and stamping pipeline are healthy** and that adding two/three modules is the change scope. If the pipeline itself needs work, add a separate stream.
- Per-product migration / onboarding effort assumes **product teams own their own model code and endpoint configuration**. If platform must do this on their behalf, multiply per-product effort by 2 – 3×.
- Effort excludes **org-wide policy work** (Defender, Sentinel, Purview onboarding) which is assumed already in place per ALZ.
- Effort excludes **non-AML data-platform work** (project data lake, Fabric/Synapse integration) — owned by the data team.
- v2 **AKS up-skilling estimate** assumes mid-level engineers with prior container experience. If the team is starting from zero, double P3.
- Australia East GPU SKU quotas often default to **0** — every GPU SKU needed in v1 (per sub) or v2 (per cluster pool) requires a Microsoft-approved raise; in capacity-constrained windows, capacity reservations or fall-back to Australia Southeast may be required.
- **Steady-state growth product-team effort** (~6,300 PD across 900 products) is roughly the same in both designs because the dominant cost is per-product model code, datastore wiring, and endpoint definition. The big v1/v2 difference at growth is **platform operating burden**, not product-team effort.
- All time estimates are **planning ranges**, not commitments. Add 20 % contingency for the first wave of any new pattern.
