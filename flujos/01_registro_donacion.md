# 01. Registrar una donación

**Activación:** un donador habilitado entrega una cantidad de un producto para un depósito concreto.

1. Comprobar que el donador existe y puede donar: `GET /api/donadores-entidades/donadores/{donadorID}/puede-donar`. No lleva body; `donadorID` es un segmento de la ruta. La respuesta debe habilitar la operación.
2. Registrar la donación en Donaciones mediante `POST /api/donaciones/donaciones` con `Content-Type: application/json`. `DonacionDTO` tiene esta forma:

   ```json
   {
     "id": null,
     "donadorID": "<id-donador>",
     "depositoID": "<id-deposito-existente>",
     "descripcion": "<detalle-de-la-donacion>",
     "productoID": "<id-producto-existente>",
     "cantidad": 10,
     "estado": null
   }
   ```

   El `id` lo genera Donaciones y debe omitirse o enviarse como `null`. `depositoID` **sí es obligatorio para el flujo**: Donaciones se lo pasa a Logística y no elige un depósito automáticamente. `cantidad` debe ser positiva y `productoID` debe existir.
3. Donaciones vuelve a comprobar el donador contra Donadores y Entidades, valida el producto, genera el ID y guarda el registro como `INGRESADA`. Publica el snapshot de `donacion` mediante su outbox.
4. En la misma operación lógica, Donaciones invoca Logística con `POST /depositos/{depositoID}/donacion` y este JSON interno:

   ```json
   { "donacionID": "<id-generado>", "productoID": "<id-producto>", "cantidad": 10 }
   ```

   La respuesta es `202 Accepted`; el paquete queda asociado al mismo depósito y donación.
5. Logística persiste el paquete y encola snapshots `paquete` y `deposito` en su outbox. Publica el mensaje de matchmaking para los Workers; no espera a que finalicen para responder al alta.
6. Verificar con `GET /api/donaciones/donaciones/{id}`, `GET /api/logistica/depositos/{depositoID}` y `GET /api/logistica/asignaciones`.

**Resultado:** donación `INGRESADA`, paquete asociado al depósito indicado y matching asíncrono en curso. La asignación/sobrante se confirma cuando los Workers informan su resultado.
