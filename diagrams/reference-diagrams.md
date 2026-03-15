# Mermaid Diagrams — Azure Architecture Reference

> Use these diagrams in VS Code with the Mermaid preview extension, or paste into any Mermaid-compatible renderer.

---

## 1. Azure Resource Hierarchy

```mermaid
graph TD
    ROOT[Management Group<br/>Root]
    MG1[Management Group<br/>Banking]
    SUB1[Subscription<br/>Lending - Production]
    SUB2[Subscription<br/>Payments - Production]
    SUB3[Subscription<br/>Shared Services]
    SUB4[Subscription<br/>Dev/Test]
    RG1[Resource Group<br/>rg-lending-app]
    RG2[Resource Group<br/>rg-lending-data]
    RG3[Resource Group<br/>rg-payments-app]
    RG4[Resource Group<br/>rg-shared-networking]
    RG5[Resource Group<br/>rg-shared-monitoring]

    ROOT --> MG1
    MG1 --> SUB1
    MG1 --> SUB2
    MG1 --> SUB3
    MG1 --> SUB4
    SUB1 --> RG1
    SUB1 --> RG2
    SUB2 --> RG3
    SUB3 --> RG4
    SUB3 --> RG5

    style ROOT fill:#1a73e8,color:#fff
    style MG1 fill:#4285f4,color:#fff
    style SUB1 fill:#34a853,color:#fff
    style SUB2 fill:#34a853,color:#fff
    style SUB3 fill:#fbbc04,color:#000
    style SUB4 fill:#ea4335,color:#fff
```

---

## 2. Hub-Spoke Network Topology

```mermaid
graph TB
    ONPREM[On-Premises<br/>Core Banking]

    subgraph "Hub VNet (10.0.0.0/16)"
        VPN[VPN Gateway /<br/>ExpressRoute]
        FW[Azure Firewall<br/>10.0.1.0/26]
        BASTION[Azure Bastion<br/>10.0.2.0/26]
        DNS[Private DNS<br/>Resolver]
    end

    subgraph "Spoke 1: Lending (10.1.0.0/16)"
        S1_WEB[Web Tier<br/>10.1.1.0/24]
        S1_APP[App Tier<br/>10.1.2.0/24]
        S1_DATA[Data Tier<br/>10.1.3.0/24]
    end

    subgraph "Spoke 2: Payments (10.2.0.0/16)"
        S2_WEB[Web Tier<br/>10.2.1.0/24]
        S2_APP[App Tier<br/>10.2.2.0/24]
        S2_DATA[Data Tier<br/>10.2.3.0/24]
    end

    subgraph "Spoke 3: Data Platform (10.3.0.0/16)"
        S3_INT[Integration<br/>10.3.1.0/24]
        S3_STORE[Storage PEs<br/>10.3.2.0/24]
        S3_ANALYTICS[Analytics<br/>10.3.3.0/24]
    end

    ONPREM <-->|ExpressRoute| VPN
    VPN <--> FW
    FW <-->|Peering| S1_WEB
    FW <-->|Peering| S2_WEB
    FW <-->|Peering| S3_INT
```

---

## 3. Identity and Access Flow

```mermaid
sequenceDiagram
    participant User
    participant WebApp as Web App
    participant EntraID as Microsoft Entra ID
    participant APIM as API Management
    participant Function as Azure Function
    participant KeyVault as Key Vault
    participant SQL as Azure SQL

    User->>WebApp: 1. Access loan portal
    WebApp->>EntraID: 2. Redirect for authentication
    EntraID->>User: 3. MFA challenge
    User->>EntraID: 4. Complete MFA
    EntraID->>WebApp: 5. Return JWT token
    WebApp->>APIM: 6. API call + JWT token
    APIM->>APIM: 7. Validate JWT (policy)
    APIM->>Function: 8. Forward request
    Function->>KeyVault: 9. Get connection string (Managed Identity)
    KeyVault->>Function: 10. Return secret
    Function->>SQL: 11. Query database
    SQL->>Function: 12. Return data
    Function->>APIM: 13. Response
    APIM->>WebApp: 14. Response
    WebApp->>User: 15. Display loan data
```

---

## 4. Event-Driven Architecture Pattern

```mermaid
graph LR
    subgraph "Producers"
        P1[Web App]
        P2[Mobile App]
        P3[Batch Job]
    end

    subgraph "Ingestion"
        EH[Event Hubs<br/>High throughput]
        SBQ[Service Bus Queue<br/>Reliable delivery]
        EG[Event Grid<br/>Reactive events]
    end

    subgraph "Processing"
        F1[Function:<br/>Stream Processor]
        F2[Function:<br/>Command Handler]
        F3[Function:<br/>Event Handler]
        SA[Stream Analytics:<br/>Real-time Analytics]
    end

    subgraph "Storage"
        COSMOS[(Cosmos DB)]
        SQL[(Azure SQL)]
        BLOB[Blob Storage]
    end

    P1 --> EH
    P2 --> SBQ
    P3 --> BLOB
    BLOB --> EG

    EH --> F1
    EH --> SA
    SBQ --> F2
    EG --> F3

    F1 --> COSMOS
    F2 --> SQL
    F3 --> SQL
    SA --> COSMOS
```

---

## 5. Multi-Region Disaster Recovery

```mermaid
graph TB
    subgraph "Users"
        CLIENT[Client Applications]
    end

    subgraph "Global Traffic"
        AFD[Azure Front Door<br/>Health probes, failover]
    end

    subgraph "Primary: North Europe"
        P_APP[App Service<br/>Zone Redundant]
        P_SQL[(Azure SQL<br/>Business Critical<br/>Zone Redundant)]
        P_COSMOS[(Cosmos DB<br/>Write Region)]
        P_SB[Service Bus<br/>Premium]
        P_KV[Key Vault]
    end

    subgraph "Secondary: West Europe"
        S_APP[App Service<br/>Standby, scaled down]
        S_SQL[(Azure SQL<br/>Auto-Failover Group<br/>Read Replica)]
        S_COSMOS[(Cosmos DB<br/>Read Region)]
        S_SB[Service Bus<br/>Geo-DR Pair]
        S_KV[Key Vault]
    end

    CLIENT --> AFD
    AFD -->|Priority 1| P_APP
    AFD -.->|Priority 2, failover| S_APP

    P_APP --> P_SQL
    P_APP --> P_COSMOS
    P_APP --> P_SB
    P_APP --> P_KV

    S_APP --> S_SQL
    S_APP --> S_COSMOS
    S_APP --> S_SB
    S_APP --> S_KV

    P_SQL -.->|Async replication| S_SQL
    P_COSMOS -.->|Multi-region replication| S_COSMOS
    P_SB -.->|Geo-DR metadata| S_SB
```

---

## 6. Storage Decision Tree

```mermaid
graph TD
    START{What type<br/>of data?}
    START -->|Structured, relational| REL{Need global<br/>distribution?}
    START -->|Semi-structured, NoSQL| NOSQL{Volume &<br/>latency?}
    START -->|Unstructured files| FILES{Analytics<br/>needed?}
    START -->|Messages / Events| MSG{Pattern?}

    REL -->|No| SQL[Azure SQL Database]
    REL -->|Yes| COSMOS_SQL[Cosmos DB<br/>SQL API]

    NOSQL -->|Low latency, global| COSMOS[Cosmos DB]
    NOSQL -->|Large volume, analytics| SYNAPSE[Synapse Analytics]

    FILES -->|Yes, big data| ADLS[Data Lake<br/>Storage Gen2]
    FILES -->|No, just storage| BLOB[Blob Storage]

    MSG -->|Reliable messaging| SB[Service Bus]
    MSG -->|High-throughput streaming| EH[Event Hubs]
    MSG -->|Event notifications| EG[Event Grid]
    MSG -->|Simple queue| QS[Queue Storage]

    style SQL fill:#4285f4,color:#fff
    style COSMOS fill:#34a853,color:#fff
    style COSMOS_SQL fill:#34a853,color:#fff
    style SYNAPSE fill:#9c27b0,color:#fff
    style ADLS fill:#ff9800,color:#fff
    style BLOB fill:#ff9800,color:#fff
    style SB fill:#e91e63,color:#fff
    style EH fill:#e91e63,color:#fff
    style EG fill:#e91e63,color:#fff
    style QS fill:#e91e63,color:#fff
```

---

## 7. Compute Decision Tree

```mermaid
graph TD
    START{What is the<br/>workload type?}
    START -->|Web app / API| WEB{Need full<br/>OS control?}
    START -->|Event-driven, short tasks| FUNC[Azure Functions]
    START -->|Container-based| CONT{Orchestration<br/>needed?}
    START -->|Batch processing| BATCH[Azure Batch]
    START -->|Legacy, full VM| VM[Virtual Machines]

    WEB -->|No, managed| APPS[App Service]
    WEB -->|Yes| VM_WEB[VM + Scale Set]

    CONT -->|Simple, single container| ACI[Container Instances]
    CONT -->|Microservices, complex| ORCH{Serverless<br/>preferred?}
    ORCH -->|Yes| ACA[Container Apps]
    ORCH -->|No, full control| AKS[AKS]

    style FUNC fill:#4285f4,color:#fff
    style APPS fill:#34a853,color:#fff
    style ACA fill:#9c27b0,color:#fff
    style AKS fill:#ff9800,color:#fff
    style VM fill:#ea4335,color:#fff
    style ACI fill:#fbbc04,color:#000
    style BATCH fill:#795548,color:#fff
```

---

## How to Preview Mermaid Diagrams in VS Code

1. Install the **"Markdown Preview Mermaid Support"** extension in VS Code
2. Open this file and press `Ctrl+Shift+V` to preview
3. Mermaid diagrams will render inline in the markdown preview

Alternatively, paste diagrams into [mermaid.live](https://mermaid.live) for an interactive editor.
