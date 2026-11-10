# 02. Matchmaking asíncrono de paquetes

**Activación:** Logística crea un paquete al recibir `POST /depositos/{id}/donacion` desde Donaciones. El camino de alta de una necesidad que encuentra stock disponible es distinto: Donadores y Entidades invoca directamente `POST /stock/asignaciones` y no depende de este mensaje de worker.

1. La API de Logística persiste el paquete y publica un evento interno de aplicación con la información de depósito, paquete, donación, producto, cantidad y algoritmo. El listener del servidor API envía el mensaje al exchange/cola RabbitMQ de matchmaking.
2. Worker 1 o Worker 2 recibe el mensaje. Ambos ejecutan el mismo proceso; el deployment API normal no debe consumir esa cola (perfil `worker` exclusivo en instancias worker).
3. El Worker consulta las necesidades pendientes con el cliente HTTP configurado por `DONADORES_URL` y obtiene la cantidad ya asignada, cuando corresponde, mediante `GET /internal/matchmaking/necesidades/{necesidadId}/cantidad-asignada` en Logística.
4. El Worker aplica las reglas del dominio: prioriza necesidades `RECURRENTE` que puedan cubrirse completamente; las `EXTRAORDINARIA` admiten cobertura parcial. El algoritmo del depósito (`SUB_ATENDIDOS` o `PRIORIDAD_POR_SCORE`) ordena candidatas. No debe asignarse más que la cantidad del paquete ni duplicar asignaciones ante reintentos.
5. El Worker devuelve el resultado a Logística con `POST /internal/matchmaking/resultados` y `Content-Type: application/json`:

   ```json
   {
     "depositoId": "<id-deposito>",
     "paqueteId": "<id-paquete>",
     "necesidadId": "<id-necesidad-o-null>",
     "cantidadAsignada": 7,
     "cantidadSobrante": 3
   }
   ```

   `depositoId`, `paqueteId`, `cantidadAsignada` y `cantidadSobrante` son requeridos; cantidades no negativas. Si no hubo asignación, `necesidadId` puede ser `null` y `cantidadAsignada` vale `0`.
6. Logística aplica el resultado idempotentemente, registra la asignación si corresponde y conserva cualquier sobrante en stock con el mismo `donacionID`; persiste snapshots de `paquete`, `asignacion` y `deposito` en el outbox CQRS.
7. Verificar con `GET /api/logistica/asignaciones`, `GET /api/logistica/stock/{productoID}` y `GET /api/logistica/depositos/{depositoID}`. Para diagnóstico por necesidad, consultar el endpoint interno de cantidad asignada.

**Concurrencia:** los dos workers pueden competir por mensajes; la API debe tratar un resultado repetido como duplicado y no incrementar dos veces las cantidades. La suma `cantidadAsignada + cantidadSobrante` debe conservar la cantidad original del paquete.
