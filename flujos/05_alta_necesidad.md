# 05. Alta de necesidad y asignación desde stock

**Activación:** una entidad benéfica declara qué producto necesita, con qué urgencia y cantidad objetivo.

1. La entidad debe existir en Donadores y Entidades. Puede consultarse con `GET /api/donadores-entidades/entidades/{id}`; si falta, crearla mediante `POST /api/donadores-entidades/entidades` según el DTO de entidad del componente.
2. Enviar `POST /api/donadores-entidades/necesidades?periodo=SEMANAL` (o `MENSUAL`) con `NecesidadMaterialDTO`:

   ```json
   {
     "id": null,
     "entidadID": "<id-entidad>",
     "nivelDeUrgencia": 4,
     "descripcion": "<detalle-de-la-necesidad>",
     "cantidadObjetivo": 10,
     "productoSolicitadoID": "<id-producto-existente>",
     "tipo": "EXTRAORDINARIA"
   }
   ```

   `id` debe ser nulo/omitirse; lo genera el servicio. Los tipos son `EXTRAORDINARIA` y `RECURRENTE`; el período se envía en query string y admite `SEMANAL` o `MENSUAL`.
3. Donadores y Entidades valida entidad, producto y campos; consulta `GET /stock/{productoID}` en Logística para conocer stock disponible. Guarda la necesidad con `cantidadAsignada=0` porque el dominio considera satisfecha la cantidad al momento de la entrega, no por la mera reserva.
4. Si el stock permite asignar, invoca sincrónicamente `POST /stock/asignaciones` en Logística con `{ "necesidadId": "<id-necesidad-generado>", "productoId": "<id-producto>", "cantidad": 10 }`. Logística devuelve asignaciones o `204 No Content` si no hay disponibles.
5. Para una necesidad `RECURRENTE`, la cobertura debe ser total; si el stock no alcanza, no se asigna parcialmente. La `EXTRAORDINARIA` puede aceptar cobertura parcial.
6. Donadores y Entidades publica snapshots de `necesidad` y `entidad` a su outbox. Para confirmar escritura usar `GET /api/donadores-entidades/necesidades/{id}`; para vistas agregadas, `GET /api/consultas/necesidades/urgentes` cuando el read model haya convergido.

**Importante:** el alta de una necesidad con stock ya disponible usa la asignación HTTP directa descrita en el paso 4; no es el mismo flujo asíncrono de Workers que se activa al ingresar una donación nueva.
