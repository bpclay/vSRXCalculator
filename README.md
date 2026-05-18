# vSRXCalculator

---

## Overview

This tool provides a side-by-side monthly cost comparison between deploying a **Juniper vSRX Next Generation Virtual Firewall** (via cloud Marketplace, PAYG licensing) and each cloud provider's **native managed firewall service** in a centralized inspection architecture using the cloud provider's native transit/hub construct.

The calculator is a single self-contained HTML file that runs in any browser with no dependencies to install. All inputs are adjustable via sliders and dropdowns; costs recalculate in real time.

The tool is currently scoped to **one use case** (symmetrical internet edge), with additional use cases planned:

| Use case | Status |
|---|---|
| Centralized inspection · Symmetrical internet edge (in → VPC → out) | ✅ Available |
| East-west VPC/VNet traffic | 🔜 Planned |
| Direct Connect / ExpressRoute head-end | 🔜 Planned |

---

## Supported Cloud Providers

| Cloud | Native Firewall | Transit/Hub Construct | vSRX Instance Family |
|---|---|---|---|
| AWS | AWS Network Firewall (ANF) | Transit Gateway (TGW) | c5 / c5a / c5n / c6in / c7i |
| GCP | Cloud NGFW Enterprise | Network Connectivity Center (NCC) | C2 standard |
| Azure | Azure Firewall Standard | Virtual WAN (Standard Hub) | Fsv2 series |

---

## Architecture Assumptions

The model assumes a **centralized inspection** pattern — a dedicated inspection VPC/VNet/project hosts the firewall, and all spoke VPCs route through it via the cloud provider's hub construct. This is the most common enterprise deployment pattern for both vSRX and native firewall options.

**Symmetrical internet edge use case:**

```
Internet → IGW/Cloud LB → Hub/TGW → Inspection VPC (Firewall) → Hub/TGW → Spoke VPC → App
App → Hub/TGW → Inspection VPC (Firewall) → Hub/TGW → IGW → Internet
```

Key implications of this flow:

- **Inbound traffic** (internet → cloud) carries no AWS/GCP/Azure data transfer charge — inbound is always free
- **Outbound traffic** (cloud → internet) incurs standard data transfer out (DTO) rates for the cloud provider — identical for both vSRX and native firewall in this model, so DTO does not differentiate the two options
- **Hub/transit data processing** is charged on **both directions** (traffic crosses the hub twice per round trip), so all per-GB hub charges are applied to **2× the per-direction traffic volume**
- **Native firewall data processing** is similarly charged on both directions — ANF, Cloud NGFW, and Azure Firewall all meter inspected bytes in both directions
- **vSRX has no per-GB firewall processing charge** — its cost is fixed compute (EC2/CE/VM) plus Marketplace software fee, regardless of traffic volume

**High Availability:** vSRX is modeled as an **HA pair (2 instances) per Availability Zone / Region**. Single-instance deployments are not modeled as they are not suitable for production.

---

## User-Configurable Inputs

| Input | Description | Notes |
|---|---|---|
| Per-direction traffic (TB/mo) | Traffic volume in one direction | Tool bills applicable charges at 2× this figure |
| Availability Zones / Regions | Number of AZs (AWS), regions (GCP), or hub deployments (Azure) | Drives firewall endpoint count and HA pair count |
| Hub attachments | TGW VPC attachments (AWS), NCC spokes (GCP), spoke VNets (Azure) | Drives attachment-hour charges |
| vSRX instance size | Compute instance for vSRX — cloud-specific families | See instance inventory section below |
| vSRX feature tier | Software feature set enabled — affects SW fee and throughput | See feature tiers section below |

---

## vSRX Feature Tiers

The Marketplace software fee and effective throughput vary by feature tier. Three tiers are modeled:

| Tier | SW fee multiplier | Throughput multiplier | Features included |
|---|---|---|---|
| Standard | 1.00× | 1.00× | Core firewall, routing, NAT, IPsec VPN, CoS |
| NGFW *(default)* | 1.60× | 0.50× | Standard + IPS + AppSecure (AppID, AppFW, AppQoS, AppTrack) |
| Premium NGFW | 2.10× | 0.35× | NGFW + Content Security (Anti-Virus, Anti-Spam, Web Filtering) |

**Throughput note:** The multipliers reflect the processing overhead of enabling IPS and Content Security on Junos. Enabling full IPS/AppSecure reduces effective throughput approximately 50% versus the large-packet baseline; Content Security adds a further ~15% reduction. These figures are consistent with Juniper's published guidance but should be validated against your specific traffic profile (packet size distribution, session count, and signature set depth all affect real-world throughput).

---

## Instance Recommendation Logic

The tool dynamically recommends an instance size based on the traffic slider and selected feature tier:

1. Convert per-direction TB/mo to average Gbps: `perDirGB × 8 ÷ (30 × 86,400 seconds)`
2. Apply a **4× peak multiplier** to account for traffic bursting above average
3. Apply the feature tier's **throughput multiplier** to each instance's NIC bandwidth ceiling
4. Select the **smallest instance whose effective throughput meets or exceeds the estimated peak**

The recommendation is advisory — users can override the selection. The indicator shows three states:

- ✓ **Recommended** — selected instance matches the recommendation
- ↑ **Oversized** — selected instance exceeds what the traffic requires; a smaller instance is suggested
- ⚠ **Undersized** — selected instance's effective throughput is insufficient for the estimated peak

**Important caveat:** NIC bandwidth is used as the throughput ceiling proxy. Actual vSRX throughput is highly sensitive to packet size distribution, concurrent session count, Junos version, and SR-IOV/DPDK configuration. Treat the recommendation as directional, not a hard sizing guarantee.

---

## vSRX Instance Inventory

### AWS — 23 instances (c5 / c5a / c5n / c6in / c7i families)

Instance eligibility sourced from **AWS Marketplace vSRX NGFW listing, Usage Costs tab (May 2026)**. SW base fees taken directly from Marketplace pricing. EC2 on-demand rates sourced from Vantage/Economize (us-east-1, Linux, May 2026).

| Instance | vCPU | Memory | EC2/hr | SW base/hr (NGFW) | NIC |
|---|---|---|---|---|---|
| c5.large | 2 | 4 GiB | $0.085 | $0.65 | up to 10 Gbps |
| c5.xlarge | 4 | 8 GiB | $0.170 | $0.75 | up to 10 Gbps |
| c5a.xlarge | 4 | 8 GiB | $0.154 | $0.75 | up to 10 Gbps |
| c6in.xlarge | 4 | 8 GiB | $0.227 | $0.75 | up to 30 Gbps |
| c7i.xlarge | 4 | 8 GiB | $0.179 | $0.75 | up to 12.5 Gbps |
| c5.2xlarge | 8 | 16 GiB | $0.340 | $0.92 | up to 10 Gbps |
| c5a.2xlarge | 8 | 16 GiB | $0.308 | $0.92 | up to 10 Gbps |
| c5n.2xlarge | 8 | 21 GiB | $0.432 | $0.92 | up to 25 Gbps |
| c6in.2xlarge | 8 | 16 GiB | $0.454 | $0.92 | up to 40 Gbps |
| c7i.2xlarge | 8 | 16 GiB | $0.357 | $0.92 | up to 12.5 Gbps |
| c5.4xlarge | 16 | 32 GiB | $0.680 | $1.77 | up to 10 Gbps |
| c5a.4xlarge | 16 | 32 GiB | $0.616 | $1.77 | up to 10 Gbps |
| c5n.4xlarge | 16 | 42 GiB | $0.864 | $1.77 | up to 25 Gbps |
| c6in.4xlarge | 16 | 32 GiB | $0.907 | $1.77 | up to 50 Gbps |
| c7i.4xlarge | 16 | 32 GiB | $0.714 | $1.77 | up to 12.5 Gbps |
| c5a.8xlarge | 32 | 64 GiB | $1.232 | $3.55 | up to 10 Gbps |
| c5n.9xlarge | 36 | 96 GiB | $1.944 | $3.55 | 50 Gbps |
| c5.9xlarge | 36 | 72 GiB | $1.530 | $3.55 | up to 10 Gbps |
| c6in.8xlarge | 32 | 64 GiB | $1.814 | $3.55 | 50 Gbps |
| c7i.8xlarge | 32 | 64 GiB | $1.428 | $3.55 | 12.5 Gbps |
| c5a.16xlarge | 64 | 128 GiB | $2.464 | $5.70 | up to 20 Gbps |
| c6in.16xlarge | 64 | 128 GiB | $3.629 | $5.70 | 100 Gbps |
| c7i.16xlarge | 64 | 128 GiB | $2.856 | $5.70 | 25 Gbps |

*SW fees shown are the NGFW tier base (1.60× multiplier applied on top of base for NGFW; 2.10× for Premium).*

### GCP — 5 instances (C2 standard series)

C2 standard is GCP's compute-optimized family and the closest equivalent to the AWS c5/c6 instance families used in the Juniper Marketplace listing. On-demand rates sourced from **GCP compute-optimized VM pricing page (us-central1, May 2026)**. SW fees are **estimated** — Juniper does not publish GCP-specific PAYG rates.

| Instance | vCPU | Memory | CE/hr | NIC |
|---|---|---|---|---|
| c2-standard-4 | 4 | 16 GiB | $0.209 | up to 10 Gbps |
| c2-standard-8 | 8 | 32 GiB | $0.418 | up to 16 Gbps |
| c2-standard-16 | 16 | 64 GiB | $0.835 | up to 32 Gbps |
| c2-standard-30 | 30 | 120 GiB | $1.566 | up to 32 Gbps |
| c2-standard-60 | 60 | 240 GiB | $3.132 | up to 32 Gbps |

### Azure — 5 instances (Fsv2 series)

Fsv2 is Azure's compute-optimized series and Microsoft's recommended family for network virtual appliance (NVA) workloads. On-demand rates sourced from **Azure VM pricing (East US, Linux, May 2026)**. SW fees are **estimated** — Azure Marketplace listing states prices are intentionally high for list; actual rates require a private offer from Juniper.

| Instance | vCPU | Memory | VM/hr | NIC |
|---|---|---|---|---|
| F4s v2 | 4 | 8 GiB | $0.169 | 3.125 Gbps |
| F8s v2 | 8 | 16 GiB | $0.338 | 6.25 Gbps |
| F16s v2 | 16 | 32 GiB | $0.676 | 12.5 Gbps |
| F32s v2 | 32 | 64 GiB | $1.352 | 16 Gbps |
| F64s v2 | 64 | 128 GiB | $2.704 | 28 Gbps |

---

## Pricing Sources & Data Confidence

### Native Firewall & Hub/Transit Charges

All native firewall and transit/hub pricing is sourced directly from provider public pricing pages and carries **high confidence**:

| Charge | Rate | Source |
|---|---|---|
| **AWS** ANF endpoint | $0.395/hr/endpoint | AWS Network Firewall pricing page |
| **AWS** ANF data processing | $0.065/GB | AWS Network Firewall pricing page |
| **AWS** TGW attachment | $0.050/hr/attachment | AWS Transit Gateway pricing page |
| **AWS** TGW data processing | $0.020/GB | AWS Transit Gateway pricing page |
| **AWS** data transfer out | $0.090/GB (first 10 TB) | AWS data transfer pricing |
| **GCP** Cloud NGFW Enterprise endpoint | $1.750/hr/endpoint | GCP Cloud NGFW pricing page |
| **GCP** Cloud NGFW Enterprise data | $0.018/GB | GCP Cloud NGFW pricing page |
| **GCP** NCC VPC spoke | $0.100/hr/spoke | GCP NCC pricing page |
| **GCP** NCC ADN fee | $0.020/GiB | GCP NCC pricing page |
| **GCP** data transfer out | $0.085/GB (us-central1) | GCP egress pricing page |
| **Azure** Firewall Standard deployment | $1.250/hr | Azure Firewall pricing page |
| **Azure** Firewall Standard data | $0.016/GB | Azure Firewall pricing page |
| **Azure** vWAN Standard Hub | $0.250/hr | Azure Virtual WAN pricing page |
| **Azure** vWAN hub data processing | $0.020/GB | Azure Virtual WAN pricing page |
| **Azure** VNet peering | $0.010/GB | Azure bandwidth pricing page |
| **Azure** data transfer out (Zone 1) | $0.087/GB | Azure bandwidth pricing page |

*Note: AWS ANF TLS inspection additional data charges were removed per AWS February 2026 price reduction and are not included in the model.*

### vSRX Charges — Data Confidence by Cloud

| Component | AWS | GCP | Azure |
|---|---|---|---|
| EC2/CE/VM compute rates | ✅ Sourced | ✅ Sourced | ✅ Sourced |
| Instance eligibility list | ✅ Sourced (Marketplace listing) | ⚠ Estimated (C2 equiv) | ⚠ Estimated (Fsv2 equiv) |
| SW base fees | ✅ Sourced (Marketplace tab) | ⚠ Estimated | ⚠ Estimated |
| Throughput baselines | ✅ Sourced (Juniper datasheet) | ⚠ Estimated (NIC ceiling proxy) | ⚠ Estimated (NIC ceiling proxy) |
| Tier multipliers (NGFW/Premium) | ✅ Sourced (Juniper guidance) | ✅ Applied uniformly | ✅ Applied uniformly |

**For GCP and Azure:** vSRX costs should be treated as **directional estimates** pending verification from Juniper. The Azure Marketplace listing explicitly states list prices are intentionally elevated; real-world rates require a private offer or BYOL engagement with Juniper/HPE.

---

## What the Model Does Not Include

The following charges are real but excluded from the current model. They apply equally to both architectures (native firewall and vSRX) unless noted:

| Excluded item | Notes |
|---|---|
| CloudWatch / Cloud Monitoring / Azure Monitor logging | Applies to both; volume-dependent |
| Elastic IPs / static IP addresses | vSRX-specific; minor at scale |
| Cross-AZ data transfer during failover | vSRX-specific; incurred during HA failover events — $0.01/GB each direction |
| Security Director / Junos Space management | vSRX-specific; Juniper management plane tooling |
| Direct Connect / ExpressRoute circuit costs | Use-case dependent; planned for future scenario |
| AWS GuardDuty, GCP SCC, Azure Defender | Security tooling that may complement either firewall option |
| BYOL licensing costs | BYOL eliminates the Marketplace SW fee but substitutes a Juniper contract — contact Juniper for pricing |
| Reserved/Committed Use discounts | Both EC2/CE/VM compute and some native firewall charges have commitment discount paths; not modeled |
| Data transfer out tiered pricing | DTO modeled at first-tier rate; blended rate will be lower at high volumes |
| GCP Sustained Use Discounts (SUD) | Automatically applied to C2 instances running >25% of month; would reduce CE costs ~20% |

---

## Key Financial Dynamics

**The fundamental cost structure difference:**

- **Native firewall** = low/predictable fixed cost + **variable per-GB charge** that scales linearly with traffic
- **vSRX (PAYG)** = high fixed cost (compute + SW fee) + **no per-GB firewall charge** beyond hub/transit costs shared with both architectures

This creates a **traffic-volume crossover point** — below it, native firewall is less expensive; above it, vSRX is less expensive. The crossover shifts based on:

- Number of AZs/regions (drives vSRX fixed cost and native endpoint cost)
- Instance size and feature tier selected (drives vSRX fixed cost)
- Cloud provider (native data processing rates vary: ANF $0.065/GB vs Azure Firewall $0.016/GB vs Cloud NGFW $0.018/GB)

**PAYG as worst case:** The model prices vSRX at Marketplace list (PAYG), which is the ceiling. Any commercial engagement with Juniper/HPE reduces vSRX costs:

- **BYOL** eliminates the Marketplace SW fee entirely (the largest variable in the vSRX stack)
- **EC2/CE/VM Reserved or Committed Use** discounts reduce the compute line 30–60% depending on term
- **Enterprise License Agreements** or private Marketplace offers can discount the PAYG SW fee itself

The native firewall options have narrower discount paths — no BYOL equivalent exists, and per-GB data processing charges have no commitment discount path.

---

## SME Feedback Requested

The following areas are where input from subject matter experts would be most valuable:

1. **vSRX instance eligibility on GCP and Azure** — Are C2 standard (GCP) and Fsv2 (Azure) the correct families? Is there a published Juniper compatibility matrix for GCP/Azure that would replace the extrapolation?

2. **vSRX SW fees on GCP and Azure** — Do you have access to current PAYG rates from the GCP or Azure Marketplace listings? The AWS rates are sourced directly; GCP/Azure are estimated.

3. **Throughput figures** — The AWS datasheet numbers (Juniper/AWS joint datasheet) are used as the basis. Do these hold for GCP/Azure on equivalent-spec instances? Is there a Juniper performance guide covering all three clouds?

4. **Feature tier multipliers** — The 0.50× (NGFW) and 0.35× (Premium) throughput multipliers are consistent with published Juniper guidance on IPS overhead. Are these conservative enough for worst-case sizing, or too aggressive?

5. **Additional use cases** — For the planned east-west VPC and Direct Connect head-end scenarios, are there architectural constraints that would change the billing model materially (e.g., east-west traffic not traversing the internet gateway, eliminating the DTO charge)?

6. **GCP Cloud NGFW tier selection** — The model uses Enterprise tier ($1.750/hr + $0.018/GB) as the ANF/Azure Firewall equivalent. Should Standard tier ($0.018/GB, no endpoint fee) be included as an option for workloads that don't require IDPS?

7. **Azure Firewall Premium** — The model uses Standard tier ($1.250/hr). Should Premium ($1.750/hr, same data rate) be included as a selectable option?

---

## Planned Enhancements

- Additional use cases (east-west, Direct Connect/ExpressRoute head-end)
- BYOL toggle to remove Marketplace SW fee and model Juniper contract scenarios
- Reserved/Committed Use discount toggle for EC2/CE/VM compute
- Multi-cloud comparison view (show all three clouds simultaneously)
- Export to CSV/spreadsheet for offline analysis

---

## Technical Notes

The calculator is a **single self-contained HTML file** with no external dependencies beyond Chart.js (loaded from CDN — requires internet connection on first load). All pricing data is hardcoded in the JavaScript configuration objects and can be updated directly.

The file is structured for expansion into a multi-scenario React application via Claude Code. Each cloud provider is a self-contained configuration object (`CLOUDS.aws`, `CLOUDS.gcp`, `CLOUDS.azure`) with its own pricing constants, calculation functions, and line-item renderers. Adding a new use case involves extending these objects with scenario-specific calc functions keyed by use case value.

---
