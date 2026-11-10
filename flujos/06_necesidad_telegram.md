# 06. Modificar una necesidad desde Telegram

**Activación:** un administrador habilitado selecciona `/admin` y usa la operación guiada de editar una necesidad, o ejecuta el comando `/modificar_necesidad`.

1. El bot recibe el comando/mensaje. Para el flujo guiado, pregunta el ID y los campos que falten y muestra los datos/resumen para confirmar antes de enviar la escritura. El catálogo `tools.json` declara la operación, método, ruta y esquema que utiliza tanto Telegram como MCP.
2. Obtener la necesidad actual con `GET /api/donadores-entidades/necesidades/{id}` (la ruta interna es `GET /necesidades/{id}`) cuando se necesite revisar los valores existentes.
3. El usuario confirma los datos nuevos: `entidadID`, `productoSolicitadoID`, `descripcion`, `cantidadObjetivo`, `nivelDeUrgencia` y `tipo`. El DTO HTTP de Donadores y Entidades también admite `id`; en `PUT` debe coincidir con `{id}`.
4. Telegram Bot envía `PUT /api/donadores-entidades/necesidades/{id}` a través del Gateway, `Content-Type: application/json`, con el DTO completo:

   ```json
   {
     "id": "<id-necesidad>",
     "entidadID": "<id-entidad>",
     "nivelDeUrgencia": 4,
     "descripcion": "<descripción-actualizada>",
     "cantidadObjetivo": 12,
     "productoSolicitadoID": "<id-producto>",
     "tipo": "EXTRAORDINARIA"
   }
   ```

5. El `ApiTransport` del bot resuelve la operación del catálogo compartido, genera `X-Trace-Id`, valida el esquema de entrada y ejecuta la solicitud contra `GATEWAY_URL`. MCP Server es un cliente separado que reutiliza `tools.json`; Telegram **no invoca al servidor MCP**.
6. Gateway enruta y elimina `/api/donadores-entidades`, Donadores y Entidades valida/persiste y publica el evento `necesidad` por outbox. El bot devuelve el resultado HTTP; la proyección Mongo se actualiza después del consumo del evento.
7. Verificar escritura con `GET /api/donadores-entidades/necesidades/{id}` y, después de convergencia, el reporte con `GET /api/consultas/necesidades/urgentes`.

**Configuración:** `integrations.gateway-url`/`GATEWAY_URL` apunta al Gateway. Las URLs directas por componente quedan sólo como overrides explícitos para pruebas; no deben configurarse en la demo desplegada.
