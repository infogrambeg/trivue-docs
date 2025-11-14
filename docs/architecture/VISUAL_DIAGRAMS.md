# Trivue Platform - Vizuelni Dijagrami Arhitekture

Ovaj dokument sadrži kompletne vizuelne dijagrame koji prikazuju arhitekturu Trivue platforme. Dijagrami su kreirani korišćenjem Mermaid syntax-e za maksimalnu kompatibilnost sa GitHub, GitLab i drugim markdown renderer-ima. Svaki dijagram vizualno predstavlja ključne arhitekturne koncepte detaljno opisane u glavnoj dokumentaciji.

## 1. High-Level System Architecture

Kompletan pregled Trivue platforme sa svim eksternim integracijama, API gateway-em, 12 biznis modula i infrastrukturnim komponentama. Ovaj dijagram prikazuje kako različiti delovi sistema komuniciraju i koji protokoli se koriste za komunikaciju.

```mermaid
%%{init: {'flowchart': {'curve': 'step'}}}%%
graph TD
    subgraph external ["🌐 External Systems"]
        MobileApp([📱 Mobile App<br/>iOS/Android])
        BackOffice([💻 Backoffice UI<br/>Next.js 16])
        POSTerminals([🏪 POS Terminals<br/>HMAC Auth])
        EmailProviders([📧 Email Providers<br/>SendGrid/SMTP])
        SMSGateway([📤 SMS Gateway<br/>Twilio/InfoBip])
    end

    subgraph gateway ["🚀 API Gateway Layer"]
        NGINX[🔒 NGINX Reverse Proxy<br/>Load Balancing<br/>SSL Termination<br/>Rate Limiting]

        subgraph apis ["API Endpoints"]
            APIGateway[🔌 REST API Gateway<br/>Port :5024<br/>HTTP/1.1<br/>Public Facing]
            GRPCEndpoint[⚡ gRPC Internal API<br/>Port :5020<br/>HTTP/2<br/>Internal Only]
        end
    end

    subgraph modules ["💎 Business Modules - Modular Monolith"]
        subgraph core ["🏛️ Core Modules"]
            Members[👥 Members Module<br/>DDD Reference<br/>• User Management<br/>• Segmentation]
            Points[💰 Points Module<br/>• Earning Rules<br/>• Transactions<br/>• Balance Tracking]
            Products[📦 Products Module<br/>• Catalog Management<br/>• Categories<br/>• Pricing Rules]
        end

        subgraph features ["🎯 Feature Modules"]
            Rewards[🏆 Rewards Module<br/>• Achievements<br/>• Tiers<br/>• Benefits]
            Campaigns[📢 Campaigns Module<br/>• Promotions<br/>• Targeting<br/>• Analytics]
            Vouchers[🎟️ Vouchers Module<br/>• Generation<br/>• Validation<br/>• Redemption]
            Referrals[🔄 Referrals Module<br/>• Tracking<br/>• Conversions<br/>• Commissions]
        end

        subgraph support ["🛠️ Support Modules"]
            Locations[📍 Locations Module<br/>• Store Management<br/>• Working Hours<br/>• Geo Data]
            Analytics[📊 Analytics Module<br/>• Reports<br/>• Metrics<br/>• KPIs]
            Integration[🔗 Integration Module<br/>Inbox Pattern<br/>• POS Sync<br/>• Outbox Processing]
            Inventory[📋 Inventory Module<br/>• Card Stock<br/>• Serial Numbers<br/>• Batches]
            Notifications[🔔 Notifications Module<br/>• Templates<br/>• Scheduling<br/>• Multi-channel]
        end
    end

    subgraph infra ["🔧 Infrastructure Layer"]
        subgraph dataLayer ["💾 Data Persistence"]
            PostgreSQL[(🐘 PostgreSQL 17<br/>12 Module Schemas<br/>• members<br/>• points<br/>• products<br/>• rewards<br/>• campaigns<br/>• locations<br/>• vouchers<br/>• referrals<br/>• analytics<br/>• integration<br/>• inventory<br/>• notifications)]
        end

        subgraph messagingLayer ["📬 Messaging & Caching"]
            Redis{{💨 Redis Cache<br/>Distributed<br/>• Query Results<br/>• Session Data<br/>• Feature Flags}}
            RabbitMQ{{🐰 RabbitMQ<br/>Message Broker<br/>• Domain Events<br/>• Async Processing<br/>• Dead Letter Queue}}
        end

        subgraph jobsLayer ["⏰ Background Processing"]
            Hangfire[/📅 Hangfire<br/>Background Jobs<br/>• Outbox Processing<br/>• Scheduled Tasks<br/>• Recurring Jobs/]
        end
    end

    subgraph monitoring ["📊 Monitoring & Observability"]
        AppInsights[☁️ Azure App Insights<br/>• APM<br/>• Error Tracking<br/>• User Analytics]
        Seq[📝 Seq Logging<br/>• Structured Logs<br/>• Log Aggregation<br/>• Search & Filter]
        Grafana[📈 Grafana Dashboards<br/>• Real-time Metrics<br/>• Alerts<br/>• Visualization]
    end

    %% External to Gateway connections
    MobileApp ==>|REST/JWT| NGINX
    BackOffice ==>|REST/JWT| NGINX
    POSTerminals ==>|HMAC Auth| NGINX

    %% Gateway to APIs
    NGINX ==>|Proxy| APIGateway
    NGINX -.->|Internal| GRPCEndpoint

    %% APIs to Core Modules
    APIGateway -->|MediatR| Members
    APIGateway -->|MediatR| Points
    APIGateway -->|MediatR| Products

    %% APIs to Feature Modules
    APIGateway -->|CQRS| Rewards
    APIGateway -->|CQRS| Campaigns
    APIGateway -->|CQRS| Vouchers
    APIGateway -->|CQRS| Referrals

    %% APIs to Support Modules
    APIGateway -->|Commands| Locations
    APIGateway -->|Queries| Analytics
    APIGateway -->|Process| Integration
    APIGateway -->|Manage| Inventory
    APIGateway -->|Send| Notifications

    %% gRPC connections (internal only)
    GRPCEndpoint -.->|Read Only| Members
    GRPCEndpoint -.->|Read Only| Points
    GRPCEndpoint -.->|Read Only| Products

    %% Module to Database connections
    Members ==>|EF Core| PostgreSQL
    Points ==>|EF Core| PostgreSQL
    Products ==>|EF Core| PostgreSQL
    Rewards ==>|EF Core| PostgreSQL
    Campaigns ==>|EF Core| PostgreSQL
    Locations ==>|EF Core| PostgreSQL
    Integration ==>|EF Core| PostgreSQL

    %% Cache connections
    Members -.->|Cache| Redis
    Points -.->|Cache| Redis
    Products -.->|Cache| Redis
    Analytics -.->|Cache| Redis
    Campaigns -.->|Cache| Redis

    %% Event Bus connections
    Integration ==>|Publish| RabbitMQ
    Members ==>|Publish| RabbitMQ
    Points ==>|Publish| RabbitMQ
    Locations ==>|Publish| RabbitMQ
    Campaigns ==>|Subscribe| RabbitMQ
    Notifications ==>|Subscribe| RabbitMQ

    %% Background Jobs
    Hangfire -.->|Process| RabbitMQ
    Hangfire -.->|Schedule| Notifications
    Hangfire -.->|Outbox| Integration

    %% External integrations
    Notifications ==>|Send| EmailProviders
    Notifications ==>|Send| SMSGateway

    %% Monitoring connections
    Members -.->|Telemetry| AppInsights
    Points -.->|Telemetry| AppInsights
    Integration -.->|Logs| Seq
    Analytics -.->|Metrics| Grafana
    Campaigns -.->|Performance| Grafana

    %% Style definitions
    classDef external fill:#e3f2fd,stroke:#1976d2,stroke-width:3px,color:#000,padding:10px
    classDef api fill:#fff3e0,stroke:#f57c00,stroke-width:3px,color:#000,padding:10px
    classDef module fill:#f3e5f5,stroke:#7b1fa2,stroke-width:3px,color:#000,padding:10px
    classDef database fill:#ffebee,stroke:#c62828,stroke-width:3px,color:#000,padding:10px
    classDef messaging fill:#fff9c4,stroke:#f9a825,stroke-width:3px,color:#000,padding:10px
    classDef cache fill:#e0f2f1,stroke:#00796b,stroke-width:3px,color:#000,padding:10px
    classDef jobs fill:#fce4ec,stroke:#ad1457,stroke-width:3px,color:#000,padding:10px
    classDef monitor fill:#f1f8e9,stroke:#558b2f,stroke-width:3px,color:#000,padding:10px

    %% Apply styles
    class MobileApp,BackOffice,POSTerminals,EmailProviders,SMSGateway external
    class NGINX,APIGateway,GRPCEndpoint api
    class Members,Points,Products,Rewards,Referrals,Locations,Vouchers,Campaigns,Analytics,Integration,Inventory,Notifications module
    class PostgreSQL database
    class RabbitMQ messaging
    class Redis cache
    class Hangfire jobs
    class AppInsights,Seq,Grafana monitor
```

## 2. Triple-Layer Event-Driven Architecture

Prikaz revolucionarnog triple-layer sistema koji kombinuje brzinu instant feedback-a sa pouzdanošću eventual consistency i performansama cache sistema. Svaki sloj ima specifičnu ulogu u garantovanju brzih response vremena i pouzdane komunikacije između modula.

```mermaid
sequenceDiagram
    participant User as 👤 User Action
    participant API as REST API
    participant Domain as Domain Aggregate
    participant Outbox as Outbox Table
    participant Layer1 as Layer 1: MediatR<br/>(Instant <1ms)
    participant Layer2 as Layer 2: RabbitMQ<br/>(Eventual 5-10min)
    participant Layer3 as Layer 3: Redis<br/>(Cache)
    participant Consumer as Event Consumer
    participant Target as Target Module

    Note over User,Target: User promeni ime lokacije

    User->>API: PUT /locations/{id}
    API->>Domain: UpdateLocationName()
    Domain->>Domain: Biznis validacija
    Domain-->>Outbox: Persist LocationUpdatedEvent

    Note over Layer1,Layer3: Paralelno procesiranje

    par Layer 1: Instant Feedback
        Outbox->>Layer1: Publish to MediatR
        Layer1->>Target: Handle instantly
        Layer1->>Layer3: Invalidate cache keys
        Note over Layer1: Backoffice vidi promene odmah!
    and Layer 2: Guaranteed Delivery
        Outbox->>Layer2: Hangfire Job (every 30s)
        Layer2->>Consumer: Process from queue
        Consumer->>Consumer: Check idempotency
        Consumer->>Target: Update if needed
        Note over Layer2: Self-healing reconciliation
    end

    API-->>User: 200 OK (instant)

    Note over Layer3: Sledeći GET će imati cache miss<br/>i učitati fresh podatke
```

## 3. ICrossModuleEvent Pattern Flow

Elegantno rešenje koje eliminiše duplo procesiranje cross-module event-a. Eventi označeni sa ICrossModuleEvent interfejsom automatski se preskaču u Layer 1 i procesiraju isključivo kroz Layer 2, što garantuje da se svaki event procesira tačno jednom.

```mermaid
%%{init: {'flowchart': {'curve': 'step'}}}%%
flowchart TD
    Start([Domain Event Raised])
    Check{{"Implements<br/>ICrossModuleEvent?"}}

    LocalEvent[Regular Domain Event]
    CrossEvent[Cross-Module Event]

    MediatR[Layer 1: MediatR<br/>Publish Locally]
    Skip[/Skip Layer 1<br/>Filter Out/]

    Outbox[(Save to Outbox Table)]

    RabbitMQ{{Layer 2: RabbitMQ<br/>Process via Inbox Pattern}}

    LocalHandlers[Local Event Handlers<br/>Process Immediately]
    CrossHandlers[Cross-Module Handlers<br/>Process Asynchronously]

    Cache{{Layer 3: Redis<br/>Invalidate Cache}}

    End([Event Processed])

    Start ==>|Trigger| Check
    Check -->|No| LocalEvent
    Check -->|Yes| CrossEvent

    LocalEvent ==>|Publish| MediatR
    LocalEvent -->|Persist| Outbox
    CrossEvent -.->|Skip| Skip
    CrossEvent -->|Persist| Outbox

    MediatR ==>|Handle| LocalHandlers
    MediatR -.->|Invalidate| Cache

    Outbox ==>|Queue| RabbitMQ
    RabbitMQ ==>|Consume| CrossHandlers

    LocalHandlers -->|Complete| End
    CrossHandlers -->|Complete| End
    Cache -.->|Updated| End

    classDef decision fill:#ffeb3b,stroke:#f57f17,stroke-width:3px,color:#000
    classDef skip fill:#ff5252,stroke:#c62828,stroke-width:3px,color:#fff
    classDef layer1 fill:#66bb6a,stroke:#2e7d32,stroke-width:3px,color:#000
    classDef layer2 fill:#42a5f5,stroke:#1565c0,stroke-width:3px,color:#000
    classDef layer3 fill:#ffa726,stroke:#ef6c00,stroke-width:3px,color:#000
    classDef storage fill:#e1bee7,stroke:#6a1b9a,stroke-width:3px,color:#000
    classDef handler fill:#c5cae9,stroke:#303f9f,stroke-width:3px,color:#000
    classDef start fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
    classDef endpoint fill:#ffccbc,stroke:#d84315,stroke-width:3px,color:#000

    class Check decision
    class Skip skip
    class MediatR layer1
    class RabbitMQ layer2
    class Cache layer3
    class Outbox storage
    class LocalHandlers,CrossHandlers handler
    class LocalEvent,CrossEvent handler
    class Start start
    class End endpoint
```

## 4. Hexagonal Architecture - Single Module Deep Dive

Detaljan prikaz hexagonal arhitekture na primeru Members modula. Prikazuje četiri sloja sa jasnom separacijom koncerna i dependency inversion principom. Domain sloj nikad ne zavisi od Infrastructure sloja - to je nepovredivo pravilo.

```mermaid
%%{init: {'flowchart': {'curve': 'step'}}}%%
graph LR
    subgraph presentation ["🎨 Presentation Layer"]
        Controller[REST Controller<br/>MembersController]
        GrpcService[gRPC Service<br/>MembersGrpcService]
        Validators[Request<br/>Validators]
    end

    subgraph application ["⚙️ Application Layer"]
        Commands[Commands<br/>• RegisterMember<br/>• UpdateProfile]
        Queries[Queries<br/>• GetMember<br/>• SearchMembers]
        Handlers[MediatR<br/>Handlers]
        EventHandlers[Domain Event<br/>Handlers]
        Mappers[AutoMapper<br/>Profiles]
    end

    subgraph domain ["💎 Domain Layer - Pure Business Logic"]
        Aggregates[Aggregates<br/>• Member<br/>• Segment]
        ValueObjects[Value Objects<br/>• Email<br/>• MemberId]
        DomainEvents[Domain Events<br/>• MemberRegistered<br/>• CardAssigned]
        DomainServices[Domain<br/>Services]
        RepoInterfaces[Repository<br/>Interfaces]
    end

    subgraph infrastructure ["🔧 Infrastructure Layer"]
        Repositories[EF Core<br/>Repositories]
        DbContext[Members<br/>DbContext]
        Messaging{{RabbitMQ<br/>Consumers}}
        Cache{{Redis Cache<br/>Service}}
        ExternalAPIs([External API<br/>Clients])
    end

    subgraph database ["💾 Database"]
        PostgreSQL[(PostgreSQL<br/>members schema)]
    end

    Controller ==>|HTTP| Commands
    Controller ==>|HTTP| Queries
    GrpcService -.->|gRPC| Queries

    Commands -->|Execute| Handlers
    Queries -->|Execute| Handlers
    Handlers ==>|Use| Aggregates
    Handlers -->|Via Interface| RepoInterfaces

    EventHandlers -.->|Subscribe| DomainEvents

    Aggregates ==>|Contains| ValueObjects
    Aggregates ==>|Raise| DomainEvents
    Aggregates -->|Use| DomainServices

    Repositories -.->|Implements| RepoInterfaces
    Repositories -->|Use| DbContext
    DbContext -->|Persist| PostgreSQL

    Messaging ==>|Trigger| EventHandlers
    Cache -.->|Speed Up| Queries
    ExternalAPIs -.->|Call| Repositories

    classDef presentation fill:#e3f2fd,stroke:#1976d2,stroke-width:3px,color:#000
    classDef application fill:#fff3e0,stroke:#f57c00,stroke-width:3px,color:#000
    classDef domain fill:#f3e5f5,stroke:#7b1fa2,stroke-width:4px,color:#000
    classDef infrastructure fill:#e8f5e9,stroke:#388e3c,stroke-width:3px,color:#000
    classDef database fill:#ffebee,stroke:#c62828,stroke-width:3px,color:#000
    classDef messaging fill:#fff9c4,stroke:#f9a825,stroke-width:3px,color:#000

    class Controller,GrpcService,Validators presentation
    class Commands,Queries,Handlers,EventHandlers,Mappers application
    class Aggregates,ValueObjects,DomainEvents,DomainServices,RepoInterfaces domain
    class Repositories,DbContext,ExternalAPIs infrastructure
    class Messaging,Cache messaging
    class PostgreSQL database
```

## 5. gRPC Internal API Architecture (4th Layer)

Opcioni četvrti sloj koji omogućava brze sinhrone read operacije između modula. gRPC server reuse-uje iste MediatR query handler-e kao REST API, dok client implementacija uključuje automatski fallback na HTTP ako gRPC nije dostupan.

```mermaid
sequenceDiagram
    participant Campaign as Campaigns Module
    participant Adapter as MembersModuleClientAdapter
    participant Feature as Feature Flag
    participant gRPC as gRPC Client<br/>(:5020)
    participant HTTP as HTTP Client<br/>(:5024)
    participant MembersAPI as Members Module
    participant Query as MediatR Query Handler
    participant DB as PostgreSQL

    Note over Campaign,DB: Campaign validira target members

    Campaign->>Adapter: GetMemberAsync(memberId)
    Adapter->>Feature: Check UseGrpcForMembers

    alt gRPC Enabled & Available
        Feature-->>Adapter: true
        Adapter->>gRPC: GetMember(request)
        gRPC->>MembersAPI: gRPC call (HTTP/2)
        MembersAPI->>Query: Handle GetMemberQuery
        Query->>DB: Fetch data
        DB-->>Query: Member data
        Query-->>MembersAPI: MemberDto
        MembersAPI-->>gRPC: Response (2-5ms)
        gRPC-->>Adapter: MemberDto
    else Fallback to HTTP
        Feature-->>Adapter: false OR gRPC failed
        Adapter->>HTTP: GET /api/members/{id}
        HTTP->>MembersAPI: REST call (HTTP/1.1)
        MembersAPI->>Query: Handle GetMemberQuery
        Query->>DB: Fetch data
        DB-->>Query: Member data
        Query-->>MembersAPI: MemberDto
        MembersAPI-->>HTTP: Response (50-100ms)
        HTTP-->>Adapter: MemberDto
    end

    Adapter-->>Campaign: MemberDto

    Note over gRPC,HTTP: gRPC je 10-20x brži (2-5ms vs 50-100ms)
```

## 6. Module Communication via Events

Konkretan primer kako Members modul komunicira sa Points modulom preko event-driven arhitekture. Prikazuje kompletni flow od registracije member-a do kreiranja points account-a sa Outbox i Inbox pattern-ima.

```mermaid
sequenceDiagram
    participant User as 👤 Admin User
    participant API as Members API
    participant MemberAgg as Member Aggregate
    participant OutboxDB as Outbox Table<br/>(members schema)
    participant MediatR as MediatR Publisher
    participant Hangfire as Hangfire Job
    participant RabbitMQ as RabbitMQ Exchange
    participant InboxDB as Inbox Table<br/>(points schema)
    participant PointsConsumer as Points Consumer
    participant PointsModule as Points Module

    User->>API: POST /members/register
    API->>MemberAgg: RegisterMember()
    MemberAgg->>MemberAgg: Validate business rules
    MemberAgg-->>OutboxDB: Save MemberRegisteredEvent

    Note over MediatR,PointsModule: Dual processing paths

    par Layer 1: Instant (pokušaj)
        OutboxDB->>MediatR: Publish event
        MediatR->>PointsModule: Try instant creation
        Note over MediatR: Ako failuje, nije kritično
    and Layer 2: Guaranteed (reconciliation)
        loop Every 30 seconds
            Hangfire->>OutboxDB: Check unprocessed
            OutboxDB-->>Hangfire: MemberRegisteredEvent
            Hangfire->>RabbitMQ: Publish to exchange
            RabbitMQ->>InboxDB: Store in inbox
            InboxDB->>PointsConsumer: Process message
            PointsConsumer->>PointsConsumer: Check idempotency
            alt Not yet processed
                PointsConsumer->>PointsModule: CreatePointsAccount()
                PointsModule-->>PointsConsumer: Account created
                PointsConsumer->>InboxDB: Mark as processed
            else Already processed
                Note over PointsConsumer: Skip (idempotent)
            end
        end
    end

    API-->>User: 201 Created
```

## 7. Database Schema-per-Module Architecture

Prikaz kako svaki od 12 modula ima sopstvenu PostgreSQL schema-u, što omogućava laku transformaciju u mikroservise. Svaki modul potpuno poseduje svoje podatke i ne deli tabele sa drugim modulima.

```mermaid
%%{init: {'flowchart': {'curve': 'step'}}}%%
graph TB
    subgraph postgres ["💾 PostgreSQL Database Server - Port 5432"]
        subgraph schemas ["📂 Module Schemas - Complete Data Isolation"]
            members[(members schema<br/>• Members<br/>• Cards<br/>• Segments<br/>• Addresses)]

            points[(points schema<br/>• PointsAccounts<br/>• Transactions<br/>• EarningRules<br/>• Balances)]

            products[(products schema<br/>• Products<br/>• Categories<br/>• Attributes<br/>• Pricing)]

            rewards[(rewards schema<br/>• Rewards<br/>• Achievements<br/>• Redemptions<br/>• Tiers)]

            referrals[(referrals schema<br/>• ReferralCodes<br/>• Tracking<br/>• Conversions)]

            locations[(locations schema<br/>• Locations<br/>• WorkingHours<br/>• Coordinates)]

            vouchers[(vouchers schema<br/>• Vouchers<br/>• Validations<br/>• Redemptions)]

            campaigns[(campaigns schema<br/>• Campaigns<br/>• Targets<br/>• Analytics)]

            analytics[(analytics schema<br/>• Events<br/>• Metrics<br/>• Reports)]

            integration[(integration schema<br/>• InboxMessages<br/>• OutboxMessages<br/>• POSDevices)]

            inventory[(inventory schema<br/>• PhysicalCards<br/>• SerialNumbers<br/>• Batches)]

            notifications[(notifications schema<br/>• Templates<br/>• History<br/>• Preferences)]
        end

        shared[(shared schema<br/>• Countries<br/>• Currencies<br/>• TimeZones)]
    end

    members -.->|Domain Events| integration
    points -.->|Domain Events| integration
    products -.->|Domain Events| integration
    locations -.->|Domain Events| integration
    rewards -.->|Domain Events| integration
    campaigns -.->|Domain Events| integration

    classDef memberSchema fill:#e3f2fd,stroke:#1976d2,stroke-width:3px,color:#000
    classDef pointSchema fill:#f3e5f5,stroke:#7b1fa2,stroke-width:3px,color:#000
    classDef productSchema fill:#e8f5e9,stroke:#388e3c,stroke-width:3px,color:#000
    classDef rewardSchema fill:#fce4ec,stroke:#c2185b,stroke-width:3px,color:#000
    classDef locationSchema fill:#e0f2f1,stroke:#00796b,stroke-width:3px,color:#000
    classDef integrationSchema fill:#fff3e0,stroke:#f57c00,stroke-width:4px,color:#000
    classDef sharedSchema fill:#eceff1,stroke:#455a64,stroke-width:3px,color:#000
    classDef otherSchema fill:#f1f8e9,stroke:#558b2f,stroke-width:3px,color:#000

    class members memberSchema
    class points pointSchema
    class products productSchema
    class rewards rewardSchema
    class locations locationSchema
    class integration integrationSchema
    class shared sharedSchema
    class referrals,vouchers,campaigns,analytics,inventory,notifications otherSchema
```

## 8. Microservices Migration Path

Vizuelni prikaz kako će platforma evoluirati od trenutnog modular monolith-a ka potpunoj mikroservisnoj arhitekturi. Plan migracije omogućava postupnu transformaciju bez prekida u radu sistema.

```mermaid
%%{init: {'flowchart': {'curve': 'step'}}}%%
graph TD
    subgraph current ["📦 Trenutno Stanje - Modular Monolith (2025)"]
        subgraph single ["Single Deployment Unit"]
            MM_API[REST API :5024<br/>+ gRPC :5020]
            MM_Modules[12 Modules<br/>Members, Points, Products...]
            MM_DB[(Single PostgreSQL<br/>12 Schemas)]
            MM_Infra{{RabbitMQ + Redis<br/>+ Hangfire}}
        end

        MM_API ==>|Direct| MM_Modules
        MM_Modules -->|EF Core| MM_DB
        MM_Modules -.->|Events| MM_Infra
    end

    Arrow1[["⬇️ Faza 1-2: Infrastructure Setup ⬇️"]]

    subgraph transition ["🔄 Prelazno Stanje (2026 Q1-Q2)"]
        subgraph gateway ["API Gateway Added"]
            T_Gateway[Kong/Ocelot<br/>Gateway]
            T_Discovery{{Consul Service<br/>Discovery}}
            T_Monolith[Monolith sa<br/>10 modula]
            T_Integration[Integration Service<br/>Prvi mikroservis]
            T_Analytics[Analytics Service<br/>Drugi mikroservis]
        end

        T_Gateway ==>|Route| T_Discovery
        T_Discovery -->|Register| T_Monolith
        T_Discovery -->|Register| T_Integration
        T_Discovery -->|Register| T_Analytics
    end

    Arrow2[["⬇️ Faza 3-5: Full Decomposition ⬇️"]]

    subgraph future ["🚀 Buduće Stanje - Microservices (2026 Q3-Q4)"]
        subgraph mesh ["Service Mesh Architecture"]
            F_Gateway[API Gateway<br/>+ Load Balancer]
            F_Discovery{{Service Discovery<br/>+ Health Checks}}

            subgraph core ["Core Services"]
                F_Members[Members Service<br/>Own DB]
                F_Points[Points Service<br/>Own DB]
                F_Products[Products Service<br/>Own DB]
                F_Rewards[Rewards Service<br/>Own DB]
            end

            subgraph supporting ["Supporting Services"]
                F_Integration[Integration Service<br/>Own DB]
                F_Analytics[Analytics Service<br/>Own DB]
                F_Notifications[Notifications Service<br/>Own DB]
            end

            subgraph infra2 ["Infrastructure"]
                F_Kafka{{Kafka<br/>Event Stream}}
                F_Redis{{Redis Cache<br/>Cluster}}
                F_Monitoring[/Distributed Tracing<br/>+ Monitoring/]
            end
        end

        F_Gateway ==>|Route| F_Discovery
        F_Discovery -.->|Discover| F_Members
        F_Discovery -.->|Discover| F_Points
        F_Discovery -.->|Discover| F_Integration

        F_Members ==>|Publish| F_Kafka
        F_Points ==>|Publish| F_Kafka
        F_Integration ==>|Subscribe| F_Kafka

        F_Members -.->|Cache| F_Redis
        F_Analytics -.->|Cache| F_Redis
    end

    classDef currentState fill:#ffeb3b,stroke:#f57c00,stroke-width:3px,color:#000
    classDef transitionState fill:#66bb6a,stroke:#2e7d32,stroke-width:3px,color:#000
    classDef futureState fill:#42a5f5,stroke:#1565c0,stroke-width:3px,color:#000
    classDef gateway fill:#ff7043,stroke:#d84315,stroke-width:4px,color:#000
    classDef service fill:#f3e5f5,stroke:#7b1fa2,stroke-width:3px,color:#000
    classDef infra fill:#e0f2f1,stroke:#00796b,stroke-width:3px,color:#000
    classDef messaging fill:#fff9c4,stroke:#f9a825,stroke-width:3px,color:#000
    classDef arrow fill:#e8eaf6,stroke:#3f51b5,stroke-width:2px,color:#000

    class MM_API,MM_Modules currentState
    class T_Integration,T_Analytics transitionState
    class F_Members,F_Points,F_Products,F_Rewards,F_Integration,F_Analytics,F_Notifications service
    class T_Gateway,F_Gateway gateway
    class MM_DB,MM_Infra,T_Discovery,F_Discovery,F_Redis infra
    class F_Kafka messaging
    class Arrow1,Arrow2 arrow
```

## 9. Event Flow Decision Matrix

Vizuelni vodič koji pomaže developerima da odluče koji layer da koriste za različite scenarije. Prikazuje decision tree sa jasnim kriterijumima za izbor između Layer 1, Layer 2 i Layer 3.

```mermaid
%%{init: {'flowchart': {'curve': 'step'}}}%%
flowchart TD
    Start([Novi Event/Operacija])

    Q1{{"Da li je<br/>cross-module?"}}
    Q2{{"Da li backoffice<br/>treba instant<br/>feedback?"}}
    Q3{{"Da li je<br/>bulk operacija?"}}
    Q4{{"Da li treba<br/>guaranteed<br/>delivery?"}}
    Q5{{"Da li su podaci<br/>često čitani?"}}

    L1[Layer 1: MediatR<br/>⚡ Instant < 1ms<br/>In-memory]
    L2[Layer 2: RabbitMQ<br/>🐰 Eventual 5-10min<br/>Persistent queue]
    L3[Layer 3: Redis<br/>💨 Cache invalidation<br/>Performance]
    L1L2[Layer 1 + Layer 2<br/>⚡+🐰 Instant + Guaranteed]
    L2Only[Samo Layer 2<br/>🐰 Async processing]

    Example1[/"📍 Location name update<br/>→ POS sync"/]
    Example2[/"👥 Member registration<br/>→ Points account"/]
    Example3[/"📢 Campaign<br/>→ 10k emails"/]
    Example4[/"📦 Product price<br/>→ Cache update"/]

    Start ==>|Begin| Q1
    Q1 -->|Da| Q4
    Q1 -->|Ne| Q2

    Q2 -->|Da| L1
    Q2 -->|Ne| Q5

    Q4 -->|Da| Q3
    Q4 -->|Ne| L1

    Q3 -->|Da| L2Only
    Q3 -->|Ne| L1L2

    Q5 -->|Da| L3
    Q5 -->|Ne| L1

    L1L2 -.->|Use Case| Example1
    L1L2 -.->|Use Case| Example2
    L2Only -.->|Use Case| Example3
    L3 -.->|Use Case| Example4

    classDef start fill:#c8e6c9,stroke:#2e7d32,stroke-width:3px,color:#000
    classDef decision fill:#ffeb3b,stroke:#f57f17,stroke-width:3px,color:#000
    classDef layer1 fill:#e8f5e9,stroke:#388e3c,stroke-width:3px,color:#000
    classDef layer2 fill:#e3f2fd,stroke:#1976d2,stroke-width:3px,color:#000
    classDef layer3 fill:#fff3e0,stroke:#f57c00,stroke-width:3px,color:#000
    classDef combined fill:#f3e5f5,stroke:#7b1fa2,stroke-width:4px,color:#000
    classDef async fill:#fce4ec,stroke:#c2185b,stroke-width:3px,color:#000
    classDef example fill:#f5f5f5,stroke:#757575,stroke-width:2px,color:#000,font-style:italic

    class Start start
    class Q1,Q2,Q3,Q4,Q5 decision
    class L1 layer1
    class L2 layer2
    class L3 layer3
    class L1L2 combined
    class L2Only async
    class Example1,Example2,Example3,Example4 example
```

## Dodatne Napomene

Svi dijagrami koriste Mermaid syntax koji je potpuno kompatibilan sa:
- GitHub markdown rendering
- GitLab markdown rendering
- VS Code sa Mermaid preview ekstenzijom
- Većinom modernih dokumentacionih alata

Za najbolji prikaz preporučujemo korišćenje GitHub-a ili alata sa Mermaid podrškom. Dijagrami su dizajnirani da budu čitljivi i u običnom markdown formatu kao kod blokovi.

---

*Dokumentovano: 2025-11-14*
*Verzija: 1.2*
*Status: Production Ready - Fixed Mermaid Rendering Issues*