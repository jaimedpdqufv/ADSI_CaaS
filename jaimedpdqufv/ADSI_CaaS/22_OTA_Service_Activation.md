# OTA Service Activation

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [enunciado.md](enunciado.md)
- [pasame las preguntas y sus respuestas a markdown.md](pasame las preguntas y sus respuestas a markdown.md)

</details>



## Purpose and Scope

This page documents the Over-The-Air (OTA) service activation mechanism used to remotely enable optional services on customer vehicles after payment. OTA activation is the technical delivery mechanism for all services sold through the CaaS platform.

**Scope of this document:**
- Pre-installed functionality model and activation mechanics
- Integration with the existing API IoT infrastructure
- Activation workflow from payment to service delivery
- Retry logic and failure handling mechanisms
- The critical "do not charge" rule for failed activations
- Technical support escalation procedures

**Related pages:**
- For service catalog and pricing models, see [Service Catalog and Pricing](#6.1)
- For maintenance requirements that may block activation, see [Maintenance-Linked Service Access](#6.3)
- For payment processing before activation, see [Payment Types and Flows](#7.1)
- For the broader IoT infrastructure, see [Vehicle Communication (IoT and OTA)](#5.3)

Sources: [enunciado.md:1-23](), [pasame las preguntas y sus respuestas a markdown.md:31-56]()

---

## Pre-Installed Functionality Model

All vehicles sold through CaaS are delivered with **complete firmware pre-installed** containing all optional features in a dormant state. The OTA activation process does not install new software or update the vehicle's firmware; it merely **enables or disables** existing functionality through configuration changes.

```mermaid
graph TB
    subgraph "Vehicle Firmware"
        BASE["Plataforma Base<br/>(Always Active)"]
        OPT1["Optional Feature 1<br/>(Dormant)"]
        OPT2["Optional Feature 2<br/>(Dormant)"]
        OPT3["Optional Feature 3<br/>(Dormant)"]
        OPTN["Optional Feature N<br/>(Dormant)"]
    end
    
    subgraph "OTA Activation"
        CONFIG["Configuration Command<br/>via API IoT"]
    end
    
    CONFIG -->|"Enable Feature 1"| OPT1
    CONFIG -->|"Enable Feature 2"| OPT2
    OPT1 -.->|"State: Active"| OPT1
    OPT2 -.->|"State: Active"| OPT2
    
    BASE -.->|"Always Enabled"| BASE
    OPT3 -.->|"Remains Dormant"| OPT3
    OPTN -.->|"Remains Dormant"| OPTN
```

**Key characteristics:**

| Aspect | Description |
|--------|-------------|
| **Software Updates** | Independent process, not managed by CaaS sales system |
| **Activation Mechanism** | Configuration flag changes sent via API IoT |
| **Hardware Requirements** | All hardware for optional features already present in vehicle |
| **Activation Speed** | Near-instantaneous once configuration received by vehicle |
| **Deactivation** | Services can be disabled by sending deactivation commands |
| **Workshop Visit** | Never required for CaaS-managed optional services |

This architecture enables:
- Fast service delivery (no software installation required)
- Reliable activation (no risk of failed downloads or corrupted installs)
- Reversibility (features can be disabled if subscription ends)
- Offline capability (vehicle doesn't need internet to use activated features, only to receive activation)

Sources: [pasame las preguntas y sus respuestas a markdown.md:55-56](), [enunciado.md:3-4]()

---

## OTA Activation Process Flow

The following sequence diagram illustrates the end-to-end flow from payment to service activation, including success and failure paths.

### OTA Activation Sequence Diagram

```mermaid
sequenceDiagram
    participant Customer
    participant WebApp as "Web/Mobile<br/>Platform"
    participant PaymentSvc as "Payment<br/>Service"
    participant OTAEngine as "OTA Activation<br/>Engine"
    participant APIIoT as "API IoT<br/>(Existing)"
    participant IoTNetwork as "IoT Network"
    participant Vehicle
    participant NotificationSvc as "Notification<br/>Service"
    participant TechSupport as "Technical<br/>Support"
    
    Customer->>WebApp: "Select service & duration"
    WebApp->>PaymentSvc: "Initiate payment"
    PaymentSvc->>Customer: "Payment confirmed"
    
    Note over PaymentSvc,OTAEngine: Payment successful,<br/>trigger activation
    
    PaymentSvc->>OTAEngine: "Activate service"
    OTAEngine->>OTAEngine: "Queue activation request"
    
    rect rgb(240, 240, 240)
        Note over OTAEngine,Vehicle: Activation Attempt Loop
        
        OTAEngine->>APIIoT: "POST /vehicle/{vin}/configure"
        APIIoT->>IoTNetwork: "Push configuration"
        IoTNetwork->>Vehicle: "Apply configuration"
        
        alt Vehicle Successfully Configured
            Vehicle->>IoTNetwork: "Configuration applied"
            IoTNetwork->>APIIoT: "Success status"
            APIIoT->>OTAEngine: "200 OK + confirmation"
            OTAEngine->>NotificationSvc: "Service activated"
            NotificationSvc->>Customer: "Service active notification"
        else Vehicle Communication Failed
            IoTNetwork-->>APIIoT: "Timeout / Error"
            APIIoT-->>OTAEngine: "4xx/5xx Error"
            OTAEngine->>OTAEngine: "Increment retry count"
            
            alt Retries Remaining
                Note over OTAEngine: Wait backoff period,<br/>retry activation
            else Max Retries Exceeded
                OTAEngine->>APIIoT: "GET /vehicle/{vin}/status"
                APIIoT->>OTAEngine: "Vehicle telemetry"
                OTAEngine->>TechSupport: "Escalate with diagnostics"
                OTAEngine->>PaymentSvc: "DO NOT CHARGE customer"
                OTAEngine->>NotificationSvc: "Activation failed"
                NotificationSvc->>Customer: "Issue notification + no charge"
            end
        end
    end
```

**Process steps:**

1. **Payment Confirmation**: Customer pays, payment service confirms success
2. **Activation Queuing**: OTA Engine receives activation request and queues it
3. **Configuration Push**: OTA Engine calls API IoT to push configuration to vehicle
4. **Vehicle Application**: IoT Network delivers configuration to physical vehicle
5. **Confirmation Loop**: System waits for vehicle to confirm configuration applied
6. **Success Path**: Notify customer and mark service as active
7. **Failure Path**: Retry with backoff, escalate to tech support if all retries fail
8. **Customer Protection**: Ensure customer is not charged if activation fails

Sources: [pasame las preguntas y sus respuestas a markdown.md:46-53](), [enunciado.md:18-20]()

---

## API IoT Integration

The CaaS platform integrates with a **pre-existing, documented, and tested** API IoT infrastructure. This API is a constraint of the system—it already exists and cannot be modified by the CaaS project.

### API IoT Architecture

```mermaid
graph TB
    subgraph "CaaS Platform"
        OTAEngine["OTA Activation Engine"]
        ConfigRepo[("Configuration<br/>Repository")]
        ServiceDB[("Service<br/>Database")]
    end
    
    subgraph "API IoT (Pre-Existing System)"
        APIEndpoint["API IoT REST Interface"]
        AuthLayer["Authentication Layer"]
        CommandQueue["Command Queue"]
        TelemetryReceiver["Telemetry Receiver"]
        StateCache[("Vehicle State Cache")]
    end
    
    subgraph "IoT Network Infrastructure"
        MessageBroker["Message Broker<br/>(e.g., MQTT)"]
        ConnMgr["Connection Manager"]
    end
    
    subgraph "Vehicle Fleet"
        V1["Vehicle 1<br/>ECU"]
        V2["Vehicle 2<br/>ECU"]
        VN["Vehicle N<br/>ECU"]
    end
    
    OTAEngine -->|"POST /vehicle/{vin}/configure"| APIEndpoint
    OTAEngine -->|"GET /vehicle/{vin}/status"| APIEndpoint
    OTAEngine -.->|"Read config templates"| ConfigRepo
    OTAEngine -.->|"Log activation attempts"| ServiceDB
    
    APIEndpoint --> AuthLayer
    AuthLayer --> CommandQueue
    AuthLayer --> StateCache
    
    CommandQueue --> MessageBroker
    TelemetryReceiver --> StateCache
    
    MessageBroker <--> ConnMgr
    ConnMgr <-->|"Bidirectional<br/>Vehicle Link"| V1
    ConnMgr <-->|"Bidirectional<br/>Vehicle Link"| V2
    ConnMgr <-->|"Bidirectional<br/>Vehicle Link"| VN
    
    V1 -->|"Status & telemetry"| TelemetryReceiver
    V2 -->|"Status & telemetry"| TelemetryReceiver
    VN -->|"Status & telemetry"| TelemetryReceiver
```

### Key API IoT Endpoints

| Endpoint | Method | Purpose | CaaS Usage |
|----------|--------|---------|------------|
| `/vehicle/{vin}/configure` | POST | Send configuration command to vehicle | Activate/deactivate services |
| `/vehicle/{vin}/status` | GET | Query current vehicle state and telemetry | Pre-activation checks, failure diagnostics |
| `/vehicle/{vin}/health` | GET | Query vehicle connectivity and health | Determine if vehicle is reachable |
| `/vehicle/{vin}/history` | GET | Retrieve configuration history | Audit trail, troubleshooting |

**Integration characteristics:**

- **Synchronicity**: API calls are synchronous (request-response), but vehicle communication is asynchronous
- **Response Model**: API IoT returns immediate acceptance (202 Accepted) and requires polling or callback for final status
- **Authentication**: OAuth 2.0 client credentials flow for CaaS service account
- **Rate Limiting**: Maximum 10 configuration commands per vehicle per hour (prevent flooding)
- **Timeout**: Configuration commands timeout after 5 minutes if vehicle doesn't respond
- **Idempotency**: Configuration commands are idempotent (safe to retry)

**Responsibility boundaries:**

| Responsibility | API IoT | CaaS OTA Engine |
|----------------|---------|-----------------|
| **Message routing to vehicle** | ✓ | |
| **Vehicle connectivity management** | ✓ | |
| **Configuration persistence on vehicle** | ✓ | |
| **Retry logic for failed activation** | | ✓ |
| **Business rule enforcement** | | ✓ |
| **Customer notification** | | ✓ |
| **Payment reversal on failure** | | ✓ |

Sources: [pasame las preguntas y sus respuestas a markdown.md:33-38]()

---

## Retry and Failure Handling

The OTA Activation Engine implements **robust retry logic with exponential backoff** to handle transient failures in vehicle communication. The system distinguishes between recoverable errors (retry) and terminal failures (escalate).

### OTA Activation State Machine

```mermaid
stateDiagram-v2
    [*] --> PaymentConfirmed: "Payment successful"
    
    PaymentConfirmed --> QueuedForActivation: "Queue activation request"
    
    QueuedForActivation --> AttemptingActivation: "Start activation"
    
    state AttemptingActivation {
        [*] --> SendingCommand
        SendingCommand --> WaitingResponse: "API IoT call sent"
        WaitingResponse --> CheckingResponse: "Response received"
        
        CheckingResponse --> Success: "200 OK + vehicle confirmed"
        CheckingResponse --> Failure: "Error response or timeout"
        
        Failure --> EvaluatingRetry: "Check retry count"
        EvaluatingRetry --> CalculateBackoff: "Retries < max"
        CalculateBackoff --> WaitingBackoff: "Delay = 2^retry * base_delay"
        WaitingBackoff --> SendingCommand: "Backoff complete"
        
        EvaluatingRetry --> MaxRetriesExceeded: "Retries >= max"
    }
    
    Success --> NotifyingCustomer: "Mark service active"
    NotifyingCustomer --> Active: "Send success notification"
    Active --> [*]
    
    MaxRetriesExceeded --> QueryingVehicleState: "GET vehicle status"
    QueryingVehicleState --> EscalatingToSupport: "Collect diagnostics"
    EscalatingToSupport --> ReversingPayment: "Create support ticket"
    ReversingPayment --> NotifyingFailure: "DO NOT CHARGE"
    NotifyingFailure --> Failed: "Notify customer of issue"
    Failed --> [*]
    
    note right of ReversingPayment
        Critical Business Rule:
        Customer NEVER pays for
        undelivered service
    end note
```

### Retry Configuration

```mermaid
graph LR
    subgraph "Retry Parameters"
        MaxRetries["Max Retries: N<br/>(configurable, e.g., 5)"]
        BaseDelay["Base Delay: T<br/>(configurable, e.g., 30s)"]
        MaxDelay["Max Delay: M<br/>(configurable, e.g., 15min)"]
    end
    
    subgraph "Backoff Calculation"
        Formula["delay = min(2^retry × T, M)"]
    end
    
    subgraph "Example Timeline"
        A1["Attempt 1: t=0"]
        A2["Attempt 2: t=30s"]
        A3["Attempt 3: t=90s"]
        A4["Attempt 4: t=210s"]
        A5["Attempt 5: t=450s"]
        A6["Attempt 6: t=900s"]
        ESC["Escalate: t=1800s"]
    end
    
    MaxRetries --> Formula
    BaseDelay --> Formula
    MaxDelay --> Formula
    
    Formula --> A1
    A1 --> A2
    A2 --> A3
    A3 --> A4
    A4 --> A5
    A5 --> A6
    A6 --> ESC
```

**Failure categories and handling:**

| Failure Type | Example | Retry? | Action |
|--------------|---------|--------|--------|
| **Transient Network** | IoT network congestion | Yes | Retry with backoff |
| **Vehicle Offline** | Vehicle in garage, no signal | Yes | Retry with backoff |
| **API IoT Error** | 5xx server error | Yes | Retry with backoff |
| **Invalid Configuration** | Malformed config data | No | Immediate escalation, code bug |
| **Vehicle Incompatible** | Service not supported by vehicle model | No | Immediate escalation, data error |
| **Authorization Failure** | Invalid API credentials | No | Immediate escalation, config error |
| **Max Retries Exceeded** | Vehicle persistently unreachable | No | Escalate to tech support |

**Critical rule: DO NOT CHARGE customer**

When activation fails after all retries are exhausted:

1. **Payment Reversal**: If payment was processed, initiate immediate refund
2. **Service Database**: Mark service as "Failed Activation" (not "Active")
3. **Customer Notification**: Send clear explanation that service was NOT delivered
4. **Audit Trail**: Log all retry attempts and final failure reason
5. **Technical Support**: Create ticket with vehicle diagnostics for manual intervention

This rule is **non-negotiable** and protects customer trust. The system must never charge for services it cannot deliver.

Sources: [pasame las preguntas y sus respuestas a markdown.md:48-53]()

---

## Customer Notifications

Customers receive **real-time notifications** about OTA activation status through multiple channels. Notifications are sent at key points in the activation process.

### Notification Flow

```mermaid
graph TD
    subgraph "Activation Events"
        E1["Payment Confirmed"]
        E2["Activation Started"]
        E3["Activation In Progress"]
        E4["Activation Successful"]
        E5["Activation Failed"]
        E6["Retry Attempt"]
    end
    
    subgraph "Notification Service"
        Router["Notification Router"]
        EmailSvc["Email Service"]
        PushSvc["Push Notification<br/>Service"]
        VehicleHMI["Vehicle HMI<br/>via API IoT"]
    end
    
    subgraph "Customer Touchpoints"
        Email["Customer Email"]
        MobileApp["Mobile App"]
        VehicleDisplay["Vehicle Display<br/>Screen"]
    end
    
    E1 --> Router
    E2 --> Router
    E3 --> Router
    E4 --> Router
    E5 --> Router
    E6 -.->|"Optional"| Router
    
    Router --> EmailSvc
    Router --> PushSvc
    Router --> VehicleHMI
    
    EmailSvc --> Email
    PushSvc --> MobileApp
    VehicleHMI --> VehicleDisplay
    
    Email -.->|"Links to"| WebPlatform["Expediente de Compra<br/>Web Platform"]
    MobileApp -.->|"In-app view"| ServiceStatus["Service Status Screen"]
```

### Notification Content by Event

| Event | Email | Push Notification | Vehicle HMI | Timing |
|-------|-------|-------------------|-------------|--------|
| **Payment Confirmed** | Receipt + activation notice | "Service purchased" | None | Immediate |
| **Activation Started** | "Activating your service" | "Activating..." | "Receiving configuration" | Within 1 minute |
| **Activation Successful** | "Service now active" | "Service ready to use" | "Feature enabled" | Immediately after confirmation |
| **First Retry** | None (silent) | None (silent) | None | N/A |
| **Activation Failed** | Detailed explanation + no charge notice | "Activation issue - not charged" | None | After all retries exhausted |

**Key notification principles:**

1. **Multi-channel redundancy**: Critical notifications (success, failure) sent to all channels
2. **Customer reassurance**: Failed activation messages emphasize "not charged"
3. **In-vehicle feedback**: Vehicle display shows activation status in real-time
4. **Expediente integration**: All activation events logged in customer's purchase record
5. **Silent retries**: Don't alarm customer with transient failures; notify only on final outcome

**Example notification messages:**

**Success:**
```
Subject: Your [Service Name] is now active

Hello [Customer Name],

Your [Service Name] has been successfully activated on your [Vehicle Model].
You can start using this feature immediately.

Activation details:
- Service: [Service Name]
- Duration: [Duration]
- Cost: [Amount]
- Activated: [Timestamp]

View full details in your expediente de compra:
[Link to Web Platform]

Thank you for choosing CaaS.
```

**Failure:**
```
Subject: Service activation issue - You have NOT been charged

Hello [Customer Name],

We were unable to activate [Service Name] on your [Vehicle Model] due to
a technical issue. We have escalated this to our technical support team.

Important: You have NOT been charged for this service.

Our team will contact you shortly to resolve this issue. If you have any
questions, please contact support at [Contact Info].

Reference number: [Ticket ID]

We apologize for the inconvenience.
```

Sources: [enunciado.md:18-20](), [pasame las preguntas y sus respuestas a markdown.md:48-53]()

---

## Technical Support Escalation

When OTA activation fails after exhausting all retries, the system **automatically escalates** to technical support with comprehensive diagnostics.

### Escalation Workflow

```mermaid
flowchart TD
    MaxRetries["Max Retries Exceeded"]
    
    MaxRetries --> CollectDiagnostics["Collect Diagnostics"]
    
    CollectDiagnostics --> GetVehicleStatus["GET /vehicle/{vin}/status"]
    CollectDiagnostics --> GetNetworkStatus["Query IoT Network<br/>Connection Status"]
    CollectDiagnostics --> GetActivationHistory["Retrieve Activation<br/>Attempt Log"]
    CollectDiagnostics --> GetCustomerInfo["Retrieve Customer<br/>& Vehicle Info"]
    
    GetVehicleStatus --> BuildTicket["Build Support Ticket"]
    GetNetworkStatus --> BuildTicket
    GetActivationHistory --> BuildTicket
    GetCustomerInfo --> BuildTicket
    
    BuildTicket --> TicketFields{{"Ticket Contains:<br/>- VIN<br/>- Customer ID<br/>- Service ID<br/>- All retry timestamps<br/>- Error responses<br/>- Vehicle telemetry<br/>- Network status"}}
    
    TicketFields --> CreateTicket["Create Ticket in<br/>Support System"]
    
    CreateTicket --> Priority{{"Determine Priority"}}
    
    Priority -->|"Customer paid"| HighPriority["HIGH Priority:<br/>Customer waiting"]
    Priority -->|"Pre-delivery activation"| MediumPriority["MEDIUM Priority:<br/>Vehicle not yet delivered"]
    
    HighPriority --> AssignTechnician["Auto-assign to<br/>On-duty Technician"]
    MediumPriority --> AssignTechnician
    
    AssignTechnician --> ReversePayment["Trigger Payment Reversal"]
    ReversePayment --> NotifyCustomer["Send Customer<br/>Failure Notification"]
    
    NotifyCustomer --> TechnicianWorks["Technician<br/>Investigates & Resolves"]
    
    TechnicianWorks --> Resolution{{"Resolution"}}
    
    Resolution -->|"Issue resolved"| ManualActivation["Manual Activation<br/>via API IoT"]
    Resolution -->|"Vehicle hardware issue"| ServiceAppointment["Schedule Service<br/>Appointment"]
    Resolution -->|"CaaS system bug"| BugReport["File Bug Report<br/>for Development"]
    
    ManualActivation --> NotifySuccess["Notify Customer:<br/>Service Now Active"]
    ServiceAppointment --> NotifyWorkshop["Notify Customer:<br/>Workshop Visit Required"]
    BugReport --> NotifyWorkaround["Notify Customer:<br/>Alternative Solution"]
```

### Support Ticket Data Structure

The escalation system creates a comprehensive support ticket containing all diagnostic information:

| Field | Source | Purpose |
|-------|--------|---------|
| **Ticket ID** | Auto-generated | Unique identifier for tracking |
| **VIN** | Service request | Identify specific vehicle |
| **Customer ID** | Service request | Contact information |
| **Customer Email** | Customer database | Notification endpoint |
| **Service ID** | Service catalog | Identify which feature to activate |
| **Service Name** | Service catalog | Human-readable service description |
| **Payment Transaction ID** | Payment service | For refund processing |
| **Payment Amount** | Payment service | For refund processing |
| **Activation Request Time** | OTA Engine | When activation was requested |
| **Retry Attempts** | OTA Engine logs | Array of all retry timestamps and responses |
| **Last Error Code** | API IoT response | Most recent failure reason |
| **Last Error Message** | API IoT response | Detailed error description |
| **Vehicle Status Snapshot** | API IoT | Telemetry at time of failure |
| **Vehicle Online Status** | API IoT | Is vehicle reachable? |
| **Network Signal Strength** | API IoT | Connection quality metrics |
| **Vehicle Firmware Version** | API IoT | For compatibility checking |
| **Configuration Payload** | Config repository | What configuration was attempted |
| **Priority** | Business rules | HIGH if customer already paid |

**Automatic actions triggered on escalation:**

1. **Payment reversal initiated** (if payment was processed)
2. **Customer notification sent** (via email and push)
3. **Service marked as "Pending Support"** in service database
4. **Ticket assigned to on-duty technician**
5. **SLA timer started** (response required within defined timeframe)

**Technician manual intervention options:**

- **Retry activation manually** if issue was transient
- **Modify configuration payload** if format issue detected
- **Schedule vehicle inspection** if hardware problem suspected
- **Escalate to API IoT team** if network infrastructure issue
- **Escalate to development** if CaaS system bug identified
- **Contact customer directly** for additional information

The goal is to **resolve activation issues quickly** while ensuring the customer is never charged for undelivered services.

Sources: [pasame las preguntas y sus respuestas a markdown.md:48-53]()

---

## Summary

OTA Service Activation is a critical component of the CaaS system, responsible for the technical delivery of purchased services to customer vehicles. Key takeaways:

| Aspect | Key Point |
|--------|-----------|
| **Delivery Model** | Pre-installed firmware + configuration activation |
| **Integration** | Existing API IoT infrastructure (constraint) |
| **Reliability** | Retry logic with exponential backoff |
| **Customer Protection** | Never charge for failed activations |
| **Failure Handling** | Automatic escalation to technical support |
| **Transparency** | Real-time notifications across multiple channels |

The system prioritizes **customer trust and satisfaction** by implementing robust failure handling and ensuring that customers are never charged for services that cannot be activated. This "customer-first" approach is embedded in the technical architecture and business logic of the OTA activation process.

Sources: [enunciado.md:1-23](), [pasame las preguntas y sus respuestas a markdown.md:31-56]()