Based on the CaaS system documentation, here's a detailed breakdown of the functional and non-functional requirements:

## Functional Requirements

### 1. **User Management and Access**
- The system must NOT allow public registration; all users must be known and controlled (customers who have purchased vehicles, employees, etc.) [1](#0-0) 
- The system must provide different access channels: Corporate Intranet for dealership employees, Web Platform for customers, and Mobile Application for customers

### 2. **Vehicle Sales and Order Management**
- The system must allow dealership staff to register vehicle sales through the corporate intranet
- The system must automatically place factory orders synchronously when a reservation is made [2](#0-1) 
- The system must receive asynchronous manufacturing status notifications from the factory [3](#0-2) 
- The system must send automatic notifications about the manufacturing status as part of its differentiating value [4](#0-3) 

### 3. **Payment Processing**
- The system must process reservation payments (señal) and final payments before vehicle registration [5](#0-4) 
- The system must handle subscription billing using a "mes vencido" (post-paid) model [6](#0-5) 
- The system must assume payment risk and deliver services before complete payment settlement [7](#0-6) 

### 4. **OTA Service Activation**
- The system must activate/deactivate optional services remotely via OTA (Over-The-Air) without requiring workshop visits [8](#0-7) 
- All functionality is pre-installed in vehicle firmware; the system only activates or deactivates features [9](#0-8) 
- When OTA activation fails, the system must retry a determined number of times [10](#0-9) 
- If activation fails after all retries, the system must proactively obtain vehicle status, send information to technical support, notify the customer, and NOT charge for the service [11](#0-10) 

### 5. **Maintenance Verification**
- The system must query the VSS (Vehicle Service System) to check maintenance status before allowing certain service activations [12](#0-11) 
- The system must block activation of specific services that depend on required maintenance for safety reasons, but cannot block already paid services or the base platform [13](#0-12) 
- The system must NEVER block the vehicle from operating, as only police have that authority [14](#0-13) 

### 6. **Service Cancellation (Desistimiento)**
- The system must implement distance selling law cancellation rights: services lasting >14 days can be returned within the first 14 days; services lasting <14 days can be returned anytime [15](#0-14) 
- The system must process refunds for services canceled within the legal period
- The system must allow customers to cancel pending services or automatic subscription renewals [16](#0-15) 

### 7. **Vehicle Registration**
- The system must register vehicles with administrative bodies (government registration) via Admin API before delivery

### 8. **Delivery Management**
- If home delivery fails due to customer absence, the system must register the vehicle's return to the dealership for customer pickup [17](#0-16) 
- If final payment fails, the system must mark the vehicle as dealership stock ("sin asignar") and the customer loses the reservation [18](#0-17) 

### 9. **Customer Data Repository (Expediente de Compra)**
- The system must maintain a centralized purchase record containing manuals, history, and invoices accessible across all customer channels

## Non-Functional Requirements

### 1. **Integration Constraints**
- The API IoT is pre-existing, documented, and tested, and CANNOT be modified by CaaS; any integration issues are attributed to the previous project [19](#0-18) 
- The VSS system is external, maintained by workshops, and CaaS has read-only access [20](#0-19) 

### 2. **Security and Access Control**
- No external SSO integration (Google, Microsoft, etc.); the system is not public and requires controlled user access [1](#0-0) 
- All users must be known and pre-provisioned (customers receive credentials via email after dealership registration)

### 3. **Reliability and Fault Tolerance**
- The system must implement retry logic with proactive status monitoring for failed OTA activations [21](#0-20) 
- The system must handle asynchronous webhook notifications from factory and payment gateway

### 4. **Legal Compliance**
- The system must comply with EU distance selling regulations (derecho de desistimiento) for service cancellations [22](#0-21) 
- The system must never prevent vehicle operation, even for maintenance non-compliance [14](#0-13) 

### 5. **Customer Protection**
- The system must never charge customers for services that fail to activate after all retry attempts [23](#0-22) 
- The system must assume financial risk by delivering services before payment settlement confirmation [7](#0-6) 

### 6. **Communication Architecture**
- The system must support synchronous API calls for order placement with the factory [24](#0-23) 
- The system must support asynchronous webhook notifications for manufacturing status updates [3](#0-2) 

### 7. **Multi-Channel Notification**
- The system must provide automatic notifications across multiple channels (email, mobile app, vehicle display) for manufacturing status and service activation events [4](#0-3) 

### 8. **Scalability**
- The system must handle bidirectional communication with the entire vehicle fleet via IoT network
- The system must support concurrent payment processing for reservations, services, and subscriptions

## Notes

The CaaS system is designed around a "Platform Base + Optional Services" model where customers purchase a base vehicle platform and can dynamically activate optional features throughout the vehicle's lifecycle. The architecture emphasizes customer protection (no charges for failed activations), legal compliance (14-day cancellation rights), and operational flexibility (risk-tolerant service delivery before payment settlement).

The system operates within significant technical constraints, particularly the pre-existing API IoT that cannot be modified, and the external VSS system that provides read-only maintenance data. These constraints directly influence the system's non-functional requirements around integration patterns and data access models.

### Citations

**File:** pasame las preguntas y sus respuestas a markdown.md (L15-16)
```markdown
P: ¿Nuestro sistema puede recibir inicios de sesión a través de cuentas externas (Google, Microsoft, etc.)? ¿Se gestiona con un SSO externo?  
R: NO estamos haciendo un sistema público. Todos los usuarios deben ser conocidos y estar controlados (clientes que han comprado coche, empleados, etc.). En nuestro sistema no tiene cabida el auto registro de usuarios.
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L24-25)
```markdown
P: ¿Qué pasa si el vehículo se entrega a domicilio y el usuario no está presente?  
R: La grúa llama al cliente; si no puede recogerlo, se devuelve al concesionario para que el cliente vaya a buscarlo. Nunca se deja en la calle por riesgos.  
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L26-27)
```markdown
P: ¿Qué ocurre si al ir a recoger el coche el cliente no tiene el dinero para el pago final?  
R: El vehículo se lo queda el concesionario y pasa a ser stock para venta inmediata. El cliente pierde la reserva. En el sistema, el vehículo existe pero queda "sin asignar" a ningún cliente.  
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L28-29)
```markdown
P: ¿El sistema debe enviar notificaciones automáticas del estado del pedido/fabricación?  
R: Sí, las notificaciones automáticas sobre el estado de fabricación forman parte del valor diferencial que se quiere proporcionar.
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L36-38)
```markdown
* Existe una **API IoT** documentada y probada para configuración remota.  
* Está preparada para CaaS, aunque no en uso hasta que finalice el proyecto.  
* Cualquier problema de integración se carga al proyecto previo.
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L43-44)
```markdown
1. **CaaS \-\> Fábrica:** Síncrono. Iniciativa de CaaS (gestión pedidos).  
2. **Fábrica \-\> CaaS:** Asíncrono. Notificaciones de estado de fabricación.
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L46-47)
```markdown
P: ¿Los servicios opcionales se activan siempre por OTA (Over The Air) o requieren taller?  
R: Los servicios vendidos por CaaS siempre se distribuyen mediante OTA. Nunca es necesario llevar el vehículo al taller para esto.  
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L51-53)
```markdown
1. Reintentar un número determinado de veces.  
2. Si falla, obtener estado del vehículo proactivamente y enviar info al servicio técnico.  
3. Comunicar al cliente y **no cobrar** el servicio no suministrado.
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L55-56)
```markdown
P: ¿Las mejoras compradas son actualizaciones de software o desbloqueos?  
R: Toda la funcionalidad ya está preinstalada en el firmware. CaaS solo realiza la activación o desactivación. Las actualizaciones de software del coche son un proceso independiente a las ventas de CaaS.
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L62-64)
```markdown

* El control se lleva mediante el sistema **VSS** (Vehicle Service System), que tiene el histórico y porcentaje de cumplimiento por bloques funcionales.  
* El cliente decide cuándo llevarlo (no podemos obligarle).
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L66-67)
```markdown
P: ¿Si no hay mantenimiento, se bloquea el coche?  
R: Nunca se puede bloquear el vehículo para evitar que circule (solo la policía puede).
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L69-70)
```markdown
* Si falta mantenimiento, se **pierde la garantía**.  
* Solo se puede evitar que se active el servicio específico que dependa de ese mantenimiento (por seguridad), pero no se bloquean servicios ya pagados ni la plataforma base.
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L72-73)
```markdown
P: ¿El estado de mantenimiento lo comunica el propio coche por IoT?  
R: No. (Se deduce que se consulta al sistema VSS de talleres).
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L80-81)
```markdown
* **Compra del coche:** Se paga una reserva y el resto al final antes de matricular.  
* **Servicios CaaS:** El pago (tarjeta) puede tener retraso. Se asume el riesgo y se entrega el servicio antes de la liquidación total.  
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L82-82)
```markdown
* **Suscripciones:** Cobro a **mes vencido**.
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L85-89)
```markdown
R: Sí, por la ley de venta a distancia (desistimiento).

* Si dura \> 14 días: Se puede devolver en los primeros 14 días.  
* Si dura \< 14 días: Se puede devolver en cualquier momento.  
* Política posible: Si se devuelve por insatisfacción, desaparece del catálogo para ese cliente (configurable).
```

**File:** pasame las preguntas y sus respuestas a markdown.md (L95-95)
```markdown
* El cliente puede anular servicios pendientes o cancelar la renovación automática de suscripciones.  
```
