# Hub-Spoke Networking

> **When to use:** Multiple environments (dev, staging, prod) or business units need isolated networks with shared services (firewall, DNS, monitoring).

---

## Pattern Overview

A central **hub** VNet hosts shared infrastructure. **Spoke** VNets are peered to the hub and isolated from each other unless routed through the hub.

```mermaid
graph TB
    subgraph Hub VNet - 10.0.0.0/16
        FW[Azure Firewall]
        VPN[VPN Gateway<br/>or ExpressRoute]
        DNS[Private DNS Zones]
        BASTION[Azure Bastion]
    end

    subgraph Spoke 1 - Prod - 10.1.0.0/16
        APP1[App Services]
        SQL1[Azure SQL]
    end

    subgraph Spoke 2 - Dev - 10.2.0.0/16
        APP2[App Services]
        SQL2[Azure SQL]
    end

    subgraph Spoke 3 - Data - 10.3.0.0/16
        ADF[Data Factory]
        ADLS[Data Lake]
    end

    subgraph On-Premises
        ONPREM[Corporate Network]
    end

    Spoke 1 <-->|peering| Hub VNet - 10.0.0.0/16
    Spoke 2 <-->|peering| Hub VNet - 10.0.0.0/16
    Spoke 3 <-->|peering| Hub VNet - 10.0.0.0/16
    Hub VNet - 10.0.0.0/16 <-->|VPN / ER| On-Premises
```

## Azure Implementation

| Component | Hub Role | Spoke Role |
|-----------|----------|------------|
| **VNet Peering** | Accepts peering from all spokes | Peers to hub; `UseRemoteGateways = true` |
| **Azure Firewall** | Central egress filtering | UDR `0.0.0.0/0` → Firewall private IP |
| **Private DNS Zones** | Hosts zones, linked to all VNets | Resolves private endpoints via hub DNS |
| **VPN/ExpressRoute** | Single connection to on-premises | Reaches on-prem through hub gateway |
| **Azure Bastion** | Secure VM access without public IPs | VMs in spokes accessed via hub Bastion |
| **NSGs** | Protect shared services subnets | Protect workload subnets |

## Banking Example — Nordic Bank Network Topology

```mermaid
graph TB
    subgraph Hub - Shared Services
        FW[Azure Firewall<br/>Egress + DNAT rules]
        ER[ExpressRoute to<br/>HQ data center]
        PDNS[Private DNS<br/>*.database.windows.net<br/>*.blob.core.windows.net]
    end

    subgraph Spoke: Core Banking - Prod
        APIM[API Management]
        FN[Functions]
        SQLP[(Azure SQL<br/>Private Endpoint)]
    end

    subgraph Spoke: Analytics
        ADF[Data Factory]
        SYN[Synapse Analytics]
        ADLS[(Data Lake<br/>Private Endpoint)]
    end

    subgraph Spoke: Dev/Test
        APPD[App Services]
        SQLD[(SQL Dev)]
    end

    Spoke: Core Banking - Prod <-->|peering| Hub - Shared Services
    Spoke: Analytics <-->|peering| Hub - Shared Services
    Spoke: Dev/Test <-->|peering| Hub - Shared Services
```

**Why hub-spoke here?**
- **Isolation:** Production banking workloads cannot be reached from dev/test.
- **Shared egress:** One Azure Firewall governs all outbound traffic — single point for audit logs.
- **Single on-prem connection:** ExpressRoute in the hub; all spokes reach HQ through it.
- **DNS consistency:** Private DNS zones in the hub resolve private endpoints across all spokes.

## Key Design Decisions

| Decision | Choice | Why |
|----------|--------|-----|
| Spoke-to-spoke traffic | Via Azure Firewall (NVA route) | Spokes don't peer directly; hub firewall inspects |
| Address space | /16 per VNet, /24 per subnet | Room for growth; avoids overlap |
| DNS | Azure Private DNS + hub-linked | No custom DNS servers needed; auto-registers endpoints |
| Gateway transit | Enabled on hub peering | Spokes share VPN/ER gateway without their own |

## Anti-Patterns to Avoid

- **Direct spoke-to-spoke peering** — Defeats isolation. Route through hub if needed.
- **Overlapping address spaces** — Plan CIDR ranges upfront across all VNets.
- **No UDR on spoke subnets** — Without route tables, spoke traffic bypasses the firewall.
- **Too many spokes** — VNet peering has limits (500 per VNet); consider Virtual WAN for large scale.
