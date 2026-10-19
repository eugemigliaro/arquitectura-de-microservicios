# Simulacro diagnóstico — Segundo parcial

**Materia:** Arquitectura de Microservicios  
**Examen de referencia:** viernes 9 de octubre de 2026  
**Formato:** 35 preguntas de opción múltiple, una sola opción correcta por pregunta.  
**Tiempo sugerido de práctica:** 75–90 minutos; no es una duración oficial.

Este simulacro fue elaborado para estudiar: no es un parcial oficial ni una predicción de sus preguntas. Los casos y distractores son de elaboración propia, basados en los siete documentos oficiales indicados al final. El primer parcial de `material/entrada/preguntas-1er-parcial.pdf` se usa únicamente como referencia de estilo y complejidad.

## Cómo responder

- Elegí A, B, C o D para cada pregunta.
- Agregá una justificación breve: una o dos frases alcanzan.
- Marcá tu confianza: **alta**, **media** o **baja**. Si acertaste por descarte o intuición, indicá confianza baja.
- Para que el diagnóstico sea útil, hacé una primera pasada sin consultar apuntes. Podés dejar una respuesta en blanco.
- Completá la tabla del final y guardá este archivo. Después pedí en el chat: «Corregí mi simulacro».

No hay respuestas correctas marcadas. Las dos preguntas ya contestadas en el chat no se repiten literalmente; este es un nuevo diagnóstico.

## 1. Testing

### Pregunta 1

Un equipo prueba el adaptador de persistencia de Pedidos contra una base efímera real. Verifica el mapeo de campos, la configuración de conexión y que una transacción fallida no deje cambios parciales; el resto de las dependencias está simulado. Según el alcance definido en el documento de Testing, ¿qué clasificación es más precisa?

- **A.** Unitario: cualquier prueba que use una base efímera evalúa una única función aislada.
- **B.** End-to-end: la presencia de una base real implica recorrer todas las integraciones del producto.
- **C.** De integración: verifica el límite entre el servicio y su persistencia, aislando el resto.
- **D.** De contrato: una transacción es un contrato entre consumidor y proveedor HTTP.

Respuesta: C
### Pregunta 2

Un test usa un repositorio que almacena objetos en un diccionario en memoria y permite consultarlos. No devuelve respuestas prefijadas ni verifica el número de invocaciones. ¿Qué doble representa?

- **A.** Un fake: implementa el comportamiento del repositorio de forma simplificada.
- **B.** Un mock: conservar datos equivale a verificar explícitamente las interacciones.
- **C.** Un stub: cualquier reemplazo de una base de datos es necesariamente un stub.
- **D.** No es un doble: una dependencia con estado convierte la prueba en end-to-end.

Respuesta: 
### Pregunta 3

Un servicio alcanza 100 % de cobertura, pero sus tests de descuentos nunca comparan el precio calculado con el requerido por el negocio. ¿Cuál es la conclusión más precisa?

- **A.** La cobertura garantiza las reglas de negocio; el error debe estar fuera del servicio.
- **B.** Alcanzar 100 % demuestra que se probaron todas las combinaciones de entradas posibles.
- **C.** La cobertura deja de ser útil y debe eliminarse del pipeline.
- **D.** Se ejecutó el código medido, pero eso no garantiza que se comprobaran los resultados correctos.

### Pregunta 4

Un consumidor espera que la API de Inventario incluya un campo que el proveedor planea retirar. Ambos equipos quieren detectar la incompatibilidad antes del despliegue sin mantener un entorno con todo el producto. ¿Qué estrategia corresponde?

- **A.** Ejecutar exclusivamente tests unitarios, cada equipo con los mocks que ya tiene.
- **B.** Verificar contra el proveedor las expectativas de interfaz declaradas por el consumidor mediante contract testing.
- **C.** Reemplazar todos los tests de ambos equipos por un único test end-to-end.
- **D.** Congelar para siempre la API: los contratos impiden cualquier evolución de una interfaz.

### Pregunta 5

El código propio supera sus pruebas funcionales, pero una librería incluida en la imagen tiene una vulnerabilidad conocida. ¿Qué control apunta específicamente a ese problema?

- **A.** SAST, porque analiza exclusivamente las dependencias distribuidas con la aplicación.
- **B.** Un linter, porque verificar convenciones demuestra que las librerías son seguras.
- **C.** SCA, porque analiza vulnerabilidades en las dependencias.
- **D.** Cobertura de código, porque ejecutar todas las líneas descarta vulnerabilidades conocidas.

## 2. Comunicación síncrona

### Pregunta 6

Una plataforma expone una API pública, tiene llamadas internas frecuentes que necesitan serialización compacta y ofrece un frontend con consultas de datos complejas. ¿Qué elección coincide mejor con los criterios de la materia?

- **A.** REST para la API pública, gRPC para las llamadas internas y GraphQL cuando la flexibilidad del frontend lo justifique.
- **B.** GraphQL para todos los bordes, porque elimina los fallos de red y el acoplamiento temporal.
- **C.** gRPC para todos los bordes, porque un único protocolo elimina la necesidad de contratos.
- **D.** REST para todo, porque combinar protocolos impide automatizar pruebas y despliegues.

### Pregunta 7

Un servicio realiza llamadas gRPC sin deadline a un proveedor que a veces responde muy lentamente. ¿Qué mejora aborda directamente el riesgo?

- **A.** Usar streaming en todas las llamadas, porque cancela automáticamente cualquier espera prolongada.
- **B.** Quitar el contrato `.proto`, porque la generación de clientes causa las esperas.
- **C.** Agregar reintentos inmediatos ilimitados para obtener una respuesta cuanto antes.
- **D.** Definir deadlines explícitos para limitar cuánto espera cada llamada y manejar su vencimiento.

### Pregunta 8

Una query GraphQL recupera veinte pedidos y cada resolver de productos hace una consulta independiente a la base. El resultado es correcto, pero aumenta el número de consultas con la cantidad de pedidos. ¿Qué medida apunta al problema descrito?

- **A.** Aumentar la profundidad máxima permitida para que el cliente incluya más campos.
- **B.** Usar DataLoader para agrupar las cargas y abordar el patrón N+1.
- **C.** Trasladar todas las reglas de negocio a los resolvers para reducir las capas del sistema.
- **D.** Devolver menos campos al cliente, lo que por sí solo garantiza una única consulta a la base.

### Pregunta 9

Llegan sostenidamente 120 trabajos por segundo y los workers pueden completar 80. Se incorpora una cola finita sin cambiar workers ni recursos. ¿Qué ocurrirá si la demanda se mantiene?

- **A.** La capacidad de procesamiento aumenta a 120 porque encolar reemplaza el trabajo del worker.
- **B.** La cola mantiene una latencia constante y puede absorber indefinidamente el exceso.
- **C.** Crecen el backlog y la espera; al saturarse la cola reaparece el rechazo o la pérdida según la política.
- **D.** El procesamiento pasa a ser inmediato porque el productor ya no espera la finalización.

### Pregunta 10

Bajo sobrecarga, un endpoint acumula pedidos cuyos clientes ya vencieron el timeout, pero el servidor continúa trabajando. Se implementa un bulkhead que limita ejecución y espera, y rechaza pronto el exceso. ¿Qué cambia principalmente?

- **A.** Se protege el servicio acotando el trabajo admitido; eso no garantiza completar los pedidos rechazados.
- **B.** Se garantiza que toda solicitud termine porque el bulkhead persiste el exceso en un broker.
- **C.** Se elimina la necesidad de idempotencia porque ya no hay pedidos duplicados.
- **D.** Se amplía automáticamente la capacidad de CPU para igualar la tasa de llegada.

## 3. DevOps y CI/CD

### Pregunta 11

Cada cambio dispara build y pruebas; los releases aprobados quedan listos para producción, pero una persona decide cuándo ejecutar el despliegue. ¿Cómo se clasifica mejor el proceso?

- **A.** Continuous Deployment: que el artefacto esté listo equivale a desplegarlo automáticamente.
- **B.** Solo CI: cualquier intervención humana impide hablar de Continuous Delivery.
- **C.** No es CI/CD: las pruebas deben ejecutarse manualmente para habilitar un release.
- **D.** Continuous Delivery: el artefacto queda listo para desplegar, sin exigir despliegue automático a producción.

### Pregunta 12

El pipeline prueba una imagen en staging. Para producción, otro job recompila el código y construye una imagen nueva, potencialmente con dependencias distintas. ¿Qué modificación respeta mejor el principio de delivery?

- **A.** Reconstruir una tercera imagen después del despliegue para comprobar que el código compila.
- **B.** Disponibilizar y promover el mismo artefacto validado, cambiando la configuración del ambiente cuando corresponda.
- **C.** Conservar el mismo nombre de tag aunque el contenido cambie, porque el nombre garantiza identidad.
- **D.** Omitir las pruebas de staging: reconstruir en producción ya garantiza reproducibilidad.

### Pregunta 13

Se actualizan progresivamente las instancias de un servicio mediante rolling deployment. La versión nueva renombra de inmediato una columna que la versión vieja todavía utiliza. ¿Cuál es el problema principal?

- **A.** Rolling requiere detener simultáneamente todas las instancias antes de migrar la base.
- **B.** Los contenedores no permiten que dos versiones de una aplicación compartan un ambiente.
- **C.** Durante la convivencia, las versiones deben ser compatibles; el cambio rompe a las instancias viejas.
- **D.** El rolling garantiza compatibilidad del esquema, por lo que el fallo debe ser exclusivamente de red.

### Pregunta 14

Un equipo quiere enviar inicialmente el 1 % del tráfico a una versión nueva, observar errores y latencia, y aumentar la exposición si las métricas son aceptables. ¿Qué estrategia describe?

- **A.** Canary: exposición progresiva y posibilidad de reducir rápidamente el tráfico ante fallos.
- **B.** Basic: apagar todas las instancias, reemplazarlas y observar después el resultado.
- **C.** Multi-service: desplegar todos los servicios juntos para garantizar su independencia.
- **D.** Rolling: reemplazar instancias sin usar la evaluación del grupo inicial para decidir la expansión.

### Pregunta 15

Una vulnerabilidad se detecta habitualmente después de llegar a producción. El equipo incorpora un control automático en el pipeline y devuelve el fallo al desarrollador con el commit que lo introdujo. ¿Qué principio aplica?

- **A.** Leftover Principle: eliminar toda intervención humana en operación.
- **B.** Continuous Deployment: publicar toda modificación aunque falle un control de seguridad.
- **C.** Testing exploratorio: reemplazar controles automatizados por recorridos manuales.
- **D.** Shift-left: detectar y comunicar el error temprano para corregirlo antes de producción.

## 4. Estado distribuido y patrones

### Pregunta 16

Dos instancias leen que queda un único cupo y ambas lo descuentan con un `GET` seguido de un `DECR`. Cada instancia protege ese código con su propio mutex local. ¿Qué falta para evitar la carrera entre instancias?

- **A.** Aumentar el TTL del cupo, porque toda clave con expiración serializa las operaciones.
- **B.** Hacer atómica la verificación de disponibilidad y la modificación sobre el estado compartido.
- **C.** Agregar otro mutex local en cada proceso, sin cambiar cómo acceden al estado compartido.
- **D.** Usar balanceo round-robin, porque repartir requests equivale a coordinar las decisiones.

### Pregunta 17

A adquiere un lock con expiración. Su lease vence y B adquiere el mismo lock. A retoma y ejecuta un `DEL` incondicional sobre la key. ¿Cómo se evita que A libere el lock de B?

- **A.** Comparar el valor con el identificador de esa adquisición y borrar únicamente si coincide, en una operación atómica.
- **B.** Usar un identificador fijo compartido por todas las instancias para simplificar la liberación.
- **C.** Hacer un `GET` y luego un `DEL` separados: no puede ocurrir ningún cambio entre ambos comandos.
- **D.** Eliminar la expiración del lock: una instancia caída ya no necesita recuperación.

### Pregunta 18

A obtiene un lease con token 33 y pausa más que el TTL. B obtiene el token 34 y escribe en el recurso protegido. Luego A retoma e intenta escribir con el token 33. ¿Qué mecanismo protege frente a esa escritura atrasada?

- **A.** El `owner-id` por sí solo: aunque el recurso no lo valide, impide cualquier escritura de A.
- **B.** Un TTL mayor: garantiza que ninguna pausa pueda superar la vigencia del lease.
- **C.** Un fencing token validado por el recurso, que rechace tokens inferiores al mayor ya observado.
- **D.** Un mutex local en A: coordina automáticamente las escrituras de todas las instancias.

### Pregunta 19

Un cliente repite una solicitud con una clave de idempotencia ya completada, pero cambia el importe respecto de la solicitud original. ¿Qué comportamiento corresponde?

- **A.** Aplicar el nuevo importe porque la clave identifica cualquier operación futura del cliente.
- **B.** Generar silenciosamente una nueva clave y cobrar otra vez.
- **C.** Devolver siempre la respuesta anterior, sin comprobar si los parámetros coinciden.
- **D.** Rechazar el uso de la misma clave con parámetros distintos; el registro debe vincularla con la operación original.

### Pregunta 20

Un consumidor de Valkey Streams cae antes de enviar `XACK` y no volverá a ejecutarse. ¿Cómo puede continuar el procesamiento otro consumidor del grupo?

- **A.** Considerando procesados todos los mensajes entregados: la entrega implica ACK automático.
- **B.** Reclamando los pendientes mediante `XAUTOCLAIM` después del tiempo mínimo de inactividad correspondiente.
- **C.** Usando Pub/Sub para recuperar el historial que conservó de los mensajes anteriores.
- **D.** Borrando el stream completo para que el productor reenvíe automáticamente cada evento.

## 5. Sistemas distribuidos

### Pregunta 21

Un cliente invoca por RPC una operación de pago y recibe timeout. No tiene información adicional sobre el servidor. ¿Qué puede concluir?

- **A.** El pago podría no haberse ejecutado o haberse ejecutado sin que llegara la respuesta; reintentar puede duplicarlo.
- **B.** El servidor abortó la operación necesariamente antes de modificar datos.
- **C.** El servidor está caído de forma permanente: un nodo lento siempre responde antes del timeout.
- **D.** RPC garantiza la misma semántica de una llamada local, por lo que un reintento no tiene riesgos.

### Pregunta 22

Una transacción mueve una unidad de Y a X y debe conservar `X + Y = 20`. Una auditoría concurrente lee un estado intermedio con `X = 11` e `Y = 10`. ¿Qué significa exigir serializabilidad?

- **A.** Que los requests ingresen al servidor en orden de llegada, aunque la auditoría vea cambios parciales.
- **B.** Que todos los servicios compartan el mismo reloj físico para fechar las escrituras.
- **C.** Que el resultado de la ejecución concurrente equivalga a algún orden secuencial válido de las transacciones.
- **D.** Que toda transacción termine antes de un timeout, independientemente de lo que observe.

### Pregunta 23

Dos clientes leen la versión 7 de un perfil. El primero actualiza y pasa a versión 8. El segundo intenta actualizar con la condición `version = 7` y no modifica ninguna fila. ¿Qué representa este mecanismo?

- **A.** Locking pesimista: el primer cliente bloqueó el perfil desde su lectura hasta su escritura.
- **B.** Locking optimista: se detectó un conflicto de versión y el segundo cliente debe manejarlo, por ejemplo releyendo y reintentando.
- **C.** 2PC: dos clientes actuaron como participantes de una transacción distribuida.
- **D.** Ausencia de concurrencia: una actualización condicional permite sobrescribir siempre el último valor.

### Pregunta 24

En 2PC, un participante persistió su preparación y votó SÍ. Luego pierde contacto con el coordinador y desconoce la decisión global. ¿Qué riesgo caracteriza al protocolo?

- **A.** Puede confirmar unilateralmente porque su voto SÍ demuestra que todos los demás votaron igual.
- **B.** Puede abortar unilateralmente al vencer su timeout, aunque otros participantes hayan confirmado.
- **C.** Puede liberar todos sus recursos y ejecutar la operación como una Saga, sin cambiar el protocolo.
- **D.** Puede quedar bloqueado esperando la decisión; votar SÍ no autoriza a decidir unilateralmente.

### Pregunta 25

Un workflow central conoce la secuencia de una compra, envía comandos por un broker a Inventario y Pagos, y decide el paso siguiente según sus respuestas. ¿Cómo se clasifica la coordinación?

- **A.** Orquestación: existe un dueño explícito del flujo, aunque el transporte sea asíncrono.
- **B.** Coreografía: cualquier uso de eventos o mensajes implica ausencia de coordinador.
- **C.** 2PC: enviar comandos a varios servicios garantiza una transacción atómica distribuida.
- **D.** Comunicación síncrona: todo flujo con un coordinador requiere llamadas bloqueantes.

## 6. Comunicación asincrónica y Kafka

### Pregunta 26

Facturación y Analytics necesitan procesar cada evento de ventas de forma independiente. Además, Facturación quiere distribuir su carga entre tres instancias. ¿Cómo deben organizarse los consumer groups?

- **A.** Todas las instancias de ambos servicios en el mismo grupo para que cada servicio reciba todos los eventos.
- **B.** Un grupo diferente por cada instancia de Facturación para repartir cada evento entre ellas una sola vez.
- **C.** Un grupo para Facturación, compartido por sus tres instancias, y otro grupo para Analytics.
- **D.** Un único consumer para todos los servicios, porque Kafka no admite lectores independientes del mismo topic.

### Pregunta 27

Un topic tiene cuatro particiones y un grupo cuenta con cuatro consumidores. El lag crece porque los handlers son lentos. Se agregan cuatro consumidores más al mismo grupo, sin cambiar particiones ni handlers. ¿Qué resultado debe esperarse?

- **A.** El throughput necesariamente se duplica: cada partición queda dividida entre dos consumidores simultáneos.
- **B.** Como máximo cuatro consumidores tendrán particiones asignadas a la vez; sumar consumidores no aumenta ese límite.
- **C.** Kafka borra el backlog al rebalancear, así que el lag vuelve inmediatamente a cero.
- **D.** Cada nuevo consumidor recibe el historial completo independientemente de los otros miembros del grupo.

### Pregunta 28

`PedidoCreado` y `PedidoPagado` pertenecen a la misma orden, pero cada mensaje usa un UUID aleatorio como key. El consumidor puede observar `PedidoPagado` antes que `PedidoCreado`. ¿Qué cambio aborda el problema de particionamiento?

- **A.** Usar el nombre del tipo de evento como key: todos los eventos de cualquier orden compartirán una partición.
- **B.** Agregar consumidores hasta que haya uno por mensaje; así se obtiene orden global.
- **C.** Quitar todas las keys: Kafka garantiza el orden global entre particiones.
- **D.** Usar una key estable por `orderId`, para mantener los eventos de la entidad en la misma partición; el handler también debe respetar su orden de procesamiento.

### Pregunta 29

Un servicio guarda la orden y su evento en una misma transacción mediante Outbox. El relay publica, pero cae antes de marcar el evento como publicado. Al reiniciar lo publica nuevamente. ¿Cuál es la interpretación correcta?

- **A.** Outbox resuelve la atomicidad del cambio y el registro del evento, pero admite publicaciones repetidas; un Inbox puede proteger el efecto de negocio del consumidor.
- **B.** Outbox falló: su garantía es que el broker entregue cada evento exactamente una vez.
- **C.** Debe marcarse publicado antes de enviarlo para evitar duplicados, aunque el proceso pueda caer entre ambos pasos.
- **D.** Basta insertar el identificador en el Inbox y aplicar el efecto en otra transacción independiente para eliminar todas las ventanas de falla.

### Pregunta 30

Un mensaje tiene un payload que el consumidor no puede deserializar. Se reintenta indefinidamente sin avanzar el offset, mientras el lag aumenta en esa partición. ¿Qué política corresponde?

- **A.** Mantener el retry infinito: cualquier error de deserialización necesariamente es transitorio.
- **B.** Reiniciar todo el cluster para que el payload se convierta automáticamente al esquema correcto.
- **C.** Limitar intentos, apartar el mensaje a una DLQ u otro topic y alertar para su revisión, gestionando la continuidad según las reglas del negocio.
- **D.** Agregar una key nueva a cada reintento para garantizar orden y evitar duplicados.

## 7. Del monolito a microservicios

### Pregunta 31

Un equipo de cuatro personas mantiene una aplicación con pocos cambios. Propone migrar únicamente porque «los microservicios son más modernos y cualquier monolito es imposible de escalar». ¿Qué evaluación coincide con la materia?

- **A.** Migrar siempre es correcto si se elige un broker de alto throughput.
- **B.** Las razones son insuficientes: debe identificarse un problema de negocio medible; un monolito también puede escalar horizontalmente.
- **C.** Migrar es obligatorio porque cada tabla necesita un servicio independiente.
- **D.** La arquitectura debe decidirse exclusivamente según la cantidad de líneas de código.

### Pregunta 32

Se quiere extraer Notificaciones. Las llamadas externas llegan por `/notifications/*`, pero el checkout y un cron invocan funciones internas del monolito. ¿Qué combinación cubre ambos puntos de acceso?

- **A.** Strangler Fig por sí solo, porque redirigir HTTP también reemplaza cualquier llamada interna.
- **B.** Un branch largo donde reescribir todo el monolito antes de desplegar algún cambio.
- **C.** Parallel Run de todas las escrituras, sin interfaces ni control de efectos secundarios.
- **D.** Strangler Fig para las rutas externas y Branch by Abstraction para las llamadas internas, incluyendo el cron.

### Pregunta 33

Un equipo compara el motor de pagos viejo y el nuevo ejecutando ambos con cada solicitud real y devolviendo al usuario solo la respuesta del viejo. Ambos motores pueden efectuar el cobro. ¿Qué debe resolver antes de hacer esta comparación?

- **A.** Los efectos secundarios: desactivarlos en la ejecución en sombra o usar una idempotencia que cubra ambas ejecuciones, para evitar cobros dobles.
- **B.** Solo la respuesta visible: si el usuario recibe una, puede ocurrir como máximo un cobro.
- **C.** Solo la cantidad de logs: la base garantiza automáticamente que dos motores no cobren el mismo pedido.
- **D.** Solo el timeout: mantener el viejo sin deadline evita que el nuevo produzca efectos.

### Pregunta 34

Durante una migración, el servicio nuevo pasó a ser el escritor y el monolito recibe una copia de sus cambios. La sincronización quedó atrasada. Se propone volver al monolito simplemente apagando un feature flag. ¿Qué falta evaluar?

- **A.** Nada: cambiar la ruta revierte automáticamente todas las escrituras realizadas.
- **B.** El tamaño del binario: los datos no influyen en el rollback de una migración.
- **C.** La reconciliación y el estado de los datos: volver a una copia atrasada puede perder visibilidad de cambios, y el flag no los revierte.
- **D.** El número de topics: usar uno solo obliga a que ambas bases siempre estén sincronizadas.

### Pregunta 35

Tras seis meses de migración hay diez servicios nuevos, pero el tiempo desde commit hasta producción no mejoró y los despliegues siguen requiriendo coordinación entre todos los equipos. ¿Cómo debe evaluarse el resultado?

- **A.** Es exitoso porque la cantidad de servicios es la métrica principal de autonomía.
- **B.** No basta contar servicios: deben contrastarse las métricas DORA y la coordinación necesaria con los objetivos y la línea de base.
- **C.** Es exitoso si todos los servicios utilizan lenguajes distintos, aunque no cambie la entrega.
- **D.** Eliminar las métricas anteriores evita comparar arquitecturas que usan tecnologías diferentes.

## Hoja de respuestas

Escribí tu letra, confianza y explicación en cada fila. Podés usar «No sé» en la justificación.

| Pregunta | Opción | Confianza | Justificación breve |
| :---- | :---- | :---- | :---- |
| 1 | C | alta |  |
| 2 | A | media |  |
| 3 | D | alta |  |
| 4 | B | alta |  |
| 5 | A | media |  |
| 6 | A | alta |  |
| 7 | D | media-baja |  |
| 8 | B | media-baja |  |
| 9 | C | media-alta |  |
| 10 | A | media |  |
| 11 | D | alta |  |
| 12 | B | alta |  |
| 13 | C | media |  |
| 14 | A | media alta |  |
| 15 | D | alta |  |
| 16 | B | media  |  |
| 17 | A | media baja |  |
| 18 | C | media baja |  |
| 19 | D | media baja |  |
| 20 | no se |  |  |
| 21 | A |  |  |
| 22 | C | alta |  |
| 23 | no se |  |  |
| 24 | D | media alta |  |
| 25 | A | media |  |
| 26 | C | media |  |
| 27 | B | baja |  |
| 28 | D |  |  |
| 29 | A |  |  |
| 30 | C |  |  |
| 31 | B |  |  |
| 32 | D |  |  |
| 33 | A |  |  |
| 34 | C |  |  |
| 35 | B |  |  |

**Tiempo empleado:**  
**Temas que sentiste más difíciles:**

## Criterio de devolución

La corrección distinguirá aciertos, errores y respuestas en blanco por tema. Para el diagnóstico, una opción correcta con justificación equivocada o confianza baja requiere repaso; una opción incorrecta con confianza alta señala una confusión prioritaria. No se inferirá dominio de todo un documento por acertar estas cinco preguntas.

Cada devolución incluirá la opción correcta, el concepto evaluado, la explicación del caso, por qué fallan los distractores y referencias al material oficial. Luego retomaremos preguntas de a una, centradas en las debilidades detectadas. Si hace falta ampliar cobertura, se evaluarán otros conceptos de los mismos siete documentos.

## Fuentes y trazabilidad

Las siguientes referencias permiten recuperar la teoría al corregir. Los escenarios concretos y sus opciones son de elaboración propia; no son citas textuales ni resoluciones oficiales. Para evitar pistas durante la práctica, las fuentes están agrupadas por bloque.

| Bloque | Fuente oficial | Páginas de apoyo |
|---|---|---|
| Testing | `T10` — Testing | [T10, p. 6], [T10, p. 7], [T10, p. 8], [T10, p. 10], [T10, p. 18], [T10, p. 19], [T10, p. 20] |
| Comunicación síncrona | `T09` — Comunicación síncrona | [T09, p. 12], [T09, p. 15], [T09, p. 17], [T09, p. 22], [T09, p. 26], [T09, p. 30], [T09, p. 33] |
| DevOps y CI/CD | `T13` — DevOps | [T13, p. 17], [T13, p. 18], [T13, p. 35], [T13, p. 40], [T13, p. 42], [T13, p. 45] |
| Estado distribuido y patrones | `T12` — Estado distribuido: patrones con Valkey | [T12, p. 8], [T12, p. 9], [T12, p. 10], [T12, p. 11], [T12, p. 19], [T12, p. 25], [T12, p. 33] |
| Sistemas distribuidos | `T11` — Sistemas distribuidos | [T11, p. 18], [T11, p. 20], [T11, p. 22], [T11, p. 27], [T11, p. 28], [T11, p. 29], [T11, p. 34] |
| Comunicación asincrónica y Kafka | `T14` — Comunicación asincrónica y Kafka | [T14, p. 35], [T14, p. 40], [T14, p. 42], [T14, p. 43], [T14, p. 47], [T14, p. 48], [T14, p. 50], [T14, p. 53] |
| Del monolito a microservicios | `T15` — Del monolito a microservicios | [T15, p. 8], [T15, p. 9], [T15, p. 23], [T15, p. 25], [T15, p. 26], [T15, p. 40], [T15, p. 46] |

**Criterio terminológico:** las preguntas de alcance de pruebas siguen `T10`. Las de pipeline y despliegue siguen `T13`. Se evita usar como equivalentes «integration» y «functional» o confundir 2PC con el locking en dos fases; las diferencias de terminología están registradas en `wiki/dudas-y-conflictos.md`.
