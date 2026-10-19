# Estado distribuido y Sistemas distribuidos — Resumen para repasar

Síntesis del material oficial: **T12 — Estado distribuido: patrones con Valkey** y **T11 — Sistemas distribuidos**. Los ejemplos señalados como didácticos son de elaboración propia.

## Cómo se conectan los dos temas

**Sistemas distribuidos:** varias máquinas colaboran enviándose mensajes. ¿Qué pasa si la red falla, un nodo cae o dos operaciones se pisan?

**Estado distribuido:** varias instancias necesitan coordinarse sobre datos y recursos. ¿Cómo evitamos reservar dos veces el último cupo, duplicar un pago o perder una tarea? [T11, p. 4] [T12, p. 3] [T12, p. 4]

---

## Parte 1. Estado distribuido

### 1. La memoria de una instancia no coordina a las demás

Un servicio puede tener varias instancias. Cada una tiene su propia memoria y puede reiniciarse o desaparecer. Por eso, una decisión sobre un recurso compartido no puede depender solamente de una variable local. [T12, p. 2] [T12, p. 5]

**Ejemplo didáctico:** queda una entrada. A y B leen «hay una». Ambas venden y recién después descuentan. El problema está en que **leer → decidir → actualizar** no se ejecutó como una única operación atómica.

Aunque `GET` y `DECR` sean individualmente atómicos, su combinación no lo es: otra instancia puede intercalarse. Hay que verificar la condición y modificar el dato sin esa ventana, por ejemplo mediante un script atómico o una actualización condicional en la base. [T12, p. 8]

| Estado | Ejemplo | Qué necesita |
|---|---|---|
| Local y efímero | Request en curso, pool de conexiones | Puede perderse con la instancia |
| Compartido de coordinación | Cupos, locks, claves de idempotencia | Atomicidad y políticas de expiración |
| Durable de negocio | Pagos, pedidos, inventario | Persistencia y fuente de verdad |
| Derivado | Caché, rankings | Poder reconstruirse; admite obsolescencia controlada |

[T12, p. 5]

### 2. Lock distribuido: una instancia a la vez

Un **mutex local** coordina threads dentro del mismo proceso. Un **lock distribuido** coordina instancias distintas mediante un servicio compartido. Valkey es un motor de datos en memoria que ofrece estructuras y operaciones para estos patrones. [T12, p. 7] [T12, p. 9]

Adquisición presentada en clase:

```text
SET lock:recurso <owner-id> NX PX 30000
```

- `NX`: crear solo si la clave no existe.
- `PX 30000`: vigencia de 30 segundos.
- `owner-id`: identifica esa adquisición del lock.

La expiración permite recuperar el recurso si el dueño cae. Para liberar, se debe **comparar el propietario y borrar de forma atómica**. Un `DEL` incondicional podría borrar el lock de otra instancia que lo adquirió después. [T12, p. 10]

**El TTL no alcanza para proteger las escrituras.** Si A pausa más que el TTL, B puede adquirir y escribir; cuando A vuelve, todavía podría intentar una escritura atrasada.

Un **fencing token** es un número creciente por adquisición. El recurso protegido conserva el mayor token observado y rechaza operaciones con uno anterior. **Owner-id protege la liberación; fencing protege el recurso frente al dueño atrasado.** El recurso tiene que validar el token. [T12, p. 11]

### 3. Elegir el patrón según la necesidad

| Necesidad | Patrón | Idea y operaciones |
|---|---|---|
| Un solo proceso en una sección crítica | Lock | Adquirir con propietario y expiración; liberar condicionalmente |
| Hasta N procesos usando recursos | Semáforo | Lista de permisos: `BLPOP` adquiere, `LPUSH` devuelve |
| Repartir tareas entre workers | Cola | `LPUSH` encola; `BRPOP`/`RPOP` extrae; los workers compiten |
| Evitar consultas repetidas a la base | Cache-aside | `GET`; si falta, consultar la base y guardar con TTL |
| Mantener un ranking | Sorted set | IDs ordenados por un valor |
| Guardar atributos de una sesión | Hash | Pares atributo–valor bajo una clave |

[T12, p. 10] [T12, p. 12] [T12, p. 14] [T12, p. 16] [T12, p. 17] [T12, p. 24]

**Cache hit:** el dato estaba en caché. **Cache miss:** se busca en la base y se carga en caché. La caché es estado derivado; puede estar desactualizada dentro de lo que admita el negocio. [T12, p. 5] [T12, p. 16]

**Trampa:** en comandos bloqueantes como `BLPOP`, timeout `0` significa esperar indefinidamente. Con un timeout finito, puede devolver `nil` si no aparece un permiso. [T12, p. 13] [T12, p. 14]

### 4. Idempotencia: repetir la solicitud sin repetir el efecto

Un timeout no prueba que la operación falló. Podría haberse realizado y haberse perdido la respuesta. También puede haber mensajes reentregados después de una caída. [T12, p. 18]

La **clave de idempotencia** identifica una operación lógica. El registro sigue su estado:

- `PROCESSING`: se está ejecutando.
- `COMPLETED`: ya terminó; se puede repetir la respuesta guardada.
- `FAILED`: falló; un reintento puede permitirse según la política.

La misma clave con parámetros diferentes debe rechazarse. **Ejemplo didáctico:** no se puede reutilizar la clave de un pago de $100 para uno de $500. [T12, p. 19]

**Registrar «procesado» y aplicar el cambio por separado deja una ventana de falla:**

| Orden | Si el proceso cae en el medio… |
|---|---|
| Marcar procesado → aplicar efecto | La próxima entrega se descarta, pero el efecto nunca ocurrió |
| Aplicar efecto → marcar procesado | La próxima entrega puede repetir el efecto |

[T12, p. 20]

El **Inbox transaccional** resuelve esa ventana para cambios dentro de la misma base:

1. Iniciar una transacción.
2. Registrar el ID del mensaje con restricción de unicidad.
3. Si es nuevo, aplicar el cambio de negocio; si ya existe, omitirlo.
4. Confirmar la transacción.
5. Enviar el ACK.

Si se pierde el ACK, la reentrega encuentra el ID y no repite el cambio. **La marca y el efecto se confirman juntos.** [T12, p. 21]

Un `SADD` puede impedir que dos workers adquieran simultáneamente el mismo trabajo, pero **adquirir no demuestra haber completado**: hacen falta finalización y recuperación explícitas. [T12, p. 22] [T12, p. 23]

### 5. Entrega de mensajes y posición del ACK

**ACK** significa confirmación de procesamiento.

| Garantía | Qué admite |
|---|---|
| At-most-once | Una entrega o ninguna: admite pérdida, sin reentrega |
| At-least-once | Reentrega hasta confirmar, mientras el mensaje siga disponible: admite duplicados |
| Efecto idempotente | Varias entregas, un único cambio de negocio |
| Exactly-once | Hay que precisar el alcance: una entrega única no garantiza un efecto externo único |

[T12, p. 26]

**ACK antes del efecto:** si el consumidor cae después del ACK, puede perderse el trabajo. **ACK después del commit:** si cae antes del ACK, puede haber reentrega; hace falta idempotencia. [T12, p. 27]

### 6. Pub/Sub y Streams

**Valkey Pub/Sub** difunde mensajes a suscriptores, pero no conserva historial. Un suscriptor que no recibe el mensaje lo pierde: entrega at-most-once. [T12, p. 25]

**Streams** conserva una secuencia de entradas. Los lectores avanzan por posición y pueden releer lo que siga retenido; el log puede recortarse. [T12, p. 29]

En un **consumer group** se reparte el trabajo entre consumidores y se espera un ACK por mensaje. Si un consumidor falla antes del ACK:

- Si vuelve, puede recuperar sus pendientes con `XREADGROUP ... 0`.
- Si no vuelve, otro puede reclamarlos con `XAUTOCLAIM`, respetando el tiempo mínimo de inactividad.

[T12, p. 32] [T12, p. 33]

**Orden del log no equivale a orden de finalización.** Dos workers paralelos pueden terminar en un orden distinto al de entrega. Si una entidad requiere orden, sus mensajes deben seguir una ruta serializada. [T12, p. 28]

---

## Parte 2. Sistemas distribuidos

### 1. Distribuir trae beneficios y costos

Los nodos colaboran mediante mensajes por una red. Distribuir puede ayudar a tolerar fallas, acercar datos al usuario o procesar información que no cabe en una máquina. A cambio, complica debugging, despliegue, latencia y coordinación de transacciones. **La distribución no es una mejora automática.** [T11, p. 2] [T11, p. 4] [T11, p. 5]

### 2. Tres fuentes de incertidumbre: red, nodos y tiempo

| Eje | Modelos que conviene distinguir |
|---|---|
| Red | Confiable: entrega, aunque puede reordenar. Fair-loss: admite pérdida, duplicación y reordenamiento. Arbitraria: un adversario puede interferir. Partición: grupos que se comunican internamente, pero no entre sí |
| Nodos | Crash-stop: se detiene permanentemente. Crash-recovery: cae, pierde memoria y puede volver. Bizantino: comportamiento arbitrario, incluso malicioso |
| Tiempo | Modelo sincrónico: límites conocidos de demora y ejecución. Asincrónico: no hay garantías de tiempo |

[T11, p. 8] [T11, p. 9] [T11, p. 10]

Sin límites de tiempo garantizados, **no responder no permite distinguir un nodo caído de uno lento**. Tampoco se dispone de memoria compartida ni de un reloj global confiable para resolver toda la coordinación. [T11, p. 11]

**Aclaración conceptual:** el modelo temporal sincrónico/asincrónico habla de garantías sobre las demoras. La comunicación síncrona/asíncrona habla de si el cliente espera una respuesta inmediata. Son distinciones diferentes. [T11, p. 10] [T09, p. 5] [T09, p. 7]

### 3. RPC: parece una función, pero cruza una red

**Remote Procedure Call** permite invocar una operación remota con una interfaz parecida a una llamada local. Esa comodidad no elimina latencia variable, fallos parciales ni respuestas perdidas. [T11, p. 14] [T11, p. 18]

**Ejemplo didáctico:** llamás a `cobrar()` y vence el timeout. Quizá no llegó la solicitud; quizá el servidor sigue ejecutándola; quizá cobró y se perdió la respuesta. Reintentar sin protección puede cobrar dos veces.

Para el examen: **timeout = no obtuve respuesta a tiempo; no demuestra que la operación no ocurrió.** [T11, p. 18]

### 4. Atomicidad y serializabilidad responden preguntas distintas

**Atomic commit:** si una operación involucra varios participantes, todos confirman o ninguno. Busca evitar un resultado parcial. [T11, p. 21]

**Serializabilidad:** las transacciones pueden ejecutarse concurrentemente, pero su resultado debe equivaler a algún orden secuencial válido. Busca evitar interferencias como lecturas inconsistentes y actualizaciones perdidas. [T11, p. 22]

**Ejemplo didáctico:** una transferencia mueve dinero entre dos cuentas mientras una auditoría lee ambas. La auditoría debe observar un resultado compatible con leer antes o después de la transferencia, sin mezclar una cuenta actualizada con la otra sin actualizar.

### 5. Control pesimista y optimista

| | Pesimista | Optimista |
|---|---|---|
| Estrategia | Bloquear antes de operar | Detectar conflictos al intentar confirmar el cambio |
| Costo típico | Esperas y locks | Reintentos |
| Conviene cuando | Los conflictos son frecuentes | Los conflictos son poco frecuentes |
| Ejemplo de clase | Reserva de asientos | Edición de perfil |

[T11, p. 27]

En **optimistic locking**, se guarda una versión. La actualización solo procede si la versión sigue siendo la leída; si otro la cambió, se detecta el conflicto y se debe manejar, por ejemplo releyendo y reintentando. [T11, p. 28]

### 6. Two-Phase Commit — 2PC

Coordina un commit atómico entre participantes mediante dos fases:

1. **Preparación:** el coordinador pide votos. Los participantes se preparan, persisten lo necesario y bloquean recursos; votar SÍ los compromete a esperar la decisión.
2. **Decisión:** el coordinador comunica commit o abort. Los participantes ejecutan la decisión y confirman con ACK.

[T11, p. 29] [T11, p. 31] [T11, p. 32]

**La trampa central:** un participante que votó SÍ y desconoce la decisión no puede confirmar ni abortar unilateralmente por timeout. Puede quedar bloqueado reteniendo recursos. [T11, p. 34]

Por eso 2PC tiene costos: locks largos, menor concurrencia, dependencia del coordinador y dificultades de escalabilidad. `T11` lo propone cuando se necesita consistencia inmediata, sin estados intermedios ni compensaciones. [T11, p. 29] [T11, p. 35]

**Cuidado con las diapositivas:** la página 30, aunque titulada 2PC, describe acumulación y liberación de locks. La wiki registra como inferencia que corresponde al control de concurrencia 2PL. Para estudiar, distinguí **coordinar el commit** de **controlar la concurrencia**. [T11, p. 30]

### 7. Orquestación y coreografía

| | Orquestación | Coreografía |
|---|---|---|
| Quién conoce el flujo completo | Un coordinador explícito | El flujo surge de las reacciones de los servicios |
| Cómo avanzan los pasos | El coordinador envía comandos y decide | Cada servicio reacciona según su responsabilidad |
| Ventaja | Hay un dueño explícito del proceso | Autonomía y menor centralización de la lógica del flujo |
| Costo | El coordinador conoce y controla la secuencia | Es más difícil reconstruir, depurar y mantener el flujo |

[T11, p. 20] [T11, p. 37]

**Usar un broker no convierte automáticamente el flujo en coreografía.** Un orquestador también puede enviar comandos y recibir respuestas asíncronas. Ambas coordinaciones admiten comunicación síncrona o asíncrona. [T11, p. 20] [T11, p. 37]

El ejemplo de venta muestra que, si falla el pago, Inventario libera la reserva y la venta termina cancelada: son acciones compensatorias. Como conexión con la unidad de Kafka, esa unidad retoma el enfoque como **Saga**, con transacciones locales y compensaciones. [T11, p. 39] [T14, p. 21]

---

## Confusiones para revisar antes del examen

- **Mutex local:** no coordina procesos distintos. [T12, p. 9]
- **Dos comandos atómicos:** no convierten su combinación en atómica. [T12, p. 8]
- **TTL de lock:** no impide que un dueño atrasado intente escribir. [T12, p. 11]
- **Marca de adquisición:** no demuestra que el trabajo terminó. [T12, p. 22]
- **ACK y commit:** son pasos diferentes; una caída entre ellos puede causar reentrega. [T12, p. 27]
- **Log ordenado:** no garantiza finalización ordenada bajo paralelismo. [T12, p. 28]
- **Timeout de RPC:** no confirma fracaso de la operación. [T11, p. 18]
- **Votar SÍ en 2PC:** no significa que ya se decidió el commit global. [T11, p. 29] [T11, p. 34]
- **Broker:** no define si hay orquestación o coreografía. [T11, p. 20] [T11, p. 37]
