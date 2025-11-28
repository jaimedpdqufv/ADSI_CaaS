# Payment Types and Flows

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [enunciado.md](enunciado.md)
- [pasame las preguntas y sus respuestas a markdown.md](pasame las preguntas y sus respuestas a markdown.md)

</details>



## Purpose and Scope

This document details all payment types within the CaaS system, their processing flows, timing characteristics, and failure handling mechanisms. It covers payments for vehicle acquisition (reservation and final payment), one-time service purchases, and subscription-based services.

For information about risk management and asynchronous settlement strategies, see [Risk Management and Settlement](#7.2). For service cancellation and refund eligibility rules, see [Service Cancellation and Refunds](#6.4). For payment failure scenarios and system behavior, see [Payment Failure Scenarios](#9.2).

---

## Payment Type Taxonomy

The CaaS system processes four distinct payment types, each with different timing characteristics, settlement patterns, and business rules:

```mermaid
graph TB
    PAYMENT_ROOT["CaaS Payment Types"]
    
    VEHICLE["Vehicle Acquisition Payments"]
    SERVICES["Service Payments"]
    
    RESERVATION["Reservation Payment<br/>(Señal)"]
    FINAL["Final Payment<br/>(Before Registration)"]
    
    ONETIME["One-time Service Payment<br/>(Pago por Uso)"]
    SUBSCRIPTION["Subscription Payment<br/>(Mes Vencido)"]
    
    PAYMENT_ROOT --> VEHICLE
    PAYMENT_ROOT --> SERVICES
    
    VEHICLE --> RESERVATION
    VEHICLE --> FINAL
    
    SERVICES --> ONETIME
    SERVICES --> SUBSCRIPTION
    
    RESERVATION -.->|"Synchronous<br/>Blocking"| RES_GATEWAY["Payment Gateway"]
    FINAL -.->|"Synchronous<br/>Blocking"| FIN_GATEWAY["Payment Gateway"]
    ONETIME -.->|"Async Settlement<br/>CaaS Assumes Risk"| ONE_GATEWAY["Payment Gateway"]
    SUBSCRIPTION -.->|"Post-paid<br/>Monthly Billing"| SUB_GATEWAY["Payment Gateway"]
```

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:77-82](), [enunciado.md:1-23]()

| Payment Type | Timing | Settlement Model | Risk Assumption | Criticality |
|-------------|--------|------------------|-----------------|-------------|
| **Reservation Payment (Señal)** | At vehicle reservation | Synchronous, blocking | None - must succeed | High - gates factory order |
| **Final Payment** | Before vehicle registration | Synchronous, blocking | None - must succeed | Critical - no vehicle without payment |
| **One-time Service Payment** | At service purchase | Async settlement | CaaS assumes settlement risk | Medium - service delivered immediately |
| **Subscription Payment (Mes Vencido)** | End of billing period | Post-paid, async | CaaS assumes collection risk | Medium - billed after consumption |

---

## Reservation Payment Flow

The reservation payment (señal) is the first financial transaction in the vehicle acquisition process. This payment triggers the factory order and reserves the vehicle for the customer.

```mermaid
sequenceDiagram
    participant Customer
    participant WebPlatform as "Web Platform /<br/>Mobile App"
    participant SalesSystem as "Corporate Intranet<br/>Sales System"
    participant PaymentGateway as "Payment Gateway"
    participant FactoryAPI as "Factory API"
    participant Database as "Order Database"
    
    Customer->>SalesSystem: "Complete vehicle selection<br/>(Plataforma Base + Options)"
    SalesSystem->>Customer: "Display reservation amount"
    
    Customer->>SalesSystem: "Initiate reservation payment"
    SalesSystem->>PaymentGateway: "POST /payments<br/>{amount, customer, vehicle}"
    
    PaymentGateway-->>SalesSystem: "Payment processing..."
    
    alt Payment Successful
        PaymentGateway-->>SalesSystem: "200 OK<br/>{transaction_id, status: confirmed}"
        SalesSystem->>Database: "Create order record<br/>(status: RESERVED)"
        SalesSystem->>FactoryAPI: "POST /orders<br/>{vehicle_spec, customer_id}"
        FactoryAPI-->>SalesSystem: "200 OK<br/>{order_id, estimated_delivery}"
        SalesSystem->>Database: "Update order<br/>(factory_order_id, status: ORDERED)"
        SalesSystem->>Customer: "Generate access credentials<br/>Send email"
        SalesSystem->>WebPlatform: "Create expediente de compra"
        WebPlatform-->>Customer: "Access to platform granted"
    else Payment Failed
        PaymentGateway-->>SalesSystem: "402 Payment Required<br/>{error_code, reason}"
        SalesSystem->>Customer: "Payment failed - retry required"
        SalesSystem->>Database: "Log failed attempt<br/>(no order created)"
    end
```

**Sources:** [enunciado.md:9-13](), [pasame las preguntas y sus respuestas a markdown.md:80]()

### Key Characteristics

- **Synchronous Processing**: The payment must succeed before proceeding with factory order
- **Blocking Operation**: Sales process cannot continue without successful reservation payment
- **Triggers Multiple Actions**:
  - Factory order placement via API
  - Customer credential generation
  - Expediente de compra creation
  - Email notification with access credentials
- **No Risk Assumption**: CaaS does not proceed unless payment is confirmed

---

## Final Payment Flow

The final payment completes the vehicle acquisition and triggers the registration and delivery process. This is the most critical payment in the system.

```mermaid
sequenceDiagram
    participant Customer
    participant WebPlatform as "Web Platform /<br/>Expediente"
    participant PaymentSystem as "Payment Processing"
    participant PaymentGateway as "Payment Gateway"
    participant AdminAPI as "Admin Registration API"
    participant Database as "Order Database"
    participant TransportSystem as "Transport Coordination"
    
    Note over Customer,TransportSystem: Vehicle manufactured and available in dealership
    
    WebPlatform->>Customer: "Notify: Vehicle ready for final payment"
    Customer->>WebPlatform: "Access expediente"
    WebPlatform->>Customer: "Display final payment amount<br/>(Total - Reservation)"
    
    Customer->>WebPlatform: "Initiate final payment"
    WebPlatform->>PaymentSystem: "Process final payment"
    PaymentSystem->>PaymentGateway: "POST /payments/final<br/>{amount, customer, vehicle}"
    
    PaymentGateway-->>PaymentSystem: "Processing..."
    
    alt Payment Successful
        PaymentGateway-->>PaymentSystem: "200 OK<br/>{transaction_id, confirmed}"
        PaymentSystem->>Database: "Update order<br/>(status: PAID, payment_complete: true)"
        PaymentSystem->>AdminAPI: "POST /register-vehicle<br/>{vehicle_id, customer_id, country}"
        AdminAPI-->>PaymentSystem: "Registration initiated"
        AdminAPI-->>PaymentSystem: "Registration complete<br/>{registration_number, plates}"
        PaymentSystem->>Database: "Update order<br/>(status: REGISTERED)"
        PaymentSystem->>TransportSystem: "Schedule delivery<br/>{customer_address, vehicle}"
        PaymentSystem->>Customer: "Payment confirmed<br/>Vehicle will be delivered"
    else Payment Failed
        PaymentGateway-->>PaymentSystem: "402 Payment Required<br/>{error}"
        PaymentSystem->>Database: "Update order<br/>(status: PAYMENT_FAILED)"
        PaymentSystem->>Database: "Mark vehicle: SIN_ASIGNAR"
        PaymentSystem->>Database: "Mark vehicle: AVAILABLE_STOCK"
        PaymentSystem->>Customer: "Payment failed<br/>Reservation lost"
        Note over Database: Vehicle immediately available for sale to other customers
    end
```

**Sources:** [enunciado.md:14-17](), [pasame las preguntas y sus respuestas a markdown.md:26-27](), [pasame las preguntas y sus respuestas a markdown.md:80]()

### Critical Business Rules

1. **Payment Must Succeed**: No vehicle delivery without complete final payment
2. **Immediate Consequence of Failure**: 
   - Vehicle marked as `SIN_ASIGNAR` (unassigned)
   - Vehicle becomes available stock for immediate sale
   - Customer loses reservation entirely
3. **Triggers Registration**: Only after successful payment does registration begin
4. **Blocking Operation**: Customer cannot proceed to delivery without payment confirmation

---

## One-Time Service Payment Flow

One-time service payments enable customers to purchase optional features (opciones disponibles) on a pay-per-use basis. These payments follow an **asynchronous settlement model** where CaaS delivers the service immediately while assuming settlement risk.

```mermaid
sequenceDiagram
    participant Customer
    participant WebPlatform as "Web/Mobile Platform"
    participant ServiceCatalog as "Service Catalog"
    participant VSSSystem as "VSS System"
    participant PaymentSystem as "Payment Processing"
    participant PaymentGateway as "Payment Gateway"
    participant OTAEngine as "OTA Activation Engine"
    participant IoTGateway as "IoT Gateway"
    participant Database as "Service Database"
    
    Customer->>WebPlatform: "Browse opciones disponibles"
    WebPlatform->>ServiceCatalog: "Get available services<br/>for customer vehicle"
    ServiceCatalog-->>WebPlatform: "Service catalog<br/>(with prices, durations)"
    
    Customer->>WebPlatform: "Select service + duration"
    
    alt Service Requires Maintenance Check
        WebPlatform->>VSSSystem: "GET /maintenance-status<br/>{vehicle_id, service_id}"
        VSSSystem-->>WebPlatform: "Maintenance status"
        
        alt Maintenance Not Current
            WebPlatform->>Customer: "Service blocked<br/>Complete maintenance first"
            Note over Customer: Flow terminates - see Service Blocking
        end
    end
    
    WebPlatform->>Customer: "Display final price"
    Customer->>WebPlatform: "Confirm purchase"
    
    WebPlatform->>PaymentSystem: "Initiate service payment"
    PaymentSystem->>PaymentGateway: "POST /payments/service<br/>{amount, customer, service}"
    
    Note over PaymentGateway: Payment processing (async settlement)
    PaymentGateway-->>PaymentSystem: "202 Accepted<br/>{transaction_id, status: pending}"
    
    critical CaaS Assumes Risk - Immediate Delivery
        PaymentSystem->>Database: "Create service activation record<br/>(status: PAYMENT_PENDING)"
        PaymentSystem->>OTAEngine: "Queue OTA activation<br/>{vehicle, service, config}"
        OTAEngine->>IoTGateway: "Send OTA command<br/>{vehicle_id, service_config}"
        
        IoTGateway-->>OTAEngine: "Command sent"
        
        alt OTA Activation Successful
            IoTGateway-->>OTAEngine: "Activation confirmed<br/>(vehicle reports success)"
            OTAEngine->>Database: "Update record<br/>(status: ACTIVE)"
            OTAEngine->>Customer: "Notify: Service activated"
            
            par Async Payment Settlement
                PaymentGateway-->>PaymentSystem: "Webhook: Payment settled<br/>{transaction_id, status: confirmed}"
                PaymentSystem->>Database: "Update record<br/>(payment_status: CONFIRMED)"
            end
        else OTA Activation Failed
            Note over OTAEngine: See OTA failure handling
            OTAEngine->>PaymentSystem: "Activation failed after retries"
            PaymentSystem->>PaymentGateway: "POST /refund<br/>{transaction_id}"
            PaymentSystem->>Database: "Update record<br/>(status: FAILED, charged: false)"
            PaymentSystem->>Customer: "Service unavailable<br/>NO CHARGE applied"
        end
    end
```

**Sources:** [enunciado.md:18-19](), [pasame las preguntas y sus respuestas a markdown.md:77-82](), [pasame las preguntas y sus respuestas a markdown.md:48-53]()

### Key Characteristics

| Aspect | Behavior |
|--------|----------|
| **Settlement Model** | Asynchronous - CaaS proceeds before bank confirms |
| **Risk Assumption** | CaaS assumes risk of payment failure/chargeback |
| **Service Delivery** | Immediate upon payment initiation (not settlement) |
| **Failure Handling** | **DO NOT CHARGE** customer if OTA activation fails |
| **Maintenance Gating** | Some services blocked if maintenance not current |
| **Customer Protection** | No charge for undelivered functionality |

### Critical Rule: No Charge for Failed Activation

The most important business rule for one-time service payments is:

**If OTA activation fails after all retry attempts, the customer must NOT be charged for the service.**

This rule protects customers from paying for functionality they cannot use and is implemented by:
1. Initiating refund immediately upon final OTA failure
2. Marking the service record as `charged: false`
3. Notifying the customer that no charge has been applied
4. Escalating to technical support for resolution

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:48-53]()

---

## Subscription Payment Flow (Mes Vencido)

Subscription payments follow a **post-paid billing model** where customers are charged at the end of each billing period for services consumed during that period. This model is referred to as "mes vencido" (expired month) billing.

```mermaid
sequenceDiagram
    participant Customer
    participant WebPlatform as "Web/Mobile Platform"
    participant SubscriptionMgmt as "Subscription Management"
    participant ServiceUsage as "Service Usage Tracker"
    participant BillingEngine as "Billing Engine"
    participant PaymentGateway as "Payment Gateway"
    participant Database as "Billing Database"
    
    Customer->>WebPlatform: "Subscribe to recurring service"
    WebPlatform->>SubscriptionMgmt: "Create subscription<br/>{customer, service, start_date}"
    
    SubscriptionMgmt->>Database: "Create subscription record<br/>(status: ACTIVE, billing_period: MONTHLY)"
    SubscriptionMgmt->>Customer: "Subscription activated<br/>Billing: mes vencido"
    
    Note over Customer,Database: Customer uses service during billing period
    
    loop During Billing Period
        Customer->>ServiceUsage: "Use subscribed service"
        ServiceUsage->>Database: "Log usage<br/>{date, service, duration}"
    end
    
    Note over BillingEngine: End of billing period (month end)
    
    BillingEngine->>Database: "Query usage for billing period<br/>{customer, start_date, end_date}"
    Database-->>BillingEngine: "Usage records"
    
    BillingEngine->>BillingEngine: "Calculate total charge<br/>(usage * rate)"
    BillingEngine->>Database: "Create invoice<br/>{amount, period, due_date}"
    
    BillingEngine->>Customer: "Send invoice notification<br/>(email + app)"
    
    BillingEngine->>PaymentGateway: "POST /payments/subscription<br/>{amount, customer, invoice_id}"
    
    alt Payment Successful
        PaymentGateway-->>BillingEngine: "200 OK<br/>{transaction_id, confirmed}"
        BillingEngine->>Database: "Update invoice<br/>(status: PAID)"
        BillingEngine->>SubscriptionMgmt: "Subscription remains active"
        BillingEngine->>Customer: "Payment confirmed"
        Note over SubscriptionMgmt: Next billing period begins
    else Payment Failed
        PaymentGateway-->>BillingEngine: "402 Payment Required"
        BillingEngine->>Database: "Update invoice<br/>(status: UNPAID)"
        BillingEngine->>SubscriptionMgmt: "Payment failed"
        
        alt Retry Logic
            Note over BillingEngine: Retry payment attempts (e.g., 3x)
            BillingEngine->>PaymentGateway: "Retry payment"
        end
        
        alt All Retries Failed
            SubscriptionMgmt->>Database: "Update subscription<br/>(status: CANCELLED_NONPAYMENT)"
            SubscriptionMgmt->>Customer: "Subscription cancelled<br/>Payment failure"
        end
    end
```

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:82]()

### Mes Vencido Billing Characteristics

| Aspect | Detail |
|--------|--------|
| **Billing Timing** | End of billing period (month end) |
| **Payment Model** | Post-paid (charge after consumption) |
| **Service Delivery** | Immediate upon subscription activation |
| **Billing Period** | Monthly (configurable) |
| **Risk** | CaaS bears collection risk |
| **Cancellation** | Automatic if payment fails after retries |

### Subscription Lifecycle States

```mermaid
stateDiagram-v2
    [*] --> ACTIVE: "Customer subscribes"
    
    ACTIVE --> BILLING: "End of month"
    BILLING --> PAYMENT_PROCESSING: "Generate invoice"
    
    PAYMENT_PROCESSING --> PAID: "Payment successful"
    PAYMENT_PROCESSING --> RETRY: "Payment failed"
    
    PAID --> ACTIVE: "Next billing period"
    
    RETRY --> PAID: "Retry successful"
    RETRY --> CANCELLED_NONPAYMENT: "All retries failed"
    
    ACTIVE --> CANCELLED_CUSTOMER: "Customer cancels"
    
    CANCELLED_NONPAYMENT --> [*]
    CANCELLED_CUSTOMER --> [*]
```

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:82]()

---

## Planned and Recurring Service Payments

The CaaS system supports future-scheduled service activations and recurring purchases, which combine aspects of both one-time and subscription payments.

```mermaid
graph TB
    subgraph "Planned Service Purchase"
        PLAN_PURCHASE["Customer purchases<br/>service for future date"]
        PLAN_PAYMENT["Immediate payment"]
        PLAN_SCHEDULE["Schedule activation<br/>for future date"]
        PLAN_ACTIVATION["OTA activation<br/>on scheduled date"]
        
        PLAN_PURCHASE --> PLAN_PAYMENT
        PLAN_PAYMENT --> PLAN_SCHEDULE
        PLAN_SCHEDULE -.->|"Wait until scheduled date"| PLAN_ACTIVATION
    end
    
    subgraph "Recurring Service Purchase"
        RECUR_SETUP["Customer sets up<br/>recurring purchase"]
        RECUR_FIRST["First payment<br/>+ activation"]
        RECUR_SCHEDULE["Schedule recurrence<br/>(e.g., monthly)"]
        RECUR_REPEAT["Automatic renewal<br/>+ payment"]
        
        RECUR_SETUP --> RECUR_FIRST
        RECUR_FIRST --> RECUR_SCHEDULE
        RECUR_SCHEDULE -.->|"Each period"| RECUR_REPEAT
        RECUR_REPEAT -.->|"Loop"| RECUR_SCHEDULE
    end
    
    PLAN_ACTIVATION -.->|"Can be recurring"| RECUR_SETUP
```

**Sources:** [enunciado.md:21-22]()

### Planned Service Characteristics

- **Payment Timing**: Immediate (at purchase time)
- **Service Activation**: Scheduled for future date
- **Cancellation Rights**: Subject to desistimiento rules (14-day window if service > 14 days)
- **OTA Delivery**: Standard retry logic applies at activation time

### Recurring Service Characteristics

- **Initial Payment**: First period charged immediately
- **Renewal Payments**: Automatic at each recurrence interval
- **Cancellation**: Customer can cancel anytime (affects future renewals only)
- **Failed Payment**: Cancels future recurrences

---

## Payment Gateway Integration Architecture

```mermaid
graph TB
    subgraph "CaaS Payment System"
        PaymentController["Payment Controller"]
        PaymentService["Payment Service"]
        PaymentRepo["Payment Repository"]
        
        PaymentController --> PaymentService
        PaymentService --> PaymentRepo
    end
    
    subgraph "Payment Gateway (External)"
        GatewayAPI["Gateway REST API"]
        CardProcessing["Card Processing"]
        BankNetwork["Banking Network"]
        
        GatewayAPI --> CardProcessing
        CardProcessing --> BankNetwork
    end
    
    subgraph "Payment Flows by Type"
        SyncFlow["Synchronous Flows:<br/>- Reservation Payment<br/>- Final Payment"]
        AsyncFlow["Async Settlement:<br/>- Service Payments<br/>- Subscription Payments"]
        
        SyncFlow -.->|"Wait for confirmation"| GatewayAPI
        AsyncFlow -.->|"Proceed immediately"| GatewayAPI
    end
    
    PaymentService -->|"POST /payments"| GatewayAPI
    GatewayAPI -.->|"Webhook callbacks"| PaymentService
    
    PaymentService --> PaymentEvents["Payment Events:<br/>- PaymentInitiated<br/>- PaymentConfirmed<br/>- PaymentFailed<br/>- PaymentSettled"]
    
    PaymentEvents -->|"Triggers"| ServiceActivation["Service Activation"]
    PaymentEvents -->|"Triggers"| Notifications["Customer Notifications"]
    PaymentEvents -->|"Triggers"| Accounting["Accounting System"]
```

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:77-82]()

### Integration Patterns by Payment Type

| Payment Type | Integration Pattern | Behavior |
|-------------|-------------------|----------|
| **Reservation** | Synchronous request-response | Wait for payment confirmation before proceeding |
| **Final Payment** | Synchronous request-response | Block until payment confirmed or failed |
| **Service Payment** | Async initiate + webhook settlement | Deliver service immediately, settle later |
| **Subscription** | Async post-paid billing | Charge at period end, use webhooks for confirmation |

---

## Payment State Machine

```mermaid
stateDiagram-v2
    [*] --> INITIATED: "Payment request"
    
    INITIATED --> PROCESSING: "Sent to gateway"
    
    PROCESSING --> CONFIRMED: "Sync: immediate confirmation"
    PROCESSING --> PENDING: "Async: accepted for processing"
    PROCESSING --> FAILED: "Declined/error"
    
    PENDING --> SETTLED: "Bank settlement complete"
    PENDING --> FAILED: "Settlement failed"
    
    CONFIRMED --> SETTLED: "Final settlement"
    
    SETTLED --> REFUNDED: "Refund requested"
    
    FAILED --> RETRY: "Retry attempt"
    RETRY --> PROCESSING
    RETRY --> ABANDONED: "Max retries exceeded"
    
    SETTLED --> [*]
    REFUNDED --> [*]
    ABANDONED --> [*]
    
    note right of PENDING
        Service delivered
        in this state for
        one-time payments
    end note
    
    note right of FAILED
        For service payments:
        DO NOT CHARGE if
        OTA activation failed
    end note
```

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:77-82](), [pasame las preguntas y sus respuestas a markdown.md:48-53]()

---

## Payment Failure Handling Summary

Payment failures are handled differently based on payment type and failure timing:

| Failure Scenario | System Behavior | Customer Impact |
|-----------------|-----------------|-----------------|
| **Reservation payment fails** | No order created, no factory order placed | Cannot proceed with purchase |
| **Final payment fails** | Vehicle marked `SIN_ASIGNAR`, becomes available stock | Loses reservation entirely |
| **Service payment fails (before OTA)** | No service activation attempted | No charge, can retry |
| **Service payment fails (after OTA success)** | Service delivered, payment still pending | CaaS assumes settlement risk |
| **OTA activation fails** | Payment refunded immediately | **NO CHARGE** - critical protection |
| **Subscription payment fails** | Retry attempts, then cancel subscription | Service terminates after grace period |

**Critical Business Rule**: The most important customer protection is: **Never charge for services that could not be activated via OTA**.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:26-27](), [pasame las preguntas y sus respuestas a markdown.md:48-53](), [pasame las preguntas y sus respuestas a markdown.md:77-82]()

---

## Payment Data Flow Summary

```mermaid
graph LR
    subgraph "Payment Initiation"
        CustomerUI["Customer Interface"]
        PaymentReq["Payment Request"]
    end
    
    subgraph "Payment Processing"
        PaymentSvc["Payment Service"]
        Gateway["Payment Gateway"]
        BankNet["Bank Network"]
    end
    
    subgraph "Business Actions"
        FactoryOrder["Factory Order<br/>(Reservation)"]
        VehicleReg["Vehicle Registration<br/>(Final Payment)"]
        OTAActivation["OTA Activation<br/>(Service Payment)"]
        BillingInvoice["Invoice Generation<br/>(Subscription)"]
    end
    
    subgraph "Data Persistence"
        OrderDB["Order Database"]
        ServiceDB["Service Database"]
        BillingDB["Billing Database"]
        AuditLog["Payment Audit Log"]
    end
    
    CustomerUI --> PaymentReq
    PaymentReq --> PaymentSvc
    PaymentSvc --> Gateway
    Gateway --> BankNet
    
    PaymentSvc -.->|"Success: Reservation"| FactoryOrder
    PaymentSvc -.->|"Success: Final"| VehicleReg
    PaymentSvc -.->|"Success: Service"| OTAActivation
    PaymentSvc -.->|"Period End"| BillingInvoice
    
    FactoryOrder --> OrderDB
    VehicleReg --> OrderDB
    OTAActivation --> ServiceDB
    BillingInvoice --> BillingDB
    
    PaymentSvc --> AuditLog
```

**Sources:** [enunciado.md:1-23](), [pasame las preguntas y sus respuestas a markdown.md:77-82]()

---

This document provides a comprehensive overview of all payment types and flows within the CaaS system. For detailed information on specific aspects:
- Risk management strategies: see [Risk Management and Settlement](#7.2)
- Refund processing: see [Service Cancellation and Refunds](#6.4)
- Payment failure scenarios: see [Payment Failure Scenarios](#9.2)
- OTA activation failures: see [OTA Activation Failures](#9.1)