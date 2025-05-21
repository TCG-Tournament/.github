# .github

# Plan de Desarrollo de la Plataforma de Gestión de Torneos TCG

## Arquitectura Basada en Microservicios

Para garantizar escalabilidad y flexibilidad a largo plazo, se adopta una **arquitectura de microservicios**. Este enfoque divide la aplicación en servicios independientes, cada uno encargado de una **funcionalidad de negocio específica**, lo que facilita escalado individual y despliegues más ágiles. Cada microservicio expone sus propias APIs REST y mantiene su propia lógica y **base de datos independiente**, siguiendo las buenas prácticas de aislamiento de datos. Además, se implementará una **API Gateway** que actúe como punto de entrada único para el frontend, enrutando las solicitudes a los microservicios correspondientes y manejando preocupaciones transversales (autenticación, rate-limiting, etc.).

**Microservicios propuestos:**

* **Servicio de Autenticación y Usuarios:** Gestiona el registro de usuarios, login seguro (con hashing de contraseñas y opcionalmente JWT para sesiones) y roles. También almacena el perfil básico del usuario y su disponibilidad horaria general.
* **Servicio de Torneos:** Permite a organizadores crear y configurar torneos. Mantiene datos de torneos (nombre, fechas de inicio/fin, número de mesas, patrocinadores asociados, tipo de torneo – individual, parejas o equipos – y extras ofertados).
* **Servicio de Inscripción (Jugadores/Equipos):** Gestiona las inscripciones de jugadores a los torneos. Controla registros individuales o en grupo (parejas/equipos) y registra la disponibilidad específica de cada participante para ese torneo (inicialmente copiada de su disponibilidad general). Este servicio también manejará la creación y gestión de equipos/parejas: por ejemplo, un jugador puede crear un equipo e invitar (o agregar) a su compañero antes de inscribir el equipo al torneo.
* **Servicio de Brackets y Partidas:** Encargado de generar los emparejamientos (brackets) y programar las partidas. Utiliza algoritmos semi-aleatorios que respetan restricciones de disponibilidad de los jugadores y la capacidad de mesas. También almacena el resultado de los enfrentamientos y avanza de ronda en ronda. Este servicio podría funcionar de manera **event-driven** – por ejemplo, al cerrar inscripciones un torneo, se emite un evento que este servicio consume para generar el bracket inicial.
* **Servicio de Notificaciones (futuro):** Aunque no es prioridad en la fase inicial, se deja prevista la posibilidad de un microservicio para notificar a los usuarios (vía email o push) sobre sus próximos enfrentamientos, cambios de horario o anuncios del torneo.
* **Servicio de Pagos (futuro):** Preparado para cuando se integre Stripe u otra pasarela. Gestionará la creación de pagos para cuotas de inscripción o compra de extras definidos por el organizador. Por ahora estará inactivo o limitado, pero su separación permite agregar pagos online más adelante sin impactar otros módulos.

Todos los microservicios se comunicarán preferiblemente a través de llamadas REST internas o mensajes en una cola (si se opta por un patrón asíncrono para ciertos procesos, como generación de enfrentamientos). Cada servicio se desplegará en un contenedor Docker individual, lo que facilita la independencia en despliegues y escalado horizontal según la carga de trabajo de cada uno. Por ejemplo, si la generación de brackets consume muchos recursos, el servicio de Brackets podría escalarse por separado sin afectar a los demás.

La arquitectura garantizará la **seguridad** en las comunicaciones usando HTTPS en todos los endpoints. El servicio de Autenticación emitirá tokens seguros (p. ej. JWT) que el API Gateway validará para autorizar las peticiones a otros microservicios. También se prevé implementar controles de acceso por roles: usuarios normales vs. organizadores vs. administradores del sistema.

*(Diagrama lógico:* la aplicación web (frontend) se comunica con una API Gateway, que enruta hacia los microservicios: Auth/Usuarios, Torneos, Inscripciones/Equipos, Brackets/Partidas, etc., cada uno con su propia base de datos. *Esto asegura bajo acoplamiento y datos descentralizados.*)\*

## Elección de Tecnologías

**Backend:** Se opta por **.NET 7 (ASP.NET Core)** para implementar los microservicios, dado que ofrece alto rendimiento, soporte multiplataforma y un ecosistema maduro. .NET facilita la construcción de APIs RESTful y dispone de integraciones para autenticación (ASP.NET Identity, JWT Bearer) y frameworks de microservicios. Cada microservicio será un proyecto ASP.NET Web API separado. Para comunicación interna asíncrona podríamos usar un bus de mensajes (por ejemplo, **RabbitMQ** o Azure Service Bus) en caso de eventos como la generación de brackets post-inscripción.

*Justificación:* .NET es robusto y cuenta con herramientas empresariales; además, el equipo tiene preferencia por .NET/Python. **Alternativa:** En caso de optar por Python, se podría usar **FastAPI** o **Django REST** para los servicios web, y libraries como Celery para tareas asíncronas (p.ej. cálculos de emparejamientos). No obstante, .NET brinda tipado estático y mejor integración out-of-the-box para muchos de nuestros requisitos (login seguro, APIs, despliegue en contenedores).

**Frontend:** Se desarrollará una aplicación web **SPA (Single Page Application)** usando **Angular 15+** (o React/Vue según preferencia, aquí asumiremos Angular por su soporte nativo i18n). Angular permite modularizar la interfaz en componentes reutilizables y gestionar estados de forma robusta. La elección de una SPA moderna asegura una experiencia de usuario fluida y carga dinámica de datos mediante llamadas AJAX/REST al backend. Se aplicará diseño **responsive** apoyándonos en un framework CSS como **Bootstrap 5** (que por defecto sigue principios *mobile-first*) para acelerar el desarrollo de una interfaz adaptable a móviles.

**Base de datos:** Se empleará un **sistema relacional SQL** (por ejemplo, **PostgreSQL** o **SQL Server**). La naturaleza relacional encaja con los datos estructurados del dominio: usuarios, torneos, inscripciones y resultados tienen claras relaciones entre sí. Además, un RDBMS facilita usar **SQL** para consultas complejas (e.g. listar partidas filtrando disponibilidad). Cada microservicio tendrá su base de datos aislada – por ejemplo, el servicio de Usuarios con la suya, el de Torneos con otra, etc. – para garantizar independencia (cada servicio administra sus datos). En .NET, se usará **Entity Framework Core** para el mapeo objeto-relacional y migraciones.

*Nota:* Aunque inicialmente el volumen de datos no es crítico, la elección de tecnologías prepara el terreno para escalar. PostgreSQL ofrece confiabilidad y escalabilidad vertical; en caso de crecimiento, se puede migrar a una solución escalable (sharding, read replicas). También se podrá incorporar un caché (Redis) para optimizar lecturas frecuentes (ej: rankings, listas de torneos) si fuera necesario en fases posteriores.

**Seguridad y otros:** Para el login seguro, ASP.NET Identity manejará el hashing de contraseñas (algoritmo BCrypt o Argon2), enforcement de políticas de contraseña y eventualmente 2FA (opcional). Las APIs requerirán autenticación mediante token JWT en cada petición. A nivel de infraestructura, Docker facilitará el empaquetado; un entorno orquestado con **Kubernetes** podría ser considerado si la escala lo justifica, aunque inicialmente bastaría con Docker Compose o despliegues en Azure App Service/Container Instances para cada servicio.

## Esquema de Base de Datos

A continuación se presenta un **modelo de datos** lógico, resaltando las entidades principales y sus relaciones. Cada microservicio mantiene solo las tablas de su dominio; sin embargo, aquí se describen de forma unificada para claridad:

* **Usuario:** Almacena la información del usuario registrado. Campos principales: `UsuarioID` (PK), nombre, apellidos, email, teléfono, contraseña\_hash, rol (jugador/organizador), disponibilidad\_general (p. ej. texto libre o estructura que indique franjas horarias disponibles). *Nota:* La disponibilidad general podría normalizarse en una tabla aparte de Disponibilidades, pero para simplicidad inicial puede guardarse como preferencia del usuario.
* **Torneo:** Detalles del torneo. Campos: `TorneoID` (PK), nombre, descripción, fecha\_inicio, fecha\_fin, num\_mesas\_disponibles, tipo\_modalidad (individual/parejas/equipos), organizadorID (FK al Usuario que lo creó), estado (inscripción abierta, en curso, finalizado), etc. **Sponsor:** Si se requiere listar patrocinadores, podemos tener una tabla `Sponsor` (`SponsorID`, nombre, logo, URL) y una tabla relación `TorneoSponsor` para vincular múltiples sponsors al torneo. Si es un simple texto/imagen, también se podría almacenar en un campo JSON o texto en Torneo indicando patrocinadores.
* **Inscripción/Participación:** Registra cada participación en un torneo. Campos: `InscripcionID` (PK), torneoID (FK a Torneo), tipo\_participante (solo, pareja, equipo), participanteID (puede referenciar a un jugador o a un equipo; ver más abajo), disponibilidad\_particular (opcionalmente, la disponibilidad confirmada para este torneo; si no se indica, se puede copiar la general del usuario). Esta tabla podría subdividirse según modalidad:

  * Para individuales: un registro por jugador (participanteID = usuarioID).
  * Para parejas/equipos: un registro por equipo (participanteID = equipoID) más registros de miembros del equipo en otra tabla.
* **Equipo/Pareja:** Representa un equipo inscrito. Campos: `EquipoID` (PK), nombre\_equipo, torneoID (FK, suponiendo equipos definidos por torneo), capitánID (opcional, un usuario líder). Si los equipos pudieran ser reutilizables en varios torneos, podríamos separarlos de Torneo, pero dado que las alineaciones pueden cambiar por torneo, es razonable definirlos en contexto de uno. **MiembrosEquipo:** tabla que relaciona `EquipoID` con `UsuarioID` (miembros).
* **Partida/Match:** Representa un enfrentamiento entre dos participantes en un torneo. Campos: `MatchID` (PK), torneoID (FK), ronda, participanteA (FK a UsuarioID o EquipoID), participanteB, horario\_programado, mesa\_asignada, resultado (ganador o puntuación). La ronda puede ser numérica (1,2,3... para eliminatoria) o etiqueta (Final, Semifinal, etc.). También se incluye un campo `estado` de la partida (pendiente, en juego, finalizada).
* **ExtrasInscripcion:** Si el organizador define **extras** (por ejemplo, venta de merchandising, camisa conmemorativa, etc.), se pueden modelar con una tabla `Extra` (id, descripción, precio, torneoID) y una tabla intermedia `ExtraInscripcion` para registrar qué inscripciones (jugadores/equipos) han solicitado qué extras. Por ejemplo, si en la inscripción el jugador marca que quiere comprar una camiseta, quedaría registrado aquí. Esto permitirá luego al organizador saber cuántos extras entregar o cobrar al inicio del torneo.

Cada microservicio manejará las tablas pertinentes:

* *Servicio de Usuarios:* Usuario (y posiblemente Disponibilidad general si separada), Roles.
* *Servicio de Torneos:* Torneo, Sponsor, TorneoSponsor, Extra.
* *Servicio de Inscripción:* Inscripción, Equipo, MiembrosEquipo, ExtraInscripcion.
* *Servicio de Brackets:* Match/Partida (y quizás un registro de Rondas si se quiere tabla aparte para las fases del torneo).

**Relaciones clave:** Un Usuario puede tener muchas Inscripciones (a distintos torneos); un Torneo tiene muchas Inscripciones; si la inscripción es de equipo, el Equipo tiene muchos Usuarios (miembros). Un Torneo tiene muchas Partidas; cada Partida referencia dos participantes (que pueden ser usuarios o equipos según modalidad). Estas relaciones se garantizarán mediante integridad referencial dentro de cada contexto de microservicio. Para las referencias inter-servicio (por ejemplo, la Inscripción guarda `usuarioID` para saber qué usuario se inscribió), no se usará foreign key a la tabla de Usuario (porque está en otra base de datos); en su lugar, se almacena el identificador y el servicio de Inscripción consultará al servicio de Usuarios cuando necesite datos del perfil (p. ej., mostrar nombre). Este desacoplamiento respeta la independencia de los servicios.

La base de datos estará normalizada para evitar duplicación de datos. Por ejemplo, los datos de contacto del jugador solo residen en la tabla de Usuario (microservicio de Usuarios) y otros servicios la piden vía API en caso necesario. Asimismo, todos los campos sensibles (contraseñas, tokens de sesión) estarán adecuadamente cifrados/hasheados.

## Componentes del Frontend

El front-end web (SPA) estará compuesto por varias vistas y componentes, siguiendo una arquitectura **modular**. A continuación, se describen los componentes/páginas principales:

* **Página de Inicio (Home):** Página de bienvenida con información general y posiblemente un listado de próximos torneos destacados. Desde aquí el usuario puede navegar a registrarse o loguearse, y ver torneos públicos.
* **Registro y Login:** Formularios para creación de cuenta y autenticación. El formulario de registro capturará nombre, apellidos, email, teléfono, contraseña y campos de disponibilidad general (por ejemplo, una serie de checkboxes o un campo de texto para "Disponibilidad horaria"). Se implementarán validaciones en el cliente (formato de email, contraseña fuerte, etc.) antes de enviar al backend.
* **Dashboard del Usuario (Perfil):** Una vez logueado, el usuario cuenta con una sección de perfil donde puede revisar y editar su información de contacto y disponibilidad general. También desde aquí puede ver sus torneos inscritos y resultados pasados. *Si el usuario tiene rol de Organizador*, este dashboard mostrará opciones extra, como administrar torneos creados por él.
* **Listado de Torneos:** Página que muestra todos los torneos disponibles (próximos o en curso). Incluye filtros (por fecha, por formato individual/por equipos, por organizador, etc.). Cada torneo listado muestra resumen de fecha, formato, cupos (inscritos vs. capacidad) y un botón para ver detalles o inscribirse.
* **Detalle de Torneo:** Vista específica de un torneo seleccionado. Muestra información completa: descripción, fechas, ubicación (si aplica), patrocinadores (logos/nombres), organizador, número de mesas, lista de inscritos (o número de inscritos), reglas adicionales si las hay, y las **opciones de inscripción**.

  * Si el torneo está **abierto** y el usuario aún no inscrito: muestra formulario de inscripción. En ese formulario, si el torneo es individual, simplemente un botón "Inscribirme" (quizá confirmando disponibilidad). Si es por parejas o equipos, se ofrecerá opción de **crear un equipo** nuevo (ingresando nombre de equipo si se desea y seleccionando al compañero de pareja o invitando miembros por usuario/email) o **unirse a un equipo existente** (previa invitación).
  * El formulario de inscripción mostrará la **disponibilidad precargada** del usuario y permitirá ajustarla para este torneo (ej.: quizás el jugador normalmente está disponible viernes, pero para este torneo específico sabe que un viernes no podrá, entonces lo indica). Así, esos datos se envían al backend.
  * También se listarán los **extras** disponibles definidos por el organizador (por ejemplo, "Comprar tapete conmemorativo - 10€"). El jugador puede marcar qué extras desea. (Sin pago online por ahora, esto solo registra la intención de compra para que la tienda/organizador lo gestione offline).
  * Tras inscripción, la vista podría mostrar mensaje de confirmación y los detalles de la inscripción (ej. "Inscrito como equipo XYZ, a la espera de inicio de torneo").
  * Si el torneo ya **inició**: la vista de detalle muestra el **bracket** o calendario de partidas. Se puede mostrar un diagrama de eliminatorias o una lista de emparejamientos por ronda. Los jugadores verán resaltado su nombre/equipo en el bracket, con horarios y mesas asignadas para sus partidas siguientes.
  * Si el usuario es el organizador de ese torneo: esta página le brinda controles administrativos, como **cerrar inscripciones**, **generar brackets** (si no automático), ingresar resultados de partidas, descalificar a alguien si necesario, y agregar notas o anuncios.
* **Administración de Torneo (Organizador):** Aunque muchos controles del organizador aparecen en la página de detalle del torneo, podríamos tener una sección dedicada donde el organizador ve *todas sus competiciones*. Desde allí puede crear un nuevo torneo (formulario de creación con todos los campos: nombre, fecha, mesas, modalidad, patrocinadores, extras, etc.), editar torneos próximos, y revisar métricas. La **creación de torneo** será un flujo guiado: elección de formato (individual/parejas/equipos y tamaño de equipo), introducción de límites (fecha límite de inscripción, máximo participantes), y adición de extras opcionales con sus descripciones/precios. También puede cargar logos de sponsors en esta interfaz.
* **Componente de Bracket/Calendario:** Un componente UI específico para visualizar los emparejamientos. En desktop podría mostrarse como un diagrama de eliminación; en mobile, tal vez como una lista de rondas desplegables con sus partidas. Este componente obtiene del backend la estructura de partidas (quién contra quién y en qué ronda) y los horarios/mesas. Permitirá actualizaciones dinámicas (por ejemplo, marcar ganador de un match si el usuario es el organizador).
* **Otros componentes utilitarios:** navegación (barra de menú con login/registro si no autenticado, o perfil si sí), selector de idioma (para soporte multilingüe), y posiblemente componentes para mensajes/alertas (ej. confirmaciones, errores). También habrá componentes para formularios reutilizables, como el sub-formulario de disponibilidad horaria (que podría usarse en registro y en inscripción).

**Diseño y UX:** El frontend seguirá principios *mobile-first* para que todas estas vistas sean cómodas en pantalla pequeña. Se priorizará una navegación sencilla: por ejemplo, en móvil se usará un menú tipo hamburguesa con las secciones principales. Los botones serán grandes y táctiles, y se reducirá la necesidad de escribir mucho texto en móvil (aprovechando selecciones predefinidas cuando sea posible, p.ej. casillas para disponibilidad).

En cuanto a *estilo*, podemos implementar un tema simple y limpio, personalizable con CSS (posiblemente con un preprocesador SASS para manejar temas de colores si cada tienda organizadora quiere branding diferente, aunque inicialmente no se pide, se puede tener en mente).

## Flujos Clave de la Aplicación

A continuación se describen los flujos más importantes del sistema paso a paso, mostrando cómo interactúan los componentes frontend con los microservicios backend:

### 1. Registro de Usuarios y Autenticación Segura

1. **Registro de nuevo usuario:** Un visitante accede a la página de registro. Rellena sus datos personales (nombre, apellido, contacto) y credenciales. También indica su disponibilidad general (por ejemplo, selecciona días/horas disponibles en un control UI). Al enviar, el **Frontend** valida los campos (formato de email, contraseña suficientemente fuerte, etc.) y luego envía la solicitud al *Servicio de Autenticación* (endpoint de **API** `/usuarios/registrar`).
2. **Creación en backend:** El Servicio de Autenticación verifica que el email no exista ya y crea el usuario en su base de datos. La contraseña se guarda hasheada con un algoritmo seguro. Opcionalmente, se envía un correo de verificación al email proporcionado (esto podría implementarse ahora o en fases posteriores). La disponibilidad general se guarda asociada al usuario.
3. **Confirmación:** El backend devuelve una respuesta de éxito. El frontend notifica al usuario (por ejemplo, "Registro exitoso, por favor verifica tu correo" si se implementó verificación). El usuario ya puede iniciar sesión.
4. **Inicio de sesión (Login):** En la página de login, el usuario ingresa email y contraseña. Al enviar, el frontend llama al endpoint `/usuarios/login` del Servicio de Autenticación. El backend valida las credenciales:

   * Si son correctas, genera un **token JWT** firmado que incluye el ID de usuario y rol, y lo devuelve.
   * El frontend almacena este token de forma segura (p. ej. en *localStorage* o en una cookie httpOnly) para adjuntarlo en futuras peticiones.
5. **Sesión iniciada:** A partir de aquí, el usuario accede a páginas autenticadas (perfil, inscripción a torneos, etc.). El frontend incluirá el token en los headers de las peticiones (Authorization: Bearer token). El API Gateway/servicios verificarán el token en cada request, garantizando que solo usuarios válidos accedan. Si el usuario tiene rol organizador, el frontend habilitará opciones extra (como crear torneos) en la interfaz.
6. *(Gestión de roles:* Por defecto, todos los registrados son jugadores. Para otorgar rol de Organizador, inicialmente un administrador podría marcar un usuario como organizador en la base de datos, o podríamos implementar una solicitud de upgrade. En fases iniciales bastará con asignarlo manualmente en BD o vía un endpoint admin.)\*

### 2. Inscripción a un Torneo (Individual, Pareja o Equipo)

Este flujo varía ligeramente según la modalidad del torneo, pero conceptualmente:

1. **Creación del torneo (Organizador):** Un usuario con rol organizador navega a "Crear Torneo". Rellena el formulario con datos: nombre del torneo, fechas, número de mesas disponibles, modalidad (ej. "parejas" de 2 jugadores, "equipos" de N jugadores, o "individual"), y agrega opcionalmente patrocinadores (ej. sube logos o ingresa nombres) y extras (cada extra con descripción y precio si corresponde). Al confirmar, el frontend envía estos datos al **Servicio de Torneos** (`/torneos/crear`). El servicio almacena el torneo (estado "inscripción abierta") y asocia el usuario como organizador. Si hay logos de sponsor, podrían almacenarse en blob storage y las URLs guardadas. El organizador queda redirigido a la página del torneo recién creado (detalle), donde inicialmente no hay inscritos.
2. **Vista del torneo para jugadores:** Otro usuario (jugador) ve el nuevo torneo en la lista de torneos disponibles. Entra al detalle `/torneos/{id}`. El frontend obtiene los datos del torneo del Servicio de Torneos. Si el jugador no está inscrito aún y las inscripciones están abiertas, verá el formulario de inscripción adecuado:

   * **Torneo individual:** simplemente un botón "Inscribirse".
   * **Torneo por parejas:** campos para inscribir a la pareja. Aquí se plantea: o bien un jugador inscribe a ambos (p. ej. debe ingresar el email o ID del compañero, quien debe tener cuenta) – posiblemente requiriendo que el compañero confirme; o implementar una mecánica de invitación. Simplificaremos asumiendo que uno inscribe al dúo: el usuario selecciona a su compañero (podemos ofrecer un buscador de usuarios registrados por email/nombre). Si el compañero no existe en sistema, habría que invitarlo a registrarse, pero para MVP podríamos requerir que ambos tengan cuenta de antemano.
   * **Torneo por equipos (más de 2):** similar a parejas pero podrá agregar múltiples miembros. El formulario permite ingresar nombre de equipo y añadir miembros (por usuario/email).
3. **Precarga de disponibilidad:** El formulario de inscripción muestra la disponibilidad general del usuario (por ejemplo, "Disponible: viernes 15-17" previamente guardado). Si el usuario necesita ajustarla para este evento (digamos no podrá tal día en particular), la edita. Esa disponibilidad particular se incluirá en la solicitud de inscripción.
4. **Selección de extras:** Si el torneo ofrece extras (listados con checkboxes), el jugador marca los que desea. Por ejemplo, "Comprar playmat conmemorativo". No se realiza pago ahora, solo se registra la selección.
5. **Enviar inscripción:** Al enviar el formulario, el frontend compila los datos y los envía al **Servicio de Inscripción** (`/inscripciones`):

   * Contiene: torneoID, usuarioID (del inscrito), más si aplica información de equipo:

     * Si es individual: solo el usuarioID.
     * Si es pareja/equipo: o bien un equipoID si ya existía, o los datos para crear uno nuevo con la lista de miembros. En el segundo caso, el Servicio de Inscripción primero creará un registro en Equipo con los miembros (verificando que todos los miembros tienen cuenta y no están ya inscritos por otro equipo), luego crea la inscripción referenciando a ese equipo.
   * Incluye la disponibilidad específica para el torneo y los extras seleccionados.
6. **Procesamiento backend:** El Servicio de Inscripción:

   * Verifica que las inscripciones estén abiertas y que no se exceda el límite de participantes (según num\_mesas u otras reglas, ej. si el torneo tiene un límite de 32 jugadores).
   * Si modalidad equipo/pareja: valida que los usuarios del equipo no se inscriban doble. Puede enviar notificaciones a los otros miembros (futuro: un email "has sido inscrito en el equipo X por tu capitán").
   * Registra la nueva inscripción (crea registro en tabla Inscripción, y registros en MiembrosEquipo si aplica). Copia la disponibilidad proporcionada a un campo de disponibilidad\_particular. Registra también cada extra seleccionado en la tabla ExtraInscripcion.
   * Retorna éxito.
7. **Confirmación al jugador:** El frontend muestra mensaje de inscripción exitosa. A partir de aquí, en la página de detalle del torneo, el usuario verá su estado (por ejemplo "Inscrito como Equipo XYZ" o "Inscripción individual confirmada") en lugar del formulario.
8. **Gestión por organizador:** El organizador al ver la página del torneo verá la lista de inscritos actualizarse en tiempo real o al refrescar. Podrá ver información relevante: por ejemplo, para equipos, ver los nombres de miembros en cada equipo inscrito, y cualquier extra solicitado por cada inscripción. Esto le permite preparar logística (ej. tener listo cierto número de camisetas de la talla, etc., aunque esos detalles de talla no los manejamos aquí).
9. **Cierre de inscripciones:** Cuando llegue la fecha de inicio (o el organizador decida manualmente), se cierra el registro. El organizador puede hacer clic en "Cerrar inscripciones" en la interfaz de torneo, lo que marcará el torneo como no aceptando más jugadores (Servicio de Torneos actualiza estado a "en curso" o "pendiente inicio"). Este evento de cierre puede desencadenar automáticamente la generación de brackets (ver siguiente flujo).

### 3. Generación de Emparejamientos (Brackets) y Asignación de Partidas

Una vez definidas las inscripciones, el sistema debe generar los enfrentamientos considerando disponibilidad y mesas:

1. **Inicio del torneo - generación de bracket:** Al cerrar inscripciones, el **Servicio de Torneos** notifica al **Servicio de Brackets** (esto puede ser mediante una llamada directa o emisión de un evento "Torneo listo para brackets" en una cola de mensajes). El Servicio de Brackets consulta la lista de participantes inscritos del torneo (vía el Servicio de Inscripción) y sus disponibilidades, así como la cantidad de mesas disponibles y rango de horarios posibles (que puede inferirse de las disponibilidades y quizás horario de apertura del evento).
2. **Algoritmo de emparejamiento:** Se utiliza un algoritmo semi-aleatorio *con restricciones*. Inicialmente, se barajan los participantes aleatoriamente para eliminar sesgos. Luego el algoritmo asigna emparejamientos para la **primera ronda** intentando maximizar la compatibilidad de horarios:

   * Recorre la lista de participantes en pares aleatorios para formar partidas. Si detecta que una pareja específica de jugadores no tiene ningún horario en común disponible, podría intercambiar parejas (swap) con otro emparejamiento para mejorar la situación. (En casos extremos, si un jugador tiene disponibilidad muy limitada que no coincide con nadie, el organizador deberá intervenir — por ahora asumiremos la mayoría tiene traslape suficiente).
   * Si hay número impar de participantes, se asigna **bye** (descanso) a uno aleatoriamente o según algún criterio (ej. el que tenga peor traslape de horarios podría darse bye en primera ronda para no forzar un mal emparejamiento).
   * Cada match de primera ronda se le asigna un **horario y mesa**: por ejemplo, se sabe que hay N mesas, y supongamos rondas consecutivas. El sistema puede crear franjas horarias por ronda (ej. Ronda1: viernes 15-17h, Ronda2: viernes 17-19h, etc., de acuerdo al calendario del torneo) e intentar ubicar cada match en una franja en la que ambos jugadores estén disponibles. Si un emparejamiento dado no cabe en la primera franja por disponibilidad, podría asignarlo a una mesa en una franja posterior.
   * El resultado es un conjunto de partidas programadas para la ronda 1.
3. **Publicación del bracket:** El Servicio de Brackets guarda las partidas de la ronda 1 en su base de datos y marca el bracket generado. Estas partidas (con horarios y mesas) se envían al Servicio de Torneos o directamente al frontend para mostrar. El jugador al revisar el torneo ahora verá con quién juega, cuándo y dónde (mesa).
4. **Desarrollo de partidas y siguientes rondas:** Conforme los jugadores juegan sus partidas (sea en persona o en línea, fuera del sistema), los resultados se reportan:

   * El organizador (o un árbitro designado) ingresa al sistema para cada partida y registra el resultado (p. ej. selecciona ganador o ingresa marcador) a través de la interfaz de torneo. Esto envía una actualización al Servicio de Brackets (`/partidas/{id}/resultado`) con el ganador.
   * El Servicio de Brackets actualiza el estado de ese match a "finalizado" y registra quién ganó.
   * Cuando *todas* las partidas de una ronda están finalizadas, el Servicio de Brackets procede a generar la **siguiente ronda**. Toma la lista de ganadores, y repite un proceso similar de emparejamiento:

     * Si es eliminación directa, simplemente empareja ganadores en pares (posiblemente arrastrando el bye si hubo).
     * Vuelve a considerar disponibilidades: ahora, puede ser que algunos jugadores originalmente tenían restricciones que ya no importan si quedaron eliminados. El algoritmo recalcula los cruces óptimos entre los que siguen en competición. A medida que el torneo avanza, los horarios posibles se acotan (por ejemplo, ya estamos en sábado para semifinales, etc.), pero el sistema intenta seguir colocando partidas en horarios factibles para ambos.
   * Nuevamente asigna mesas y horarios para la nueva ronda. Como ya se consumieron algunas franjas, podría asignar la siguiente ronda más tarde ese mismo día u otro día, respetando disponibilidad. (Esto es complejo, pero se puede simplificar asumiendo rondas consecutivas y jugadores que no pueden jugar simultáneamente en dos mesas).
5. **Notificaciones:** Idealmente, cada vez que se asigna una partida, el sistema notificaría a los jugadores involucrados. Por ejemplo, un correo automático: "Ronda 2: Jugador A vs Jugador B el Sábado a las 10:00 en Mesa 3". Esto sería manejado por el Servicio de Notificaciones (si implementado). En la primera versión, podría bastar con que los jugadores consulten la web para ver las actualizaciones.
6. **Finalización del torneo:** El proceso continúa hasta que queda el ganador final. El bracket completo queda almacenado. El torneo se marca como "finalizado" en el Servicio de Torneos. Se podría generar automáticamente un reporte o clasificación final.
7. **Sponsors y premios:** Si hay patrocinadores, sus logos pueden mostrarse durante todo el torneo en la web (por ejemplo, "Patrocinado por ..."). Si en el futuro se gestionan premios, el sistema podría también registrar qué premios corresponden al ganador o finalistas, aunque esto sería una extensión futura.

**Consideraciones adicionales:** El organizador puede ajustar manualmente emparejamientos o reprogramar partidas si surgen imprevistos (por ejemplo, un jugador llega tarde). Para soportar esto, el frontend de organizador podría permitir arrastrar y cambiar horarios o editar el bracket, enviando cambios al backend. Esta funcionalidad avanzada podría implementarse en fases posteriores si se requiere.

## Soporte Multilingüe desde el Inicio

Desde la concepción del proyecto se incorporará la **internacionalización (i18n)** para soportar múltiples idiomas (inicialmente español e inglés, posiblemente más). Esto implica varias prácticas:

* **Interfaz traducible:** Todos los textos de la UI (etiquetas, botones, mensajes) no estarán hardcodeados, sino externalizados en archivos de recursos de idioma (por ejemplo, archivos JSON o de recursos de Angular). Cada cadena tendrá una clave, y habrá un fichero por idioma. Esto facilita actualizar traducciones sin tocar el código. Por ejemplo, `{ "login_button": { "es": "Iniciar Sesión", "en": "Login" } }`.
* **Soporte en frontend:** Angular ofrece el módulo i18n y pipes para traducción. Se configurará el framework para cargar el idioma según preferencia del usuario o configuración del navegador. Habrá un selector de idioma en la página (ej. un menú para Español/Inglés) que al pulsarlo cargará las nuevas cadenas en tiempo real.
* **Soporte en backend:** Aunque la mayor parte de contenido textual está en el frontend, el backend también puede enviar mensajes (ej. emails de notificación, validaciones de error). Para ello, los microservicios pueden utilizar librerías de internacionalización (p. ej. en .NET usar ResourceManager o archivos .resx por cultura). Otra opción es que el backend trabaje en un idioma neutro (inglés) y el frontend traduzca mensajes de error comunes. Pero para una mejor UX, podemos hacer que las API acepten un header "Accept-Language" para devolver mensajes en el idioma del usuario.
* **Datos multilingües:** Si se desea que contenidos dinámicos (descripciones de torneos, nombres de extras, etc.) estén en varios idiomas, habría que extender el modelo de datos para almacenarlos. Por simplicidad inicial, se almacenarán en el idioma que ingrese el organizador (probablemente español). En futuro, se podría permitir al organizador proporcionar traducciones de la descripción del torneo para otros idiomas, guardando campos adicionales o tablas de localización. Esto no se implementa de inicio, pero la arquitectura lo podrá incorporar sin problema.
* **Formato local:** Al ser multi-país, se cuidará el formato de fechas, horas y números. En frontend, usar APIs de Internationalization del navegador o bibliotecas (Moment.js, Intl) para mostrar fechas en formato local apropiado. Por ejemplo, en español "25/05/2025 17:00" vs en inglés "5/25/2025 5:00 PM". Esto se hará de forma automática según el locale activo.
* **Contenido estático traducido:** Cualquier página informativa o email template se preparará en versiones por idioma. E.g., correos de confirmación en ES y EN.
* **Mejores prácticas:** Se seguirá la regla de *no incrustar texto en código*, siempre usar recursos externos para facilitar traducción. Asimismo, se usará UTF-8 en todos los sistemas para soportar caracteres especiales y distintos alfabetos.

Implementando multilingüe desde el inicio evitamos refactorizaciones mayores más adelante. Esto también mejora la accesibilidad y alcance global de la plataforma.

## Diseño **Mobile-First** y Optimización Responsive

La aplicación se diseñará bajo el principio **mobile-first**, es decir, primero optimizada para dispositivos móviles y luego adaptada a pantallas más grandes. Esto garantiza una mejor experiencia dado que la mayoría de usuarios potenciales podrían interactuar desde su smartphone. Aspectos clave de este enfoque:

* **Priorizar contenido esencial:** En móviles, el espacio es limitado. Se mostrará en primer lugar la información y acciones más importantes para el usuario (por ejemplo, en la vista de torneo, en móvil se ve inmediatamente el botón de inscribirse o su próximo enfrentamiento). Elementos menos prioritarios se pueden ocultar tras expansiones o en secciones secundarias. Esto mejora la usabilidad y enfoque del usuario.
* **Layout adaptable:** Se utilizarán unidades relativas y rejillas flexibles (Flexbox/CSS Grid) de Bootstrap con breakpoints mobile-first (comenzando con estilos para \~360px ancho e ir expandiendo). La estructura se pensará inicialmente para una columna vertical (scroll) en móviles, y en tablets/desktop aprovechar mayor anchura para mostrar columnas adicionales o información extra. Ejemplo: en móvil, la lista de torneos es una columna de tarjetas apiladas; en desktop, puede ser una tabla o grid de tarjetas en varias columnas.
* **Navegación móvil:** Se implementará un menú responsive. En móvil, los menús y opciones (perfil, listar torneos, etc.) se ocultarán tras un ícono de menú hamburguesa para maximizar espacio. Los botones táctiles serán suficientemente grandes y separados para evitar toques erróneos (guidelines de tamaño mínimo de objetivos táctiles).
* **Rendimiento en móviles:** Se minimizarán recursos para móviles, cargando sólo lo necesario. Imágenes (ej. logos de sponsors) se optimizarán en tamaño/peso y uso de formatos modernos. El enfoque mobile-first también implica mejorar tiempos de carga en redes móviles más lentas. Se aplicarán técnicas como lazy loading de imágenes/brackets y minificación/concatenación de archivos CSS/JS.
* **Testing en dispositivos reales:** El diseño responsive será validado en varios tamaños de pantalla y dispositivos comunes. Se probará en móviles de gama baja también, para asegurar que la app funciona fluidamente (sin congelamientos).
* **Beneficios:** Este enfoque no solo mejora la **UX en smartphones**, sino que también es bien visto por SEO, ya que los motores de búsqueda utilizan la indexación mobile-first. Google premiará nuestro sitio si ofrece excelente experiencia móvil, lo que podría atraer más usuarios. Además, comenzando por móvil se garantiza que las funcionalidades básicas estén optimizadas; luego es más fácil ampliar a desktop agregando mejoras visuales sin alterar la base (contenido y funcionalidad central).
* **Desktop enhancements:** En pantallas mayores, se “enriquece” la UI pero sin cambiar la lógica. Ej: mostrar la lista de inscritos junto al bracket en la vista de torneo (dos columnas) o gráficos adicionales. Esto se logra con CSS adaptativo aprovechando la base mobile.
* **Posible PWA:** Opcionalmente, se puede convertir la SPA en una **Progressive Web App** para permitir que usuarios instalen la aplicación en móvil y reciban notificaciones push. Esto aprovecharía el diseño responsive y mejoraría aún más la experiencia móvil, pero se consideraría en fases posteriores.

En resumen, el diseño mobile-first nos asegura una aplicación accesible y cómoda en el dispositivo más usado, sin sacrificar a futuro la experiencia en desktop, que será igualmente buena gracias al enfoque responsive.

## Roadmap de Desarrollo en Fases

Dado que no hay una fecha de entrega fija ni un volumen de usuarios definido desde el inicio, se propone un desarrollo **iterativo por fases**, entregando primero un producto mínimo viable y luego agregando funcionalidades. A continuación se detallan las fases planificadas, con sus objetivos, funcionalidades incluidas y responsables de implementación en cada caso:

### Fase 1: MVP – Gestión Básica de Torneos

En esta fase se construye la base del sistema con las funciones esenciales para operar un torneo simple individual sin pagos en línea. El objetivo es tener un sistema funcional donde jugadores puedan registrarse, organizadores crear torneos, jugadores inscribirse individualmente, y generar un bracket básico considerando disponibilidad.

**Funcionalidades Clave y Responsables (Fase 1):**

| Funcionalidad                                                | Responsable(s)                                                                         |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------- |
| Registro y Login de usuarios (JWT, hashing)                  | Backend (Auth Service), Frontend (formulario UI)                                       |
| Perfil de usuario con disponibilidad general                 | Backend (Auth – guardar pref.), Frontend (página perfil)                               |
| Rol de Organizador (asignación manual inicialmente)          | Backend (Auth – campo rol y autorizaciones)                                            |
| Creación de torneos (individual)                             | Backend (Torneo Service – API crear/editar torneo), Frontend (formulario creacion)     |
| Listado y detalle de torneos                                 | Backend (Torneo Service – listar/buscar), Frontend (páginas listar/detalle)            |
| Inscripción individual a torneo                              | Backend (Inscripción Service – crear inscripción), Frontend (UI en detalle torneo)     |
| Copia de disponibilidad a inscripción                        | Backend (Inscripción – lógica copiar/almacenar disp.), Frontend (prellenar form)       |
| Generación automática de bracket (emparejamientos básicos)   | Backend (Brackets Service – algoritmo ronda 1), **Backend** (posible script/algoritmo) |
| Asignación básica de mesas/horarios en ronda 1               | Backend (Brackets – placeholder simple, ej. todas partidas misma hora inicialmente)    |
| Visualización de enfrentamientos (bracket)                   | Backend (Brackets – proveer datos), Frontend (componente bracket simplificado)         |
| Registro de resultado de partida (manual)                    | Backend (Brackets – actualizar ganador), Frontend (opción en UI organizador)           |
| Notificaciones básicas (ej. confirmar inscripción por email) | Backend (Auth/Inscripción – enviar email), **Opcional en F1** (puede postergar)        |
| Estructura i18n preparada en frontend (ES/EN)                | Frontend (set up traducciones estáticas)                                               |
| Diseño responsive inicial (mobile-first CSS)                 | Frontend (desarrollo UI adaptativa)                                                    |
| Deployment de MVP (contenedores, base de datos)              | DevOps (Docker Compose env, pipeline CI/CD básico)                                     |

*Notas:* En Fase 1, el bracket podría generarse de forma simplificada (totalmente aleatorio ignorando disponibilidad para la primera entrega, o asumiendo todos disponibles en horario del evento) si el algoritmo complejo no está listo. La prioridad es probar el flujo completo. La consideración de disponibilidad en emparejamientos podría afinarse en Fase 2.
Responsabilidades: En equipos pequeños, un mismo desarrollador puede cubrir varias; aquí se listan por rol. El Tech Lead/Architect apoyará en el diseño de la API y estructura de microservicios. Un QA deberá probar registro, inscripción y flujo de torneo completo manualmente.

### Fase 2: Funcionalidades Avanzadas y Modalidades de Equipo

Esta fase extiende la plataforma para soportar parejas y equipos, mejora el algoritmo de programación de partidas con disponibilidad, e introduce los extras y preparación para pagos. También se robustecen aspectos del sistema tras feedback del MVP (seguridad, notificaciones, etc.).

**Funcionalidades Clave y Responsables (Fase 2):**

| Funcionalidad                                                                                  | Responsable(s)                                                                                                                                                |
| ---------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Soporte de torneos por *parejas* y *equipos* (N jugadores)                                     | Backend (Inscripción – entidades Equipo, miembros), Frontend (form registro equipo, selección miembros)                                                       |
| Gestión de equipos (crear equipo, invitación/confirmación miembros si necesario)               | Backend (Inscripción – lógica invitación o código equipo), Frontend (UI para gestión equipo en inscripción)                                                   |
| Mejoras en algoritmo de emparejamiento considerando disponibilidad (todas rondas)              | Backend (Brackets – optimización algoritmo, quizá implementar algoritmo de disponibilidad con backtracking/ILP)                                               |
| Programación de partidas multi-ronda con disponibilidad y mesas (calendario)                   | Backend (Brackets – asignar rondas a horarios escalonados), Backend (Torneo – definir slots de tiempo posibles)                                               |
| Actualización automática de siguientes rondas en bracket                                       | Backend (Brackets – detectar fin de ronda, generar siguiente)                                                                                                 |
| Visualización mejorada de bracket (por rondas, con progresión)                                 | Frontend (mejorar componente bracket UI, quizás usar biblioteca)                                                                                              |
| Funcionalidad de **extras** en torneos: definir extras (organizador) y elegir extras (jugador) | Backend (Torneo – modelo Extra), Backend (Inscripción – guardar selecciones), Frontend (UI en creación torneo para extras; UI en form inscripción)            |
| Listado para organizador de extras solicitados                                                 | Backend (Inscripción – consulta), Frontend (sección en detalle torneo para organizador)                                                                       |
| Inicio de integración de pagos (Stripe) – **diseño**                                           | Backend (Pagos – crear microservicio stub que registre intención de pago, integra Stripe sandbox), Frontend (UI mostrar precios extras y msg “pago en sitio”) |
| Soporte multilenguaje completo (verificar todas páginas en EN/ES)                              | Frontend (traducción de todo texto añadido en F2), QA (verificación idioma)                                                                                   |
| Endurecimiento de seguridad (ej. políticas de contraseña, verificación email obligatoria)      | Backend (Auth – reglas de password, email verify flow), DevOps (configurar SMTP o servicio email)                                                             |
| Notificaciones por correo de enfrentamientos programados                                       | Backend (Notificaciones – enviar email con info de match), DevOps (servicio email)                                                                            |
| Pruebas de carga iniciales (escalar microservicios si es necesario)                            | DevOps/Backend (testing concurrent users, ajustar replicas)                                                                                                   |

En esta fase, el trabajo de backend es mayor (nuevas entidades y lógica compleja de emparejamientos), mientras frontend se enfoca en formularios adicionales y visualización avanzada. Se introducirán pruebas unitarias especialmente para el algoritmo de brackets y lógica de inscripción, para asegurar calidad conforme crece la complejidad.

### Fase 3: Monetización y Expansión de Características

La tercera fase incorpora la funcionalidad de pagos en línea y explora características para mejorar la plataforma y atender a negocios (tiendas) como organizadores. También se pulen detalles finales para prepararse a escalar la aplicación a un público mayor.

**Funcionalidades Clave y Responsables (Fase 3):**

| Funcionalidad                                                                                                    | Responsable(s)                                                                                                               |
| ---------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Integración completa de Stripe** para pagos opcionales (ej. pagar inscripción o extras)                        | Backend (Pago Service – integrar API Stripe, manejar webhooks), Frontend (checkout UI, indicar precio total)                 |
| Opciones de pago definidas por organizador (ej. costo de inscripción, costo de cada extra)                       | Backend (Torneo – campos de precios), Frontend (mostrar monto, pasar a Pasarela)                                             |
| Flujo de pago: del carrito de inscripción a Stripe y confirmación                                                | Backend (Pago – crear sesión de pago Stripe, redirigir), Frontend (manejar redirección y confirmar pago completado)          |
| Distinción de cuentas de **Tienda** (organizadores comerciales)                                                  | Backend (Auth – tipo de organizador: tienda vs particular), Frontend (quizá insignia de tienda)                              |
| Funcionalidades extra para tiendas (publicar perfil de tienda, logo, administrar múltiples torneos de su tienda) | Backend (extender Usuario/Organizador con perfil tienda), Frontend (páginas de perfil tienda con lista de torneos, branding) |
| Sistema de comentarios o rating post-torneo (para feedback)                                                      | Frontend (form feedback), Backend (nuevo microservicio o en Torneo – guardar valoraciones)                                   |
| Estadísticas y reportes (número de jugadores, torneos, etc.)                                                     | Backend (Torneo/Inscripción – endpoints de métricas), Frontend (dashboard admin/organizador con gráficas)                    |
| Optimización de rendimiento y escalabilidad final                                                                | DevOps/Backend (deploy en Kubernetes, auto-scaling config, caching donde aplique)                                            |
| Testing completo e inicio de pruebas beta con usuarios reales                                                    | QA/PM (beta testing, gather feedback)                                                                                        |

En Fase 3, cerramos el círculo añadiendo el componente financiero y mejorando la propuesta de valor para organizadores comerciales. La integración de Stripe implicará coordinar front y back: el backend generará las órdenes de pago y procesará confirmaciones (webhooks de Stripe indicando pago realizado, para marcar inscripciones pagadas), mientras el frontend redirigirá al checkout seguro de Stripe.

La distinción de tiendas permitirá, por ejemplo, que una tienda vea todos sus torneos y quizás una opción de destacar su marca. Esto sienta las bases para **futuras fases**, donde se podrían incluir: aplicaciones móviles nativas si se requiere (aunque quizás innecesario si la PWA funciona bien), funcionalidades sociales (amistades, chat entre participantes), o un marketplace de productos de TCG integrado.

Cada fase deberá ser validada con los usuarios finales (jugadores y organizadores) para ajustar la dirección del desarrollo. Como no hay una fecha límite rígida, podemos iterar hasta lograr un sistema sólido y completo.

Finalmente, mediante este roadmap modular, el proyecto puede crecer de un MVP básico a una plataforma robusta de gestión de torneos, cumpliendo con todos los requisitos iniciales y siendo flexible para futuras necesidades. Cada fase entrega valor usable, y la arquitectura de microservicios garantiza que podamos escalar y modificar componentes individualmente conforme la base de usuarios y las características se amplíen en el tiempo.

**Referencias:** Hemos seguido lineamientos de arquitectura de microservicios de la industria, aplicado mejores prácticas de internacionalización y diseño responsive para asegurar que la plataforma sea técnicamente sólida, fácil de mantener y centrada en la mejor experiencia para todos los usuarios (jugadores, organizadores particulares y tiendas). Cada decisión tecnológica y de diseño tomada busca cumplir con los requerimientos planteados de forma eficiente y escalable.

