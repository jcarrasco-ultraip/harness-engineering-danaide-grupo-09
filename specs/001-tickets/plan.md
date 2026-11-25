# Plan — MDA-1: crear y consultar tickets

## Enfoque

Implementar una API FastAPI pequeña con autenticación de identidades sintéticas, autorización por operación y persistencia en PostgreSQL, separando contratos HTTP, reglas de acceso y consultas a datos. Crear primero pruebas pytest de cada comportamiento, observar su fallo por la regla pendiente y escribir la implementación mínima para hacerlas pasar. Aplicar el alcance autorizado antes de consultar, contar, ordenar y paginar; proteger la creación con validación y una transacción. Entregar migración inicial, carga reproducible de datos sintéticos, Docker Compose y CI para ejecutar las mismas pruebas, incluida la conservación de datos tras reiniciar la API. El alcance termina en R1–R31 del [spec](spec.md), bajo la [constitución](../../constitution.md); no incorpora atención, SLA, reportes ni administración de catálogos.

## Decisiones por defecto

Son decisiones de implementación y contrato técnico para esta fase; no agregan criterios de aceptación al spec. Las tres definiciones funcionales acordadas se documentan más abajo y ya están incorporadas al spec como R29–R31.

| Decisión | Por qué | Alternativa descartada | Requisitos |
| --- | --- | --- | --- |
| Aplicación síncrona, SQLAlchemy con psycopg y migraciones Alembic; sesiones por solicitud | Una sola vía de acceso a PostgreSQL, transacciones y esquema reproducible | Mezclar SQL directo y ORM o introducir concurrencia asíncrona desde el inicio | R6–R11, R16–R28 |
| Tablas de identidades, clientes, categorías, sesiones y tickets; claves foráneas y restricciones para invariantes | Impedir referencias inválidas y persistir sin depender de memoria del proceso | Almacenes en memoria o duplicación de clientes/categorías en tickets | R1, R6–R11, R16 |
| Identificadores de ticket enteros generados por PostgreSQL; `created_at` en `timestamptz`, generado por el servidor y devuelto en UTC | Unicidad concurrente, orden de desempate inequívoco e instantes comparables | Contador en memoria o fecha enviada por el cliente | R11, R16, R19, R21–R22, R28 |
| `POST /auth/login` recibe JSON `username`, `password`; devuelve token opaco Bearer aleatorio con vencimiento configurable (default: 1 hora); persistir solo su hash | Permite invalidar por expiración y evita implementar firma JWT; identidad y rol salen de la base | Tokens sin vencimiento o aceptar rol/cliente declarados por el solicitante | R1–R4 |
| Contraseñas con Argon2id; secretos desde entorno, sin valores reales en repo; semilla sintética explícita e idempotente | Reproducibilidad sin contraseñas en texto plano en almacenamiento ni secretos de ejecución versionados | Credenciales reales, contraseñas persistidas en claro o seed implícito al importar módulos | R1–R2; constitución: seguridad y datos |
| `POST /tickets`, `GET /tickets/{id}`, `GET /tickets`; JSON con `id`, `client_id`, `category`, `priority`, `description`, `created_at`, `status`, `assignee_id`, `level` | Contrato explícito común para creación y detalle; valores iniciales `abierto`, `null`, `1` | DTO distintos que omitan campos requeridos | R6–R16 |
| Login/consulta/listado exitosos: 200; creación: 201; autenticación inválida: 401; permiso denegado: 403; ticket no encontrado: 404; entrada inválida: 422 | Semántica HTTP consistente y verificable | Responder siempre 200 o revelar existencia con 403 en detalle ajeno | R1–R9, R11–R15, R20, R23, R26, R29–R31 |
| Errores JSON con código estable y campos inválidos, sin reflejar valores recibidos; ajeno e inexistente usan exactamente el mismo 404 | Facilita pruebas y evita devolver descripciones o credenciales en errores | Errores SQL, trazas o mensajes distintos según existencia | R2–R5, R8–R9, R14–R15, R23, R26, R29–R31 |
| Query params `client_id`, `category`, `priority`, `created_from`, `created_to`, `sort_by=created_at\|priority`, `sort_direction=asc\|desc`, `page`, `page_size`; respuesta `{items, page, page_size, total}` | Hace concretos filtros y metadatos sin modificar sus reglas | Orden libre con nombres de columnas SQL o paginación sin total | R17–R28 |
| Una función de alcance por identidad, reutilizada en detalle, selección y conteo; filtros parametrizados | El total debe respetar el mismo aislamiento que las filas | Filtrar en Python después de paginar o contar globalmente | R12–R14, R17–R20, R25 |
| Prioridad ordenada mediante rango Baja=1, Media=2, Alta=3, Urgente=4; desempate siempre `id ASC`; offset calculado después de filtrar y ordenar | Respeta orden de negocio y recorrido estable cuando el conjunto no cambia | Orden alfabético, desempate descendente u orden posterior a paginación | R19, R21–R22, R25, R28 |
| Conteo e items dentro de una transacción de lectura con snapshot consistente en PostgreSQL | Evita contradicción entre total e items de una respuesta | Dos consultas con snapshots independientes | R17, R25, R27; R28 solo promete estabilidad sin cambios |
| pytest unitario para reglas e integración HTTP/repositorio contra PostgreSQL real; Compose para entorno y CI | Verifica tipos, restricciones y orden de la base elegida | SQLite como sustituto de las pruebas de persistencia | R1–R31; constitución: calidad |

## Casos que el spec no detalla

### Defaults técnicos y pruebas asociadas

| Caso | Tratamiento propuesto | Prueba prevista |
| --- | --- | --- |
| Token ausente, malformado, desconocido o vencido | 401 sin datos ni escrituras; operaciones recibe 403 en las tres rutas de tickets | `test_authentication_required` parametrizado por ruta/token y `test_operations_denied`; R3–R4 |
| Usuario inexistente frente a contraseña incorrecta | Mismo código y cuerpo genérico; no devolver usuario ni contraseña | `test_login_failure_is_generic`; R2 |
| Varias creaciones simultáneas | IDs distintos y datos íntegros; no prometer IDs consecutivos | `test_concurrent_creation_unique_ids` contra PostgreSQL; R11 |
| Límite de descripción con espacios y Unicode | Contar caracteres del texto recibido, antes de cualquier transformación; conservar descripción aceptada exactamente | `test_description_boundaries`: 1, 5.000 y 5.001 caracteres, espacios alrededor, Unicode y solo blancos; R8, R11–R12 |
| Cliente manda campos internos como `status`, `level`, `assignee_id`, `id`, `created_at` | Rechazar con 422 y señalar campos extra; ninguna escritura | `test_server_fields_cannot_be_set`; R10–R11 |
| Cadenas de catálogo con distinta capitalización o espacios alrededor | Usar valores exactos del catálogo; sin normalización silenciosa | `test_catalog_values_are_exact`; R9, R23 |
| Fecha sin zona o equivalente con otro offset | Rechazar fecha sin zona; comparar instantes equivalentes y conservar inclusividad | `test_date_filter_requires_timezone`, `test_date_offsets_and_inclusive_bounds`; R19, R23 |
| Empate de prioridad/fecha entre páginas | Fixtures con empates, orden secundario por ID ascendente incluso con orden principal descendente | `test_sort_ties_and_page_walk` en las cuatro combinaciones de orden; R21–R22, R28 |
| Parámetros de paginación omitidos individualmente | Aplicar default solo al omitido; no corregir silenciosamente entradas inválidas | `test_independent_pagination_defaults`, `test_invalid_pagination`; R24–R26 |
| Cero resultados y página fuera de rango | `items=[]`; mantener página y tamaño pedidos; total filtrado/autorizado | `test_empty_and_out_of_range_pages`; R17, R25, R27 |
| Cliente filtra por sí mismo o intenta otro ID inexistente | Admitir el propio; denegar otro ID antes de consultar su existencia | `test_client_filter_scope`; R17, R20 |
| Error durante inserción y reintento | Rollback; no dejar ticket parcial. No agregar garantía de idempotencia HTTP | `test_failed_insert_rolls_back`; R6–R11, R16 |
| Reinicio y carga inicial repetida | Mantener volumen de PostgreSQL; reiniciar proceso API; seed no duplica ni borra datos | `test_ticket_survives_api_restart`, `test_seed_is_idempotent`; R1, R9, R16 |
| Logs y respuestas de error de validación | Excluir cuerpo de tickets, credenciales, tokens, cabeceras de autorización y secretos; no usar el error por defecto si refleja `input` sensible | `test_sensitive_data_not_in_logs_or_errors`, con marcadores sintéticos en login y payload inválido; R2–R3, R8; constitución: seguridad y datos |

### Definiciones funcionales acordadas en el spec

Las tres omisiones detectadas quedaron resueltas en R29–R31 del [spec](spec.md). Se conservan P1–P3 como referencia de la discusión; no hay decisiones funcionales pendientes en estos casos.

| ID | Regla acordada | Prueba prevista | Requisitos |
| --- | --- | --- | --- |
| P1 | Soporte filtra por `client_id` inexistente: HTTP 422, señalar `client_id` y no devolver tickets. Un cliente que filtra por otro ID sigue recibiendo denegación, exista o no; un cliente existente sin tickets sí da lista vacía y total 0 | `test_agent_filter_unknown_client_returns_422`, `test_client_filter_scope`, `test_agent_filter_existing_client_without_tickets` | R20, R27, R29 |
| P2 | Parámetros de consulta desconocidos o repetidos en el listado: HTTP 422, señalar los parámetros inválidos y no devolver tickets, aunque los valores repetidos sean iguales | `test_unknown_and_repeated_query_parameters`, parametrizado por cada parámetro admitido y por valores iguales/distintos | R30 |
| P3 | Creación con tipos estrictos: `client_id` entero positivo; categoría, prioridad y descripción cadenas. Rechazar `null`, tipos incorrectos y conversiones automáticas con HTTP 422, indicar campos inválidos y no insertar | `test_creation_strict_types_and_nulls`, con cada campo en `null`, números, booleanos, objetos/listas incompatibles; ID 0, negativo, decimal y cadena numérica; caso válido de control | R8–R9, R31 |

No se detectan contradicciones entre constitución y spec. Las URLs de Jira siguen pendientes según el propio spec; no bloquean esta planificación. Las dudas sobre fases futuras siguen en el backlog.

## Validación

- Mantener trazabilidad R1–R31 en [tasks.md](tasks.md); todo test de aceptación debe señalar requisito, datos sintéticos, acción y resultado. Incluir los tests de R29–R31 y distinguir cliente inexistente (422 para soporte) de cliente existente sin tickets (lista vacía).
- Antes de cada implementación, ejecutar su test y guardar comando y fallo esperado: debe fallar por el comportamiento ausente, no por problemas de entorno. Reejecutar hasta verde y conservar evidencia en el PR; un import inexistente solo sirve como primer smoke de arranque.
- Ejecutar unitarios y pruebas HTTP/integración con pytest sobre una base PostgreSQL de pruebas aislada, migrada y con datos controlados. Usar al menos dos clientes, dos identidades de cliente, un agente y una identidad operaciones; cubrir permisos en cada ruta, no solo en una función de autorización.
- Verificar límites 0/1/20/100/101 en paginación según corresponda; descripciones 5.000/5.001; fechas inclusivas con offsets y empates; filtros combinados; totales aislados; recorrido sin omisiones ni duplicados con datos invariables.
- Probar persistencia reiniciando realmente la API con la misma base y volumen, no solo abriendo una sesión nueva. Probar unicidad concurrente y rollback sobre PostgreSQL.
- Documentar como interfaz de ejecución `docker compose up --build` y `docker compose run --rm test` (servicio `test` separado de la base de desarrollo). CI ejecutará el mismo suite completo desde una base limpia; fijar dependencias y verificar migración y seed. No se afirma que estos comandos existan todavía.
- Cierre de implementación: suite verde, evidencia de rojo/verde, revisión de contrato OpenAPI y ausencia de secretos/datos sensibles en logs, revisión de otro integrante y CI aprobada antes de merge. No marcar MDA-1 en backlog antes del merge.

Esta entrega contiene planificación: todavía no hay implementación ni pruebas ejecutables de MDA-1.
