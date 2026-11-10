# 07. Procesar progreso de misiones e insignias

**Activación:** configuración de una misión/insignia, asignación de una misión a un donador y ejecución manual o periódica del cálculo.

1. Crear insignia con `POST /api/incentivos/insignias` y body `InsigniaDTO` `{ "id": null, "nombre": "<nombre>", "descripcion": "<detalle>" }`.
2. Crear misión con `POST /api/incentivos/misiones` y body `MisionDTO` `{ "id": null, "nombre": "<nombre>", "insigniaID": "<id-insignia>", "categoriaInicio": "OCASIONAL", "categoriaFin": "COLABORADOR", "tipo": "DONACIONES_EXITOSAS" }`. Las categorías y tipos deben ser valores existentes de sus enums; `TipoMisionEnum`: `COMPLETITUD`, `DONACIONES_EXITOSAS`, `DONACIONES_ASCENDENTES`, `REVOLUCION_DONADORA`.
3. Asignar la misión al donador con `POST /api/incentivos/misiones/donador/{donadorID}` enviando el `MisionDTO` de la misión (incluido su ID). La operación responde `204 No Content`.
4. Ejecutar manualmente `POST /api/incentivos/procesamiento/{donadorID}` sin body; responde `204 No Content`. El scheduler automático utiliza `incentivos.cron.inicio-ms` y `incentivos.cron.intervalo-ms` (defaults de 60 segundos) para procesar los IDs elegibles.
5. Incentivos consulta la actividad/datos del donador en los módulos integrados, calcula progreso y condiciones, persiste el resultado de forma idempotente y emite `incentivo_donador` por outbox.
6. Verificar con `GET /api/incentivos/misiones/donador/{donadorID}`, `GET /api/incentivos/insignias/donador/{donadorID}` y, tras el consumo del evento, `GET /api/consultas/donadores/{donadorID}/ficha`.

**Resultado:** el progreso/insignias se actualizan en Incentivos; el read model refleja el estado con consistencia eventual. El procesamiento manual devuelve no-content, por lo que la verificación se hace con GET, no interpretando el código 204 como un resumen de progreso.
