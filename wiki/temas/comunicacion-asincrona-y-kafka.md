# Comunicación asincrónica y Kafka

## Por qué asíncrono

La unidad retoma la demo de sobrecarga de [comunicación síncrona](comunicacion-sincrona.md): el flujo síncrono perdía casi todos los pedidos y la cola no perdía ninguno, pero llevaba la latencia a decenas de segundos. [T14, p. 5] Las cifras exactas de esa demo están en `T09`; el resumen de `T14` las redondea, ver [Dudas y conflictos](../dudas-y-conflictos.md). [T09, p. 29]

| Aspecto | Síncrono | Asíncrono |
|---|---|---|
| Acoplamiento temporal | Alto: emisor y receptor deben estar disponibles a la vez | Bajo: el broker media en el tiempo |
| Latencia percibida | La del eslabón más lento del camino | La del primer hop, el encolado |
| Falla en cascada | Se propaga directo | Amortiguada por el broker |
| Complejidad operativa | Baja | Alta: broker, DLQ, idempotencia, observabilidad |
| Trazabilidad | Directa, por el call stack | Indirecta: hay que reconstruir la cadena de eventos |

Fuente de la tabla: [T14, p. 6]. En la comunicación asíncrona el emisor no espera respuesta inmediata, lo que da desacoplamiento temporal y espacial a cambio de un intermediario —el message broker— y de más complejidad operacional. Los brokers presentados son RabbitMQ, un broker tradicional y flexible; Apache Kafka, un log distribuido de alto throughput; y NATS, ultraliviano y cloud-native. [T14, p. 7]

No conviene usar async cuando se necesita la respuesta para decidir el paso siguiente, el flujo es simple con un único consumidor, no se puede pagar la operación de un broker —equipo, monitoreo, on-call— o la consistencia inmediata importa más que el desacople. Async no es progreso automático: se paga con complejidad operativa lo que se gana en resiliencia y escala. [T14, p. 8]

## Fundamentos de mensajería

| | Cola (queue) | Publish/Subscribe |
|---|---|---|
| Entrega | Un mensaje a un grupo consumidor | Un mensaje a N suscriptores, de muchos grupos |
| Caso de uso | Distribuir trabajo entre workers | Notificar a múltiples servicios |
| Ejemplo | Procesar un pago con un worker | `OrderCreated` hacia pago, email y analytics |
| Herramientas | Colas de RabbitMQ, AWS SQS | Topics de Kafka, exchanges de RabbitMQ |

Fuente de la tabla: [T14, p. 10].

**CloudEvents** es una especificación de la CNCF para describir eventos, agnóstica de transporte y de lenguaje, usable sobre HTTP, Kafka, AMQP o NATS. Sus atributos comunes son `id`, `source`, `type`, `specversion` y `data`; el ejemplo agrega `time` y `datacontenttype`, con un `type` como `com.itba.order.created`. **Convención de cátedra:** todos los labs event-driven usan CloudEvents, y los que tienen comunicación asíncrona incluyen un `asyncapi.yaml`. [T14, p. 11]

**NATS** es un binario Go sin dependencias que ofrece pub/sub, request-reply y queue groups. Sin JetStream, si nadie está escuchando el mensaje se pierde; JetStream agrega un stream persistente con offset por consumidor, que sobrevive a un consumidor caído. [T14, p. 12]

| | NATS | RabbitMQ | Kafka |
|---|---|---|---|
| Modelo | Pub/sub + request-reply | Broker con exchanges | Log distribuido |
| Persistencia | Opcional (JetStream) | Sí, hasta consumir | Sí, con retención configurable |
| Setup | Un binario, sin configuración | Erlang y plugins | Cluster de brokers |
| Ideal para | Mensajería simple y rápida | Routing flexible | Alto throughput, replay, event sourcing |

Fuente de la tabla: [T14, p. 13].

## Event-Driven Architecture

En EDA los servicios emiten eventos que describen hechos ocurridos y otros reaccionan; el emisor no sabe ni le interesa quién escucha. Un **evento** dice “algo pasó” (`OrderCreated`): está en pasado, es inmutable y no tiene expectativa. Un **comando** dice “hacé algo” (`ProcessPayment`): es imperativo y espera resultado. [T14, p. 15] La distinción coincide con la de command y domain event en [DDD táctico](ddd-tactico-y-eventstorming.md). [T03, p. 38]

La coreografía de venta vista antes —cada servicio reacciona a lo que pasó y nadie orquesta desde afuera— ya es EDA aplicada a un flujo de negocio; esta unidad le agrega el broker real. [T14, p. 16]

| Forma | Qué lleva | Costo |
|---|---|---|
| Event notification | Payload mínimo (`OrderCreated { orderId }`); quien necesite más, pregunta | Liviano, pero genera llamadas de vuelta |
| Event-carried state transfer | Estado completo (`orderId`, ítems, total, cliente); nadie vuelve a preguntar | Autosuficiente, pero duplica datos entre servicios |
| Comando | No es un evento: instrucción dirigida con expectativa de resultado (`ChargePayment`) | — |

Fuente de la tabla: [T14, p. 17].

**Claim check:** un state transfer sin límites produce eventos gigantes con imágenes, PDFs o PII pesada. El evento lleva entonces un puntero (`uri`, `blobId`) al payload guardado en un store (S3, una base, object storage); quien necesita el detalle lo descarga y el resto reacciona solo al hecho. El bus transporta el hecho, no el archivo, y no es todo o nada: puede llevar metadata rica y dejar el blob afuera. [T14, p. 18]

**CQRS light:** el modelo que escribe no tiene por qué ser el que se consulta. El servicio de escritura emite eventos y uno o más read models (projections) los consumen y materializan vistas optimizadas para lectura; con Kafka, el mismo log alimenta facturación, analytics y una proyección de “mis órdenes” sin acoplar al writer. No es Event Sourcing completo, donde el historial es la única fuente de verdad: solo separa write path de read path, y el bus es el puente. [T14, p. 19]

## Saga en la práctica

La unidad retoma el flujo venta → reserva de stock → cobro, con compensación si algo falla: `SALE_REQUESTED`, `STOCK_RESERVED`, `PAYMENT_FAILED` con compensación `ROLLBACK_INVENTORY`, y cierre `CANCELED`. [T14, p. 21] Así nombra **Saga** al ejemplo de coreografía de [sistemas distribuidos](sistemas-distribuidos.md). [T11, p. 38] [T11, p. 39]

En la práctica cada flecha es un evento CloudEvents publicado en un topic de NATS, como `orders.inventory.reserve-requested`, `orders.inventory.reserved` u `orders.payment.failed`; nadie llama a nadie y cada servicio se suscribe a lo que le importa. [T14, p. 22] La compensación coreografiada es una cadena: Payment publica `PaymentFailed`; Inventory, suscripto, publica `InventoryReleased`; Orders, suscripto a este, publica `OrderCancelled`. Son tres saltos y tres servicios sin un dueño único del flujo de compensación. [T14, p. 23]

A medida que un flujo crece en pasos, la coreografía pura se vuelve difícil de trazar y depurar. Por eso Uber construyó Cadence (~2016), un motor de orquestación de workflows durable para el ciclo de vida de un pedido; sus autores fundaron luego Temporal, su sucesor open source. [T14, p. 24] Con orquestación, un workflow llama a activities como funciones normales y la compensación deja de ser “publicar y esperar que alguien reaccione” para convertirse en una rama `try`/`except` en el mismo lugar que el resto de la lógica. [T14, p. 25]

- **Durabilidad:** el motor persiste el estado del workflow en cada paso. Si el worker cae justo después de reservar inventario, al reiniciar retoma ahí, sin perder la compensación ni cobrar dos veces. Es la misma preocupación que resuelve el outbox, aplicada al orquestador. [T14, p. 26]
- **Timeouts:** cada paso y la Saga entera necesitan un deadline explícito. Al vencer, se toma la rama de compensación como ante un `PaymentFailed`; en Temporal y Cadence los timers son durables. Sin timeout, la compensación es teoría y el flujo se pudre en un estado intermedio. [T14, p. 27]

| | Coreografía | Orquestación (Temporal) |
|---|---|---|
| Dueño del flujo | Implícito, repartido entre servicios | Explícito: un workflow |
| Debugging | Difícil, sin dueño único | Directo: un trace por Saga |
| Acoplamiento | Bajo: cada servicio conoce su parte | El orquestador conoce todos los pasos |
| Escalabilidad y autonomía de equipos | Alta | Media: depende del orquestador |
| Conviene cuando… | Pocos pasos, equipos autónomos | Sagas largas, muchos pasos, visibilidad end-to-end |

Fuente de la tabla: [T14, p. 28].

## Cómo funciona Kafka

Kafka aparece cuando integrar sistemas a gran escala se vuelve un problema en sí mismo: los datos se intercambian continuamente, las integraciones punto a punto acoplan, productores y consumidores trabajan a ritmos distintos, y muchas aplicaciones deben reaccionar en tiempo real y conservar historial para reprocesar. [T14, p. 30]

### Log, topics y particiones

La idea central es el **log**: una secuencia ordenada de registros inmutables que solo se escribe al final. Responde qué pasó y en qué orden, y leer no elimina el dato. Consumir no es sacar un mensaje: es avanzar una posición de lectura. [T14, p. 31]

Un **topic** agrupa eventos del mismo tipo, pero no es una cola: los eventos permanecen según la política de retención y cada consumidor lleva su propio offset. Se divide en **particiones** para paralelizar, y Kafka garantiza orden dentro de una partición, no entre particiones del mismo topic. [T14, p. 32]

La **key** decide la partición: `partición = hash(key) % cantidad_de_particiones`. Misma key, misma partición y orden relativo preservado para esa entidad; keys típicas son `orderId`, `customerId` o `accountId`. Sin key, Kafka reparte round-robin sin garantía de orden. [T14, p. 33] La demo concreta el hash como Murmur2 sobre los bytes de la key, `(hash & 0x7fffffff) % 3` para tres particiones, y remarca que no existe orden global entre particiones. [T14, p. 34]

**Anti-pattern de key:** el bug de orden casi nunca es “Kafka se rompió”. Sin key, eventos de la misma orden caen en particiones distintas y el consumidor puede ver `Paid` antes que `Created`; con una key inestable (un `uuid()` por mensaje o un campo que cambia), la entidad salta de partición; con una key demasiado gruesa (todo por `tenantId`), una partición queda caliente y el resto ociosa. La key es parte del contrato de orden y se elige por la entidad cuyo orden relativo importa. [T14, p. 35] Es la versión Kafka de la regla general de `T12`: si una entidad requiere orden, sus mensajes deben compartir una ruta serializada. [T12, p. 28]

### Brokers, producers y consumers

Un cluster tiene varios brokers y las particiones se distribuyen entre ellos: el topic es lógico, el broker es físico. Cada partición puede tener réplicas, una líder y el resto seguidoras; si un broker cae, otra réplica asume como líder y Kafka tolera la falla sin perder datos. [T14, p. 36]

El producer escribe records en un topic, con o sin key. El consumer lee con modelo **pull**: decide hasta dónde avanzó y puede reiniciarse y continuar. Ninguno necesita conocer al otro, solo el topic. [T14, p. 37]

Cada consumer lleva su **offset** —la próxima posición a leer— guardado aparte del log, que no cambia cuando se lee. Por eso dos consumers pueden estar en offsets distintos del mismo topic y Kafka permite **replay**: volver el offset a 0 sin perder ni duplicar el historial. [T14, p. 38] En la demo, sobre un log de seis eventos, Facturación tiene próximo offset 4 y lag 2, y Analytics próximo offset 2 y lag 4. [T14, p. 39]

### Consumer groups

Dentro de un consumer group, cada partición se asigna a un solo consumer a la vez. El paralelismo máximo es la cantidad de particiones: un quinto consumer con cuatro particiones queda ocioso. Si un consumer cae, el grupo rebalancea y redistribuye sus particiones. [T14, p. 40] La demo reparte cuatro particiones entre dos consumers, P0 y P2 para uno, P1 y P3 para el otro. [T14, p. 41]

| Mismo consumer group | Distintos consumer groups |
|---|---|
| Competing consumers: cada mensaje lo procesa un miembro | Fan-out: cada grupo recibe todos los mensajes |
| Escala horizontal de workers sobre el mismo topic | Facturación, email y analytics leen el mismo historial a su ritmo |
| Equivale al modelo cola | Equivale a pub/sub y habilita CQRS |

Fuente de la tabla: [T14, p. 42].

### Consumer lag

**Lag** es la diferencia entre el final del log y el offset del consumer. Es la métrica operativa principal: la pregunta no es si el broker está arriba, sino si el consumo está al día. Las respuestas típicas son más consumers (hasta la cantidad de particiones), más particiones, optimizar el handler o aceptar latencia en el pico. Kafka no es un buffer infinito e invisible: el lag es la factura del desacoplamiento temporal. [T14, p. 43]

### Cuándo usar Kafka

| Usar Kafka cuando… | No usarlo cuando… |
|---|---|
| Varios sistemas deben reaccionar al mismo evento | Alcanza una llamada síncrona simple |
| Se necesita desacople temporal y absorber picos | Se necesita request/response inmediato |
| Se necesita historial, auditoría o replay | No hay múltiples consumidores ni necesidad de historial |
| El volumen de eventos es alto | La complejidad no se justifica: NATS o una cola simple resuelven mejor |

Fuente de la tabla: [T14, p. 44].

## Patrones de confiabilidad

### Dual write y Transactional Outbox

Un servicio que actualiza su base y después publica un evento puede caer entre ambos pasos: la orden queda guardada y nadie se entera, o el evento sale y el commit falla. Es el **dual write**, porque base y broker no comparten transacción. Publicar “a mano” antes o después del commit es un anti-pattern; lo correcto es Outbox o CDC. [T14, p. 46]

El **Transactional Outbox** no publica el evento: lo guarda como una fila más en la misma base que el cambio de negocio.

1. El request escribe la orden en `orders` y el evento en `outbox` en una sola transacción: con commit existen las dos, con rollback ninguna.
2. Un proceso aparte, el **relay** (poller o CDC), lee las filas pendientes y las publica al broker.
3. Recién cuando el broker confirma, el relay marca la fila como publicada.

La atomicidad se resuelve con la transacción de la base, no con el broker. [T14, p. 47] En el ejemplo concreto, la fila de outbox guarda el id de CloudEvents, el `aggregate_id` que se usa como key de Kafka, el tipo versionado (`order.created.v1`) y el payload; el relay selecciona pendientes con `FOR UPDATE SKIP LOCKED`, publica, espera el ack y actualiza `published_at`. [T14, p. 48]

| Falla | Resultado |
|---|---|
| Crash antes del `COMMIT` | Ni orden ni evento |
| Kafka caído | La orden entra; el evento espera en outbox |
| Relay cae antes de publicar | Sigue pendiente y se publica después |
| Relay cae entre publicar y el `UPDATE` | Se publica dos veces |

Fuente de la tabla: [T14, p. 48]. La garantía es **at-least-once**: nunca se pierde, pero puede duplicarse, y el consumidor deduplica por id con un Inbox. [T14, p. 48]

| Poller | CDC (Change Data Capture) |
|---|---|
| Un worker hace `SELECT` periódico de pendientes y publica | Herramientas como Debezium leen el write-ahead log de la base y publican a Kafka |
| Simple; la latencia es el intervalo de poll | Sin código de aplicación publicando; menos latencia artificial |
| Cuidado con locks y con marcar “publicado” de forma idempotente | Agrega una pieza operativa, el connector |

Fuente de la tabla: [T14, p. 49]. La lectura externa `B01` describe el mismo patrón y el mismo trade-off; ver [DDD complementario](ddd-complementario.md).

### Transactional Inbox y consumidor idempotente

El **Inbox** es el espejo del outbox del lado consumidor: llega un mensaje, quizá reintento; en una misma transacción se inserta `message_id` en `inbox` y se aplica el efecto de negocio; si el id ya existía, la clave duplicada lleva a rollback y no-op. En ambos casos el consumer luego hace commit del offset. El outbox evita “guardé y no avisé”; el inbox, “avisé y apliqué dos veces”. [T14, p. 50]

La lámina resume: at-least-once en el bus más Outbox e Inbox en los bordes equivale aproximadamente a un efecto exactamente una vez **a nivel de negocio**. [T14, p. 50] Es la misma idea que `T12` llama efecto idempotente, con Valkey en lugar de una base. [T12, p. 21] [T12, p. 26]

Regla para el TP: Kafka, como casi todo broker, entrega at-least-once en la práctica y los reintentos van a ocurrir. La clave de dedupe es el id de CloudEvents o una business key estable, y el efecto debe ser seguro ante repetición mediante Inbox, upsert o “ya procesado → ACK”. Creer que el exactly-once (EOS) de Kafka vuelve innecesaria la idempotencia es un anti-pattern: no reemplaza el diseño del handler. [T14, p. 51]

### Retries, DLQ y poison messages

- **Reintentos con backoff:** un fallo transitorio se reintenta espaciando los intentos.
- **Dead-letter queue (DLQ):** los mensajes que fallaron N veces se apartan para no bloquear al resto.
- **Poison message:** payload corrupto o bug de deserialización; reintentarlo para siempre no lo arregla y va a la DLQ para revisión manual. [T14, p. 52]

En Kafka, reintentar para siempre es peor de lo que parece: un consumer que no avanza su offset atrasa toda esa partición —el lag crece solo ahí— y la entidad asociada se pudre mientras el resto del grupo sigue. La regla es tope de intentos, luego DLQ u otro topic, luego alerta humana. [T14, p. 53]

### Correlación y evolución de esquema

La trazabilidad async se resuelve en el contrato del evento, no solo con un APM. `correlationId` identifica el flujo de negocio entero, como el `orderId` o el id de la Saga, y lo comparten todos sus eventos; `causationId` es el id del evento o request que causó este, y arma el árbol padre → hijo. Viajan como extensiones CloudEvents y se retoman con OpenTelemetry en la clase de Observabilidad. [T14, p. 54]

Los consumidores no se despliegan al mismo tiempo que el productor, así que el contrato debe tolerarlo: versionar el `type` (`com.itba.order.created.v2`) o mantener compatibilidad hacia atrás, y preferir cambios aditivos con campos opcionales antes que renombrar o borrar. Un breaking change mal anunciado se convierte en poison messages en cascada; AsyncAPI y CloudEvents son el contrato que evita ese silent break. [T14, p. 55]

## Relación con estado distribuido

La unidad no repite lo visto con Valkey: semánticas de entrega, mecánica de claves y ventanas de falla de idempotencia, orden bajo concurrencia entre instancias e Inbox con Valkey. Ese material está en [Estado distribuido](estado-distribuido.md) y se recomienda repasarlo con la mecánica de Kafka presente. [T14, p. 56] Para profundizar, la cátedra menciona otro deck de Kafka —segments, tuning de replicación, pipelines de datos y event sourcing— que no está incorporado en este repositorio, y la documentación de Temporal. [T14, p. 62]

## Checklist y anti-patterns

| Pregunta | Decisión |
|---|---|
| ¿Necesitás la respuesta ya para seguir? | Síncrono |
| ¿Un evento con varias reacciones independientes? | Pub/sub, EDA, fan-out con distintos consumer groups |
| ¿Replay o varios consumidores sobre el mismo historial? | Kafka |
| ¿Mensajería simple y liviana sin gran volumen? | NATS |
| ¿Separar escritura de lecturas optimizadas? | CQRS light sobre el mismo log |
| ¿Payload pesado o PII? | Claim check, no fat event |
| ¿Una transacción de negocio cruza varios servicios? | Saga con timeouts: coreografía con pocos pasos y equipos autónomos, orquestación con muchos pasos y necesidad de visibilidad |
| ¿Dual write entre base y broker? | Outbox (poller o CDC) e Inbox en el consumidor |

Fuente de la tabla: [T14, p. 58].

Los anti-patterns enumerados son: [T14, p. 59] [T14, p. 60]

- **Monolito distribuido vía eventos:** todos escuchan todo y nadie entiende el flujo.
- **God topic:** un único topic `events` para todo el sistema; conviene usar topics y tipos acotados.
- **Chatty o ping-pong events:** A emite, B emite, A reacciona; es RPC disfrazado de EDA.
- **Sync-over-async:** pedir respuesta inmediata sobre el canal asíncrono, o polling y request-reply para todo.
- **Acoplamiento temporal disfrazado:** “es async”, pero el flujo no avanza si el otro no responde ya.
- **Async sin observabilidad:** sin `correlationId`, `causationId` ni traces.
- **Dual write sin Outbox**, **asumir exactly-once**, **key ausente o inestable** y **retry forever sin DLQ**.
- **Fat events o PII en el bus:** payloads enormes o sensibles viajando a N consumidores.
- **Ignorar el consumer lag:** el broker está verde mientras el SLA de frescura ya se rompió o el disco se llena.
- **Orquestador-monolito:** un workflow que conoce los internos de cada servicio.
- **Compensación síncrona sin deadline:** liberar stock “cuando se pueda” sin timeout de Saga.

## Aplicación al Álbum 2026

*Inferencia:* la consigna del Álbum admite que demoren el porcentaje de avance, el progreso de retos, los rankings y la actividad reciente, y sus escenarios de aceptación incluyen eventos de apertura o intercambio duplicados, eventos de progreso fuera de orden y actualización diferida. [E01, p. 5] [E01, p. 6] Esta unidad aporta herramientas para esos casos: consumidores idempotentes con Inbox, keys por entidad para preservar orden relativo, CQRS light para las vistas que pueden demorar y consumer lag como medida de esa demora. Ninguna fuente oficial asigna todavía estos patrones a requisitos concretos de la entrega.
