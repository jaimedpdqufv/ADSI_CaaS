# Customer-Facing Platforms

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [enunciado.md](enunciado.md)
- [pasame las preguntas y sus respuestas a markdown.md](pasame las preguntas y sus respuestas a markdown.md)

</details>



## Purpose and Scope

This page documents the customer-facing digital platforms that enable customers to interact with the CaaS system. It covers the web platform, mobile application, and the expediente de compra (purchase record) system that forms the core of the customer experience.

For information about the business processes customers perform through these platforms (purchasing vehicles, acquiring services), see [Vehicle Purchase and Delivery Process](#4.1) and [Service Acquisition and Management](#4.2). For technical details on backend integrations, see [Integration Architecture](#3.3).

**Sources:** [enunciado.md:1-23](), [pasame las preguntas y sus respuestas a markdown.md:1-104]()

---

## Platform Overview

The CaaS system provides two primary customer-facing platforms:

| Platform | Access Method | Primary Use Cases |
|----------|---------------|-------------------|
| **Web Platform** | Browser-based via corporate website | Initial registration review, document access, service browsing from desktop |
| **Mobile Application** | Native smartphone app | Service activation, vehicle status monitoring, direct vehicle control |

Both platforms provide access to the **expediente de compra** (purchase record), which serves as the single source of truth for all customer-related data, documents, and transaction history.

```mermaid
graph TB
    subgraph "Customer Access Channels"
        CUSTOMER["Customer"]
        WEB["Web Platform<br/>(Browser)"]
        MOBILE["Mobile Application<br/>(Native App)"]
    end
    
    subgraph "Core Customer Data"
        EXPEDIENTE["Expediente de Compra<br/>Purchase Record<br/>- Manuals<br/>- Maintenance History<br/>- Invoices<br/>- Service Catalog Access"]
    end
    
    subgraph "Backend Services"
        CAAS_CORE["CaaS Core Services"]
        SERVICE_CAT["Service Catalog"]
        PAYMENT["Payment Processing"]
        NOTIF["Notification Engine"]
    end
    
    subgraph "Vehicle Infrastructure"
        VEHICLE["Customer's Vehicle"]
        IOT["IoT Network"]
    end
    
    CUSTOMER -->|"Accesses via browser"| WEB
    CUSTOMER -->|"Installs and uses"| MOBILE
    
    WEB --> EXPEDIENTE
    MOBILE --> EXPEDIENTE
    
    EXPEDIENTE --> CAAS_CORE
    CAAS_CORE --> SERVICE_CAT
    CAAS_CORE --> PAYMENT
    
    MOBILE -.->|"Direct connection for<br/>operational control"| VEHICLE
    
    CAAS_CORE --> IOT
    IOT --> VEHICLE
    
    NOTIF -->|"Email"| CUSTOMER
    NOTIF -->|"Push notifications"| MOBILE
    NOTIF -->|"Vehicle display"| VEHICLE
```

**Diagram: Customer-Facing Platform Architecture**

**Key Architectural Characteristics:**

- **Unified Data Model**: Both platforms access the same expediente de compra
- **Dual Mobile Paths**: Mobile app communicates with CaaS backend for services AND directly with vehicle for operational control
- **Multi-Channel Notifications**: Customers receive updates via email, mobile push, and vehicle display

**Sources:** [enunciado.md:10-12](), [enunciado.md:17-20]()

---

## Expediente de Compra (Purchase Record)

The expediente de compra is the central data structure containing all customer-related information. It is accessible through both the web platform and mobile application.

### Contents

The expediente de compra includes:

| Component | Description | Update Frequency |
|-----------|-------------|------------------|
| **Vehicle Manuals** | Digital copies of vehicle documentation | Static (updated on vehicle delivery) |
| **Maintenance History** | Complete service record from VSS system | Updated after each workshop visit |
| **Invoice Archive** | All purchase and service invoices | Updated on each transaction |
| **Service History** | Record of activated optional services | Real-time (updated on activation/cancellation) |
| **Manufacturing Status** | Current state during vehicle production | Real-time during manufacturing phase |
| **Plan Comercial** | Contracted commercial plan details | Static (defined at purchase) |

### Data Flow

```mermaid
graph LR
    subgraph "Data Sources"
        INTRANET["Corporate Intranet<br/>(Sales Registration)"]
        FACTORY_API["Factory API<br/>(Manufacturing Status)"]
        VSS["VSS System<br/>(Maintenance Records)"]
        PAYMENT_SYS["Payment System<br/>(Invoices)"]
        OTA["OTA Engine<br/>(Service Activations)"]
    end
    
    subgraph "Expediente de Compra"
        CUSTOMER_PROFILE["Customer Profile<br/>& Credentials"]
        VEHICLE_DATA["Vehicle Data<br/>& Manuals"]
        COMMERCIAL_PLAN["Plan Comercial<br/>(Base + Options)"]
        TRANSACTION_HIST["Transaction History<br/>& Invoices"]
        SERVICE_HIST["Service History<br/>& Activations"]
        MAINT_HIST["Maintenance History<br/>& Status"]
    end
    
    subgraph "Customer Platforms"
        WEB_INTERFACE["Web Platform"]
        MOBILE_INTERFACE["Mobile App"]
    end
    
    INTRANET -->|"Initial registration"| CUSTOMER_PROFILE
    INTRANET -->|"Commercial plan"| COMMERCIAL_PLAN
    
    FACTORY_API -->|"Status updates"| VEHICLE_DATA
    VSS -->|"Query for history"| MAINT_HIST
    PAYMENT_SYS --> TRANSACTION_HIST
    OTA --> SERVICE_HIST
    
    CUSTOMER_PROFILE --> WEB_INTERFACE
    VEHICLE_DATA --> WEB_INTERFACE
    COMMERCIAL_PLAN --> WEB_INTERFACE
    TRANSACTION_HIST --> WEB_INTERFACE
    SERVICE_HIST --> WEB_INTERFACE
    MAINT_HIST --> WEB_INTERFACE
    
    CUSTOMER_PROFILE --> MOBILE_INTERFACE
    VEHICLE_DATA --> MOBILE_INTERFACE
    COMMERCIAL_PLAN --> MOBILE_INTERFACE
    TRANSACTION_HIST --> MOBILE_INTERFACE
    SERVICE_HIST --> MOBILE_INTERFACE
    MAINT_HIST --> MOBILE_INTERFACE
```

**Diagram: Expediente de Compra Data Flow**

**Sources:** [enunciado.md:11-12](), [pasame las preguntas y sus respuestas a markdown.md:28-29]()

---

## Web Platform

The web platform provides browser-based access to the CaaS system. Customers access it via the corporate website using credentials provided by email after vehicle purchase registration.

### Features and Capabilities

```mermaid
graph TB
    subgraph "Web Platform Features"
        LOGIN["Login Page<br/>(Email-based credentials)"]
        DASHBOARD["Customer Dashboard"]
        
        subgraph "Expediente Access"
            DOCS["Document Library<br/>(Manuals)"]
            INVOICES["Invoice History"]
            MAINT_VIEW["Maintenance Records<br/>(from VSS)"]
            STATUS["Manufacturing Status<br/>(during production)"]
        end
        
        subgraph "Service Management"
            CATALOG["Service Catalog<br/>(Opciones Disponibles)"]
            SERVICE_DETAIL["Service Detail View<br/>(Pricing & Duration)"]
            PURCHASE_FLOW["Service Purchase Flow"]
            PAYMENT_WEB["Payment Interface"]
        end
        
        subgraph "Account Management"
            PROFILE["Profile Settings"]
            PAYMENT_METHODS["Payment Methods"]
            NOTIF_PREFS["Notification Preferences"]
        end
    end
    
    LOGIN --> DASHBOARD
    DASHBOARD --> DOCS
    DASHBOARD --> INVOICES
    DASHBOARD --> MAINT_VIEW
    DASHBOARD --> STATUS
    
    DASHBOARD --> CATALOG
    CATALOG --> SERVICE_DETAIL
    SERVICE_DETAIL --> PURCHASE_FLOW
    PURCHASE_FLOW --> PAYMENT_WEB
    
    DASHBOARD --> PROFILE
    DASHBOARD --> PAYMENT_METHODS
    DASHBOARD --> NOTIF_PREFS
```

**Diagram: Web Platform Feature Map**

### User Access Flow

1. **Credential Delivery**: After sales registration at dealership, customer receives email with access credentials [enunciado.md:11]()
2. **Authentication**: Customer logs in using email address and provided password
3. **Dashboard Access**: Upon login, customer sees personalized dashboard with expediente data
4. **Service Browsing**: Customer can browse opciones disponibles (optional services)
5. **Transaction Execution**: Purchase flow includes service selection, payment, and confirmation

### Technical Characteristics

- **No Self-Registration**: System does not allow public registration. All users must be pre-registered through dealership sales process [pasame las preguntas y sus respuestas a markdown.md:15-16]()
- **No External SSO**: No integration with Google, Microsoft, or other external identity providers [pasame las preguntas y sus respuestas a markdown.md:15-16]()
- **Controlled User Base**: All customers are known entities who have purchased vehicles

**Sources:** [enunciado.md:10-12](), [enunciado.md:18-19](), [pasame las preguntas y sus respuestas a markdown.md:15-16]()

---

## Mobile Application

The mobile application provides smartphone-based access to CaaS services with enhanced capabilities for vehicle interaction and real-time notifications.

### Dual Communication Paths

The mobile application has two distinct communication paths:

```mermaid
graph TB
    subgraph "Mobile Application"
        APP["Mobile App"]
        
        subgraph "App Features"
            EXPEDIENTE_VIEW["Expediente View"]
            SERVICE_ACTIVATION["Service Activation"]
            VEHICLE_STATUS["Vehicle Status Monitor"]
            DIRECT_CONTROL["Direct Vehicle Control"]
        end
    end
    
    subgraph "Communication Paths"
        PATH_1["Path 1: Backend Services<br/>(via CaaS API)"]
        PATH_2["Path 2: Direct Vehicle<br/>(via Bluetooth/WiFi)"]
    end
    
    subgraph "Backend"
        CAAS_API["CaaS Core API"]
        OTA_ENGINE["OTA Engine"]
        IOT_GATEWAY["IoT Gateway"]
    end
    
    subgraph "Vehicle"
        VEHICLE_SYSTEM["Vehicle Systems"]
        VEHICLE_DISPLAY["Vehicle Display"]
    end
    
    EXPEDIENTE_VIEW --> PATH_1
    SERVICE_ACTIVATION --> PATH_1
    PATH_1 --> CAAS_API
    CAAS_API --> OTA_ENGINE
    OTA_ENGINE --> IOT_GATEWAY
    
    DIRECT_CONTROL --> PATH_2
    VEHICLE_STATUS --> PATH_2
    PATH_2 -.->|"Direct link"| VEHICLE_SYSTEM
    
    IOT_GATEWAY --> VEHICLE_SYSTEM
    VEHICLE_SYSTEM --> VEHICLE_DISPLAY
```

**Diagram: Mobile App Communication Architecture**

**Path 1 - Backend Services:**
- Expediente de compra access
- Service catalog browsing
- Payment processing
- Service activation requests
- Notification reception

**Path 2 - Direct Vehicle Connection:**
- Real-time vehicle status monitoring
- Operational control (lock/unlock, climate control, etc.)
- Local configuration without internet dependency

**Sources:** [enunciado.md:17](), High-Level Architecture Diagram 1

### Vehicle Linking Process

The mobile application is linked to the customer's vehicle upon delivery:

1. **Vehicle Delivery**: Vehicle is transported to customer address [enunciado.md:17]()
2. **App Binding**: Mobile application is directly linked to the acquired vehicle [enunciado.md:17]()
3. **Operational Enablement**: Vehicle functionality is enabled through the app [enunciado.md:17]()

### Service Activation Display

When customers purchase optional services, the mobile app provides real-time activation status:

```mermaid
stateDiagram-v2
    [*] --> ServicePurchased: "Customer pays for service"
    ServicePurchased --> OTAInProgress: "OTA activation initiated"
    OTAInProgress --> ActivationSuccess: "OTA successful"
    OTAInProgress --> ActivationRetry: "OTA fails - retry"
    ActivationRetry --> OTAInProgress: "Retry attempt"
    ActivationRetry --> ActivationFailed: "All retries exhausted"
    
    ActivationSuccess --> [*]: "App shows: Service Active<br/>Vehicle display confirms"
    ActivationFailed --> [*]: "App shows: Activation Failed<br/>Customer NOT charged"
    
    note right of ActivationSuccess
        Customer receives notification:
        - Mobile app notification
        - Vehicle display message
    end note
    
    note right of ActivationFailed
        Customer receives notification:
        - Mobile app alert
        - Technical support contact
        - Refund processed automatically
    end note
```

**Diagram: Service Activation Status Flow in Mobile App**

The mobile application displays activation status received from both:
- **CaaS backend**: Overall service status and OTA progress
- **Vehicle information system**: Confirmation of feature activation on vehicle [enunciado.md:19-20]()

**Sources:** [enunciado.md:19-20](), [pasame las preguntas y sus respuestas a markdown.md:48-53]()

---

## Platform Access and Security

### User Provisioning

```mermaid
sequenceDiagram
    participant Dealership as "Dealership Staff<br/>(Corporate Intranet)"
    participant CaaS as "CaaS System"
    participant EmailSvc as "Email Service"
    participant Customer as "Customer"
    participant WebApp as "Web Platform"
    participant MobileApp as "Mobile App"
    
    Dealership->>CaaS: "Register sale<br/>(customer data + plan comercial)"
    CaaS->>CaaS: "Generate credentials<br/>(email + password)"
    CaaS->>EmailSvc: "Send credentials"
    EmailSvc->>Customer: "Email with access credentials"
    
    Customer->>WebApp: "Login with credentials"
    WebApp->>CaaS: "Authenticate"
    CaaS-->>WebApp: "Access granted"
    
    Customer->>MobileApp: "Login with same credentials"
    MobileApp->>CaaS: "Authenticate"
    CaaS-->>MobileApp: "Access granted"
```

**Diagram: User Provisioning and Authentication Flow**

### Security Characteristics

| Security Feature | Implementation |
|------------------|----------------|
| **User Origin** | All users created through dealership sales registration |
| **Authentication** | Email + password (no external SSO) |
| **Authorization** | Role-based: customers see only their own expediente |
| **Self-Registration** | **Disabled** - no public registration allowed |
| **External Identity Providers** | **Not supported** - no Google/Microsoft SSO |
| **User Verification** | Users are known entities who have purchased vehicles |

### Rationale for Closed System

The CaaS platform is intentionally **not a public system**. All users must be:
- Known to the organization (customers, employees)
- Created through controlled processes (sales registration)
- Associated with specific vehicles or roles

This design ensures:
- Customer data privacy
- Vehicle security
- Controlled access to premium services
- Clear audit trails for transactions

**Sources:** [enunciado.md:8-11](), [pasame las preguntas y sus respuestas a markdown.md:15-16]()

---

## Technical Architecture

### Platform Integration with Backend Services

```mermaid
graph TB
    subgraph "Customer Platforms"
        WEB_CLIENT["Web Platform<br/>(Browser Client)"]
        MOBILE_CLIENT["Mobile App<br/>(Native Client)"]
    end
    
    subgraph "API Layer"
        API_GATEWAY["API Gateway"]
        AUTH_SERVICE["Authentication Service"]
    end
    
    subgraph "Business Services"
        EXPEDIENTE_SVC["Expediente Service"]
        ORDER_SVC["Order Management Service"]
        SERVICE_CATALOG_SVC["Service Catalog Service"]
        PAYMENT_SVC["Payment Processing Service"]
        NOTIFICATION_SVC["Notification Service"]
    end
    
    subgraph "Integration Layer"
        FACTORY_INT["Factory Integration"]
        VSS_INT["VSS Integration"]
        IOT_INT["IoT Gateway"]
        PAYMENT_GW["Payment Gateway"]
    end
    
    subgraph "Data Stores"
        CUSTOMER_DB[("Customer Database")]
        ORDER_DB[("Order Database")]
        SERVICE_DB[("Service Database")]
    end
    
    WEB_CLIENT -->|"HTTPS/REST"| API_GATEWAY
    MOBILE_CLIENT -->|"HTTPS/REST"| API_GATEWAY
    
    API_GATEWAY --> AUTH_SERVICE
    AUTH_SERVICE --> CUSTOMER_DB
    
    API_GATEWAY --> EXPEDIENTE_SVC
    API_GATEWAY --> ORDER_SVC
    API_GATEWAY --> SERVICE_CATALOG_SVC
    API_GATEWAY --> PAYMENT_SVC
    
    EXPEDIENTE_SVC --> CUSTOMER_DB
    ORDER_SVC --> ORDER_DB
    SERVICE_CATALOG_SVC --> SERVICE_DB
    PAYMENT_SVC --> PAYMENT_GW
    
    ORDER_SVC --> FACTORY_INT
    EXPEDIENTE_SVC --> VSS_INT
    SERVICE_CATALOG_SVC --> IOT_INT
    
    NOTIFICATION_SVC --> WEB_CLIENT
    NOTIFICATION_SVC --> MOBILE_CLIENT
    
    FACTORY_INT -.->|"Async status"| NOTIFICATION_SVC
    IOT_INT -.->|"OTA status"| NOTIFICATION_SVC
    PAYMENT_GW -.->|"Payment confirmation"| NOTIFICATION_SVC
```

**Diagram: Technical Integration Architecture**

### API Communication Patterns

The customer-facing platforms interact with backend services using the following patterns:

| Operation | Platform | Pattern | Example |
|-----------|----------|---------|---------|
| **View Expediente** | Web/Mobile | Synchronous GET | Retrieve customer data, documents, history |
| **Browse Services** | Web/Mobile | Synchronous GET | Query service catalog with filters |
| **Purchase Service** | Web/Mobile | Synchronous POST | Submit payment and service request |
| **Receive Notifications** | Mobile | Push (Async) | Manufacturing status updates |
| **Query Maintenance** | Web/Mobile | Synchronous GET | Check VSS maintenance status |
| **Monitor OTA Status** | Mobile | Polling/WebSocket | Real-time activation progress |

**Sources:** High-Level Architecture Diagrams 2 and 4

---

## Notification System

The CaaS platform employs a multi-channel notification strategy to ensure customers stay informed throughout their journey.

### Notification Channels

```mermaid
graph TB
    subgraph "Notification Triggers"
        FACTORY_STATUS["Factory Status Update"]
        VEHICLE_READY["Vehicle Ready"]
        PAYMENT_CONFIRM["Payment Confirmed"]
        OTA_SUCCESS["OTA Activation Success"]
        OTA_FAILURE["OTA Activation Failure"]
        MAINT_DUE["Maintenance Due"]
    end
    
    subgraph "Notification Engine"
        NOTIF_ENGINE["Notification Engine<br/>(Event Processor)"]
        ROUTING["Channel Router"]
    end
    
    subgraph "Delivery Channels"
        EMAIL["Email Service"]
        PUSH["Mobile Push Notifications"]
        VEHICLE_MSG["Vehicle Display Messages"]
        SMS["SMS Gateway"]
    end
    
    subgraph "Customer Touchpoints"
        CUSTOMER_EMAIL["Customer Email"]
        CUSTOMER_MOBILE["Customer Mobile App"]
        CUSTOMER_VEHICLE["Vehicle Information System"]
    end
    
    FACTORY_STATUS --> NOTIF_ENGINE
    VEHICLE_READY --> NOTIF_ENGINE
    PAYMENT_CONFIRM --> NOTIF_ENGINE
    OTA_SUCCESS --> NOTIF_ENGINE
    OTA_FAILURE --> NOTIF_ENGINE
    MAINT_DUE --> NOTIF_ENGINE
    
    NOTIF_ENGINE --> ROUTING
    
    ROUTING --> EMAIL
    ROUTING --> PUSH
    ROUTING --> VEHICLE_MSG
    ROUTING --> SMS
    
    EMAIL --> CUSTOMER_EMAIL
    PUSH --> CUSTOMER_MOBILE
    VEHICLE_MSG --> CUSTOMER_VEHICLE
    SMS --> CUSTOMER_EMAIL
```

**Diagram: Multi-Channel Notification System**

### Notification Types and Delivery

| Notification Type | Email | Mobile Push | Vehicle Display | SMS |
|-------------------|-------|-------------|-----------------|-----|
| **Credentials Sent** | ✓ | - | - | - |
| **Manufacturing Status** | ✓ | ✓ | - | Optional |
| **Vehicle Ready** | ✓ | ✓ | - | ✓ |
| **Payment Confirmation** | ✓ | ✓ | - | - |
| **Service Activated** | ✓ | ✓ | ✓ | - |
| **OTA Failure** | ✓ | ✓ | - | ✓ |
| **Maintenance Due** | ✓ | ✓ | ✓ | Optional |

### Critical Notification Rules

**Manufacturing Status Notifications:**
- Automatic notifications during vehicle production are part of the value proposition [pasame las preguntas y sus respuestas a markdown.md:28-29]()
- Status updates sent at key manufacturing milestones
- Enables customer to plan for vehicle receipt

**OTA Activation Notifications:**
- **Success Case**: Confirmation sent via app notification and vehicle display [enunciado.md:19-20]()
- **Failure Case**: Customer notified immediately with technical support escalation [pasame las preguntas y sus respuestas a markdown.md:48-53]()
- **Critical Rule**: If OTA fails after all retries, customer is notified AND NOT CHARGED [pasame las preguntas y sus respuestas a markdown.md:48-53]()

**Sources:** [enunciado.md:11](), [enunciado.md:13](), [enunciado.md:19-20](), [pasame las preguntas y sus respuestas a markdown.md:28-29](), [pasame las preguntas y sus respuestas a markdown.md:48-53]()

---

## Platform Responsibilities Summary

### Web Platform Responsibilities

- Initial customer onboarding and credential reception
- Desktop-friendly expediente browsing
- Document access (manuals, invoices)
- Service catalog exploration with detailed information
- Service purchase and payment processing
- Account and payment method management

### Mobile Application Responsibilities

- On-the-go expediente access
- Real-time service activation monitoring
- Direct vehicle status monitoring and control
- Push notification reception
- Vehicle linking and operational enablement
- Immediate updates on OTA activation status

### Shared Responsibilities

Both platforms provide:
- Access to the complete expediente de compra
- Service catalog browsing and purchasing
- Payment processing capabilities
- Transaction history viewing
- Maintenance record access (queried from VSS)
- Manufacturing status tracking (during production phase)

**Sources:** [enunciado.md:10-23](), High-Level Architecture Diagrams