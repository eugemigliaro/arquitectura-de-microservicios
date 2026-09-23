# Estado distribuido y patrones con Valkey

## Un servicio, muchas instancias

Un microservicio es una unidad lógica desplegada como varios procesos. Cada instancia tiene memoria y ciclo de vida propios, el balanceador puede mandar cada request a una instancia distinta, y el autoscaling, los reinicios y los despliegues las reemplazan continuamente. [T12, p. 2]

Por eso aparecen carreras entre instancias: dos clientes reservan el mismo recurso, ambas instancias consultan la base, ven “disponible” y confirman. La secuencia consultar-decidir-actualizar debe preservar la invariante de negocio como una única operación. [T12, p. 3]

Las necesidades de sincronización se agrupan en coordinación —exclusión mutua, control de capacidad a N procesos y deduplicación— y en estado y comunicación —una representación común de datos, propagación de cambios y distribución de trabajo entre consumidores—. [T12, p. 4]

| Tipo de estado | Ejemplos | Propiedad esperada |
|---|---|---|
| Local y efímero | Request en curso, pool de conexiones, buffer | Puede perderse con la instancia |
| Compartido de coordinación | Cupos, leases, claves de idempotencia | Operaciones atómicas y expiración |
| Durable de negocio | Pedidos, pagos, inventario | Fuente de verdad y persistencia |
| Derivado o cacheado | Rankings, consultas frecuentes, vistas | Reconstruible; admite obsolescencia controlada |

Fuente de la tabla: [T12, p. 5]. La regla que se desprende es que las decisiones sobre recursos compartidos no pueden depender solo del estado local de una instancia. [T12, p. 5]

## Valkey

Valkey es un motor de datos in-memory, distribuido y de código abierto, nacido como fork de Redis 7.2.4 y que conservó inicialmente su protocolo, comandos y estructuras. Se publica con licencia BSD-3-Clause y gobernanza neutral en la Linux Foundation. Surgió en marzo de 2024, cuando Redis Ltd. anunció que las nuevas versiones abandonarían BSD por RSALv2 y SSPLv1; desde mayo de 2025 Redis 8 volvió a ofrecer una opción open source con AGPLv3, y ambos siguen como proyectos separados. [T12, p. 7]

## Atomicidad y locks

Un `GET` seguido de un `DECR` condicional no es atómico: entre la lectura y la escritura otra instancia puede ver el mismo valor y tomar la misma decisión. Una operación atómica verifica la precondición, modifica y devuelve el resultado sin intercalaciones; puede lograrse con una operación nativa, un script, una transacción o una actualización condicional en la base. [T12, p. 8]

Un mutex local serializa los threads de un proceso, pero dos procesos pueden entrar a la vez al recurso. El lock distribuido abarca varias instancias mediante un servicio compartido, cuesta red y almacenamiento, y como la caída del proceso no lo libera, requiere expiración. [T12, p. 9]

- **Adquisición:** `SET lock:recurso <owner-id> NX PX 30000` crea la key solo si no existe, con vigencia en milisegundos, y el `owner-id` identifica esa adquisición. [T12, p. 10]
- **Liberación condicional:** un script compara el valor con el `owner-id` y solo entonces borra, de forma atómica, para no liberar el lock de otro. [T12, p. 10]
- **Fencing token:** si una instancia pausa más que el TTL, su lease expira y otra lo adquiere. Cada adquisición recibe un token estrictamente creciente, y el recurso protegido rechaza escrituras con un token menor al mayor observado. El `owner-id` protege la liberación; el fencing token protege el recurso frente a operaciones atrasadas. [T12, p. 11]

## Estructuras y patrones

| Patrón | Estructura y comandos | Uso y comportamiento |
|---|---|---|
| Semáforo | Lista con un elemento por recurso; `BLPOP` para adquirir y `LPUSH` para liberar | Controla recursos limitados; con timeout distinto de 0, `BLPOP` devuelve `nil` si la lista sigue vacía [T12, p. 12] [T12, p. 13] |
| Cola de mensajes | Una lista por cola; `LPUSH` para encolar y `BRPOP`/`RPOP` para consumir | Varios workers compiten; timeout 0 bloquea indefinidamente [T12, p. 14] |
| Cache-aside / query caching | `GET` de una key derivada de la tabla y el id o de un `sha1` de la consulta; en un miss se consulta la base y se hace `SET ... EX 60` | Valkey compartido entre microservicios, cada uno con su base [T12, p. 15] [T12, p. 16] |
| Top N | Sorted set de IDs y valores, reordenado ante cada cambio | Rankings, conteo distribuido de eventos y procesamiento por lotes [T12, p. 17] |
| Hash map distribuido | Hash de pares atributo-valor bajo una key, con set, incremento, decremento y borrado | CRUD en memoria, típico para sesiones [T12, p. 24] |
| Pub/Sub | `PUBLISH` a canales creados al publicar; suscripción a uno o varios | Sin persistencia; entrega at-most-once: si el suscriptor no lo recibe, se pierde [T12, p. 25] |

## Duplicados e idempotencia

Las ejecuciones duplicadas son normales: un timeout no dice si la operación falló o si se perdió la respuesta; clientes, SDKs, gateways y servicios intermedios reintentan; un consumidor puede caer después del commit y antes del ACK; y un mensaje pendiente puede reentregarse durante la recuperación. [T12, p. 18]

Una clave de idempotencia pasa por `PROCESSING` y termina en `COMPLETED`, donde se repite la respuesta guardada, o en `FAILED`, donde un reintento puede estar permitido. El registro puede guardar identidad del cliente y la operación, hash de parámetros, estado y propietario, código y cuerpo de respuesta, timestamps y expiración. Reusar la misma key con parámetros distintos debe rechazarse. [T12, p. 19]

La deduplicación tiene dos ventanas de falla. Registrar antes del efecto y caer deja el mensaje marcado sin cambio aplicado, y las entregas siguientes se descartan; registrar después y caer hace que una nueva entrega vuelva a aplicar el cambio. Por eso el registro y el efecto de negocio deben compartir una frontera atómica, o debe existir un protocolo de recuperación. [T12, p. 20]

El **inbox transaccional** cumple esa condición: en una transacción inserta el `message_id` en una tabla con restricción `UNIQUE` (`ON CONFLICT DO NOTHING`), aplica el cambio de dominio solo si era nuevo y hace commit; recién después envía `XACK`. Si el ACK se pierde, la próxima entrega encuentra el identificador y omite el cambio. [T12, p. 21]

Con un set, `SADD claims job:42` devuelve 1 al primer worker y 0 a los siguientes, lo que impide la ejecución concurrente. Pero no prueba que el proceso haya terminado: la marca necesita expiración, finalización y recuperación explícitas. [T12, p. 22] [T12, p. 23]

## Semánticas de entrega

| Semántica | Qué garantiza | Qué admite |
|---|---|---|
| At-most-once | Entrega una vez o ninguna; no reentrega | Pérdida; ejemplo: Valkey Pub/Sub |
| At-least-once | Reintenta hasta la confirmación; no pierde mientras el mensaje esté disponible | Entregas y ejecuciones repetidas |
| Efecto idempotente | La infraestructura puede entregar varias veces, pero el consumidor reconoce el mensaje y aplica el cambio una sola vez | — |
| Exactly-once | Debe especificarse su alcance | Una entrega única no garantiza un efecto externo único; requiere coordinar consumo, estado y efectos |

Fuente de la tabla: [T12, p. 26]. Esta formulación precisa la afirmación de la lectura externa sobre “exactly once” mediante inbox; ver [DDD complementario](ddd-complementario.md).

La posición del ACK decide la falla: con ACK antes del efecto, una caída intermedia pierde el mensaje porque no se reentrega; con ACK después del commit, una caída antes del ACK provoca reentrega y un posible efecto doble si el consumidor no es idempotente. [T12, p. 27]

## Streams

Un stream es una secuencia ordenada donde el productor solo agrega al final (`XADD`). Cada lector recuerda su última posición, puede releer desde el principio y existen estrategias para recortar el comienzo y liberar memoria: lo anterior a la posición del consumidor más atrasado queda elegible para truncado. [T12, p. 29] [T12, p. 30]

La figura distingue consumidores independientes que leen todo con `XREAD` de un consumer group que se reparte el trabajo con `XREADGROUP` y confirma con `XACK`. [T12, p. 31] En un grupo, cada consumer tiene identidad única, cada elemento va a un solo consumidor del grupo, siempre se entrega el primer ID nunca consumido y se espera un ACK por mensaje. [T12, p. 32]

Si un consumidor falla antes del ACK, al volver puede leer sus pendientes con `XREADGROUP ... 0`; si su caída es permanente, otro puede reclamarlos con `XAUTOCLAIM` pasado un tiempo mínimo de inactividad. [T12, p. 33]

El log conserva el orden de sus IDs, pero repartirlo entre consumidores paralelos hace que el orden de finalización pueda diferir del de entrega, y productores independientes no establecen un orden causal. Si una entidad requiere orden, sus mensajes deben compartir una ruta serializada. [T12, p. 28]
