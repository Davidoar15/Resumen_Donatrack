# 08. Consultas integradas y reportes

**Activación:** una pantalla, usuario o cliente MCP solicita información agregada de más de un módulo.

1. El cliente llama al Gateway bajo `/api/consultas/...`; Gateway elimina `/api/consultas` y reenvía al Servicio de Consultas.
2. Servicio de Consultas lee sólo MongoDB; no realiza llamadas sincrónicas a los cuatro módulos de escritura.
3. `GET /donadores/{id}/ficha`: ficha del donador, donaciones, quejas, entidades beneficiadas e incentivos.
4. `GET /necesidades/urgentes?urgenciaMinima=...&tipo=...&categoriaID=...&subcategoriaID=...&limite=...`: lista urgencias y stock/asignaciones.
5. `GET /depositos/reporte?depositoID=...&desde=yyyy-MM-dd&hasta=yyyy-MM-dd`: ocupación, algoritmo, stock/asignaciones, entregas y tiempos en el rango.
6. `GET /calidad?limiteDonadores=...`: agregaciones de calidad de donación/quejas.
7. `GET /estado`, `GET /recursos/{tipo}` y `GET /recursos/{tipo}/{id}` sirven para verificar estado de colecciones y snapshots.

**Consistencia:** el reporte puede tardar algunos segundos en reflejar una escritura. Consultar primero el recurso `rm_*` y estado; no confundir consistencia eventual con pérdida de datos.
