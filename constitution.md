# Constitución de Mesa de Ayuda

## Misión y audiencia

Construir una API para gestionar tickets de soporte: creación, consulta, asignación,
respuesta, cierre y escalamiento entre los niveles 1, 2 y 3. El servicio permitirá
controlar los SLA de primera respuesta y resolución por prioridad, marcar y escalar
los tickets vencidos y consultar reportes operativos.

Las funcionalidades se incorporarán por fases, según las historias del
[README](README.md#historias-cargadas-en-jira) y el [backlog](specs/backlog.md).

La audiencia comprende:

- **Clientes:** consumen la API directamente para crear y consultar sus tickets.
- **Agentes de soporte:** atienden tickets de los clientes. El nivel de soporte es
  un atributo del agente, no un rol distinto por nivel.
- **Operaciones:** consulta el volumen de tickets abiertos por nivel y el
  cumplimiento de SLA.

## Stack

- Python como lenguaje y FastAPI para la API.
- PostgreSQL como base de datos.
- pytest para las pruebas.
- Docker Compose para ejecutar el servicio y sus dependencias.

Las decisiones técnicas adicionales se documentarán al planificar cada fase. Los
specs describirán comportamiento observable, sin elegir tecnologías o mecanismos
de implementación.

## Calidad

- Cada fase tendrá criterios de aceptación verificables antes de escribir código.
- Las reglas de negocio y la API se verificarán con pytest; la persistencia se
  verificará con pruebas de integración contra PostgreSQL.
- Las pruebas cubrirán los casos válidos, los errores de validación y los permisos
  de cada rol, incluido el aislamiento entre clientes.
- El entorno y los datos iniciales deberán permitir reproducir la ejecución y las
  pruebas con Docker Compose.
- La CI deberá aprobar las comprobaciones y pruebas de la fase antes del merge.

## Seguridad y datos

La autenticación y la autorización son obligatorias desde MDA-1. Inicialmente se
utilizará inicio de sesión con credenciales sintéticas precargadas. La creación de
usuarios, el registro público y la recuperación de contraseñas quedan fuera de
MDA-1. También se precargarán los clientes y el catálogo sintético de categorías;
su administración queda fuera de esa fase.

| Rol | Permisos |
| --- | --- |
| Cliente | Crear tickets para su propio identificador de cliente y consultar únicamente los tickets de ese cliente. El identificador enviado debe coincidir con el autorizado para su identidad. |
| Agente de soporte | Crear tickets para cualquier cliente existente y consultar todos los tickets. La asignación, respuesta, cierre y escalamiento se incorporarán en sus respectivas fases, con sus reglas de autorización. |
| Operaciones | Consultar los reportes de MDA-7. No crear ni modificar tickets, ni acceder a sus descripciones. |

- La autorización se verificará en cada operación. Los permisos no concedidos
  explícitamente se denegarán.
- Las consultas y los listados de clientes no revelarán tickets de otros clientes.
  Consultar un ticket ajeno tendrá la misma respuesta que consultar uno inexistente.
- Todos los datos, incluidas identidades, credenciales, clientes, categorías,
  descripciones y ejemplos, serán sintéticos. No se usarán datos reales de personas
  ni de sistemas.
- Las contraseñas no se almacenarán en texto plano. Los secretos de ejecución no
  se incorporarán al repositorio ni se expondrán en respuestas o registros.
- Los registros no incluirán credenciales ni descripciones de tickets.

## Forma de trabajo

1. Mantener esta constitución como acuerdo del proyecto; sus cambios requieren
   acuerdo del equipo.
2. Revisar el backlog entre fases y tomar la primera fase sin marcar.
3. Resolver las preguntas funcionales y escribir `specs/NNN-<tema>/spec.md` antes
   de implementar. El spec enlazará la historia e incluirá contexto, historias,
   criterios EARS con identificadores, fuera de alcance y preguntas abiertas.
4. Trabajar en una rama por fase, con la clave `MDA-n` en su nombre.
5. Abrir el PR contra `main` del fork del grupo, con el alcance implementado y la
   evidencia de validación. Otro integrante deberá revisarlo.
6. Mergear con CI aprobada y revisión del equipo. El merge cierra la fase; solo
   entonces se marca su casilla en el backlog.

Las dudas de fases futuras se resolverán al preparar sus specs, sin ampliar por
anticipado el alcance de la fase actual.
