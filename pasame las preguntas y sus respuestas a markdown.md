# **Recopilación de Preguntas y Respuestas \- Práctica CaaS**

**Asignatura:** Práctica entregable de la convocatoria ordinaria

**Profesor:** Antonio Pantoja Valero

Este documento resume las dudas planteadas en el foro y las respuestas proporcionadas por el profesor, organizadas por temáticas.

## **1\. Definición de Actores y Sistema**

P: ¿Un elemento puede ser un actor para un caso y para otro distinto pertenecer a nuestro sistema?  
R: Es imposible. Un actor es siempre algo externo a nosotros (el sistema de información) a quien damos servicio. Dentro del sistema los componentes colaboran, pero el actor siempre es externo.  
P: ¿El vehículo actúa como un actor que recibe y envía información?  
R: (Aclaración del profesor): No entiende bien la referencia a "actor" en este contexto. Los vehículos están preparados para recibir configuraciones y devolver estado, pero son parte de la infraestructura con la que nos comunicamos, no usuarios externos en el sentido estricto de actores humanos que inician casos de uso primarios, aunque interactúan.  
P: ¿Nuestro sistema puede recibir inicios de sesión a través de cuentas externas (Google, Microsoft, etc.)? ¿Se gestiona con un SSO externo?  
R: NO estamos haciendo un sistema público. Todos los usuarios deben ser conocidos y estar controlados (clientes que han comprado coche, empleados, etc.). En nuestro sistema no tiene cabida el auto registro de usuarios.

## **2\. Fabricación, Transporte y Entrega**

P: ¿Quién se encarga de transportar el vehículo desde la fábrica hasta la localidad?  
R: Se subcontrata a una empresa de transporte. Lo importante es registrar cuándo se embarca (proveedor/fábrica) y cuándo se recibe (concesionario).  
P: ¿La fecha de entrega la planifica la fábrica, el concesionario o el proveedor externo?  
R: La fecha de entrega del vehículo siempre la determina la fábrica en función de su planificación.  
P: ¿Qué pasa si el vehículo se entrega a domicilio y el usuario no está presente?  
R: La grúa llama al cliente; si no puede recogerlo, se devuelve al concesionario para que el cliente vaya a buscarlo. Nunca se deja en la calle por riesgos.  
P: ¿Qué ocurre si al ir a recoger el coche el cliente no tiene el dinero para el pago final?  
R: El vehículo se lo queda el concesionario y pasa a ser stock para venta inmediata. El cliente pierde la reserva. En el sistema, el vehículo existe pero queda "sin asignar" a ningún cliente.  
P: ¿El sistema debe enviar notificaciones automáticas del estado del pedido/fabricación?  
R: Sí, las notificaciones automáticas sobre el estado de fabricación forman parte del valor diferencial que se quiere proporcionar.

## **3\. Funcionamiento Técnico (API, IoT y OTA)**

P: ¿Nosotros activamos las prestaciones o damos la instrucción al fabricante? ¿Es necesario un TSP (Telematics Service Provider)?  
R: La fábrica y CaaS son la misma empresa. Los vehículos ya se comunican con los servidores (unidireccional inicialmente para posición).

* Existe una **API IoT** documentada y probada para configuración remota.  
* Está preparada para CaaS, aunque no en uso hasta que finalice el proyecto.  
* Cualquier problema de integración se carga al proyecto previo.

P: Detalles sobre las APIs con fábrica:  
R:

1. **CaaS \-\> Fábrica:** Síncrono. Iniciativa de CaaS (gestión pedidos).  
2. **Fábrica \-\> CaaS:** Asíncrono. Notificaciones de estado de fabricación.

P: ¿Los servicios opcionales se activan siempre por OTA (Over The Air) o requieren taller?  
R: Los servicios vendidos por CaaS siempre se distribuyen mediante OTA. Nunca es necesario llevar el vehículo al taller para esto.  
P: ¿Qué pasa si falla la activación OTA?  
R:

1. Reintentar un número determinado de veces.  
2. Si falla, obtener estado del vehículo proactivamente y enviar info al servicio técnico.  
3. Comunicar al cliente y **no cobrar** el servicio no suministrado.

P: ¿Las mejoras compradas son actualizaciones de software o desbloqueos?  
R: Toda la funcionalidad ya está preinstalada en el firmware. CaaS solo realiza la activación o desactivación. Las actualizaciones de software del coche son un proceso independiente a las ventas de CaaS.

## **4\. Mantenimiento y Garantía**

P: ¿Quién maneja el mantenimiento?  
R: Lo realiza la red de talleres oficiales o talleres multimarca homologados.

* El control se lleva mediante el sistema **VSS** (Vehicle Service System), que tiene el histórico y porcentaje de cumplimiento por bloques funcionales.  
* El cliente decide cuándo llevarlo (no podemos obligarle).

P: ¿Si no hay mantenimiento, se bloquea el coche?  
R: Nunca se puede bloquear el vehículo para evitar que circule (solo la policía puede).

* Si falta mantenimiento, se **pierde la garantía**.  
* Solo se puede evitar que se active el servicio específico que dependa de ese mantenimiento (por seguridad), pero no se bloquean servicios ya pagados ni la plataforma base.

P: ¿El estado de mantenimiento lo comunica el propio coche por IoT?  
R: No. (Se deduce que se consulta al sistema VSS de talleres).

## **5\. Pagos, Facturación y Cancelaciones**

P: ¿Hay una entidad que compruebe los pagos al banco?  
R:

* **Compra del coche:** Se paga una reserva y el resto al final antes de matricular.  
* **Servicios CaaS:** El pago (tarjeta) puede tener retraso. Se asume el riesgo y se entrega el servicio antes de la liquidación total.  
* **Suscripciones:** Cobro a **mes vencido**.

P: ¿Se puede cancelar un servicio antes de tiempo?  
R: Sí, por la ley de venta a distancia (desistimiento).

* Si dura \> 14 días: Se puede devolver en los primeros 14 días.  
* Si dura \< 14 días: Se puede devolver en cualquier momento.  
* Política posible: Si se devuelve por insatisfacción, desaparece del catálogo para ese cliente (configurable).

P: ¿Qué pasa si roban el vehículo?  
R:

* No hay relación directa entre robo y servicios ya entregados.  
* El cliente puede anular servicios pendientes o cancelar la renovación automática de suscripciones.  
* No se devuelven importes de servicios ya suministrados si pasaron los 14 días de desistimiento.

## **6\. Documentación y Especificación**

P: ¿Cómo documentar las APIs en los requisitos?  
R: Poner un breve resumen de lo que hace y referencia al documento técnico. Diferenciar entre:

* **APIs existentes (Restricciones):** Ya existen y se deben usar.  
* **APIs a desarrollar:** Forman parte del proyecto y tienen requisitos.