# Payment and Billing

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [enunciado.md](enunciado.md)
- [pasame las preguntas y sus respuestas a markdown.md](pasame las preguntas y sus respuestas a markdown.md)

</details>



## Purpose and Scope

This page documents the payment processing architecture, billing models, and financial workflows within the CaaS platform. It covers all payment types (reservation, final, service), settlement strategies, and financial risk management.

For information about specific service payment flows and OTA activation consequences, see [Service Lifecycle Management](#6). For detailed refund policies related to desistimiento (right of withdrawal), see [Service Cancellation and Refunds](#6.4). For payment failure scenarios, see [Payment Failure Scenarios](#9.2).

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:75-96](), [enunciado.md:1-23]()

---

## Payment Architecture Overview

The CaaS platform implements a multi-tiered payment architecture supporting three distinct payment types with different processing characteristics, risk profiles, and business rules.

### Payment System Components

```mermaid
graph TB
    subgraph "Customer Channels"
        WEB["Web Platform"]
        MOBILE["Mobile Application"]
        INTRANET["Corporate Intranet<br/>(Dealership)"]
    end
    
    subgraph "Payment Processing Layer"
        PAYMENT_ORCHESTRATOR["Payment Orchestrator"]
        RESERVATION_PROCESSOR["Reservation Payment<br/>Processor"]
        FINAL_PROCESSOR["Final Payment<br/>Processor"]
        SERVICE_PROCESSOR["Service Payment<br/>Processor"]
        SUBSCRIPTION_PROCESSOR["Subscription Billing<br/>Processor"]
    end
    
    subgraph "Risk Management"
        RISK_ENGINE["Risk Assessment Engine"]
        SETTLEMENT_MONITOR["Settlement Monitor<br/>(Async Tracking)"]
        IMMEDIATE_DELIVERY["Immediate Delivery<br/>Decision Engine"]
    end
    
    subgraph "External Payment Infrastructure"
        PAYMENT_GATEWAY["Payment Gateway"]
        BANK_SETTLEMENT["Bank Settlement<br/>(Asynchronous)"]
        CARD_NETWORKS["Card Networks"]
    end
    
    subgraph "Service Delivery"
        OTA_ENGINE["OTA Activation Engine"]
        VEHICLE_ASSIGNMENT["Vehicle Assignment<br/>Manager"]
        FACTORY_API["Factory Order API"]
    end
    
    subgraph "Financial Records"
        BILLING_DB[("Billing Database")]
        INVOICE_GEN["Invoice Generator"]
        REFUND_PROCESSOR["Refund Processor"]
    end
    
    WEB --> PAYMENT_ORCHESTRATOR
    MOBILE --> PAYMENT_ORCHESTRATOR
    INTRANET --> PAYMENT_ORCHESTRATOR
    
    PAYMENT_ORCHESTRATOR --> RESERVATION_PROCESSOR
    PAYMENT_ORCHESTRATOR --> FINAL_PROCESSOR
    PAYMENT_ORCHESTRATOR --> SERVICE_PROCESSOR
    PAYMENT_ORCHESTRATOR --> SUBSCRIPTION_PROCESSOR
    
    RESERVATION_PROCESSOR --> RISK_ENGINE
    FINAL_PROCESSOR --> RISK_ENGINE
    SERVICE_PROCESSOR --> RISK_ENGINE
    
    RISK_ENGINE --> IMMEDIATE_DELIVERY
    IMMEDIATE_DELIVERY --> PAYMENT_GATEWAY
    
    PAYMENT_GATEWAY --> CARD_NETWORKS
    CARD_NETWORKS -.Async Settlement.-> BANK_SETTLEMENT
    BANK_SETTLEMENT -.Confirmation.-> SETTLEMENT_MONITOR
    SETTLEMENT_MONITOR --> BILLING_DB
    
    RESERVATION_PROCESSOR --> FACTORY_API
    FINAL_PROCESSOR --> VEHICLE_ASSIGNMENT
    SERVICE_PROCESSOR --> OTA_ENGINE
    
    PAYMENT_ORCHESTRATOR --> INVOICE_GEN
    INVOICE_GEN --> BILLING_DB
    
    REFUND_PROCESSOR --> PAYMENT_GATEWAY
    REFUND_PROCESSOR --> BILLING_DB
    
    SUBSCRIPTION_PROCESSOR --> BILLING_DB
```

**Architectural Responsibilities:**

| Component | Responsibility | Risk Characteristics |
|-----------|---------------|---------------------|
| `RESERVATION_PROCESSOR` | Process initial reservation payment (señal) | Low risk - small amount, triggers factory order |
| `FINAL_PROCESSOR` | Process final vehicle payment before registration | High risk - large amount, blocks vehicle delivery |
| `SERVICE_PROCESSOR` | Process one-time service payments | Medium risk - immediate OTA delivery despite async settlement |
| `SUBSCRIPTION_PROCESSOR` | Bill monthly subscriptions at end of period (mes vencido) | High risk - post-paid model with collection risk |
| `IMMEDIATE_DELIVERY` | Decide whether to deliver service before settlement confirmation | Core risk management component |
| `SETTLEMENT_MONITOR` | Track async bank settlements and reconcile with delivered services | Detects settlement failures post-delivery |

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:78-82](), [enunciado.md:13-19]()

---

## Payment Types

The CaaS platform processes three distinct payment types, each with unique characteristics and business rules.

### Payment Type Taxonomy

| Payment Type | Purpose | Timing | Amount | Settlement Risk | Refundable |
|--------------|---------|--------|--------|-----------------|------------|
| **Reservation Payment** | Secure vehicle allocation | At sales registration | Partial (señal/deposit) | Low | No (lost if final payment fails) |
| **Final Payment** | Complete vehicle purchase | Before vehicle registration | Remaining balance | None (blocking) | No |
| **Service Payment (One-time)** | Activate optional service | At service purchase | Service price | Medium (async settlement) | Yes (desistimiento rules apply) |
| **Service Payment (Subscription)** | Monthly recurring service | End of month (mes vencido) | Monthly fee | High (post-paid) | Yes (can cancel anytime) |

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:78-96](), [enunciado.md:13-21]()

### Reservation Payment Flow

```mermaid
sequenceDiagram
    participant DEALER as Dealership Staff<br/>(Intranet)
    participant INTRANET as Corporate Intranet
    participant PAY_PROC as Reservation Processor
    participant GATEWAY as Payment Gateway
    participant FACTORY as Factory API
    participant CUSTOMER as Customer
    
    DEALER->>INTRANET: Register sale<br/>(customer data + plan comercial)
    INTRANET->>PAY_PROC: Request reservation payment
    PAY_PROC->>GATEWAY: Process señal payment
    GATEWAY-->>PAY_PROC: Payment initiated
    
    Note over PAY_PROC: Assumes successful settlement
    
    PAY_PROC->>FACTORY: POST /orders<br/>(automatic order placement)
    FACTORY-->>PAY_PROC: Order confirmation
    
    PAY_PROC->>CUSTOMER: Email credentials<br/>+ order confirmation
    
    Note over GATEWAY,PAY_PROC: Async settlement occurs<br/>(reconciled later)
```

**Critical Rule:** Once reservation payment is initiated, the factory order is placed immediately. This creates financial exposure if the payment later fails, but optimizes manufacturing lead time.

**Sources:** [enunciado.md:8-13](), [pasame las preguntas y sus respuestas a markdown.md:78-82]()

### Final Payment Flow

```mermaid
sequenceDiagram
    participant CUSTOMER as Customer
    participant PLATFORM as Web/Mobile Platform
    participant FINAL_PROC as Final Payment Processor
    participant GATEWAY as Payment Gateway
    participant VEHICLE_MGT as Vehicle Assignment Manager
    participant ADMIN_API as Administrative API
    
    PLATFORM->>CUSTOMER: Notify: Vehicle ready<br/>in dealership
    CUSTOMER->>PLATFORM: Initiate final payment
    PLATFORM->>FINAL_PROC: Request final payment
    FINAL_PROC->>GATEWAY: Process payment
    GATEWAY-->>FINAL_PROC: Payment result
    
    alt Payment Successful
        FINAL_PROC->>VEHICLE_MGT: Confirm payment
        VEHICLE_MGT->>ADMIN_API: Register vehicle<br/>(matricular)
        ADMIN_API-->>VEHICLE_MGT: Registration complete
        VEHICLE_MGT->>CUSTOMER: Schedule delivery
    else Payment Failed
        FINAL_PROC->>VEHICLE_MGT: Mark vehicle "sin asignar"
        VEHICLE_MGT->>VEHICLE_MGT: Convert to stock<br/>for immediate sale
        FINAL_PROC->>CUSTOMER: Notify: Reservation lost
    end
```

**Critical Rule:** Final payment is **blocking**. If payment fails, the vehicle immediately becomes unassigned stock. Customer loses their reservation entirely. This is non-negotiable for cash flow protection.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:26-27](), [enunciado.md:14-17]()

### Service Payment Flow (One-time)

```mermaid
sequenceDiagram
    participant CUSTOMER as Customer
    participant EXPEDIENTE as Expediente System
    participant SERVICE_PROC as Service Payment Processor
    participant GATEWAY as Payment Gateway
    participant OTA as OTA Activation Engine
    participant IOT as IoT Gateway
    participant VEHICLE as Vehicle
    
    CUSTOMER->>EXPEDIENTE: Browse opciones disponibles
    EXPEDIENTE->>CUSTOMER: Display service catalog<br/>+ pricing
    CUSTOMER->>EXPEDIENTE: Select service + duration
    EXPEDIENTE->>SERVICE_PROC: Initiate payment
    SERVICE_PROC->>GATEWAY: Process payment
    GATEWAY-->>SERVICE_PROC: Payment initiated
    
    Note over SERVICE_PROC: Immediate delivery decision<br/>(assumes settlement risk)
    
    SERVICE_PROC->>OTA: Trigger OTA activation
    OTA->>IOT: Send configuration command
    IOT->>VEHICLE: Apply configuration
    VEHICLE-->>IOT: Configuration status
    
    alt OTA Success
        IOT-->>OTA: Activation confirmed
        OTA->>SERVICE_PROC: Service delivered
        SERVICE_PROC->>CUSTOMER: Notify: Service active
        
        Note over GATEWAY,SERVICE_PROC: Async settlement<br/>tracked separately
    else OTA Failure (after retries)
        IOT-->>OTA: Activation failed
        OTA->>SERVICE_PROC: Delivery failed
        SERVICE_PROC->>GATEWAY: Void/refund payment
        SERVICE_PROC->>CUSTOMER: Notify: NOT CHARGED
    end
```

**Critical Rule:** If OTA activation fails after all retries, the customer is **NEVER CHARGED** for the service. This is a core customer protection policy. The payment is voided or refunded if already settled.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:48-53](), [enunciado.md:18-19]()

### Subscription Billing Flow (Mes Vencido)

```mermaid
sequenceDiagram
    participant CUSTOMER as Customer
    participant PLATFORM as Web/Mobile Platform
    participant SUB_PROC as Subscription Processor
    participant USAGE_TRACKER as Usage Tracker
    participant GATEWAY as Payment Gateway
    participant BILLING_DB as Billing Database
    
    CUSTOMER->>PLATFORM: Subscribe to service
    PLATFORM->>SUB_PROC: Activate subscription
    SUB_PROC->>USAGE_TRACKER: Start tracking usage
    
    Note over CUSTOMER,USAGE_TRACKER: Customer uses service<br/>throughout month
    
    USAGE_TRACKER->>USAGE_TRACKER: Month ends
    USAGE_TRACKER->>SUB_PROC: Trigger mes vencido billing
    SUB_PROC->>BILLING_DB: Calculate charges
    BILLING_DB-->>SUB_PROC: Total amount due
    
    SUB_PROC->>GATEWAY: Charge for consumed services
    GATEWAY-->>SUB_PROC: Payment result
    
    alt Payment Successful
        SUB_PROC->>BILLING_DB: Record payment
        SUB_PROC->>USAGE_TRACKER: Continue subscription
    else Payment Failed
        SUB_PROC->>USAGE_TRACKER: Cancel subscription
        SUB_PROC->>CUSTOMER: Notify: Subscription cancelled
    end
```

**Billing Model:** **Mes vencido** (post-paid) - customers are charged at the **end of each month** for services consumed during that month. This increases customer satisfaction but creates collection risk.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:82]()

---

## Payment Processing Architecture

### Payment Gateway Integration Pattern

The CaaS platform integrates with external payment gateways using an **asynchronous settlement pattern** where service delivery occurs before final settlement confirmation.

```mermaid
graph LR
    subgraph "CaaS Platform"
        PAYMENT_INIT["Payment Initiation"]
        DELIVERY_ENGINE["Service Delivery Engine"]
        SETTLEMENT_RECONCILE["Settlement Reconciliation"]
    end
    
    subgraph "Payment Gateway"
        AUTHORIZATION["Authorization"]
        CAPTURE["Capture"]
        SETTLEMENT["Settlement"]
    end
    
    subgraph "Banking Network"
        CARD_NETWORK["Card Networks"]
        BANK["Issuing Bank"]
    end
    
    PAYMENT_INIT -->|1. Authorize payment| AUTHORIZATION
    AUTHORIZATION -->|2. Authorization OK| PAYMENT_INIT
    PAYMENT_INIT -->|3. Immediate delivery| DELIVERY_ENGINE
    
    DELIVERY_ENGINE -->|4. Deliver service to customer| DELIVERY_ENGINE
    
    AUTHORIZATION -.5. Async: Capture funds.-> CAPTURE
    CAPTURE -.6. Async: Send to network.-> CARD_NETWORK
    CARD_NETWORK -.7. Async: Settlement.-> BANK
    BANK -.8. Async: Confirmation.-> SETTLEMENT
    SETTLEMENT -.9. Async: Notify CaaS.-> SETTLEMENT_RECONCILE
```

**Timeline Analysis:**

| Time | CaaS Action | Payment Status | Risk Level |
|------|-------------|----------------|------------|
| T+0 seconds | Payment initiated | Authorization only | **CaaS assumes risk** |
| T+1 second | Service delivered to customer | Not settled | **Maximum exposure** |
| T+2-4 hours | (waiting) | Capture processing | Elevated risk |
| T+24-72 hours | (waiting) | Settlement in progress | Moderate risk |
| T+3-5 days | Settlement confirmation received | Confirmed | Risk resolved |

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:78-82]()

### Risk Assumption Strategy

```mermaid
graph TD
    PAYMENT_INIT["Payment Initiated"]
    RISK_EVAL["Risk Evaluation"]
    
    PAYMENT_INIT --> RISK_EVAL
    
    RISK_EVAL --> DECISION{Service Type?}
    
    DECISION -->|Reservation| RESERVE_FLOW["Low Risk:<br/>Deliver immediately<br/>(small amount)"]
    DECISION -->|Final Payment| FINAL_FLOW["No Risk:<br/>Blocking operation<br/>(must confirm)"]
    DECISION -->|Service Payment| SERVICE_FLOW["Medium Risk:<br/>Deliver immediately<br/>(async settlement)"]
    DECISION -->|Subscription| SUB_FLOW["High Risk:<br/>Post-paid model<br/>(collect after use)"]
    
    RESERVE_FLOW --> DELIVER["Deliver Service"]
    SERVICE_FLOW --> DELIVER
    SUB_FLOW --> MONTH_END["Bill at Month End"]
    
    FINAL_FLOW --> BLOCK["Block Until Confirmed"]
    BLOCK --> DELIVER
    
    DELIVER --> SETTLE_WAIT["Wait for Settlement"]
    MONTH_END --> SETTLE_WAIT
    
    SETTLE_WAIT --> SETTLE_RESULT{Settlement Result}
    
    SETTLE_RESULT -->|Success| RECORD_SUCCESS["Record Successful<br/>Transaction"]
    SETTLE_RESULT -->|Failure| HANDLE_FAIL["Handle Settlement<br/>Failure"]
    
    HANDLE_FAIL --> ALREADY_DELIVERED{Service Already<br/>Delivered?}
    
    ALREADY_DELIVERED -->|Yes - OTA Success| WRITE_OFF["Write-off Loss<br/>(customer keeps service)"]
    ALREADY_DELIVERED -->|Yes - OTA Failed| NO_LOSS["No Loss<br/>(customer not charged)"]
    ALREADY_DELIVERED -->|No - Pending| CANCEL_SERVICE["Cancel Pending<br/>Service"]
```

**Business Rationale:** CaaS prioritizes **customer experience over financial risk** by delivering services immediately upon payment authorization, before settlement confirmation. This creates financial exposure but improves customer satisfaction significantly.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:78-82]()

---

## Billing Models

### Pago por Uso (Pay-per-Use)

The fundamental billing model for optional services is **pago por uso** (pay-per-use), where customers pay for each service activation based on duration.

**Pricing Structure:**

| Duration Type | Payment Model | Example |
|--------------|---------------|---------|
| **Temporary** | One-time payment for fixed period | "Heated seats for 30 days: €50" |
| **Permanent** | One-time payment for vehicle lifetime | "50% power increase: €2,000" |
| **Planned** | One-time payment for future activation | "Winter mode from Dec 1-Feb 28: €150" |
| **Recurring** | Subscription with monthly billing | "Entertainment package: €30/month" |

**Sources:** [enunciado.md:3-5](), [enunciado.md:21]()

### Mes Vencido (Post-paid) Billing

For **subscription services**, CaaS implements **mes vencido** billing, a post-paid model where customers are charged at the end of each billing period for services consumed.

```mermaid
graph LR
    START["Subscription Activated"]
    USE["Customer Uses Service<br/>(Month 1)"]
    END_MONTH["Month 1 Ends"]
    BILL["Generate Bill<br/>for Month 1"]
    CHARGE["Charge Customer"]
    
    START --> USE
    USE --> END_MONTH
    END_MONTH --> BILL
    BILL --> CHARGE
    
    CHARGE --> SUCCESS{Payment<br/>Successful?}
    
    SUCCESS -->|Yes| CONTINUE["Continue to Month 2"]
    SUCCESS -->|No| CANCEL["Cancel Subscription"]
    
    CONTINUE --> USE2["Customer Uses Service<br/>(Month 2)"]
    USE2 --> END_MONTH
```

**Advantages:**
- Lower friction for customer adoption (no upfront payment)
- Aligns billing with actual usage
- Customer only pays for what they've consumed

**Risks:**
- Collection risk for services already delivered
- Requires dunning processes for failed payments
- Potential write-offs for uncollectable amounts

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:82]()

---

## Refund and Cancellation Policies

### Desistimiento (Right of Withdrawal)

CaaS complies with **distance selling regulations** that grant customers a **right of withdrawal (desistimiento)** for remotely purchased services.

**Refund Rules by Service Duration:**

```mermaid
graph TD
    CANCEL_REQUEST["Customer Requests<br/>Service Cancellation"]
    
    CANCEL_REQUEST --> SERVICE_DURATION{Service<br/>Duration?}
    
    SERVICE_DURATION -->|Duration > 14 days| LONG_SERVICE["Long-duration Service"]
    SERVICE_DURATION -->|Duration <= 14 days| SHORT_SERVICE["Short-duration Service"]
    
    LONG_SERVICE --> TIME_CHECK{Time Since<br/>Purchase?}
    TIME_CHECK -->|<= 14 days| REFUND_LONG["REFUND GRANTED<br/>(within withdrawal period)"]
    TIME_CHECK -->|> 14 days| NO_REFUND_LONG["NO REFUND<br/>(service consumed)"]
    
    SHORT_SERVICE --> REFUND_SHORT["REFUND GRANTED<br/>(anytime during duration)"]
    
    REFUND_LONG --> PROCESS_REFUND["Process Refund"]
    REFUND_SHORT --> PROCESS_REFUND
    NO_REFUND_LONG --> NOTIFY["Notify Customer:<br/>No refund available"]
    
    PROCESS_REFUND --> DEACTIVATE["Deactivate Service<br/>(if still active)"]
```

**Refund Policy Table:**

| Service Duration | Refund Window | Business Logic |
|-----------------|---------------|----------------|
| **> 14 days** | First 14 days after purchase | Customer can test service and withdraw if unsatisfied |
| **≤ 14 days** | Anytime during service duration | Service hasn't fully elapsed, so withdrawal always possible |
| **Subscription** | Anytime (cancel future billing) | Can cancel to stop future charges, but no refund for current period already used |

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:84-89]()

### Special Scenarios

#### Vehicle Theft

```mermaid
graph TD
    THEFT["Vehicle Stolen"]
    
    THEFT --> SERVICES{Services Status?}
    
    SERVICES --> DELIVERED["Already Delivered<br/>Services"]
    SERVICES --> PENDING["Pending/Future<br/>Services"]
    
    DELIVERED --> DESIST_CHECK{Within Desistimiento<br/>Period?}
    
    DESIST_CHECK -->|Yes| MAY_REFUND["May Refund<br/>(if eligible)"]
    DESIST_CHECK -->|No| NO_REFUND_THEFT["NO REFUND<br/>(service consumed)"]
    
    PENDING --> CAN_CANCEL["Customer CAN Cancel<br/>(no delivery yet)"]
    
    CAN_CANCEL --> CANCEL_PENDING["Cancel Service"]
    CANCEL_PENDING --> REFUND_PENDING["Refund Pending<br/>Payments"]
    
    NO_REFUND_THEFT --> INSURANCE["Customer Deals with<br/>Insurance Separately"]
```

**Key Rule:** Vehicle theft does **not** automatically trigger special refund logic. Already-delivered services follow normal desistimiento rules. Customers can cancel pending services and future subscriptions.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:91-96]()

---

## Payment Failure Handling

### OTA Activation Failure → No Charge

The most critical payment protection rule in the CaaS system is: **if OTA activation fails, the customer is NOT charged**.

```mermaid
sequenceDiagram
    participant CUSTOMER as Customer
    participant SERVICE_PROC as Service Payment Processor
    participant OTA as OTA Activation Engine
    participant IOT as IoT Gateway
    participant GATEWAY as Payment Gateway
    
    CUSTOMER->>SERVICE_PROC: Purchase service
    SERVICE_PROC->>GATEWAY: Initiate payment authorization
    GATEWAY-->>SERVICE_PROC: Authorization OK
    
    Note over SERVICE_PROC: Payment authorized<br/>but not captured yet
    
    SERVICE_PROC->>OTA: Trigger activation
    OTA->>IOT: Attempt 1
    IOT-->>OTA: Failed
    OTA->>IOT: Attempt 2
    IOT-->>OTA: Failed
    OTA->>IOT: Attempt N
    IOT-->>OTA: Failed
    
    OTA->>OTA: Retrieve vehicle status
    OTA->>OTA: Send to tech support
    
    OTA->>SERVICE_PROC: Activation FAILED
    SERVICE_PROC->>GATEWAY: VOID authorization<br/>DO NOT CAPTURE
    SERVICE_PROC->>CUSTOMER: Notify: Service not delivered<br/>NO CHARGE applied
    
    Note over CUSTOMER,GATEWAY: Customer never charged<br/>for undelivered service
```

**Implementation Requirements:**
- Payment **authorization** (hold) occurs immediately
- Payment **capture** only occurs after OTA confirms successful activation
- If activation fails, authorization is **voided** before capture
- If already captured (due to gateway timing), **refund** is issued immediately

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:48-53]()

### Final Payment Failure

```mermaid
graph TD
    READY["Vehicle Ready<br/>at Dealership"]
    NOTIFY["Notify Customer:<br/>Make final payment"]
    
    READY --> NOTIFY
    NOTIFY --> ATTEMPT["Customer Attempts<br/>Final Payment"]
    
    ATTEMPT --> RESULT{Payment<br/>Result?}
    
    RESULT -->|Success| REGISTER["Register Vehicle<br/>(Matriculate)"]
    REGISTER --> SCHEDULE["Schedule Delivery<br/>to Customer"]
    
    RESULT -->|Failure| UNASSIGN["Mark Vehicle<br/>'sin asignar'"]
    UNASSIGN --> STOCK["Convert to<br/>Dealership Stock"]
    STOCK --> IMMEDIATE_SALE["Available for<br/>Immediate Sale"]
    
    UNASSIGN --> LOSE_RESERVATION["Customer Loses<br/>Reservation"]
    LOSE_RESERVATION --> FORFEIT["Forfeit Reservation<br/>Payment"]
```

**Consequences of Final Payment Failure:**

| Action | System State | Customer Impact |
|--------|--------------|-----------------|
| Vehicle Status | Changed to `"sin asignar"` | Loses assigned vehicle |
| Inventory | Becomes stock for immediate sale | Another customer can purchase |
| Reservation Payment | Forfeited (not refunded) | Loses deposit |
| Customer Record | Reservation marked cancelled | Must start new purchase process |

**Business Rationale:** Protects company cash flow and prevents manufactured vehicles from sitting unsold. Harsh but necessary boundary.

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:26-27]()

### Subscription Payment Failure

```mermaid
graph TD
    MONTH_END["Month Ends"]
    CALCULATE["Calculate Charges<br/>for Services Used"]
    
    MONTH_END --> CALCULATE
    CALCULATE --> ATTEMPT["Attempt to Charge<br/>Customer"]
    
    ATTEMPT --> RESULT{Payment<br/>Successful?}
    
    RESULT -->|Yes| CONTINUE["Continue Subscription<br/>into Next Month"]
    
    RESULT -->|No| RETRY["Retry Payment<br/>(configurable attempts)"]
    
    RETRY --> RETRY_RESULT{Retry<br/>Successful?}
    
    RETRY_RESULT -->|Yes| CONTINUE
    RETRY_RESULT -->|No| CANCEL["Cancel Subscription"]
    
    CANCEL --> DEACTIVATE["Deactivate Service<br/>via OTA"]
    CANCEL --> DUNNING["Send to Collections<br/>(for outstanding amount)"]
    CANCEL --> NOTIFY["Notify Customer"]
```

**Collection Process:**
1. **First Attempt:** At month end (mes vencido billing)
2. **Retries:** Configurable retry schedule (e.g., +3 days, +7 days)
3. **Cancellation:** If all retries fail, cancel subscription
4. **Service Deactivation:** Remove service via OTA
5. **Collections:** Outstanding amount sent to dunning process

**Write-off Risk:** Services already consumed but not paid for represent a loss. This is the inherent risk of the mes vencido model.

**Sources:** [pasame las preguntas y sua respuestas a markdown.md:82]()

---

## Financial Records and Invoicing

### Invoice Generation Flow

```mermaid
graph LR
    subgraph "Payment Events"
        RES_PAY["Reservation<br/>Payment"]
        FINAL_PAY["Final<br/>Payment"]
        SERVICE_PAY["Service<br/>Payment"]
        SUB_PAY["Subscription<br/>Billing"]
    end
    
    subgraph "Invoice Generation"
        INVOICE_GEN["Invoice<br/>Generator"]
        TEMPLATE["Invoice<br/>Templates"]
    end
    
    subgraph "Storage and Delivery"
        BILLING_DB[("Billing<br/>Database")]
        EXPEDIENTE["Expediente<br/>de Compra"]
        EMAIL["Email<br/>Service"]
    end
    
    RES_PAY --> INVOICE_GEN
    FINAL_PAY --> INVOICE_GEN
    SERVICE_PAY --> INVOICE_GEN
    SUB_PAY --> INVOICE_GEN
    
    INVOICE_GEN --> TEMPLATE
    TEMPLATE --> INVOICE_GEN
    
    INVOICE_GEN --> BILLING_DB
    INVOICE_GEN --> EXPEDIENTE
    INVOICE_GEN --> EMAIL
    
    EXPEDIENTE --> CUSTOMER["Customer Access<br/>(Web/Mobile)"]
```

**Invoice Types:**

| Invoice Type | Trigger | Contents | Storage |
|-------------|---------|----------|---------|
| **Reservation Invoice** | Reservation payment successful | Deposit amount, vehicle details, order number | Expediente + Billing DB |
| **Final Payment Invoice** | Final payment successful | Remaining balance, vehicle VIN, registration details | Expediente + Billing DB |
| **Service Invoice** | Service payment successful + OTA confirmed | Service description, duration, price | Expediente + Billing DB |
| **Monthly Subscription Invoice** | End of month billing | List of services used, usage days, total charge | Expediente + Billing DB |
| **Refund Credit Note** | Desistimiento refund processed | Original invoice reference, refund amount, reason | Expediente + Billing DB |

All invoices are stored in the customer's **Expediente de Compra** and accessible via web/mobile platforms.

**Sources:** [enunciado.md:11]()

---

## Summary

The CaaS payment and billing architecture implements a **customer-first financial strategy** that prioritizes user experience while carefully managing financial risk:

**Key Characteristics:**

1. **Risk Assumption:** Delivers services immediately despite asynchronous payment settlement
2. **Customer Protection:** Never charges for failed OTA activations
3. **Legal Compliance:** Implements desistimiento (withdrawal) rules per EU distance selling regulations
4. **Flexible Billing:** Supports multiple models (one-time, temporary, permanent, subscription)
5. **Post-paid Option:** Mes vencido billing for subscriptions reduces adoption friction
6. **Strict Boundaries:** Final payment failure results in immediate loss of vehicle reservation

**Financial Risk Profile:**

| Risk Area | Exposure | Mitigation |
|-----------|----------|------------|
| OTA Failure | Medium | Void/refund if activation fails |
| Service Settlement Delay | Low-Medium | Monitor settlement, write off rare failures |
| Subscription Non-payment | High | Dunning process, service deactivation |
| Final Payment Failure | None | Blocking operation, vehicle returns to stock |

This architecture balances **business agility** (fast service delivery) with **customer protection** (no charge for failed delivery) and **financial prudence** (strict controls on high-value transactions).

**Sources:** [pasame las preguntas y sus respuestas a markdown.md:75-96](), [enunciado.md:1-23]()