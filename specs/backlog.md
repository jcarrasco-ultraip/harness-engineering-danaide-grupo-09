# Backlog de Mesa de Ayuda

Una fase por historia, en orden de implementación. Se toma la primera casilla sin
marcar y se prepara su spec antes de implementar. La casilla se marca cuando el
merge del PR cierra la fase, según la [constitución](../constitution.md).

Las historias se enlazan a su sección en el README porque todavía no hay URLs de
Jira disponibles.

- [ ] **001 — MDA-1: Crear y consultar tickets.**
  [Historia](../README.md#historias-cargadas-en-jira) ·
  [Spec](001-tickets/spec.md).
  Entregar autenticación con credenciales sintéticas, permisos por rol, creación,
  consulta por identificador y listado con filtros, orden y paginación. Usar
  clientes y categorías sintéticos precargados. No administrar usuarios ni
  catálogos.

- [ ] **002 — MDA-2: Asignar y responder tickets.**
  [Historia](../README.md#historias-cargadas-en-jira).
  Entregar asignación y respuestas en tickets existentes.
  Antes del spec: definir quién asigna y reasigna, quién responde, visibilidad de
  las respuestas, necesidad de notas internas y qué respuesta cuenta para el SLA.

- [ ] **003 — MDA-3: Cerrar tickets.**
  [Historia](../README.md#historias-cargadas-en-jira).
  Entregar el cierre de un ticket con las validaciones de estado acordadas.
  Antes del spec: definir quién cierra, si se exige motivo y si se admite reapertura.

- [ ] **004 — MDA-5: Escalar de nivel (1, 2 y 3).**
  [Historia](../README.md#historias-cargadas-en-jira).
  Entregar escalamiento manual entre niveles antes de automatizarlo por SLA.
  Antes del spec: definir permisos, condiciones, motivo, efecto sobre la asignación
  y comportamiento cuando el ticket ya está en nivel 3.

- [ ] **005 — MDA-4: SLA por prioridad.**
  [Historia](../README.md#historias-cargadas-en-jira).
  Entregar el cálculo de vencimientos de primera respuesta y resolución por
  prioridad.
  Antes del spec: definir duraciones, horario corrido o hábil, inicio, pausas,
  criterio de cumplimiento y efecto de los cambios de prioridad.

- [ ] **006 — MDA-6: SLA vencido: marcar el ticket y escalarlo.**
  [Historia](../README.md#historias-cargadas-en-jira).
  Entregar detección de vencimientos, marcado y escalamiento usando las reglas de
  MDA-4 y MDA-5.
  Antes del spec: definir frecuencia, tratamiento de ambos vencimientos, prevención
  de escalados repetidos y comportamiento de tickets cerrados o en nivel 3.

- [ ] **007 — MDA-7: Reportes para operaciones.**
  [Historia](../README.md#historias-cargadas-en-jira).
  Entregar reportes de tickets abiertos por nivel y cumplimiento de SLA, accesibles
  a operaciones sin exponer descripciones.
  Antes del spec: definir período de consulta, población y cálculo de cumplimiento,
  y tratamiento de tickets todavía abiertos.
