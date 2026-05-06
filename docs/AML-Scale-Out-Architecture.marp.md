---
marp: true
theme: default
paginate: true
backgroundColor: #fff
color: #333
style: |
  section {
    font-family: 'Segoe UI', sans-serif;
  }
  h1, h2, h3 {
    color: #0078D4;
  }
  table {
    font-size: 0.75em;
  }
  blockquote {
    background: #e8f4fd;
    border-left: 4px solid #0078D4;
    padding: 0.5em 1em;
    font-size: 0.85em;
  }
  .columns {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 1em;
  }
---

# Azure Machine Learning
## Scale-Out Reference Architecture

**Extending the subscription-stamping model to AML**

ML Platform Team · April 2026

---

## Agenda

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

## The Problem: One Subscription, Too Many Workloads

- **100 ML products × 3 environments** = 300 AML workspaces
- Each workspace: **14-node batch compute cluster**
- All workloads are **batch**

| AML Limit | Default cap | What we need |
|---|---|---|
| Endpoints / sub / region | **100** | 300 |
| Deployments / sub / region | **500** | 300+ |
| Compute targets / region | **500** (max 2,500) | 300 clusters |
| VM cores / family / region | 24–300 | 4,200 nodes |

> These limits are **regional per subscription**. The only structural fix is **more subscriptions**.

---

## Target Architecture: Subscription Fan-Out

Split 100 products across **12 AML-dedicated subscriptions** using the existing stamping pipeline.

```
Existing ALZ (Corp application-landing-zone MG)
├── DEV  → 4 subs (pg01–pg04, 25 products each)
├── TEST → 4 subs (pg01–pg04)
└── PROD → 4 subs (pg01–pg04)
```

**Totals:**
- 12 AML subscriptions
- No new management groups
- No new platform subs
- Reuses existing stamping pipeline

---

## The AML Stamp: What Gets Deployed Per Product

<div class="columns">
<div>

### `aml-sub-shared`
*Once per subscription*

- Shared ACR (Premium, PE)
- Private DNS zone links
- Diagnostic settings → ALZ LA

</div>
<div>

### `aml-product-stamp`
*Once per product per env*

- AML Workspace (`AllowOnlyApprovedOutbound`)
- ADLS Gen2
- Key Vault
- Application Insights
- User-assigned managed identity
- 14-node AmlCompute (min=0, max=14)
- Batch endpoint (≤ 20 deployments)
- Private endpoints (WS, storage, KV, ACR)
- CMK encryption (prd only)

</div>
</div>

---

## How Products Map to Subscriptions

### The ceiling formula

$$\text{pg\_index} = \lceil \frac{\text{product\_id}}{25} \rceil$$

| product_id | ceil(id / 25) | Subscription |
|---|---|---|
| 1 | 1 | `sub-aml-prd-pg01` |
| 25 | 1 | `sub-aml-prd-pg01` |
| 26 | 2 | `sub-aml-prd-pg02` |
| 100 | 4 | `sub-aml-prd-pg04` |
| **101** | **5** | `sub-aml-prd-pg05` **(new!)** |

> **Why 25?** Targets 25% of the 100-endpoint cap → **≥ 4× growth headroom**.
> **Properties:** Stateless · Stable · Trivially auditable

---

## Worked Example: Onboarding Product 101

**Intake:** `product_id=101, region=australiaeast, sku=Standard_DS3_v2`
*(No env field — all 3 environments always created)*

**Allocator:** `ceil(101/25) = 5` → `sub-aml-{env}-pg05` → does not exist → **vend**

---

## Worked Example: Pipeline Steps

*Runs 3× in parallel for dev / tst / prd:*

| Step | Module | Action | Time |
|---|---|---|---|
| 1 | Existing pipeline | Create subscription | ~5 min |
| 2 | Existing pipeline | MG placement + policy | ~2 min |
| 3 | Existing pipeline | Spoke VNet + hub peering | ~5 min |
| 4 | `aml-sub-shared` | Shared ACR + DNS links | ~8 min |
| 5 | `aml-product-stamp` | Workspace + all resources | ~15 min |

**Data scientist receives:**

| Env | Workspace | Endpoint | RG |
|---|---|---|---|
| dev | `mlw-dev-p101-aue` | `bep-p101-scoring` | `rg-aml-dev-p101` |
| tst | `mlw-tst-p101-aue` | `bep-p101-scoring` | `rg-aml-tst-p101` |
| prd | `mlw-prd-p101-aue` | `bep-p101-scoring` | `rg-aml-prd-p101` |

> **Products 102–125:** Skip steps 1–4 → only **~15 min**

---

## Full Limit Audit

At **25 products per sub**, every limit has ≥ 4× headroom:

| Limit | Default | Per-sub (25 prods) | % used | Solved? |
|---|---|---|---|---|
| Endpoints | 100 | 25 | 25% | ✅ |
| Deployments | 500 | 25–500 | 5–100% | ✅ (≤10/EP avg) |
| Compute targets | 500 (max 2,500) | 25 | 5% | ✅ |
| VM cores / family | 24–300 | 350 nodes × cores | Exceeds default | ⚠️ Quota raise |

**VM core mitigations:**
1. Automated `Microsoft.Quota` raise at vend time
2. `min_instances=0` on every cluster
3. **Quota Groups** from day one

---

## Scaling Beyond 100 Products

Growth is **linear** — just vend more subscriptions:

| Products | PG / env | AML subs | Action |
|---|---|---|---|
| **100** (today) | 4 | 12 | Baseline + Quota Groups |
| 200 | 8 | 24 | Vend pg05–pg08 |
| 400 | 16 | 48 | Same pattern |
| 1,000 | 40 | 120 | Add second region |

**Early-warning triggers (auto-alert):**
- Workspaces ≥ 20 / sub
- Endpoints ≥ 75 / sub
- Deployments ≥ 375 / sub
- Core quota ≥ 75%

---

## Security & Identity Model

<div class="columns">
<div>

### Network
- `AllowOnlyApprovedOutbound`
- `public_network_access = false`
- All traffic via private endpoints
- ALZ central DNS zones

### Encryption
- **prd:** CMK (Key Vault Crypto User)
- **dev/tst:** Microsoft-managed keys

</div>
<div>

### Identity
- User-assigned MI per workspace
  `id-aml-{env}-p{nnn}`
- RBAC roles:
  - Storage Blob Data Contributor
  - AcrPull
  - Key Vault Secrets User
- **No SAS tokens, no account keys**

### Tags (Policy-enforced)
`product_id` · `pg_index` · `cost_center`
`owner` · `environment` · `managed_by`

</div>
</div>

---

## Key Decisions

| # | Decision | Why |
|---|---|---|
| D1 | env × product-group → 12 subs | Solves endpoint + compute caps, 4× headroom |
| D2 | 25 products per sub | 25% of endpoint limit = room to grow |
| D3 | 1 workspace / product / env | CAF cloud-scale analytics guidance |
| D4 | Extend existing stamping pipeline | No new pipeline; 2 new IaC modules only |
| D6 | Batch EPs + ≤20 deployments/EP | Maximise model versions per endpoint |
| D9 | User-assigned managed identity | Credential-free ADLS/ACR/KV access |
| D11 | CMK for prd only | Balance security with dev velocity |
| D12 | `AllowOnlyApprovedOutbound` | Lock down workspace egress |

---

## What We're Not Changing

<div class="columns">
<div>

### Already in place ✅
- Azure Landing Zones (MGs, policies, hub-spoke)
- Subscription stamping pipeline
- Hub VNet, Firewall, ExpressRoute
- Central Log Analytics, Sentinel, Defender
- Entra ID groups, PIM
- Private DNS zones

</div>
<div>

### What's new 🆕
- **2 IaC modules**
  - `aml-sub-shared`
  - `aml-product-stamp`
- **12 AML subscriptions** under existing Corp MG
- AML-specific Azure Policy
  - PE-only workspaces
  - Required tags
  - Allowed VM SKUs

</div>
</div>

---

## Next Steps

| Step | Owner | Target |
|---|---|---|
| Review & approve architecture | Architecture board | Week 1 |
| Implement `aml-sub-shared` module | Platform engineering | Week 2–3 |
| Implement `aml-product-stamp` module | Platform engineering | Week 3–5 |
| Pilot: migrate 5 products | ML platform + 2 product teams | Week 5–7 |
| File VM core quota raises (prd) | Platform engineering | Week 5 |
| Full rollout: remaining 95 products | ML platform | Week 8–14 |
| Decommission old single-sub workspaces | ML platform | Week 15+ |

---

## Appendix: Microsoft Learn References

- [Manage quotas for Azure ML](https://learn.microsoft.com/azure/machine-learning/how-to-manage-quotas)
- [Azure subscription service limits](https://learn.microsoft.com/azure/azure-resource-manager/management/azure-subscription-service-limits)
- [CAF subscription design](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-area/resource-org-subscriptions)
- [Subscription vending](https://learn.microsoft.com/azure/cloud-adoption-framework/ready/landing-zone/design-area/subscription-vending)
- [Deployment Stamps pattern](https://learn.microsoft.com/azure/architecture/patterns/deployment-stamp)
- [AML for cloud-scale analytics](https://learn.microsoft.com/azure/cloud-adoption-framework/scenarios/cloud-scale-analytics/best-practices/azure-machine-learning)
- [Batch endpoints](https://learn.microsoft.com/azure/machine-learning/concept-endpoints-batch)
- [Quota Groups](https://learn.microsoft.com/azure/quotas/quota-groups)
