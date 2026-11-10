# 04. Registrar una queja sobre una donación entregada

**Activación:** el receptor reclama una donación que ya fue confirmada como `ACEPTADA`.

1. Enviar `POST /api/donaciones/donaciones/{donacionID}/quejas` a través del Gateway con `Content-Type: application/json` y el único campo requerido por `QuejaRequest`:

   ```json
   { "descripcion": "<detalle-del-reclamo>" }
   ```

2. Donaciones valida que la donación exista y esté aceptada, registra la queja y conoce el `donadorID` asociado a esa donación.
3. Donaciones notifica a Donadores y Entidades mediante `POST /donadores/{donadorID}/quejas`, enviando el DTO interno `QuejaDTO` con `id=null`, `donacionID`, `donadorID`, `fecha` (fecha actual ISO `yyyy-MM-dd`) y `descripcion`.
4. Donadores y Entidades comprueba que la queja no esté ya registrada para esa donación, valida la relación donador/donación y persiste la queja. Para verificar, usar `GET /api/donadores-entidades/donadores/{donadorID}/quejas`.
5. Donaciones y Donadores y Entidades publican sus cambios mediante outbox. Incentivos puede reevaluar el progreso en el procesamiento explícito `POST /api/incentivos/procesamiento/{donadorID}` o en el scheduler periódico; no se debe asumir que la publicación de la queja invoca instantáneamente el cron.
6. Consultar `GET /api/incentivos/misiones/donador/{donadorID}` y `GET /api/incentivos/insignias/donador/{donadorID}` para revisar el efecto; la ficha CQRS se consulta después de la convergencia del read model con `GET /api/consultas/donadores/{donadorID}/ficha`.

**Regla:** no repetir la notificación si Consultas aún no refleja el cambio. Verificar primero la queja en el módulo dueño.
