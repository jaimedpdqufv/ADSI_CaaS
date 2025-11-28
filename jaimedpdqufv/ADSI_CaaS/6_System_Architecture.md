# System Architecture

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [enunciado.md](enunciado.md)
- [pasame las preguntas y sus respuestas a markdown.md](pasame las preguntas y sus respuestas a markdown.md)

</details>



## Purpose and Scope

This document provides comprehensive technical architecture documentation for the Car as a Service (CaaS) platform. It defines the overall system structure, major subsystems, architectural patterns, and design decisions that govern how the platform operates.

For business context and model explanation, see [Business Model and Concept](#2). For detailed subsystem descriptions, see [High-Level Architecture](#3.1), [Core Technical Components](#3.2), and [Integration Architecture](#3.3). For specific integration patterns with external systems, see [External System Integrations](#5).

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:1-104](), [enunciado.md:1-23]()

---

## Architectural Overview

The CaaS platform employs a **hub-and-spoke integration architecture** with the central CaaS core orchestrating all interactions between customers, dealerships, manufacturing facilities, vehicles, and third-party services. The system is organized into distinct architectural layers with clear separation of concerns.

### Architectural Principles

| Principle | Description | Rationale |
|-----------|-------------|-----------|
| **Centralized Orchestration** | All business logic flows through the CaaS core platform | Single source of truth for business rules and state management |
| **Async-First Integration** | External systems communicate primarily via asynchronous patterns | Enables system resilience and non-blocking operations |
| **Customer-Favorable Failure Handling** | Default to customer protection in failure scenarios | Builds trust and ensures legal compliance |
| **Service Delivery Before Settlement** | Activate services immediately upon payment initiation | Optimizes customer experience despite increased financial risk |
| **No Vehicle Blocking** | Never disable base platform functionality | Legal and safety requirement |

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:31-73]()

---

## System Decomposition

### Major Subsystems

```mermaid
graph TB
    subgraph "Customer Channels"
        WEB["Web Platform"]
        MOBILE["Mobile Application"]
        EXPEDIENTE["Expediente de Compra<br/>(Purchase Records)"]
    end
    
    subgraph "Enterprise Channels"
        INTRANET["Corporate Intranet"]
        SALES["Sales Registration Module"]
    end
    
    subgraph "CaaS Core Platform"
        ORDER["Order Management"]
        CATALOG["Service Catalog"]
        PAYMENT["Payment Processing"]
        NOTIFICATION["Notification Engine"]
        OTA["OTA Activation Engine"]
        CREDENTIAL["Credential Management"]
    end
    
    subgraph "Integration Gateways"
        FACTORY_GW["Factory Gateway"]
        VSS_GW["VSS Gateway"]
        IOT_GW["IoT Gateway"]
        PAYMENT_GW["Payment Gateway"]
        ADMIN_GW["Admin Gateway"]
    end
    
    subgraph "Data Stores"
        CUSTOMER_DB[("Customer Database")]
        ORDER_DB[("Order Database")]
        SERVICE_DB[("Service Database")]
        BILLING_DB[("Billing Database")]
        CONFIG_DB[("Configuration Repository")]
    end
    
    WEB --> EXPEDIENTE
    MOBILE --> EXPEDIENTE
    INTRANET --> SALES
    
    EXPEDIENTE --> ORDER
    EXPEDIENTE --> CATALOG
    SALES --> ORDER
    
    ORDER --> PAYMENT
    CATALOG --> PAYMENT
    PAYMENT --> OTA
    
    ORDER --> FACTORY_GW
    ORDER --> ADMIN_GW
    CATALOG --> VSS_GW
    OTA --> IOT_GW
    
    PAYMENT --> PAYMENT_GW
    
    ORDER --> NOTIFICATION
    OTA --> NOTIFICATION
    SALES --> CREDENTIAL
    CREDENTIAL --> NOTIFICATION
    
    ORDER --> ORDER_DB
    EXPEDIENTE --> CUSTOMER_DB
    CATALOG --> SERVICE_DB
    PAYMENT --> BILLING_DB
    OTA --> SERVICE_DB
    OTA --> CONFIG_DB
```

**Subsystem Responsibilities:**

**Customer Channels:**
- `Web Platform`: Browser-based access to purchase records and service catalog
- `Mobile Application`: Native mobile access with direct vehicle linking capability
- `Expediente de Compra`: Unified customer data repository containing manuals, maintenance history, invoices

**Enterprise Channels:**
- `Corporate Intranet`: Dealership-facing system for sales operations
- `Sales Registration Module`: Captures customer data and commercial plan details during dealership sales

**CaaS Core Platform:**
- `Order Management`: Handles vehicle reservations, orders, and manufacturing tracking
- `Service Catalog`: Manages available optional services (opciones disponibles) with pricing and rules
- `Payment Processing`: Orchestrates all payment types (reservation, final, service, subscription)
- `Notification Engine`: Multi-channel notifications (email, push, SMS, vehicle display)
- `OTA Activation Engine`: Manages service activation via IoT with retry logic
- `Credential Management`: Generates and distributes customer access credentials

**Integration Gateways:**
- `Factory Gateway`: Bidirectional communication with manufacturing facilities
- `VSS Gateway`: Queries Vehicle Service System for maintenance records
- `IoT Gateway`: Commands vehicles and receives telemetry via API IoT
- `Payment Gateway`: Processes card payments with async settlement
- `Admin Gateway`: Registers vehicles with government administrative bodies

**Sources:** [enunciado.md:1-23](), [pasame las preguntas y sus respuestas a markdown.md:8-53]()

---

## Layered Architecture View

```mermaid
graph TB
    subgraph "Presentation Layer"
        P1["Web UI"]
        P2["Mobile UI"]
        P3["Intranet UI"]
    end
    
    subgraph "Application Services Layer"
        A1["Customer Management"]
        A2["Order Orchestration"]
        A3["Service Lifecycle Management"]
        A4["Payment Orchestration"]
        A5["Notification Dispatcher"]
    end
    
    subgraph "Domain Logic Layer"
        D1["Sales Domain<br/>Reservations, Plans, Credentials"]
        D2["Catalog Domain<br/>Opciones Disponibles, Pricing"]
        D3["Activation Domain<br/>OTA, Retry, Status"]
        D4["Billing Domain<br/>Payments, Refunds, Subscriptions"]
        D5["Maintenance Domain<br/>VSS Integration, Service Gating"]
    end
    
    subgraph "Integration Layer"
        I1["Factory API Client"]
        I2["Factory Webhook Receiver"]
        I3["VSS Query Client"]
        I4["IoT API Client"]
        I5["Payment API Client"]
        I6["Admin API Client"]
    end
    
    subgraph "Infrastructure Layer"
        INF1["Database Repositories"]
        INF2["Message Queue"]
        INF3["Logging & Monitoring"]
        INF4["Configuration Management"]
    end
    
    P1 --> A1
    P1 --> A3
    P2 --> A1
    P2 --> A3
    P3 --> A2
    
    A1 --> D1
    A2 --> D1
    A2 --> D2
    A3 --> D2
    A3 --> D3
    A3 --> D5
    A4 --> D4
    A5 --> D1
    A5 --> D3
    
    D1 --> I1
    D1 --> I6
    D2 --> I3
    D3 --> I4
    D4 --> I5
    
    I2 --> D1
    
    D1 --> INF1
    D2 --> INF1
    D3 --> INF1
    D3 --> INF2
    D4 --> INF1
    
    I1 --> INF3
    I2 --> INF3
    I3 --> INF3
    I4 --> INF3
    I5 --> INF3
    I6 --> INF3
    
    D3 --> INF4
```

**Layer Responsibilities:**

**Presentation Layer**: UI components for different user types (customers, dealership employees). No business logic.

**Application Services Layer**: Coordinates multiple domain objects to fulfill use cases. Orchestrates workflows across domains.

**Domain Logic Layer**: Core business rules and entities. Each domain is isolated and represents a cohesive business capability:
- `Sales Domain`: Manages the purchase process from reservation through delivery
- `Catalog Domain`: Defines available services, pricing models, and eligibility rules
- `Activation Domain`: Handles OTA delivery, failure handling, and retry mechanisms
- `Billing Domain`: Processes payments, manages subscriptions (mes vencido), handles refunds (desistimiento)
- `Maintenance Domain`: Gates service access based on VSS maintenance status

**Integration Layer**: Adapters for external systems. Isolates integration complexity from domain logic. Note the hybrid pattern with Factory: synchronous client for order placement (`Factory API Client`) and asynchronous webhook receiver (`Factory Webhook Receiver`) for status updates.

**Infrastructure Layer**: Technical services used across all layers. Message queues support async processing of OTA activations and notifications.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:31-53](), [enunciado.md:8-23]()

---

## Data Architecture

### Data Segregation by Domain

The system employs **database-per-domain** pattern to ensure clear data ownership and enable independent evolution:

```mermaid
graph LR
    subgraph "Customer Data Domain"
        CUST_IDENTITY["Customer Identity<br/>Email, Phone, Address"]
        CUST_CREDS["Access Credentials<br/>Username, Password Hash"]
        CUST_PREFS["Preferences<br/>Notification Settings"]
    end
    
    subgraph "Order Data Domain"
        ORDER_HDR["Order Header<br/>Customer, Plan, Status"]
        ORDER_DETAILS["Order Details<br/>Platform Base, Options"]
        VEHICLE_REG["Vehicle Registration<br/>VIN, Plate, Status"]
        MANUFACTURING["Manufacturing Status<br/>Factory Updates"]
    end
    
    subgraph "Service Data Domain"
        SERVICE_CATALOG["Service Definitions<br/>Options, Pricing, Rules"]
        SERVICE_ACTIVATION["Activation Records<br/>Service, Date, Status"]
        SUBSCRIPTION["Subscription Records<br/>Recurring Services"]
    end
    
    subgraph "Billing Data Domain"
        PAYMENT_TRANS["Payment Transactions<br/>Type, Amount, Status"]
        INVOICE["Invoices<br/>Customer, Items, Total"]
        REFUND["Refund Records<br/>Desistimiento, Date"]
    end
    
    subgraph "Configuration Data Domain"
        OTA_CONFIG["OTA Configurations<br/>Service to Vehicle Mapping"]
        RETRY_POLICY["Retry Policies<br/>Attempts, Intervals"]
        MAINT_RULES["Maintenance Rules<br/>Service Dependencies"]
    end
```

**Data Flow Patterns:**

| Flow | Pattern | Example |
|------|---------|---------|
| **Customer → Order** | Reference by ID | Customer ID stored in Order Header |
| **Order → Service** | Event-driven | Order completion triggers service catalog availability |
| **Payment → Activation** | Transactional | Payment success atomically queues OTA activation |
| **Service → Configuration** | Lookup | Service activation reads OTA configuration for vehicle commands |

**Sources:** [enunciado.md:1-23]()

---

## Integration Communication Patterns

### Factory Integration (Hybrid Pattern)

```mermaid
sequenceDiagram
    participant CAAS as "CaaS Order Management"
    participant FACTORY_API as "Factory API"
    participant FACTORY_SYS as "Factory Systems"
    participant WEBHOOK as "CaaS Webhook Receiver"
    participant NOTIF as "Notification Engine"
    
    Note over CAAS,FACTORY_API: Synchronous Order Placement
    CAAS->>FACTORY_API: POST /orders<br/>{customer, platform, options}
    FACTORY_API->>FACTORY_SYS: Create Manufacturing Order
    FACTORY_API-->>CAAS: 200 OK<br/>{orderId, estimatedDate}
    
    Note over FACTORY_SYS,WEBHOOK: Asynchronous Status Updates
    FACTORY_SYS->>FACTORY_API: Manufacturing Started
    FACTORY_API->>WEBHOOK: POST /webhooks/factory<br/>{orderId, status: "IN_PROGRESS"}
    WEBHOOK->>NOTIF: Send Customer Notification
    
    FACTORY_SYS->>FACTORY_API: Manufacturing Complete
    FACTORY_API->>WEBHOOK: POST /webhooks/factory<br/>{orderId, status: "READY"}
    WEBHOOK->>NOTIF: Send Customer Notification
```

**Rationale**: Synchronous order placement ensures immediate confirmation and error handling. Asynchronous status updates prevent blocking and enable long-running manufacturing processes without maintaining connections.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:40-44]()

---

### VSS Integration (Query Pattern)

```mermaid
sequenceDiagram
    participant CUSTOMER as "Customer"
    participant CATALOG as "Service Catalog"
    participant VSS_GW as "VSS Gateway"
    participant VSS as "VSS System"
    
    CUSTOMER->>CATALOG: Request Service Purchase
    
    Note over CATALOG,VSS: Maintenance Check for Dependent Services
    CATALOG->>VSS_GW: Query Maintenance Status<br/>{vin, serviceRequirements}
    VSS_GW->>VSS: GET /vehicles/{vin}/maintenance
    VSS-->>VSS_GW: {status, lastService, completionPercentage}
    VSS_GW-->>CATALOG: Maintenance Status
    
    alt Maintenance Current
        CATALOG-->>CUSTOMER: Service Available
    else Maintenance Overdue
        CATALOG-->>CUSTOMER: Service Blocked<br/>Complete Maintenance First
    end
```

**Key Architectural Constraint**: Vehicles do NOT report maintenance status directly. The VSS system is the authoritative source, maintained independently by workshops. CaaS queries VSS when needed for service gating decisions.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:60-73]()

---

### IoT Integration (Command Pattern with Retry)

```mermaid
sequenceDiagram
    participant PAYMENT as "Payment Processing"
    participant OTA as "OTA Activation Engine"
    participant IOT_API as "API IoT<br/>(Existing, Documented)"
    participant IOT_NET as "IoT Network"
    participant VEHICLE as "Vehicle"
    participant NOTIF as "Notification Engine"
    participant TECH as "Technical Support"
    
    PAYMENT->>OTA: Payment Successful<br/>{service, vehicle, customer}
    
    loop Retry Logic (N Attempts)
        OTA->>IOT_API: Send OTA Command<br/>{vin, configurationPayload}
        IOT_API->>IOT_NET: Push Configuration
        IOT_NET->>VEHICLE: Apply Configuration
        
        alt Activation Successful
            VEHICLE-->>IOT_NET: Status: Success
            IOT_NET-->>IOT_API: Telemetry: Activated
            IOT_API-->>OTA: Success Response
            OTA->>NOTIF: Notify Customer: Service Active
        else Activation Failed
            VEHICLE-->>IOT_NET: Status: Failed
            IOT_NET-->>IOT_API: Telemetry: Error
            IOT_API-->>OTA: Failure Response
            
            alt Retries Remaining
                Note over OTA: Wait and Retry
            else No Retries Remaining
                OTA->>IOT_API: Query Vehicle Status<br/>Proactively
                IOT_API-->>OTA: Vehicle Status Details
                OTA->>TECH: Escalate<br/>{service, vehicle, errorDetails}
                OTA->>NOTIF: Notify Customer:<br/>Technical Issue
                Note over OTA: DO NOT CHARGE CUSTOMER
            end
        end
    end
```

**Critical Architectural Constraint**: The API IoT is pre-existing, documented, and tested. It cannot be modified. All integration must conform to its existing interface.

**Critical Business Rule**: If OTA activation fails after all retries, the customer is NOT charged for the service. This is a customer protection mechanism and legal requirement.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:31-56]()

---

### Payment Integration (Async with Risk Assumption)

```mermaid
sequenceDiagram
    participant CUSTOMER as "Customer"
    participant SERVICE_CAT as "Service Catalog"
    participant PAYMENT as "Payment Processing"
    participant PGW as "Payment Gateway"
    participant OTA as "OTA Activation"
    participant BANK as "Bank Settlement<br/>(Async)"
    
    CUSTOMER->>SERVICE_CAT: Purchase Service
    SERVICE_CAT->>PAYMENT: Initiate Payment<br/>{amount, customer, service}
    PAYMENT->>PGW: Process Payment<br/>{cardDetails, amount}
    PGW-->>PAYMENT: Payment Initiated
    
    Note over PAYMENT,OTA: Immediate Delivery Despite Async Settlement
    PAYMENT->>OTA: Activate Service<br/>(Assumes Risk)
    OTA->>CUSTOMER: Service Activated
    
    Note over PGW,BANK: Asynchronous Settlement
    PGW->>BANK: Settlement Request
    
    alt Settlement Successful
        BANK-->>PGW: Confirmation
        PGW->>PAYMENT: Webhook: Payment Confirmed
    else Settlement Failed
        BANK-->>PGW: Failure
        PGW->>PAYMENT: Webhook: Payment Failed
        Note over PAYMENT: Handle Chargeback Risk<br/>Already Delivered Service
    end
```

**Architectural Trade-off**: CaaS prioritizes customer experience by activating services immediately upon payment initiation, before bank settlement completes. This introduces financial risk (chargebacks, failed settlements) but significantly improves perceived service quality.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:76-82]()

---

## State Management Architecture

### Vehicle and Service State Model

```mermaid
stateDiagram-v2
    [*] --> OrderCreated: Dealership Registration
    
    OrderCreated --> FactoryOrdered: Reservation Payment
    FactoryOrdered --> Manufacturing: Factory Accepts
    Manufacturing --> VehicleReady: Manufacturing Complete
    VehicleReady --> PendingFinalPayment: Customer Notified
    
    PendingFinalPayment --> Registered: Final Payment Success
    PendingFinalPayment --> StockForSale: Final Payment Failed
    
    StockForSale --> [*]: Vehicle Reassigned
    
    Registered --> InTransit: Admin Registration Complete
    InTransit --> Delivered: Customer Receives
    InTransit --> AtDealership: Delivery Failed<br/>Customer Unavailable
    
    AtDealership --> Delivered: Customer Pickup
    
    Delivered --> Operational: Mobile App Linked
    
    state Operational {
        [*] --> BaseActive: Base Platform Only
        
        BaseActive --> ServiceRequested: Customer Requests Service
        ServiceRequested --> MaintenanceCheck: System Validation
        
        MaintenanceCheck --> PaymentRequired: Maintenance OK
        MaintenanceCheck --> BaseActive: Maintenance Overdue<br/>Service Blocked
        
        PaymentRequired --> OTAQueue: Payment Success
        PaymentRequired --> BaseActive: Payment Failed
        
        OTAQueue --> ServiceActive: OTA Success
        OTAQueue --> TechSupport: OTA Failed<br/>After Retries
        
        TechSupport --> BaseActive: No Charge
        
        ServiceActive --> BaseActive: Service Used
        
        ServiceActive --> Refunded: Desistimiento<br/>Within 14 Days
        Refunded --> BaseActive
    }
```

**State Transitions and Business Rules:**

| Transition | Trigger | Business Rule |
|------------|---------|---------------|
| `OrderCreated → FactoryOrdered` | Reservation payment received | Automatic API call to factory |
| `PendingFinalPayment → StockForSale` | Final payment fails | Vehicle marked "sin asignar", becomes immediately available for sale |
| `InTransit → AtDealership` | Customer not available | Vehicle returned to dealership, never left unattended |
| `MaintenanceCheck → BaseActive` | Maintenance overdue | Service blocked but base platform remains operational |
| `OTAQueue → TechSupport` | All retry attempts exhausted | Customer NOT charged, technical escalation |
| `ServiceActive → Refunded` | Customer cancels within 14 days | Desistimiento (legal right of withdrawal) honored |

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:18-96](), [enunciado.md:1-23]()

---

## Deployment Architecture

### Multi-Tenant Considerations

The system must support multiple countries with varying administrative requirements:

```mermaid
graph TB
    subgraph "CaaS Core Services<br/>(Multi-Tenant)"
        CORE_APP["Application Services<br/>Country-Agnostic"]
        CORE_DB["Central Databases<br/>Customer, Order, Service"]
    end
    
    subgraph "Country-Specific Adapters"
        ADMIN_ES["Admin Gateway ES<br/>Spanish Bodies"]
        ADMIN_FR["Admin Gateway FR<br/>French Bodies"]
        ADMIN_DE["Admin Gateway DE<br/>German Bodies"]
    end
    
    subgraph "Shared Integration Services"
        FACTORY_INT["Factory Integration<br/>Single Factory System"]
        VSS_INT["VSS Integration<br/>Regional VSS Instances"]
        IOT_INT["IoT Gateway<br/>Global Vehicle Fleet"]
        PAYMENT_INT["Payment Gateway<br/>Multi-Currency Support"]
    end
    
    CORE_APP --> ADMIN_ES
    CORE_APP --> ADMIN_FR
    CORE_APP --> ADMIN_DE
    
    CORE_APP --> FACTORY_INT
    CORE_APP --> VSS_INT
    CORE_APP --> IOT_INT
    CORE_APP --> PAYMENT_INT
    
    CORE_APP --> CORE_DB
```

**Design Considerations:**
- **Core services** remain country-agnostic
- **Administrative registration** uses country-specific adapters implementing common interface
- **Payment processing** supports multiple currencies and payment methods per country
- **VSS integration** may connect to regional instances but presents unified interface

**Sources:** [enunciado.md:15-16]()

---

## Notification Architecture

### Multi-Channel Notification Delivery

```mermaid
graph TB
    subgraph "Notification Triggers"
        T1["Order Status Change<br/>(Factory Webhooks)"]
        T2["OTA Activation Result<br/>(Success/Failure)"]
        T3["Payment Events<br/>(Confirmation/Failure)"]
        T4["Credential Generation<br/>(New Customer)"]
        T5["Maintenance Reminder<br/>(Scheduled)"]
    end
    
    subgraph "Notification Engine"
        ROUTER["Notification Router<br/>Event to Channel Mapping"]
        TEMPLATE["Template Renderer<br/>Localization Support"]
        QUEUE["Message Queue<br/>Async Processing"]
    end
    
    subgraph "Delivery Channels"
        EMAIL["Email Service<br/>SMTP/SendGrid"]
        PUSH["Push Notification<br/>Mobile App"]
        SMS["SMS Gateway<br/>Urgent Messages"]
        VEHICLE["Vehicle Display<br/>Via IoT API"]
    end
    
    T1 --> ROUTER
    T2 --> ROUTER
    T3 --> ROUTER
    T4 --> ROUTER
    T5 --> ROUTER
    
    ROUTER --> TEMPLATE
    TEMPLATE --> QUEUE
    
    QUEUE --> EMAIL
    QUEUE --> PUSH
    QUEUE --> SMS
    QUEUE --> VEHICLE
```

**Notification Routing Rules:**

| Event | Email | Push | SMS | Vehicle Display |
|-------|-------|------|-----|-----------------|
| **Order Status Update** | ✓ | ✓ | - | - |
| **Vehicle Ready** | ✓ | ✓ | ✓ | - |
| **OTA Success** | ✓ | ✓ | - | ✓ |
| **OTA Failed** | ✓ | ✓ | - | - |
| **Credentials Generated** | ✓ | - | - | - |
| **Payment Confirmation** | ✓ | ✓ | - | - |
| **Maintenance Due** | ✓ | ✓ | ✓ | ✓ |

**Sources:** [enunciado.md:11-20](), [pasame las preguntas y sus respuestas a markdown.md:28-29]()

---

## Security Architecture

### Authentication and Authorization Model

```mermaid
graph TB
    subgraph "User Types"
        CUST["Customer<br/>Vehicle Owner"]
        EMP["Employee<br/>Dealership Staff"]
        ADMIN["Administrator<br/>System Operators"]
    end
    
    subgraph "Authentication"
        AUTH["Authentication Service"]
        CRED_STORE["Credential Store<br/>Password Hashes"]
        SESSION["Session Management<br/>Token Generation"]
    end
    
    subgraph "Authorization"
        AUTHZ["Authorization Service"]
        ROLE_DB["Role Definitions"]
        POLICY["Access Control Policies"]
    end
    
    subgraph "Protected Resources"
        EXPEDIENTE["Customer Expediente"]
        INTRANET["Sales Intranet"]
        ADMIN_PANEL["Admin Functions"]
    end
    
    CUST --> AUTH
    EMP --> AUTH
    ADMIN --> AUTH
    
    AUTH --> CRED_STORE
    AUTH --> SESSION
    
    SESSION --> AUTHZ
    AUTHZ --> ROLE_DB
    AUTHZ --> POLICY
    
    AUTHZ --> EXPEDIENTE
    AUTHZ --> INTRANET
    AUTHZ --> ADMIN_PANEL
```

**Access Control Rules:**

| User Type | Access Scope | Key Restrictions |
|-----------|--------------|------------------|
| **Customer** | Own expediente, own vehicle services | Cannot view other customers' data |
| **Employee (Dealership)** | Sales registration, order management | Cannot modify existing customer credentials |
| **Administrator** | System configuration, all records | Cannot initiate OTA commands directly (automated only) |

**Authentication Flow:**
1. Customers receive credentials via email after dealership registration
2. No self-registration or external SSO (Google, Microsoft, etc.)
3. All users must be explicitly provisioned in the system
4. Mobile app links to specific vehicle after delivery (one-to-one binding)

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:15-16]()

---

## Failure Handling and Resilience

### Circuit Breaker Patterns

The system implements circuit breaker patterns for critical external dependencies:

```mermaid
graph LR
    subgraph "OTA Activation Circuit Breaker"
        OTA_CB["Circuit Breaker"]
        OTA_RETRY["Retry Logic<br/>Exponential Backoff"]
        OTA_FALLBACK["Fallback:<br/>Technical Support"]
    end
    
    subgraph "Factory API Circuit Breaker"
        FACTORY_CB["Circuit Breaker"]
        FACTORY_FALLBACK["Fallback:<br/>Queue Order"]
    end
    
    subgraph "VSS Query Circuit Breaker"
        VSS_CB["Circuit Breaker"]
        VSS_FALLBACK["Fallback:<br/>Allow Service<br/>(Assume Current)"]
    end
    
    subgraph "Payment Circuit Breaker"
        PAY_CB["Circuit Breaker"]
        PAY_FALLBACK["Fallback:<br/>Reject Transaction"]
    end
```

**Failure Handling Strategies by Subsystem:**

| Subsystem | Failure Mode | Strategy | Customer Impact |
|-----------|--------------|----------|-----------------|
| **OTA Activation** | IoT API unavailable | Retry with exponential backoff (N attempts) → Escalate to tech support | Service NOT activated, NOT charged |
| **Factory API** | Factory system down | Queue order for later submission | Delayed order placement, customer notified |
| **VSS Query** | VSS unavailable | Allow service purchase (assume maintenance current) | Service activated optimistically |
| **Payment Gateway** | Payment system down | Reject transaction immediately | Customer must retry, no service delivered |
| **Admin Registration** | Admin API fails | Retry synchronously (blocking), escalate after timeout | Delivery delayed, customer notified |

**Critical Principle**: When in doubt, favor the customer. If OTA fails or payment confirmation is delayed, do NOT charge the customer for undelivered services.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:46-53]()

---

## Subscription and Billing Architecture

### Mes Vencido (Post-Paid) Model

```mermaid
sequenceDiagram
    participant CUSTOMER as "Customer"
    participant CATALOG as "Service Catalog"
    participant SUBSCRIPTION as "Subscription Manager"
    participant VEHICLE as "Vehicle<br/>(Usage Tracking)"
    participant BILLING as "Billing Engine"
    participant PAYMENT as "Payment Processing"
    
    CUSTOMER->>CATALOG: Subscribe to Recurring Service
    CATALOG->>SUBSCRIPTION: Create Subscription<br/>{service, startDate, frequency}
    SUBSCRIPTION->>VEHICLE: Activate Service
    
    Note over VEHICLE,BILLING: Month of Usage
    loop Daily
        VEHICLE->>SUBSCRIPTION: Usage Telemetry
    end
    
    Note over SUBSCRIPTION,BILLING: End of Month
    SUBSCRIPTION->>BILLING: Generate Invoice<br/>{customer, services, usage}
    BILLING->>PAYMENT: Charge Mes Vencido<br/>(Post-Paid)
    
    alt Payment Success
        PAYMENT-->>BILLING: Confirmation
        BILLING->>CUSTOMER: Invoice + Receipt
        SUBSCRIPTION->>VEHICLE: Continue Service
    else Payment Failed
        PAYMENT-->>BILLING: Failure
        BILLING->>SUBSCRIPTION: Cancel Subscription
        SUBSCRIPTION->>VEHICLE: Deactivate Service
        BILLING->>CUSTOMER: Service Cancelled<br/>Payment Issue
    end
```

**Billing Cycle Rules:**
- **One-time services**: Charge immediately upon purchase
- **Subscription services**: Charge at month end (mes vencido) for consumed services
- **Failed subscription payment**: Automatic cancellation, no further charges
- **Refunds**: Subject to desistimiento rules (14-day window for services > 14 days duration)

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:76-96]()

---

## Technology Stack Implications

Based on the architectural patterns and requirements, the system likely employs:

### Backend Services
- **API Framework**: RESTful API design for synchronous operations (Factory order placement, VSS queries)
- **Webhook Receivers**: Async endpoint handlers for Factory status updates and payment confirmations
- **Message Queue**: Required for async OTA activation processing and notification distribution
- **Scheduled Jobs**: Cron-based or triggered jobs for subscription billing (end-of-month), maintenance reminders

### Data Storage
- **Relational Database**: Transactional consistency for orders, payments, customer data
- **Document Store** (possible): Configuration repository for OTA service definitions
- **Cache Layer**: Performance optimization for service catalog queries, maintenance status lookups

### Integration Infrastructure
- **HTTP Client Libraries**: For calling Factory API, VSS API, IoT API, Payment Gateway, Admin API
- **Retry Mechanisms**: Built-in or library-based retry logic with exponential backoff for OTA activations
- **Circuit Breakers**: Fault tolerance for external dependencies

### Security Components
- **Password Hashing**: Secure credential storage (bcrypt, Argon2)
- **Token Management**: Session or JWT tokens for stateless authentication
- **TLS/HTTPS**: Encrypted communication for all external APIs and customer-facing platforms

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:1-104](), [enunciado.md:1-23]()

---

## Summary

The CaaS system architecture is built on the following key principles:

1. **Centralized Orchestration**: All business logic flows through the central CaaS platform
2. **Async-First Integration**: External systems communicate primarily via asynchronous patterns with sync for critical transactions
3. **Customer Protection**: Failure handling defaults to customer-favorable outcomes (no charge for failed OTA)
4. **Legal Compliance**: Desistimiento (withdrawal rights) and maintenance warranty rules embedded in architecture
5. **Risk-Tolerant Delivery**: Services activated before payment settlement completes
6. **Multi-Tenant Design**: Country-specific adapters for administrative requirements
7. **State-Driven Workflows**: Explicit state machines for vehicles, orders, services, and payments

For detailed component descriptions, see [Core Technical Components](#3.2). For integration patterns and external system details, see [Integration Architecture](#3.3) and [External System Integrations](#5).

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:1-104](), [enunciado.md:1-23]()