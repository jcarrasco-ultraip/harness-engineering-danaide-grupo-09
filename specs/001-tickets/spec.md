# 001 — Crear y consultar tickets

**Historia de origen:** [MDA-1 — Crear y consultar tickets](../../README.md#historias-cargadas-en-jira).

**Acuerdos aplicables:** [Constitución](../../constitution.md).

**Fase:** primera casilla sin marcar del [backlog](../backlog.md).

## Contexto

Los clientes necesitan registrar solicitudes de soporte directamente mediante la
API y consultar su seguimiento. Los agentes necesitan crear tickets en nombre de
clientes y consultar los tickets existentes. Esta fase establece esas operaciones
con autenticación, aislamiento entre clientes y validación de datos desde el inicio.

Los tickets nacen abiertos, sin agente asignado y en nivel 1. Las acciones de
atención y los SLA se incorporarán en fases posteriores.

### Datos y convenciones de la fase

- Cada identidad tiene un rol: cliente, agente de soporte u operaciones. Una
  identidad cliente está vinculada a un identificador de cliente autorizado.
- Las identidades, las credenciales y los clientes son sintéticos y están
  precargados. Los clientes se referencian por su identificador existente.
- El catálogo inicial contiene estas categorías sintéticas predefinidas:
  **Accesos**, **Conectividad**, **Aplicaciones** y **Equipamiento**.
- Las prioridades admitidas son **Baja**, **Media**, **Alta** y **Urgente**. Ese es
  también su orden de menor a mayor prioridad.
- Los cuatro campos de creación son obligatorios: cliente, categoría, prioridad y
  descripción. Los valores textuales no pueden estar vacíos ni contener solo
  espacios en blanco. La descripción admite hasta 5.000 caracteres, contando el
  texto recibido.
- Las fechas representan instantes con zona horaria. Los límites del filtro de
  creación son inclusivos y pueden enviarse por separado.
- Los filtros combinados se aplican conjuntamente. El orden se resuelve antes de
  paginar y siempre se desempata por identificador ascendente.
- Las páginas comienzan en 1; el tamaño predeterminado es 20 y el máximo es 100.
- En el cuerpo JSON de creación, `client_id` es un entero positivo;
  `category`, `priority` y `description` son cadenas. Los cuatro campos tienen
  tipos estrictos: no admiten `null` ni conversiones automáticas de tipos.
- El listado admite únicamente estos parámetros de consulta: `client_id`,
  `category`, `priority`, `created_from`, `created_to`, `sort_by`,
  `sort_direction`, `page` y `page_size`. Cada uno admite una sola ocurrencia.
- La lista vacía de R27 corresponde a solicitudes válidas. Un filtro por cliente
  inexistente usado por soporte se rechaza según R29. Para un cliente que filtre
  por otro identificador, se mantiene la denegación de R20, exista o no ese cliente.

## Historias de usuario

Todas las historias siguientes desarrollan MDA-1:

- **H1 — Crear como cliente:** como cliente autenticado, quiero crear un ticket
  para mi identificador de cliente y recibir su identificador y fecha de creación.
- **H2 — Consultar como cliente:** como cliente autenticado, quiero consultar un
  ticket propio o listar mis tickets con filtros, orden y paginación.
- **H3 — Trabajar como soporte:** como agente autenticado, quiero crear tickets
  para clientes existentes y consultar todos los tickets, pudiendo filtrar por cliente.

## Criterios de aceptación en EARS

### Autenticación y permisos

- **R1.** CUANDO una persona presente credenciales sintéticas precargadas válidas,
  el sistema DEBERÁ autenticarla y reconocer su rol y, cuando corresponda, su
  identificador de cliente autorizado.
- **R2.** SI las credenciales son inválidas, ENTONCES el sistema DEBERÁ rechazar el
  inicio de sesión sin indicar cuál de los datos de acceso es incorrecto.
- **R3.** SI una solicitud de creación o consulta de tickets carece de autenticación
  válida, ENTONCES el sistema DEBERÁ rechazarla sin crear tickets ni devolver datos
  de tickets.
- **R4.** SI una identidad con rol operaciones solicita crear, consultar por
  identificador o listar tickets, ENTONCES el sistema DEBERÁ denegar la operación.
- **R5.** SI un cliente solicita crear un ticket con un identificador de cliente
  distinto del autorizado para su identidad, ENTONCES el sistema DEBERÁ denegar la
  creación sin registrar el ticket.

### Creación

- **R6.** CUANDO un cliente autenticado envíe los cuatro campos válidos para su
  propio identificador de cliente, el sistema DEBERÁ registrar el ticket asociado
  a ese cliente.
- **R7.** CUANDO un agente de soporte autenticado envíe los cuatro campos válidos
  para un cliente existente, el sistema DEBERÁ registrar el ticket asociado al
  cliente indicado.
- **R8.** SI falta un campo obligatorio, algún texto está vacío o contiene solo
  espacios en blanco, o la descripción supera los 5.000 caracteres, ENTONCES el
  sistema DEBERÁ rechazar la creación e informar los campos inválidos sin registrar
  el ticket.
- **R9.** SI el cliente no existe, la categoría no pertenece al catálogo inicial o
  la prioridad no es Baja, Media, Alta o Urgente, ENTONCES el sistema DEBERÁ rechazar
  la creación e informar el campo inválido sin registrar el ticket.
- **R10.** CUANDO se cree un ticket, el sistema DEBERÁ establecer su estado en
  abierto, su asignación sin agente y su nivel en 1.
- **R11.** CUANDO se cree un ticket, el sistema DEBERÁ asignarle un identificador
  único y una fecha de creación, y devolverlos junto con los cuatro campos y los
  valores iniciales de estado, asignación y nivel.

- **R31.** SI una identidad autorizada envía al crear un ticket `null` o un tipo
  incorrecto en cualquiera de los cuatro campos, o un `client_id` que no sea un
  entero positivo, ENTONCES el sistema DEBERÁ rechazar la creación con HTTP 422,
  indicar los campos inválidos y no registrar el ticket, sin convertir tipos
  automáticamente. Las cadenas numéricas, los decimales y los booleanos no son
  identificadores de cliente válidos; los números, booleanos, objetos y listas
  no son cadenas válidas. Los objetos y listas tampoco son identificadores
  válidos. Los valores del tipo correcto siguen sujetos a R8 y R9.

### Consulta por identificador

- **R12.** CUANDO un cliente autenticado consulte un ticket de su propio cliente,
  el sistema DEBERÁ devolver su identificador, cliente, categoría, prioridad,
  descripción, fecha de creación, estado, asignación y nivel.
- **R13.** CUANDO un agente de soporte autenticado consulte un ticket existente,
  el sistema DEBERÁ devolver los mismos datos, cualquiera sea el cliente del ticket.
- **R14.** SI un cliente consulta un ticket ajeno o inexistente, ENTONCES el
  sistema DEBERÁ devolver la misma respuesta de no encontrado, sin diferencias de
  contenido o resultado que revelen la existencia del ticket ajeno.
- **R15.** SI un agente de soporte consulta un identificador inexistente, ENTONCES
  el sistema DEBERÁ responder que el ticket no fue encontrado.
- **R16.** CUANDO se consulte un ticket previamente creado, incluso después de
  reiniciar el servicio, el sistema DEBERÁ conservar y devolver sus datos registrados.

### Listado, filtros y orden

- **R17.** CUANDO un cliente autenticado liste tickets, el sistema DEBERÁ incluir
  únicamente tickets de su cliente, tanto en los resultados como en sus totales.
- **R18.** CUANDO un agente de soporte autenticado liste tickets, el sistema DEBERÁ
  considerar los tickets de todos los clientes, salvo los excluidos por los filtros.
- **R19.** CUANDO una identidad autorizada liste tickets con filtros de categoría,
  prioridad o fecha de creación desde y/o hasta, el sistema DEBERÁ devolver solamente
  tickets que satisfagan todos los filtros enviados, incluyendo ambos límites de
  fecha cuando estén presentes.
- **R20.** CUANDO un agente de soporte filtre el listado por cliente, el sistema
  DEBERÁ devolver solamente tickets del cliente indicado. SI un cliente intenta
  usar ese filtro para otro cliente, ENTONCES el sistema DEBERÁ denegar la solicitud.
- **R21.** CUANDO una identidad autorizada solicite orden por fecha de creación o
  por prioridad, ascendente o descendente, el sistema DEBERÁ aplicar el orden
  solicitado y desempatar por identificador ascendente.
- **R22.** CUANDO se liste sin indicar orden, el sistema DEBERÁ ordenar por fecha
  de creación descendente, con desempate por identificador ascendente.
- **R23.** SI un filtro de categoría o prioridad contiene un valor fuera de su
  catálogo, una fecha no es válida, el límite desde es posterior al límite hasta,
  o el campo o sentido de orden no está admitido, ENTONCES el sistema DEBERÁ rechazar
  el listado e informar el parámetro inválido.

- **R29.** SI un agente de soporte autenticado filtra el listado por un
  identificador de cliente inexistente, ENTONCES el sistema DEBERÁ rechazar la
  solicitud con HTTP 422, indicar `client_id` como parámetro inválido y no devolver
  datos de tickets.
- **R30.** SI una identidad autorizada envía en el listado un parámetro de
  consulta no admitido o repite uno de los parámetros, incluso con valores
  iguales, ENTONCES el sistema DEBERÁ rechazar la solicitud con HTTP 422, indicar
  los parámetros inválidos y no devolver datos de tickets.

### Paginación

- **R24.** CUANDO se liste sin indicar página o tamaño, el sistema DEBERÁ utilizar
  la página 1 y el tamaño 20 para los parámetros omitidos.
- **R25.** CUANDO se solicite una página y un tamaño válidos, el sistema DEBERÁ
  devolver la porción correspondiente del conjunto autorizado, filtrado y ordenado,
  junto con la página, el tamaño y el total de tickets que coinciden antes de paginar.
- **R26.** SI la página no es un entero mayor o igual a 1 o el tamaño no es un
  entero entre 1 y 100, ENTONCES el sistema DEBERÁ rechazar el listado e informar
  el parámetro inválido.
- **R27.** CUANDO no haya coincidencias o la página solicitada esté más allá de la
  última con resultados, el sistema DEBERÁ devolver una lista vacía y conservar los
  datos de paginación y el total correspondiente.
- **R28.** MIENTRAS el conjunto de tickets no cambie, CUANDO se recorran páginas
  consecutivas con los mismos filtros, orden y tamaño, el sistema DEBERÁ permitir
  recorrer los resultados sin repetir ni omitir tickets entre páginas.

## Fuera de alcance

- Creación o administración de usuarios, registro público y recuperación de
  contraseñas. Las cuentas y credenciales de esta fase están precargadas.
- Administración de clientes o categorías.
- Asignación y reasignación, respuestas y notas internas: MDA-2.
- Cierre y eventual reapertura: MDA-3.
- Escalamiento de nivel: MDA-5.
- Cálculo de SLA y sus vencimientos: MDA-4.
- Marcado y escalamiento automático por SLA vencido: MDA-6.
- Reportes de operaciones: MDA-7.
- Edición o eliminación de tickets e interfaz gráfica.

## Preguntas abiertas

No quedan preguntas funcionales bloqueantes para MDA-1 con los acuerdos actuales.
La selección concreta del catálogo inicial se documenta en este spec y puede
revisarse con el equipo antes de implementar.

Las preguntas sobre atención, cierre, escalamiento, SLA y reportes se conservan
en el backlog para resolverlas antes del spec de cada fase. Las URLs de Jira
siguen pendientes; mientras tanto, la referencia de MDA-1 es el README.
