# 09. Recuperar depósitos faltantes en MongoDB

**Activación:** el depósito existe en PostgreSQL/Logística, pero no aparece en `rm_deposito` o `/depositos/reporte` de Consultas.

1. Confirmar escritura: `GET /api/logistica/depositos` y `GET /api/logistica/depositos/{id}`.
2. Verificar `GET /api/consultas/estado` y `GET /api/consultas/recursos/deposito/{id}`.
3. Ejecutar `POST /api/logistica/admin/read-model/republicar` con header `X-Admin-Key: <READ_MODEL_ADMIN_KEY>`. El controlador recorre depósitos y encola snapshots de depósitos, paquetes y asignaciones existentes.
4. Logística escribe primero cada evento en `read_model_outbox` dentro de la transacción; el sender publica usando `donatrack.readmodel.exchange`/`readmodel.changed` y marca enviado sólo después del confirm del broker.
5. Consultas valida `version` y `schemaVersion=1`, persiste en `rm_deposito` y refresca `q_depositos`. El evento se procesa idempotentemente por `eventId`.
6. Verificar `GET /api/consultas/recursos/deposito/{id}` y `GET /api/consultas/depositos/reporte?depositoID={id}`.
7. Si el snapshot está en `rm_deposito` pero el agregado no coincide, enviar `POST /api/consultas/admin/reconstruir` con el mismo header.

**Causa corregida:** el publicador anterior de Logística omitía `version` y `schemaVersion`, obligatorios en `ConsumidorEventos`; por eso el evento era rechazado antes de crear el snapshot. Ahora el outbox de Logística añade ambos campos y admite republicación histórica.
