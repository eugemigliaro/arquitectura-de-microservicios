# Sistemas distribuidos

## Centralizar o distribuir

Distribuir no es gratis: un sistema centralizado conserva un único artefacto desplegable, debugging directo, transacciones ACID reales, latencia mínima y consistencia inmediata. Se pierden esas ventajas a cambio de otras, por lo que distribuir no siempre es buena idea. [T11, p. 2]

Las razones presentadas para distribuir son que la aplicación sea inherentemente distribuida, como un mensaje entre dos celulares; mejorar la confiabilidad, porque el sistema sigue funcionando aunque falle un nodo; mejorar la performance acercando el dato al usuario; resolver problemas cuyos datos no caben en una máquina; y tolerar la falla de algunas partes. [T11, p. 5]

El modelo básico son nodos que intercambian mensajes por un canal, sea HTTP, Kafka u otro, ya como request o como evento. En la arquitectura tipo, una capa de middleware distribuido se ubica entre las aplicaciones y los sistemas operativos locales de cada máquina, conectadas por la red. [T11, p. 4] [T11, p. 6]

## Modelos de falla

El comportamiento de un sistema distribuido se describe por tres ejes: la red, los nodos y el tiempo. [T11, p. 7]

| Eje | Modelos | Idea clave |
|---|---|---|
| Red | Enlace confiable; fair-loss; enlaces arbitrarios; demoras arbitrarias; partición de red | En un enlace confiable un mensaje enviado llega, aunque tal vez reordenado; en fair-loss puede perderse, duplicarse o reordenarse; en un enlace arbitrario un adversario puede espiar, modificar, eliminar o replicar mensajes, lo que se mitiga con TLS; una partición separa grupos que sí se comunican internamente [T11, p. 8] |
| Nodos | Crash-stop; crash-recovery; bizantino | El nodo se detiene para siempre, o se cae perdiendo el estado en memoria y puede volver, o hace cualquier cosa, incluso comportarse maliciosamente. Un nodo no sabe si otro está correcto o fallando [T11, p. 9] |
| Tiempo | Sincrónico; asincrónico | El modelo sincrónico acota la latencia y la velocidad de ejecución; el asincrónico no da garantías de tiempo [T11, p. 10] |

En la práctica la sincronía se degrada: la pérdida obliga a reintentar, la congestión genera colas, la red se reconfigura y los nodos sufren pausas por el scheduler, el recolector de basura, page faults, swap o thrashing. [T11, p. 10]

La conclusión de la unidad: no hay reloj global confiable ni memoria compartida, la red falla “y miente”, no se puede distinguir un nodo caído de uno lento y no existe un estado único global. [T11, p. 11]

## RPC

RPC es el paradigma dominante para la comunicación síncrona entre servicios: busca que invocar una función en otra máquina se sienta igual que una llamada local, con transparencia de ubicación. [T11, p. 14] El ejemplo de pago muestra cómo un stub del cliente serializa (marshal) los argumentos en un mensaje, el servidor los deserializa, ejecuta la implementación real y devuelve el resultado por el mismo camino mientras el cliente espera. [T11, p. 16] [T11, p. 17]

La interfaz es simple, pero el comportamiento no: la latencia es variable, hay fallos parciales, un timeout no demuestra que la operación falló y los reintentos pueden duplicarla. [T11, p. 18] Estos problemas conectan con [comunicación síncrona](comunicacion-sincrona.md) y con la idempotencia desarrollada en [estado distribuido](estado-distribuido.md).

## Orquestación, atomicidad y concurrencia

Un orquestador —un servicio dedicado o parte de uno existente— define y controla todo el flujo, envía comandos explícitos a cada participante, decide el siguiente paso según cada respuesta y es el único que conoce la secuencia completa. Puede comunicarse de forma síncrona o asíncrona. [T11, p. 20]

Cuando una operación modifica datos en varios nodos, el desafío es que el resultado no quede parcial: atomic commit busca que todos los participantes confirmen o ninguno. Si uno confirma y otro falla, el sistema queda inconsistente. [T11, p. 21]

Con transacciones concurrentes, una ejecución es **serializable** si produce el mismo resultado que algún orden secuencial válido; esto evita lecturas inconsistentes y pérdidas de actualización. [T11, p. 22] El ejemplo parte de `X = 10`, `Y = 10` y la invariante `X + Y = 20`: una transferencia suma 1 a `X` y resta 1 a `Y` mientras una auditoría lee ambos. Los órdenes T1→T2 y T2→T1 muestran resultados consistentes, pero intercalar la auditoría entre las dos escrituras imprime `11, 10`, una suma de 21 que “crea dinero”. [T11, p. 23] [T11, p. 24] [T11, p. 25] [T11, p. 26]

| | Control pesimista | Control optimista |
|---|---|---|
| Filosofía | Prevenir antes que lamentar | Confiar y verificar |
| Mecanismo | Bloquear antes de operar | Detectar conflictos al finalizar |
| Costo | Esperas y locks | Reintentos |
| Cuándo | Conflictos frecuentes | Conflictos poco frecuentes |
| Ejemplo | Reserva de asiento de avión | Edición de perfil de usuario |

Fuente de la tabla: [T11, p. 27]. El locking optimista se implementa con versionado (`version`, `updatedAt`), compare-and-swap y reintentos automáticos: si la versión leída cambió antes del `UPDATE`, hay conflicto y se reintenta. [T11, p. 28]

## Two-Phase Commit

2PC coordina una transacción real entre varios nodos: todos confirman o todos abortan. En la fase de preparación cada participante persiste su intención y bloquea recursos; en la de decisión, el coordinador decide ejecutar o no. La figura muestra `Prepare` y votos en la primera fase, y `Decision` y `Ack` en la segunda. Sus costos son que es bloqueante, los locks largos afectan la concurrencia, el coordinador es un punto único de falla y escala con dificultad. [T11, p. 29] [T11, p. 31]

La lámina siguiente explica la acumulación de locks hasta el final de la transacción para forzar la serialización, advierte que liberarlos a mitad de camino rompe la serializabilidad y que este locking produce deadlocks con facilidad. [T11, p. 30] *Inferencia:* esa descripción corresponde al locking en dos fases (2PL), una técnica de control de concurrencia distinta del protocolo de commit 2PC, aunque suelen usarse juntas; ver [Dudas y conflictos](../dudas-y-conflictos.md).

| Escenario de crash | Consecuencia |
|---|---|
| Un participante cae antes de votar | El coordinador nunca puede commitear [T11, p. 32] |
| Un participante cae después de votar SÍ | Se comprometió aunque todavía no ejecutó; los demás harán su parte, por lo que su voto debe ser durable [T11, p. 32] |
| Llegan commits repetidos | El participante reenvía el ACK [T11, p. 33] |
| El coordinador cae antes de enviar commits | Se aborta [T11, p. 33] |
| El coordinador cae después de algunos commits | Debe recordar la transacción de forma durable [T11, p. 33] |
| Vence un timeout | El coordinador puede abortar unilateralmente; los participantes no pueden decidir solos y deben mantener el lock indefinidamente [T11, p. 34] |

Conviene cuando se necesita consistencia inmediata, sin estados intermedios ni compensaciones: sistemas financieros core, transacciones entre bases de datos, operaciones legales o regulatorias y sistemas legacy. [T11, p. 35]

## Coreografía

En la coreografía no hay orquestador: cada servicio reacciona a eventos o invoca a otros según su rol, y el flujo de negocio queda implícito en la suma de las partes. Suele basarse en eventos, aunque puede ser síncrona. No tiene punto único de falla, escala bien, reduce el acoplamiento y permite evolucionar cada servicio por separado; a cambio, el flujo es difícil de entender y mantener, el debugging es complejo por falta de dueño y requiere observabilidad. [T11, p. 37]

El ejemplo visual, verificado sobre las láminas, recorre una venta con Kafka:

1. `microservice-sale` guarda la venta como `PENDING` y publica `CREATED_SALE`.
2. `microservice-inventory` separa stock (`SEPARATED`) y publica `UPDATED_INVENTORY`.
3. `microservice-payment` registra `PAID` y publica `VALIDATED_PAYMENT`, que lleva la venta a `FINALIZED`. [T11, p. 38]

Ante un fallo, el pago publica `PAYMENT_FAILED`; inventario hace `ROLLBACK` y publica `ROLLBACK_INVENTORY`; y la venta queda `CANCELED`. [T11, p. 39] *Inferencia:* es una saga coreografiada con acciones compensatorias, la alternativa a 2PC cuando se aceptan estados intermedios; la cátedra no usa todavía la palabra saga en esta unidad, aunque el programa la anticipa. [T01, p. 11]
