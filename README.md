# Trivue Platform - Vizuelni Dijagrami Procesa i Integracija

**Datum:** 2025-11-07
**Verzija:** v2.13 (Simplified)
**Status:** Production-Ready

> **NAPOMENA:** Dijagrami su pojednostavljeni za bolju čitljivost - fokus na ključnim komponentama i vezama.
> Za detaljne flow-ove pogledati sequence dijagrame (#2, #3, #4, #6, #7).

---

## 1. Architecture Overview - 11 Modula sa Event Flows

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'background':'#ffffff',
  'primaryColor':'#0b5fff',
  'secondaryColor':'#0f172a',
  'tertiaryColor':'#10b981',
  'lineColor':'#334155',
  'textColor':'#0f172a',
  'fontSize':'16px',
  'nodeTextColor':'#0f172a',
  'edgeLabelBackground':'#e6f2ff',
  'clusterBkg':'#f8fafc'
}}}%%
graph LR
    classDef core fill:#0b5fff,stroke:#062e8a,stroke-width:2px,color:#ffffff,font-weight:600;
    classDef integration fill:#ef4444,stroke:#7f1d1d,stroke-width:2px,color:#ffffff,font-weight:600;
    classDef support fill:#f59e0b,stroke:#78350f,stroke-width:2px,color:#ffffff,font-weight:600;
    classDef infra fill:#94a3b8,stroke:#475569,stroke-width:1px,color:#0f172a,font-weight:600;
    classDef external fill:#7c3aed,stroke:#4c1d95,stroke-width:2px,color:#ffffff,font-weight:600;

    POS[POS Devices]:::external
    Mobile[Mobile App]:::external
    ESIR[ESIR Fiscal]:::external

    Members[MEMBERS]:::core
    Points[POINTS]:::core
    Rewards[REWARDS]:::core
    Integration[INTEGRATION]:::integration
    Locations[LOCATIONS]:::integration
    Campaigns[CAMPAIGNS]:::support

    DB[(PostgreSQL)]:::infra
    Cache[(Redis)]:::infra
    MQ[RabbitMQ]:::infra

    POS -->|HMAC| Integration
    Mobile -->|JWT| Members
    Integration -->|Async| ESIR

    Members -.->|Event| Points
    Members -.->|Event| Campaigns
    Integration -.->|Event| Points

    Members --> DB
    Points --> DB
    Integration --> DB
    Locations --> DB
    Rewards --> DB
    Campaigns --> DB

    Members --> Cache
    Points --> Cache

    Integration --> MQ
    Points --> MQ
```

---

## 2. Triple-Layer Event-Driven Architecture

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'background':'#ffffff',
  'textColor':'#0f172a',
  'fontSize':'15px',
  'noteBkgColor':'#f1f5f9',
  'noteTextColor':'#0f172a',
  'lineColor':'#334155'
}}}%%
sequenceDiagram
    autonumber
    participant Location as Locations Module
    participant Outbox as Outbox Interceptor
    participant L1 as Layer 1: MediatR\n(INSTANT)
    participant L2 as Layer 2: RabbitMQ\n(EVENTUAL)
    participant L3 as Layer 3: Redis Cache\n(PERFORMANCE)
    participant Integration as Integration Module
    participant DB as PostgreSQL

    Note right of Location: larger-font, high-contrast labels
    Location->>Location: location.Update("New Name")
    Location->>Location: RaiseDomainEvent(LocationUpdated)
    Location->>DB: SaveChanges() - ATOMIC
    DB-->>Outbox: Commit + Outbox INSERT

    Note over Outbox: ICrossModuleEvent Filtering
    Outbox->>Outbox: Is ICrossModuleEvent?
    Outbox->>Outbox: YES → SKIP Layer 1

    Note over L2: Hangfire Job (every 30s)
    L2->>Outbox: ProcessOutboxMessages()
    Outbox-->>L2: LocationUpdatedEvent
    L2->>L2: Publish to RabbitMQ
    L2->>Integration: Consume from Queue

    Integration->>Integration: Inbox Pattern - Check EventId
    Integration->>Integration: NOT processed → Continue
    Integration->>DB: Denormalize LocationName in POS Devices
    Integration->>L3: INVALIDATE cache keys
    L3-->>L3: Remove "integration:pos-devices:*"

    Note over Integration: Handler executed ONCE (no duplication)
```

---

## 3. POS Integration Flow - Transaction Recording

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'background':'#ffffff',
  'textColor':'#0f172a',
  'fontSize':'15px',
  'noteBkgColor':'#f8fafc',
  'noteTextColor':'#0f172a',
  'lineColor':'#334155'
}}}%%
sequenceDiagram
    autonumber
    participant POS as POS Device
    participant HMAC as HMAC Middleware
    participant API as Integration API
    participant Trans as Transaction Aggregate
    participant Points as Points Module
    participant ESIR as ESIR Fiscalization
    participant RabbitMQ as RabbitMQ
    participant DB as PostgreSQL

    POS->>HMAC: POST /api/pos/transactions\nHeaders: ApiKey, Timestamp, Signature
    HMAC->>HMAC: Validate Timestamp (±5 min)
    HMAC->>HMAC: Lookup PosDevice by ApiKey
    HMAC->>HMAC: Compute HMAC-SHA256(secret, payload)
    HMAC->>HMAC: Compare Signatures
    alt Signature Valid
        HMAC->>API: Continue (set PosDeviceId in context)
        API->>Trans: Transaction.Create(items, totalAmount)
        Trans->>Trans: Validate business rules
        Trans->>Trans: RaiseDomainEvent(TransactionCreated)
        API->>DB: SaveChanges() - ATOMIC
        DB-->>API: Transaction committed

        Note over API: Layer 1 - MediatR (INSTANT)
        API->>Points: Publish TransactionCreatedEvent
        Points->>Points: Calculate points (amount + bonus)
        Points->>DB: Create PointsBatch

        Note over API: Layer 2 - Async Fiscalization
        API->>ESIR: FiscalizeAsync(transaction)
        ESIR-->>API: FiscalReceiptNumber + QR Code
        API->>DB: Update Transaction.Fiscalize()

        API-->>POS: 201 Created + TransactionDto + Fiscal Receipt
    else Signature Invalid
        HMAC->>HMAC: Increment FailedAttempts
        HMAC->>HMAC: Auto-block after 5 failures
        HMAC-->>POS: 401 Unauthorized
    end
```

---

## 4. Member Registration Flow - Cross-Module Events

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'background':'#ffffff',
  'textColor':'#0f172a',
  'fontSize':'15px',
  'noteBkgColor':'#f8fafc',
  'noteTextColor':'#0f172a',
  'lineColor':'#334155'
}}}%%
sequenceDiagram
    autonumber
    participant Mobile as Mobile App
    participant Members as Members Module
    participant Outbox as Outbox Pattern
    participant MQ as RabbitMQ
    participant Points as Points Module
    participant Campaigns as Campaigns Module
    participant Referrals as Referrals Module
    participant DB as PostgreSQL

    Mobile->>Members: POST /api/members/register\n{email, password, referralCode}
    Members->>Members: Member.Register()
    Members->>Members: Validate email uniqueness
    Members->>Members: BCrypt hash password
    Members->>Members: RaiseDomainEvent(MemberRegistered)
    Members->>DB: SaveChanges() + Outbox INSERT

    Note over Outbox: Layer 1 SKIPPED (ICrossModuleEvent)
    Outbox->>MQ: Hangfire Job → RabbitMQ Publish

    par Points Module
        MQ->>Points: MemberRegisteredEvent
        Points->>Points: CreatePointsAccount(memberId)
        Points->>DB: Insert PointsAccount (balance: 0)
    and Campaigns Module
        MQ->>Campaigns: MemberRegisteredEvent
        Campaigns->>Campaigns: Enroll in Welcome Campaign
        Campaigns->>DB: Create MemberAchievement
    and Referrals Module (if referralCode)
        MQ->>Referrals: MemberRegisteredEvent
        Referrals->>Referrals: Award referrer bonus (100 points)
        Referrals->>DB: Create Referral record
        Referrals->>Points: Publish PointsEarnedEvent
    end

    Members-->>Mobile: 201 Created + MemberDto
```

---

## 5. Deployment Architecture - Docker Containers

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'background':'#ffffff',
  'textColor':'#0f172a',
  'fontSize':'15px',
  'lineColor':'#334155'
}}}%%
graph LR
    classDef infra fill:#94a3b8,stroke:#475569,stroke-width:1px,color:#0f172a,font-weight:600
    classDef api fill:#0b5fff,stroke:#062e8a,stroke-width:2px,color:#ffffff,font-weight:600
    classDef external fill:#7c3aed,stroke:#4c1d95,stroke-width:2px,color:#ffffff,font-weight:600

    LB[Load Balancer]:::infra

    API1[API-1]:::api
    API2[API-2]:::api
    API3[API-N]:::api

    PG[(PostgreSQL Primary)]:::infra
    PG_R[(PG Replica)]:::infra

    Redis[(Redis Master)]:::infra
    RMQ[RabbitMQ]:::infra

    POS[POS Devices]:::external
    Mobile[Mobile]:::external

    LB --> API1
    LB --> API2
    LB --> API3

    API1 --> PG
    API2 --> PG
    API3 --> PG

    API1 -.-> PG_R
    API2 -.-> PG_R

    PG -.-> PG_R

    API1 --> Redis
    API2 --> Redis

    API1 --> RMQ
    API2 --> RMQ

    POS --> LB
    Mobile --> LB
```

---

## 6. HMAC Authentication Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'background':'#ffffff',
  'textColor':'#0f172a',
  'fontSize':'15px',
  'noteBkgColor':'#f1f5f9',
  'noteTextColor':'#0f172a',
  'lineColor':'#334155'
}}}%%
sequenceDiagram
    autonumber
    participant POS as POS Device
    participant Middleware as HMAC Middleware
    participant Cache as Redis (Nonce Cache)
    participant DB as PostgreSQL
    participant API as API Controller

    POS->>POS: Generate Nonce (unique)
    POS->>POS: timestamp = Unix(now)
    POS->>POS: payload = JSON(request body)
    POS->>POS: signature = HMAC-SHA256(ApiKey:timestamp:nonce:payload, secretKey)

    POS->>Middleware: POST /api/pos/transactions\nHeaders:\nX-POS-ApiKey: pk_live_XXX\nX-POS-Timestamp: 1699123456\nX-POS-Nonce: abc123\nX-POS-Signature: xyz789

    Middleware->>Middleware: Extract Headers
    Middleware->>Middleware: Validate Timestamp\n(|now - timestamp| <= 5 min)

    alt Timestamp Expired
        Middleware-->>POS: 401 Unauthorized\n"Request expired"
    end

    Middleware->>Cache: Check Nonce exists?
    Cache-->>Middleware: Nonce found

    alt Nonce Already Used
        Middleware-->>POS: 401 Unauthorized\n"Duplicate request (replay attack)"
    end

    Middleware->>DB: GetPosDeviceByApiKey(apiKey)
    DB-->>Middleware: PosDevice (with SecretKeyHash)

    alt Device Not Found or Blocked
        Middleware-->>POS: 401 Unauthorized\n"Invalid API key"
    end

    Middleware->>Middleware: expectedSignature = HMAC-SHA256(\n  ApiKey:timestamp:nonce:payload,\n  secretKey from DB\n)
    Middleware->>Middleware: Compare Signatures\n(constant-time comparison)

    alt Signature Invalid
        Middleware->>DB: Increment FailedAttempts
        Middleware->>DB: Auto-block if FailedAttempts >= 5
        Middleware-->>POS: 401 Unauthorized\n"Invalid signature"
    else Signature Valid
        Middleware->>Cache: Store Nonce (TTL: 10 min)
        Middleware->>DB: Reset FailedAttempts = 0
        Middleware->>Middleware: Set HttpContext.Items["PosDeviceId"]
        Middleware->>API: Continue to Controller
        API-->>POS: 200 OK + Response Data
    end
```

---

## 7. Rewards Redemption Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'background':'#ffffff',
  'textColor':'#0f172a',
  'fontSize':'15px',
  'noteBkgColor':'#f8fafc',
  'noteTextColor':'#0f172a',
  'lineColor':'#334155'
}}}%%
sequenceDiagram
    autonumber
    participant Member as Member (Mobile App)
    participant Rewards as Rewards Module
    participant Points as Points Module
    participant Inventory as Inventory Module
    participant DB as PostgreSQL
    participant RabbitMQ as RabbitMQ

    Member->>Rewards: POST /api/rewards/{id}/redeem\n{memberId}
    Rewards->>Rewards: Validate Reward exists & Active
    Rewards->>Points: CheckPointsBalance(memberId)
    Points-->>Rewards: Balance: 500 points

    alt Insufficient Points
        Rewards-->>Member: 400 Bad Request\n"Insufficient points"
    end

    Rewards->>Inventory: CheckStockAvailability(rewardId)
    Inventory-->>Rewards: Stock: 10 items

    alt Out of Stock
        Rewards-->>Member: 409 Conflict\n"Out of stock"
    end

    Rewards->>Rewards: Redemption.Create(memberId, rewardId)
    Rewards->>Rewards: RaiseDomainEvent(RewardRedeemed)
    Rewards->>DB: SaveChanges() + Outbox INSERT

    Note over Rewards: Layer 1 - MediatR (INSTANT)
    Rewards->>Points: Publish RewardRedeemedEvent
    Points->>Points: DeductPoints(memberId, pointsCost)
    Points->>DB: Create PointsTransaction (deduction)

    Rewards->>Inventory: ReserveStock(rewardId, quantity: 1)
    Inventory->>DB: Decrement InventoryCard.Quantity

    Note over Rewards: Layer 2 - Eventual Reconciliation
    Rewards->>RabbitMQ: Publish RewardRedeemedEvent
    RabbitMQ->>Points: Consume (idempotent check)
    RabbitMQ->>Inventory: Consume (idempotent check)

    Rewards-->>Member: 200 OK + RedemptionDto\n{redemptionId, voucherCode, expiresAt}
```

---

## 8. Campaign Achievement Progress Flow

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'background':'#ffffff',
  'textColor':'#0f172a',
  'fontSize':'15px',
  'lineColor':'#334155'
}}}%%
graph LR
    TransEvent[Transaction Event]
    LoginEvent[Login Event]

    Handler[Achievement Handler]
    Progress[Achievement Progress]
    BonusPoints[Award Bonus]

    Push[Push Notification]
    Email[Email]

    TransEvent --> Handler
    LoginEvent --> Handler

    Handler --> Progress
    Progress --> BonusPoints
    Progress --> Push
    Progress --> Email
```

---

## 9. Database Schema - Module Isolation

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'background':'#ffffff',
  'textColor':'#0f172a',
  'fontSize':'14px',
  'lineColor':'#334155'
}}}%%
erDiagram
    MEMBERS ||--o{ MEMBER_CONSENTS : has
    MEMBERS ||--o{ MEMBER_CARDS : has
    MEMBERS {
        uuid Id
        string Email
        string PasswordHash
        string FirstName
        string LastName
    }

    POINTS_ACCOUNTS ||--o{ POINTS_TRANSACTIONS : has
    POINTS_ACCOUNTS {
        uuid Id
        uuid MemberId
        int Balance
        int LifetimeEarned
    }

    POS_DEVICES ||--o{ TRANSACTIONS : records
    TRANSACTIONS ||--o{ TRANSACTION_ITEMS : contains
    TRANSACTIONS {
        uuid Id
        uuid PosDeviceId
        uuid MemberId
        decimal TotalAmount
        string FiscalReceiptNumber
    }

    LOCATIONS ||--o{ POS_DEVICES : hosts
    LOCATIONS {
        uuid Id
        string Name
        string CountryCode
    }

    REWARDS ||--o{ REDEMPTIONS : has
    REDEMPTIONS {
        uuid Id
        uuid MemberId
        uuid RewardId
        int PointsCost
    }
```

---

## 10. CI/CD Pipeline - Automated Deployment

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'background':'#ffffff',
  'textColor':'#0f172a',
  'fontSize':'15px',
  'lineColor':'#334155'
}}}%%
graph LR
    subgraph "Developer Workflow"
        Dev[Developer\nCode Changes]
        PR[Pull Request\nGitHub]
    end

    subgraph "CI Pipeline - GitHub Actions"
        Build[Build\n.NET 9 SDK]
        Test[Run Tests\n1,287 tests]
        Docker[Build Docker Images\nAPI + Backoffice]
        Push[Push to Registry\nDocker Hub / ACR]
    end

    subgraph "CD Pipeline - Kubernetes"
        Helm[Helm Chart Apply\nUpdate Deployment]
        RollingUpdate[Rolling Update\nZero Downtime]
        HealthCheck[Health Checks\nLiveness/Readiness]
        Migrate[Auto-Migrate DB\nEF Core MigrateAsync]
    end

    subgraph "Production Environment"
        K8s[Kubernetes Cluster\n3 API Pods]
        Monitor[Monitoring\nApplication Insights]
        Alerts[Alerts\nSlack/Email]
    end

    Dev --> PR
    PR -->|Merge to main| Build
    Build --> Test
    Test -->|Pass| Docker
    Docker --> Push
    Push --> Helm
    Helm --> Migrate
    Migrate --> RollingUpdate
    RollingUpdate --> HealthCheck
    HealthCheck -->|Pass| K8s
    K8s --> Monitor
    Monitor -->|Failures| Alerts

    classDef dev fill:#9cf,stroke:#333,stroke-width:2px
    classDef ci fill:#9f9,stroke:#333,stroke-width:2px
    classDef cd fill:#fc9,stroke:#333,stroke-width:2px
    classDef prod fill:#f96,stroke:#333,stroke-width:2px

    class Dev,PR dev
    class Build,Test,Docker,Push ci
    class Helm,RollingUpdate,HealthCheck,Migrate cd
    class K8s,Monitor,Alerts prod
```

---

## 11. External System Integrations - Current & Planned

```mermaid
%%{init: {'theme': 'base', 'themeVariables': {
  'background':'#ffffff',
  'textColor':'#0f172a',
  'fontSize':'15px',
  'lineColor':'#334155'
}}}%%
graph LR
    API[Trivue API]

    POS[POS Systems]
    ESIR[ESIR Fiscal]
    Backoffice[Admin Panel]

    Mobile[Mobile Apps]
    WebPortal[Web Portal]
    CDP[CDP Platform]
    Banking[Core Banking]

    Email[SMTP Email]
    Monitor[Monitoring]

    POS -->|HMAC| API
    API -->|Async| ESIR
    Backoffice -->|JWT| API

    Mobile -.->|Planned| API
    WebPortal -.->|Planned| API
    API -.->|Planned| CDP
    API -.->|Planned| Banking

    API --> Email
    API --> Monitor
```

---

## Legenda

**Boje i Simboli:**

- 🟦 **Plavo:** Core moduli (Members, Points, Rewards)
- 🟩 **Zeleno:** Marketing moduli (Campaigns, Referrals, Vouchers)
- 🟥 **Crveno:** Integration moduli (Integration, Locations, Products)
- 🟨 **Žuto:** Support moduli (Analytics, Inventory)
- ⬜ **Sivo:** Infrastructure (PostgreSQL, Redis, RabbitMQ)
- 🟪 **Ljubičasto:** External systems (POS, ESIR, Mobile)

**Event Flow Stilovi:**

- **Puna linija (→):** Synchronous API calls
- **Isprekidana linija (-.->):** Asynchronous events (RabbitMQ)
- **Debela linija:** Critical path
- **Tanka linija:** Support/optional path

---

## Napomene

1. **Triple-Layer Architecture** je centralna inovacija - omogućava INSTANT feedback (Layer 1) + EVENTUAL consistency (Layer 2) + CACHE invalidation (Layer 3)

2. **ICrossModuleEvent Pattern** eliminiše duplo procesiranje - cross-module eventi se preskačaju u Layer 1 i procesiraju SAMO kroz Layer 2 (RabbitMQ)

3. **HMAC Authentication** omogućava sigurnu POS integraciju bez OAuth2 kompleksnosti - replay attack prevention, auto-blocking, per-device keys

4. **Horizontal Scaling** je trivijalno - stateless API sa load balancer-om, PostgreSQL read replicas, Redis cluster

5. **Microservices Migration** je jednostavna - moduli već komuniciraju preko events, ekstraktovanje zahteva samo deployment kao nezavisni container

---

**Kraj dokumenta.**

**Verzija:** 1.0
**Datum:** 2025-11-07
**Autor:** Trivue Architecture Team - Zoran Stevovic
