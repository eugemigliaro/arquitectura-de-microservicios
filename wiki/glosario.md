# Glosario

Definiciones breves y enlaces al apunte donde se desarrolla cada término. Toda definición específica de la materia debe incluir una cita trazable.

| Término | Definición |
|---|---|
| Acoplamiento | Grado de dependencia entre servicios; es bajo cuando un cambio local no fuerza cambios coordinados en sus consumidores. [T02, p. 20] Ver [Diseño de servicios](temas/diseno-de-servicios.md). |
| Aggregate | Conjunto de entidades y value objects tratado como unidad y frontera de consistencia transaccional. [T03, p. 37] Ver [DDD táctico](temas/ddd-tactico-y-eventstorming.md). |
| Aggregate Root | Entidad raíz mediante la cual el exterior accede al aggregate. [T03, p. 37] |
| Anticorruption Layer (ACL) | Adaptador que traduce entre modelos y evita que el lenguaje de otro contexto contamine el propio. [T03, p. 26] Ver [DDD estratégico](temas/ddd-estrategico.md). |
| Bounded Context (BC) | Límite explícito dentro del cual un modelo de dominio y su lenguaje son válidos y consistentes. [T03, p. 21] |
| Business capability | Capacidad que un contexto ofrece al negocio o a otros contextos; expresa comportamiento, no solo acceso CRUD a datos. [T02, p. 25] |
| Capability de Linux | Privilegio de kernel independiente que reemplaza el modelo de root como conjunto indivisible de permisos. [T04, p. 33] Ver [Contenedores](temas/contenedores-linux.md). |
| cgroup | Mecanismo del kernel que contabiliza y limita recursos como CPU, memoria, PIDs e I/O para grupos de procesos. [T04, p. 16] [T04, p. 18] |
| Cohesión | Grado en que las responsabilidades de un servicio se relacionan con un propósito bien definido. [T02, p. 22] |
| Command | Intención expresada en imperativo que entra a un aggregate o contexto y puede rechazarse. [T03, p. 38] |
| Conformist | Relación en la que un contexto downstream adopta sin traducción el modelo del upstream. [T03, p. 25] |
| Contenedor | Proceso del host con vistas aisladas, recursos limitados y filesystem propio; usa el kernel del host. [T04, p. 12] [T04, p. 13] |
| Context Mapping | Técnica para representar límites, dependencias, influencia y traducciones entre bounded contexts. [T03, p. 24] |
| Contract testing | Verificación automatizada de las expectativas entre consumidores y proveedor sin exigir un entorno integrado completo. [T02, p. 33] |
| Copia apartada | Copia reservada temporalmente para un intento de intercambio; una oferta activa por sí sola no la aparta. [E01, p. 1] [E01, p. 2] |
| Copia ofrecible | Copia disponible y repetida que puede respaldar ofertas sin que eso cree reservas adicionales. [E01, p. 1] [E01, p. 2] Ver [Entrega Álbum](temas/entrega-album-2026.md). |
| Core Domain | Subdominio que contiene una diferencia competitiva y merece máxima atención de diseño. [T03, p. 20] |
| Domain | Área de conocimiento, lenguaje y reglas del problema de negocio; no equivale al código. [T03, p. 19] |
| Domain Event | Hecho de negocio ya ocurrido, nombrado en pasado y propagable hacia otros interesados. [T03, p. 38] |
| Entity | Objeto definido por una identidad que persiste aunque cambien sus atributos. [T03, p. 35] |
| Event Modeling | Método complementario que intercala wireframes, commands, eventos y read models en un storyboard de cambios y vistas. Fuente externa: [B01, p. 7]. |
| EventStorming | Técnica colaborativa y visual que parte de eventos de dominio para descubrir procesos, aggregates y bounded contexts. [T03, p. 41] |
| Generic Subdomain | Parte necesaria pero no diferenciadora que suele poder resolverse con un producto o servicio de mercado. [T03, p. 20] |
| Idempotencia | Propiedad por la cual procesar nuevamente la misma operación o mensaje no cambia el estado final. La consigna exige ese comportamiento ante reintentos y duplicados; la definición técnica se amplía en una fuente externa. [E01, p. 3] [E01, p. 4] [B01, p. 13] |
| Ley de Conway | Observación de que la estructura de un sistema tiende a reflejar la estructura de comunicación de la organización. [T02, p. 24] |
| Microservicio | Servicio con interfaz definida que puede ejecutarse, desplegarse y escalarse por separado. [T02, p. 4] [T02, p. 42] Ver [Fundamentos](temas/fundamentos-de-microservicios.md). |
| Monolito distribuido | Conjunto de despliegues separados que continúa acoplado por datos o diseño. [T02, p. 32] |
| Namespace | Vista aislada de un recurso del kernel, como procesos, red, montajes, UTS, IPC o usuarios. [T04, p. 14] |
| OCI | Conjunto de contratos estándar para empaquetar imágenes y ejecutar contenedores mediante runtimes intercambiables. [T04, p. 29] |
| Open Host Service (OHS) | Interfaz estable mediante la cual un contexto ofrece capacidades a múltiples consumidores. [T03, p. 27] |
| OverlayFS | Filesystem de unión que combina capas de solo lectura con una capa escribible y presenta una vista `merged`. [T04, p. 19] [T04, p. 21] |
| Published Language | Contrato público de integración, separado del modelo interno, como OpenAPI, AsyncAPI, JSON Schema o un esquema de eventos. [T03, p. 27] |
| runc | Runtime OCI que configura aislamiento, límites, rootfs y seguridad antes de ejecutar el proceso del contenedor. [T04, p. 31] |
| seccomp | Mecanismo que filtra las syscalls permitidas a un proceso. [T04, p. 32] [T05, p. 48] |
| Separate Ways | Decisión de no integrar dos contextos y aceptar duplicación para preservar autonomía. [T03, p. 28] |
| Shared Kernel | Subconjunto de modelo compartido cuya evolución requiere coordinación entre los equipos participantes. [T03, p. 28] |
| Subdomain | Recorte del problema de negocio dentro del dominio general. [T03, p. 19] [T03, p. 22] |
| Supporting Subdomain | Parte específica y necesaria del negocio que no constituye su diferenciador principal. [T03, p. 20] |
| Transactional Outbox | Patrón complementario que persiste el cambio de negocio y el evento pendiente en una transacción local para evitar el dual write. Fuente externa: [B01, p. 11]. |
| Ubiquitous Language | Vocabulario preciso construido y usado por negocio y desarrollo dentro de un bounded context. [T03, p. 14] [T03, p. 17] |
| Value Object | Objeto sin identidad propia, definido por sus atributos y normalmente inmutable. [T03, p. 36] |
| Whiteout | Marca en la capa superior de OverlayFS que oculta un archivo de una capa inferior sin modificar esa capa. [T04, p. 21] [T05, p. 50] |
