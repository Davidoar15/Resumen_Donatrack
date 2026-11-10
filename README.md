# DonaTrack — resumen del sistema

DonaTrack coordina donaciones, donadores, entidades benéficas, necesidades, depósitos, entregas e incentivos. Los módulos mantienen sus propios datos y reglas; el API Gateway concentra el acceso externo; Servicio de Consultas materializa un read model CQRS en MongoDB para reportes integrados.

## Componentes y responsabilidades

| Componente | Responsabilidad |
|---|---|
| **API Gateway** | Entrada HTTP única `/api/...`, enrutamiento a los módulos, propagación de `X-Trace-Id` y CORS. |
| **Donaciones** | Catálogos de categorías/productos/identificadores, registro y estados de donación, historial y quejas. |
| **Donadores y Entidades** | Alta y estado de donadores, entidades, necesidades, períodos y validación `puede-donar`. |
| **Logística API** | Depósitos, algoritmo, paquetes/stock, asignaciones y confirmación de entregas. |
| **Workers 1 y 2** | Consumidores RabbitMQ independientes para matchmaking asíncrono; ejecutan el procesamiento distribuido sin exponer API de escritura ni persistencia propia. |
| **Incentivos** | Misiones, insignias, asignaciones y procesamiento de progreso/categorías. |
| **Servicio de Consultas** | Consume eventos versionados y materializa snapshots `rm_*` + agregados `q_*` en MongoDB. |
| **MCP Server** | Expone herramientas de DonaTrack a clientes MCP usando el Gateway. |
| **Telegram Bot** | Interfaz conversacional que consume el mismo catálogo `tools.json` y ejecuta HTTP al Gateway; es un cliente separado de MCP Server. |
| **Testing** | Colecciones, pruebas, smoke checks y configuración existente de observabilidad. |

## Bases técnicas y operativas

- Java 21/Spring Boot en los servicios de dominio, Gateway y Consultas; workers Logística en proceso/servicio separado.
- PostgreSQL en módulos que requieren persistencia relacional; MongoDB Atlas en Consultas como modelo de lectura; RabbitMQ/CloudAMQP para eventos y procesamiento asíncrono.
- El cliente externo debe usar el Gateway. Las URLs de cada servicio se reservan para health checks, administración y diagnóstico controlado.
- El flujo de lectura usa exchange `donatrack.readmodel.exchange`, routing key `readmodel.changed` y queue `donatrack.readmodel.v1.events`. Contrato: `eventId`, `source`, `resourceType`, `resourceId`, `operation`, `occurredAt`, `payload`, `version` positivo y `schemaVersion=1`.
- Los publicadores persisten primero un outbox en la misma transacción que su cambio de dominio; un sender publica con confirmación del broker. El consumidor aplica snapshots idempotentes por `eventId` y versión.
- Logística implementa outbox y endpoint protegido de republicación histórica; esta corrección permite recuperar depósitos ya persistidos que no llegaron al modelo de lectura.
- MCP y Telegram no son origen de reglas de negocio: ambos consumen el catálogo compartido `tools.json` y llaman al API Gateway de forma independiente.

## Flujos

Cada documento en [`flujos/`](flujos/) describe activación, secuencia, rutas, datos y condiciones de éxito. En solicitudes del Gateway, el prefijo `/api/{componente}` se elimina antes de llegar al componente. Los paths de servicio están indicados sin ese prefijo.

| Documento | Flujo |
|---|---|
| [01](flujos/01_registro_donacion.md) | Registrar donación y preparar matchmaking. |
| [02](flujos/02_matchmaking_asincronico.md) | Procesamiento asíncrono en Workers. |
| [03](flujos/03_registrar_entrega.md) | Registrar entrega y actualizar necesidad/estado. |
| [04](flujos/04_queja_e_incentivos.md) | Queja, notificación y consecuencias en incentivos. |
| [05](flujos/05_alta_necesidad.md) | Registrar necesidad y publicación para asignación. |
| [06](flujos/06_necesidad_telegram.md) | Modificar necesidad desde Telegram (con MCP como cliente independiente del mismo catálogo). |
| [07](flujos/07_procesamiento_misiones.md) | Procesamiento de misiones e insignias. |
| [08](flujos/08_consultas_integradas.md) | Consultas agregadas y reportes. |
| [09](flujos/09_reparacion_depositos_readmodel.md) | Recuperar snapshots de depósitos y reconstruir reportes. |

## URLs Render de referencia

| Servicio | URL base |
|---|---|
| Gateway | `https://api-gateway-donatrack.onrender.com` |
| Consultas | `https://donatrack-servicio-consultas.onrender.com` |
| Logística API | `https://donatrack-api-logistica-diseno-en-beta.onrender.com` |
| Worker 1 | `https://donatrack-logistica-worker-1.onrender.com` |
| Worker 2 | `https://donatrack-logistica-worker-2.onrender.com` |
| Donaciones | `https://donaciones-xouj.onrender.com` |
| Donadores y Entidades | `https://entrega-2-lolasimone.onrender.com` |
| Incentivos | `https://entrega-2-melinazingales.onrender.com` |

Las URLs son puntos de configuración observados en los artefactos compartidos; confirmar en el dashboard de Render que sigan vigentes antes de una demostración.
