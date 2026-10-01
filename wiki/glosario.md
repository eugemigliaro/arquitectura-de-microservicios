# Glosario

Definiciones breves y enlaces al apunte donde se desarrolla cada término. Toda definición específica de la materia debe incluir una cita trazable.

| Término | Definición |
|---|---|
| Acoplamiento | Grado de dependencia entre servicios; es bajo cuando un cambio local no fuerza cambios coordinados en sus consumidores. [T02, p. 20] Ver [Diseño de servicios](temas/diseno-de-servicios.md). |
| ADR (Architecture Decision Record) | Registro de una decisión de arquitectura significativa, es decir, costosa de cambiar o que afecta a más de un servicio. Incluye contexto, al menos dos alternativas reales, decisión y consecuencias. [E04] Ver [Segunda entrega](temas/segunda-entrega-album-arquitectura.md). |
| Análisis de dominio | Trabajo de descubrir y acordar el vocabulario, las fronteras, el modelo interno, las invariantes y los hechos relevantes del negocio antes del diseño técnico. En la primera entrega del Álbum excluye implementación y arquitectura. [E02] Ver [Primera entrega DDD](temas/primera-entrega-album-ddd.md). |
| Aggregate | Conjunto de entidades y value objects tratado como unidad y frontera de consistencia transaccional. [T03, p. 37] Ver [DDD táctico](temas/ddd-tactico-y-eventstorming.md). |
| Aggregate Root | Entidad raíz mediante la cual el exterior accede al aggregate. [T03, p. 37] |
| Anticorruption Layer (ACL) | Adaptador que traduce entre modelos y evita que el lenguaje de otro contexto contamine el propio. [T03, p. 26] Ver [DDD estratégico](temas/ddd-estrategico.md). |
| Atributo de calidad | Requisito de calidad expresado como escenario medible, con estímulo, entorno, respuesta y medida, en lugar de un adjetivo como "escalable". [E04] |
| Bind mount | Montaje que presenta el mismo archivo o directorio del host dentro del mount namespace de un contenedor; el host conserva la gestión del path y de sus permisos. [T08, p. 35] [T08, p. 38] Ver [Docker CLI, redes y volúmenes](temas/docker-cli-redes-y-volumenes.md). |
| Blue/Green deployment | Estrategia que prepara un ambiente paralelo al operativo y usa un distribuidor de tráfico para decidir cuál recibe las requests. [T13, p. 41] Ver [DevOps y CI/CD](temas/devops-y-ci-cd.md). |
| Bounded Context (BC) | Límite explícito dentro del cual un modelo de dominio y su lenguaje son válidos y consistentes. [T03, p. 21] |
| Bridge de Docker | Switch L2 en software que conecta interfaces `veth` de varios network namespaces; el engine agrega IPAM, rutas y reglas de NAT. [T08, p. 20] [T08, p. 21] |
| Broker de mensajes | Intermediario de la comunicación asíncrona que media en el tiempo entre emisor y receptor; la cátedra presenta RabbitMQ, Kafka y NATS. [T14, p. 6] [T14, p. 7] Ver [Comunicación asincrónica y Kafka](temas/comunicacion-asincrona-y-kafka.md). |
| Build context | Conjunto de archivos que el cliente envía al engine antes de ejecutar un Dockerfile; el build aislado solo puede acceder a ese conjunto. [T06, p. 23] Ver [Imágenes](temas/imagenes-de-contenedores.md). |
| Bulkhead | Límite de concurrencia y espera que rechaza rápidamente el exceso de trabajo para proteger la capacidad y memoria del servicio. [T09, p. 26] Ver [Comunicación síncrona](temas/comunicacion-sincrona.md). |
| Business capability | Capacidad que un contexto ofrece al negocio o a otros contextos; expresa comportamiento, no solo acceso CRUD a datos. [T02, p. 25] |
| C4 (modelo) | Vistas de arquitectura por niveles. La segunda entrega pide el nivel 1, de contexto (sistema, actores y sistemas externos), y el nivel 2, de contenedores (servicios, almacenes, broker y demás piezas desplegables). [E04] |
| Canary deployment | Despliegue que envía la nueva versión primero a un grupo de usuarios y la amplía progresivamente; limita el impacto de una falla y facilita el rollback. [T13, p. 42] |
| Capability de Linux | Privilegio de kernel independiente que reemplaza el modelo de root como conjunto indivisible de permisos. [T04, p. 33] Ver [Contenedores](temas/contenedores-linux.md). |
| CDC (Change Data Capture) | Técnica que lee el write-ahead log de la base y publica los cambios al broker, por ejemplo con Debezium; es una de las formas de vaciar un outbox. [T14, p. 49] |
| cgroup | Mecanismo del kernel que contabiliza y limita recursos como CPU, memoria, PIDs e I/O para grupos de procesos. [T04, p. 16] [T04, p. 18] |
| Claim check | Patrón en el que el evento lleva un puntero (`uri`, `blobId`) al payload pesado guardado en un store, en lugar del payload mismo. [T14, p. 18] |
| CloudEvents | Especificación de la CNCF para describir eventos con atributos comunes como `id`, `source`, `type`, `specversion` y `data`, agnóstica de transporte y lenguaje; es convención de cátedra en los labs event-driven. [T14, p. 11] |
| Cohesión | Grado en que las responsabilidades de un servicio se relacionan con un propósito bien definido. [T02, p. 22] |
| Command | Intención expresada en imperativo que entra a un aggregate o contexto y puede rechazarse. [T03, p. 38] |
| Conformist | Relación en la que un contexto downstream adopta sin traducción el modelo del upstream. [T03, p. 25] |
| Consumer group | Grupo de consumidores que se reparte el trabajo: cada elemento se entrega a un solo miembro. En un stream de Valkey se espera un ACK por mensaje; en Kafka cada partición se asigna a un solo consumer del grupo, el paralelismo máximo es la cantidad de particiones y grupos distintos reciben todos los mensajes (fan-out). [T12, p. 32] [T14, p. 40] [T14, p. 42] Ver [Estado distribuido](temas/estado-distribuido.md). |
| Consumer lag | Diferencia entre el final del log y el offset de un consumer; la métrica operativa principal de un sistema asíncrono. [T14, p. 43] |
| Contenedor | Proceso del host con vistas aisladas, recursos limitados y filesystem propio; usa el kernel del host. [T04, p. 12] [T04, p. 13] |
| Context Mapping | Técnica para representar límites, dependencias, influencia y traducciones entre bounded contexts. [T03, p. 24] |
| Continuous Delivery | Práctica que agrega a CI tests de integración, performance y UAT, dejando cada artefacto listo para desplegarse sin incluir el despliegue. [T13, p. 17] |
| Continuous Deployment | Práctica que despliega automáticamente a producción todo release que pasa las pruebas; es una decisión de negocio. [T13, p. 18] |
| Continuous Integration (CI) | Ejecución automática de build y tests con cada commit y PR, con ramas de feature de vida corta. [T13, p. 16] Ver [DevOps y CI/CD](temas/devops-y-ci-cd.md). |
| Contract testing | Verificación automatizada de las expectativas entre consumidores y proveedor sin exigir un entorno integrado completo. [T02, p. 33] [T10, p. 10] Ver [Testing](temas/testing.md). |
| Copia apartada | Copia reservada temporalmente para un intento de intercambio; una oferta activa por sí sola no la aparta. [E01, p. 1] [E01, p. 2] La consigna general la define como reservada "para una oferta o un intercambio en curso" y la aparta al publicar; el conflicto está en [Dudas y conflictos](dudas-y-conflictos.md). [E03, p. 1] [E03, p. 4] |
| Copia ofrecible | Copia disponible y repetida que puede respaldar ofertas sin que eso cree reservas adicionales. [E01, p. 1] [E01, p. 2] Ver [Entrega Álbum](temas/entrega-album-2026.md). |
| Core Domain | Subdominio que contiene una diferencia competitiva y merece máxima atención de diseño. [T03, p. 20] |
| Coreografía | Coordinación sin orquestador central en la que cada servicio reacciona a eventos o invoca a otros según su rol; el flujo queda implícito. [T11, p. 37] Ver [Sistemas distribuidos](temas/sistemas-distribuidos.md). |
| correlationId / causationId | Extensiones del evento que identifican, respectivamente, el flujo de negocio completo y el evento o request que causó el actual, para reconstruir la cadena asíncrona. [T14, p. 54] |
| CQRS | Separación entre el modelo que escribe y el que se consulta. En su versión light, read models alimentados por los eventos del writer, sin Event Sourcing completo. [T14, p. 19] |
| DevOps | Prácticas, herramientas y filosofía de trabajo que automatizan los procesos entre el desarrollo y la puesta en producción; DevSecOps suma la seguridad como prioridad. [T13, p. 3] |
| Digest de imagen | Hash SHA256 derivado del manifiesto que permite referenciar de forma inmutable el contenido desplegado. [T06, p. 36] |
| Dead-letter queue (DLQ) | Destino aparte para los mensajes que fallaron N veces, de modo que no bloqueen al resto. [T14, p. 52] |
| Deadline | Límite temporal que el cliente propaga con una llamada; evita que una operación lenta bloquee indefinidamente y extienda el fallo por la cadena. [T09, p. 12] |
| DNAT | Traducción de la dirección o puerto de destino que Docker usa al publicar un puerto del host hacia un contenedor. [T08, p. 22] |
| Domain | Área de conocimiento, lenguaje y reglas del problema de negocio; no equivale al código. [T03, p. 19] |
| Domain Event | Hecho de negocio ya ocurrido, nombrado en pasado y propagable hacia otros interesados. [T03, p. 38] |
| Docker Compose | Herramienta que describe en YAML una aplicación multicontenedor con servicios, redes, volúmenes, configuración y dependencias para orquestación local. [T07, p. 3] [T07, p. 4] Ver [Docker Compose](temas/docker-compose.md). |
| Docker daemon | Servicio `dockerd` que expone la API consumida por la CLI y gestiona imágenes, redes y volúmenes antes de delegar la ejecución en `containerd` y `runc`. [T08, p. 4] |
| Dual write | Escribir en la base y publicar al broker sin una transacción compartida; una caída entre ambos pasos deja un estado inconsistente. [T14, p. 46] |
| Efecto idempotente | Resultado en el que la infraestructura puede entregar un mensaje varias veces pero el consumidor reconoce su identidad y aplica el cambio de negocio una sola vez. [T12, p. 26] |
| Entity | Objeto definido por una identidad que persiste aunque cambien sus atributos. [T03, p. 35] |
| Event-carried state transfer | Forma de evento que transporta el estado completo para que el consumidor no vuelva a consultar; a cambio duplica datos entre servicios. [T14, p. 17] |
| Event-Driven Architecture (EDA) | Estilo en el que los servicios emiten eventos sobre hechos ocurridos y otros reaccionan, sin que el emisor sepa quién escucha. [T14, p. 15] Ver [Comunicación asincrónica y Kafka](temas/comunicacion-asincrona-y-kafka.md). |
| Event notification | Forma de evento con payload mínimo; quien necesita más datos consulta al emisor. [T14, p. 17] |
| Expand/contract | Técnica que la segunda entrega menciona como ejemplo para desplegar migraciones de esquema y cambios de contrato sin interrumpir el servicio, explicitando su orden respecto del despliegue de los servicios. [E04] *Conocimiento general:* primero se agrega lo nuevo de forma compatible, luego se migran productores y consumidores, y al final se elimina lo viejo. |
| Event Modeling | Método complementario que intercala wireframes, commands, eventos y read models en un storyboard de cambios y vistas. Fuente externa: [B01, p. 7]. |
| EventStorming | Técnica colaborativa y visual que parte de eventos de dominio para descubrir procesos, aggregates y bounded contexts. [T03, p. 41] |
| Fencing token | Número estrictamente creciente entregado en cada adquisición de un lease; el recurso rechaza operaciones con un token menor al mayor observado. [T12, p. 11] |
| Generic Subdomain | Parte necesaria pero no diferenciadora que suele poder resolverse con un producto o servicio de mercado. [T03, p. 20] |
| GraphQL | Lenguaje de consulta con schema tipado que permite al cliente pedir los campos necesarios; la cátedra lo orienta a BFF y frontends complejos. [T09, p. 14] [T09, p. 17] |
| gRPC | RPC tipado basado en Protocol Buffers y HTTP/2, con generación de código y streaming; se propone para comunicación interna service-to-service. [T09, p. 11] [T09, p. 17] |
| Healthcheck | Comando periódico ejecutado dentro del contenedor que informa estado `starting`, `healthy` o `unhealthy` según su código de salida; no reinicia por sí solo. [T07, p. 17] |
| Idempotencia | Propiedad por la cual procesar nuevamente la misma operación o mensaje no cambia el estado final. La consigna exige ese comportamiento ante reintentos y duplicados. La cátedra modela el ciclo de vida de una clave de idempotencia y exige que el registro y el efecto compartan una frontera atómica. [E01, p. 3] [E01, p. 4] [T12, p. 19] [T12, p. 20] Ver [Estado distribuido](temas/estado-distribuido.md). |
| Imagen de contenedor | Metadata y archivos organizados en capas inmutables y reutilizables, necesarios para crear un contenedor. [T06, p. 7] Ver [Imágenes](temas/imagenes-de-contenedores.md). |
| Inbox transaccional | Tabla de identificadores de mensaje con restricción `UNIQUE` que se inserta en la misma transacción que el cambio de dominio, para detectar reentregas; es el espejo del outbox del lado consumidor. [T12, p. 21] [T14, p. 50] |
| Integration test | Prueba de la interacción mínima entre servicios o con límites como HTTP y persistencia, aislando el resto cuando corresponde. [T10, p. 7] [T10, p. 8] La cátedra también usó oralmente `functional testing` como denominación; el material escrito separa alcance e intención. [N-2026-09-11-integration-test-functional-testing] [T10, p. 9] |
| Invariante de oro | Condición que debe valer tras cualquier secuencia de operaciones, fallos, reintentos y eventos duplicados o desordenados. Exige cantidades no negativas, ninguna copia perdida ni acreditada dos veces, contenido de sobre estable, ningún intercambio completado reejecutado y ninguna copia apartada por una oferta cancelada, vencida o rechazada de forma terminal. [E03, p. 5] Ver [Consigna general](temas/consigna-general-album-2026.md). |
| Kafka | Broker basado en un log distribuido, particionado y replicado, orientado a alto throughput, historial y replay. [T14, p. 7] [T14, p. 13] [T14, p. 36] |
| Layer cache | Reutilización de capas de build; un miss invalida la capa afectada y las posteriores. [T06, p. 20] [T06, p. 21] |
| Ley de Conway | Observación de que la estructura de un sistema tiende a reflejar la estructura de comunicación de la organización. [T02, p. 24] |
| Lock distribuido | Exclusión mutua entre instancias mediante un servicio compartido; necesita propietario y expiración porque la caída del proceso no lo libera. [T12, p. 9] [T12, p. 10] |
| Microservicio | Servicio con interfaz definida que puede ejecutarse, desplegarse y escalarse por separado. [T02, p. 4] [T02, p. 42] Ver [Fundamentos](temas/fundamentos-de-microservicios.md). |
| Monolito distribuido | Conjunto de despliegues separados que continúa acoplado por datos o diseño. [T02, p. 32] |
| Multi-stage build | Dockerfile con etapas separadas, donde la imagen de runtime recibe solo los artefactos copiados desde la etapa de compilación. [T06, p. 27] |
| Namespace | Vista aislada de un recurso del kernel, como procesos, red, montajes, UTS, IPC o usuarios. [T04, p. 14] |
| NATS | Broker ultraliviano con pub/sub, request-reply y queue groups; sin JetStream el mensaje se pierde si nadie está escuchando. [T14, p. 12] |
| OCI | Conjunto de contratos estándar para empaquetar imágenes y ejecutar contenedores mediante runtimes intercambiables. [T04, p. 29] |
| Offset | Próxima posición a leer de un consumer, guardada aparte del log; moverla hacia atrás permite replay. [T14, p. 38] |
| Open Host Service (OHS) | Interfaz estable mediante la cual un contexto ofrece capacidades a múltiples consumidores. [T03, p. 27] |
| Orquestador | Componente que define el flujo completo, envía comandos a cada participante, decide el siguiente paso y es el único que conoce la secuencia. [T11, p. 20] |
| OverlayFS | Filesystem de unión que combina capas de solo lectura con una capa escribible y presenta una vista `merged`. [T04, p. 19] [T04, p. 21] |
| Partición | Subdivisión de un topic de Kafka para paralelizar; el hash de la key elige la partición y el orden solo se garantiza dentro de ella. [T14, p. 32] [T14, p. 33] |
| Pipeline | Secuencia de etapas automatizadas de CI/CD, compuestas por jobs que corren en runners, que ante una falla vuelve al inicio para corregirla. [T13, p. 27] [T13, p. 29] |
| Poison message | Mensaje que no puede procesarse por payload corrupto o bug de deserialización; reintentarlo no lo arregla y debe ir a la DLQ. [T14, p. 52] |
| Published Language | Contrato público de integración, separado del modelo interno, como OpenAPI, AsyncAPI, JSON Schema o un esquema de eventos. [T03, p. 27] |
| Punto caliente (hot spot) | Recurso que concentra escrituras concurrentes bajo carga, como un código viral, una figurita muy buscada o una oferta popular, y puede volverse cuello de botella. [E03, p. 12] |
| Readiness | Señal de que un servicio puede aceptar trabajo; en Compose, `service_healthy` permite esperar esa condición. [T07, p. 18] En la segunda entrega, una readiness que siempre responde OK es un error de diseño. [E04] |
| Registry de imágenes | Catálogo público o privado que almacena imágenes y sus tags. [T06, p. 34] |
| REST | Estilo de API basado en recursos y verbos HTTP, habitualmente con JSON; la cátedra lo presenta como opción predeterminada para APIs públicas. [T09, p. 8] [T09, p. 17] |
| Rolling deployment | Actualización progresiva de los nodos hasta reemplazarlos todos; exige que convivan versiones retrocompatibles. [T13, p. 40] |
| RPC | Remote Procedure Call: paradigma que hace que invocar una función en otra máquina se vea como una llamada local, con transparencia de ubicación. [T11, p. 14] |
| runc | Runtime OCI que configura aislamiento, límites, rootfs y seguridad antes de ejecutar el proceso del contenedor. [T04, p. 31] |
| Saga | Transacción de negocio que cruza servicios como secuencia de pasos locales con compensaciones ante fallas, coordinada por coreografía u orquestación y con timeouts explícitos. [T14, p. 21] [T14, p. 27] [T14, p. 28] |
| seccomp | Mecanismo que filtra las syscalls permitidas a un proceso. [T04, p. 32] [T05, p. 48] |
| Separate Ways | Decisión de no integrar dos contextos y aceptar duplicación para preservar autonomía. [T03, p. 28] |
| Serializabilidad | Propiedad de una ejecución concurrente que produce el mismo resultado que algún orden secuencial válido de sus transacciones. [T11, p. 22] |
| Shared Kernel | Subconjunto de modelo compartido cuya evolución requiere coordinación entre los equipos participantes. [T03, p. 28] |
| Shift-left | Cultura de devolver al desarrollador cuanto antes los errores detectados por el pipeline, antes de que el problema llegue a producción. [T13, p. 45] |
| SLO | Objetivo de nivel de servicio para una operación crítica, como la latencia p99 al abrir un sobre o aceptar una oferta, o una disponibilidad mínima. La consigna pide definirlos y explicar cómo se validan. [E03, p. 12] [E04] |
| Subdomain | Recorte del problema de negocio dentro del dominio general. [T03, p. 19] [T03, p. 22] |
| Supporting Subdomain | Parte específica y necesaria del negocio que no constituye su diferenciador principal. [T03, p. 20] |
| Tag de imagen | Puntero con nombre hacia una imagen; tags como `latest` pueden moverse y no identifican contenido de manera inmutable. [T06, p. 35] |
| Test double | Sustituto usado durante una prueba: un stub devuelve respuestas fijas, un mock además verifica interacciones y un fake implementa comportamiento real simplificado. [T10, p. 6] |
| tmpfs | Filesystem residente en RAM, efímero y contabilizado contra el límite de memoria del contenedor; se usa para secretos o temporales que no deben persistir. [T08, p. 39] |
| Topic | Agrupación lógica de eventos del mismo tipo en Kafka; no es una cola, porque los eventos permanecen según la retención y cada consumidor lleva su offset. [T14, p. 32] |
| Transactional Outbox | Patrón que persiste el cambio de negocio y el evento pendiente en una transacción local para evitar el dual write; un relay los publica después con garantía at-least-once. [T14, p. 47] [T14, p. 48] También en la fuente externa [B01, p. 11]. Ver [Comunicación asincrónica y Kafka](temas/comunicacion-asincrona-y-kafka.md). |
| Two-Phase Commit (2PC) | Protocolo de commit atómico con fase de preparación, que persiste la intención y bloquea recursos, y fase de decisión del coordinador; es bloqueante y tiene un punto único de falla. [T11, p. 29] |
| Ubiquitous Language | Vocabulario preciso construido y usado por negocio y desarrollo dentro de un bounded context. [T03, p. 14] [T03, p. 17] |
| Valkey | Motor de datos in-memory, distribuido y open source, fork de Redis 7.2.4 bajo BSD-3-Clause y gobernanza de la Linux Foundation. [T12, p. 7] |
| Value Object | Objeto sin identidad propia, definido por sus atributos y normalmente inmutable. [T03, p. 36] |
| Volumen nombrado | Almacenamiento gestionado por Docker fuera de la unión OverlayFS y desacoplado del ciclo de vida del contenedor. [T08, p. 35] [T08, p. 37] |
| Whiteout | Marca en la capa superior de OverlayFS que oculta un archivo de una capa inferior sin modificar esa capa. [T04, p. 21] [T05, p. 50] |
