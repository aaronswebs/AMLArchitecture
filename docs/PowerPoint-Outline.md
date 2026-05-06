# PowerPoint Copilot Outline: AML Scale-Out Architecture

Use this outline as a prompt for Microsoft PowerPoint Copilot to generate a presentation. Copy the text below into Copilot's "Create a presentation about…" prompt.

---

## Prompt for PowerPoint Copilot

Create a professional presentation with the following slides. Use a clean, modern design with Azure blue (#0078D4) accents. Include diagrams where indicated.

---

### Slide 1 — Title Slide
**Title:** Azure Machine Learning — Scale-Out Reference Architecture
**Subtitle:** Extending the subscription-stamping model to AML
**Footer:** ML Platform Team · April 2026

---

### Slide 2 — Agenda
1. The problem: subscription limits
2. Current state vs target state
3. The AML stamp — what gets deployed
4. How products map to subscriptions (the ceiling formula)
5. Worked example: onboarding Product 101
6. Full limit audit
7. Scaling to 200+ products
8. Security & identity model
9. Key decisions
10. Next steps

---

### Slide 3 — The Problem: One Subscription, Too Many Workloads
**Key message:** Everything in a single subscription hits hard AML regional caps.

- 100 ML products × 3 environments = 300 AML workspaces
- Each workspace: 14-node batch compute cluster
- All workloads are batch

**Table: Where we hit limits**

| AML Limit | Default cap | What we need |
|---|---|---|
| Endpoints per subscription per region | 100 | 300 |
| Deployments per subscription per region | 500 | 300+ |
| Total compute targets per region | 500 (max 2,500) | 300 clusters |
| VM cores per family per region | 24–300 | 4,200 nodes worth |

**Callout box:** These limits are regional per subscription. The only structural fix is more subscriptions.

---

### Slide 4 — Target Architecture: Subscription Fan-Out
**Key message:** Split 100 products across 12 AML-dedicated subscriptions using the existing stamping pipeline.

**Diagram (hierarchy):**
- Existing ALZ (Corp application-landing-zone MG)
  - DEV → 4 subscriptions (pg01–pg04, 25 products each)
  - TEST → 4 subscriptions (pg01–pg04)
  - PROD → 4 subscriptions (pg01–pg04)

**Totals:** 12 AML subscriptions · No new MGs · No new platform subs · Reuses existing stamping pipeline

---

### Slide 5 — The AML Stamp: What Gets Deployed Per Product
**Key message:** Two new IaC modules plug into the existing stamping pipeline.

**Module 1 — `aml-sub-shared` (once per subscription):**
- Shared Azure Container Registry (Premium, private endpoint)
- Private DNS zone links to ALZ central zones
- Diagnostic settings → ALZ central Log Analytics

**Module 2 — `aml-product-stamp` (once per product per environment):**
- AML Workspace (network isolation = AllowOnlyApprovedOutbound, public access disabled)
- ADLS Gen2 (workspace default store)
- Key Vault
- Application Insights
- User-assigned managed identity
- 14-node AmlCompute cluster (min=0, max=14)
- Batch endpoint (≤ 20 deployments)
- Private endpoints (workspace, storage, KV, ACR)
- CMK encryption (production only)

---

### Slide 6 — How Products Map to Subscriptions: The Ceiling Formula
**Key message:** A simple, stateless, deterministic formula maps every product to its subscription.

**Formula:** `pg_index = ceil(product_id / 25)`

**Example table:**

| product_id | ceil(id / 25) | Subscription |
|---|---|---|
| 1 | 1 | sub-aml-prd-pg01 |
| 25 | 1 | sub-aml-prd-pg01 |
| 26 | 2 | sub-aml-prd-pg02 |
| 100 | 4 | sub-aml-prd-pg04 |
| 101 | 5 | sub-aml-prd-pg05 (new!) |

**Why 25?** Targets 25% of the 100-endpoint cap → ≥ 4× growth headroom on every limit.

**Properties:** Stateless (no registry), stable (never changes), trivially auditable.

---

### Slide 7 — Worked Example: Onboarding Product 101
**Key message:** End-to-end flow from intake to workspace.

**Step 0 — Intake:**
Data scientist submits: product_id=101, region=australiaeast, sku=Standard_DS3_v2
(No environment field — all 3 envs are always created)

**Allocator:** ceil(101/25) = 5 → sub-aml-{env}-pg05 → does not exist → vend

**Pipeline steps (runs 3× in parallel for dev/tst/prd):**

| Step | Who | What | Time |
|---|---|---|---|
| 1 | Existing pipeline | Create subscription | ~5 min |
| 2 | Existing pipeline | MG placement + policy | ~2 min |
| 3 | Existing pipeline | Spoke VNet + hub peering | ~5 min |
| 4 | aml-sub-shared | Shared ACR + DNS links | ~8 min |
| 5 | aml-product-stamp | Workspace + all resources | ~15 min |

**Data scientist receives:**

| Env | Workspace | Endpoint | RG |
|---|---|---|---|
| dev | mlw-dev-p101-aue | bep-p101-scoring | rg-aml-dev-p101 |
| tst | mlw-tst-p101-aue | bep-p101-scoring | rg-aml-tst-p101 |
| prd | mlw-prd-p101-aue | bep-p101-scoring | rg-aml-prd-p101 |

**Products 102–125:** Skip steps 1–4 (sub already exists). Only ~15 min.

---

### Slide 8 — Full Limit Audit: Every AML Cap Covered
**Key message:** At 25 products per sub, every limit has ≥ 4× headroom.

| Limit | Default | Per-sub usage (25 products) | % used | Solved? |
|---|---|---|---|---|
| Endpoints | 100 | 25 | 25% | ✅ |
| Deployments | 500 | 25–500 | 5–100% | ✅ (keep ≤10 per EP avg) |
| Compute targets | 500 (max 2,500) | 25 | 5% | ✅ |
| VM cores per family | 24–300 | 350 nodes × cores | Exceeds default | ⚠️ Needs quota raise |

**VM core fix:** Automated quota raise at vend time + min_instances=0 + Quota Groups from day one.

---

### Slide 9 — Scaling Beyond 100 Products
**Key message:** Growth is linear — just vend more subscriptions.

| Products | Product-groups/env | AML subs | Action |
|---|---|---|---|
| 100 (today) | 4 | 12 | Baseline + Quota Groups |
| 200 | 8 | 24 | Vend pg05–pg08 |
| 400 | 16 | 48 | Same pattern |
| 1,000 | 40 | 120 | Add second region |

**Early-warning triggers (auto-alert):**
- Workspaces ≥ 20 per sub
- Endpoints ≥ 75 per sub
- Deployments ≥ 375 per sub
- Core quota ≥ 75%

---

### Slide 10 — Security & Identity Model
**Key message:** Zero-trust by default; no stored credentials.

**Network:**
- Workspace: AllowOnlyApprovedOutbound + public_network_access = false
- All traffic via private endpoints through ALZ central DNS zones

**Identity:**
- User-assigned managed identity per workspace (id-aml-{env}-p{nnn})
- RBAC roles: Storage Blob Data Contributor (ADLS), AcrPull (ACR), Key Vault Secrets User (KV)
- No SAS tokens, no account keys

**Encryption:**
- CMK for production (Key Vault Crypto User role on managed identity)
- Microsoft-managed keys for dev/tst

**Tags enforced by Azure Policy:**
- product_id, pg_index, cost_center, owner, environment, managed_by

---

### Slide 11 — Key Decisions Summary

| # | Decision | Why |
|---|---|---|
| D1 | env × product-group → 12 subs | Solves endpoint + compute caps with 4× headroom |
| D2 | 25 products per sub | 25% of endpoint limit = room to grow |
| D3 | 1 workspace per product per env | CAF cloud-scale analytics guidance |
| D4 | Extend existing stamping pipeline | No new pipeline; 2 new IaC modules only |
| D6 | Batch endpoints + ≤20 deployments/EP | Batch-only workload; maximise model versions |
| D9 | User-assigned managed identity | Credential-free ADLS/ACR/KV access |
| D11 | CMK for prd only | Balance security with dev velocity |
| D12 | AllowOnlyApprovedOutbound | Lock down workspace egress |

---

### Slide 12 — What We're Not Changing
**Key message:** This architecture extends, it doesn't replace.

**Already in place (no changes):**
- Azure Landing Zones (MGs, policies, hub-and-spoke)
- Subscription stamping pipeline
- Hub VNet, Firewall, ExpressRoute
- Central Log Analytics, Sentinel, Defender
- Entra ID groups, PIM
- Private DNS zones

**What's new:**
- 2 IaC modules (aml-sub-shared, aml-product-stamp)
- 12 AML-dedicated subscriptions under existing Corp MG
- AML-specific Azure Policy (PE-only workspaces, required tags, allowed VM SKUs)

---

### Slide 13 — Next Steps

| Step | Owner | Target |
|---|---|---|
| Review & approve architecture | Architecture board | Week 1 |
| Implement aml-sub-shared module | Platform engineering | Week 2–3 |
| Implement aml-product-stamp module | Platform engineering | Week 3–5 |
| Pilot: migrate 5 products to new stamp | ML platform + 2 product teams | Week 5–7 |
| File VM core quota raises for prd subs | Platform engineering | Week 5 |
| Full rollout: remaining 95 products | ML platform | Week 8–14 |
| Decommission old single-subscription workspaces | ML platform | Week 15+ |

---

### Slide 14 — Appendix: Microsoft Learn References

- Manage quotas for Azure ML: learn.microsoft.com/azure/machine-learning/how-to-manage-quotas
- Azure subscription service limits: learn.microsoft.com/azure/azure-resource-manager/management/azure-subscription-service-limits
- CAF subscription design: learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-subscriptions
- Subscription vending: learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-area/subscription-vending
- Deployment Stamps pattern: learn.microsoft.com/azure/architecture/patterns/deployment-stamp
- AML for cloud-scale analytics: learn.microsoft.com/azure/cloud-adoption-framework/scenarios/cloud-scale-analytics/best-practices/azure-machine-learning
- Batch endpoints: learn.microsoft.com/azure/machine-learning/concept-endpoints-batch
- Quota Groups: learn.microsoft.com/azure/quotas/quota-groups
