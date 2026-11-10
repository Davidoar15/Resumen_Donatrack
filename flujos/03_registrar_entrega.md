# 03. Registrar la entrega física de un paquete

**Activación:** el depósito entrega un paquete asignado a la entidad y se confirma la recepción física.

1. Obtener el paquete/asignación desde `GET /api/logistica/asignaciones` o `GET /api/logistica/asignaciones/{asignacionID}`. Confirmar que el paquete corresponde a esa entrega.
2. Enviar `POST /api/logistica/entregas` con `Content-Type: application/json` y el `PaqueteDTO` completo:

   ```json
   {
     "id": "<id-paquete>",
     "donacionID": "<id-donacion>",
     "producto": "<id-producto>",
     "cantidad": 7
   }
   ```

   Para múltiples paquetes usar `POST /api/logistica/entregas/lote` con `{ "paqueteIds": ["<id-paquete-1>", "<id-paquete-2>"] }`.
3. Logística valida y registra la entrega/cambio de estado e historial. El endpoint individual devuelve `200` con mensaje de confirmación.
4. Logística notifica a Donaciones mediante `FachadaDonacionesHttp`; Donaciones cambia el estado de la donación a `ACEPTADA` cuando se cumplen las reglas y publica su snapshot.
5. Logística actualiza la necesidad en Donadores y Entidades por la asignación entregada; la cantidad satisfecha se actualiza con la identidad de entrega para soportar reintentos.
6. Verificar `GET /api/donaciones/donaciones/{donacionID}`, `GET /api/donaciones/donaciones/{donacionID}/historial` y `GET /api/donadores-entidades/necesidades/{necesidadID}`.
7. La proyección MongoDB es eventual. Para el reporte integrado, verificar el estado/evento y consultar el endpoint de Consultas cuando haya consumido los snapshots.

**Resultado:** entrega trazable, estado de donación actualizado y necesidad satisfecha en función de la cantidad asignada efectivamente entregada.
