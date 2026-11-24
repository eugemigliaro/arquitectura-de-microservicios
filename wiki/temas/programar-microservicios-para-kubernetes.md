# Programar microservicios para Kubernetes

## Fuente y alcance

Síntesis de `T18`, presentación de la cátedra con los docentes de la materia y el subtítulo "Patrones de diseño para aplicaciones cloud-native". **Inferencia de catalogación:** el ciclo 2026 se asigna por la configuración de la materia y la fecha de creación del PDF; no figura en la portada. [T18, p. 1] La unidad toma los objetos de [Kubernetes — Parte 1](kubernetes-parte-1.md) y [Parte 2](kubernetes-parte-2.md) desde el lado del código: qué tiene que hacer el servicio para que la plataforma pueda reemplazarlo, escalarlo y apagarlo sin perder trabajo. `T21` la retoma como base de la resiliencia. [T21, p. 26] Ver [Resiliencia y escalabilidad en Kubernetes](resiliencia-y-escalabilidad.md).

## La división de responsabilidades

La idea central es separar la lógica de negocio de la operación y **no reimplementar el control plane dentro del servicio**. [T18, p. 2]

| La aplicación se ocupa de | Kubernetes se ocupa de |
|---|---|
| Reglas de negocio y contratos | Crear, reemplazar y distribuir Pods |
| Idempotencia y transacciones | Descubrir servicios y enrutar tráfico |
| ACK/NACK, timeouts y compensaciones | Programar Jobs y CronJobs |
| Saber cuándo está lista para trabajar | Aplicar límites y escalar réplicas |

Fuente: [T18, p. 2]. La regla de cierre: antes de implementar un mecanismo operacional en código, preguntar qué abstracción ya ofrece Kubernetes. **La plataforma administra procesos; el código protege la semántica del negocio.** [T18, p. 32]

| Microservicio aislado | Servicio diseñado para Kubernetes |
|---|---|
| Descubre Pods e IPs manualmente | Se conecta a Services por DNS |
| Tiene un supervisor propio | Expone salud y gestiona señales |
| Mantiene sesiones en memoria | Tolera réplicas reemplazables |
| Reintenta indefinidamente | Restringe los retries a operaciones seguras |

Fuente: [T18, p. 3].

## 1. El contrato del contenedor

| Aspecto | Correcto | Antipatrón |
|---|---|---|
| Proceso | En foreground, recibe las señales reales | Daemonizar o usar un supervisor casero |
| Terminación | `SIGTERM` y cierre con plazo | Ignorar `SIGTERM` |
| Logs | Estructurados a `stdout`/`stderr` | Solo a un archivo en disco local |
| Configuración | Variables de entorno o archivos montados | Valores embebidos en la imagen |
| Persistencia | Estado externo cuando corresponde | Depender de la capa escribible |

Fuente: [T18, p. 5]. Es el mismo contrato que ya pedían las imágenes y Compose; ver [Imágenes de contenedores](imagenes-de-contenedores.md).

### Terminación gradual

Kubernetes inicia el cierre y el proceso tiene que recorrer cinco etapas antes de que venza el plazo: [T18, p. 6]

1. **`SIGTERM`:** empieza la terminación.
2. **Drenar:** dejar de aceptar trabajo nuevo.
3. **Finalizar:** completar o cancelar lo que está en curso.
4. **Cerrar:** conexiones, ACK pendientes y flush.
5. **Salir** antes del plazo.

Si se agota el *grace period*, el proceso recibe `SIGKILL`. Un servidor HTTP termina las requests en curso; un worker deja de consumir y confirma solo el trabajo completado. [T18, p. 6] El ejemplo en Go usa `signal.NotifyContext` con `SIGTERM` y `SIGINT`, espera la señal y llama a `srv.Shutdown` con un contexto de 25 segundos. **El tiempo de drenado tiene que ser coherente con `terminationGracePeriodSeconds`.** [T18, p. 7] El período de gracia empieza antes de `preStop` e incluye el hook. [T16, p. 21]

### Configuración, logs y almacenamiento

- **Configuración:** variables de entorno y montajes; *fail fast* si es inválida; no reconstruir la imagen por ambiente. [T18, p. 8]
- **Observabilidad:** logs a `stdout`/`stderr`, correlación y trazas, sin rotación casera de logs. [T18, p. 8] Ver [Observabilidad](observabilidad.md). [T23, p. 9]
- **Almacenamiento:** la capa escribible es efímera, `emptyDir` sirve solo para datos temporales, y lo que debe persistir va a un PVC o a un almacenamiento externo de objetos. [T18, p. 8] Ver la duración de cada opción en [Kubernetes — Parte 2](kubernetes-parte-2.md). [T17, p. 28]

## 2. Estado, red y recursos

### Instancias reemplazables

| Antipatrón | Diseño apropiado |
|---|---|
| Sesiones en memoria por Pod | Estado persistente fuera del Pod |
| Uploads solo en disco local | Archivos en un almacenamiento apropiado |
| Contadores críticos locales | Operaciones atómicas o en la base |
| Caché como fuente de verdad | Caché local solo como optimización |

Fuente: [T18, p. 10]. La prueba mental es preguntarse **qué pasa si este Pod desaparece ahora mismo**. [T18, p. 10]

### Descubrimiento

El cliente resuelve un nombre como `payment-api.default.svc`; el Service (VIP y DNS estables) reenvía a los Pods `Ready` listados en su EndpointSlice. La plataforma se ocupa del descubrimiento y del enrutamiento hacia destinos disponibles; el cliente, de los timeouts, la reconexión y la semántica segura de los reintentos. [T18, p. 11] Ver [Services y descubrimiento](kubernetes-parte-2.md#services-y-descubrimiento). [T17, p. 10]

### Requests, limits y concurrencia

| Recurso | Qué hace Kubernetes | Qué debe hacer el código |
|---|---|---|
| CPU | Requests, limits y *throttling* | Limitar el paralelismo y medir p95/p99 |
| Memoria | Memory limit y `OOMKilled` | Acotar buffers, cachés y colas |
| Disco | Ephemeral storage y volúmenes | No crecer sin límite en `tmp` o logs |
| Réplica | El HPA ajusta la cantidad de Pods | Escalar sin estado local obligatorio |

Fuente: [T18, p. 12].

Sin límites, una instancia acepta 1.000 tareas simultáneas con un buffer sin cota y termina con latencia creciente y OOM. Con límites usa un *worker pool* o semáforo, prefetch y cola acotados, backpressure y métricas. **La plataforma escala instancias; el servicio regula el trabajo dentro de cada instancia.** [T18, p. 13] Es el mismo argumento del bulkhead en [Comunicación síncrona](comunicacion-sincrona.md). [T09, p. 26]

## 3. Ciclo de vida y disponibilidad

### Probes

| Probe | Pregunta | Efecto si falla |
|---|---|---|
| `startupProbe` | ¿Terminó el arranque inicial? | Protege de reinicios prematuros |
| `readinessProbe` | ¿Puede atender tráfico ahora? | El Pod sale de los endpoints |
| `livenessProbe` | ¿El proceso quedó irrecuperable? | El kubelet reinicia el contenedor |

Fuente: [T18, p. 15].

| Semántica incorrecta | Contrato operativo |
|---|---|
| Siempre devuelve HTTP 200 | Liveness detecta un bloqueo local |
| Liveness depende de PostgreSQL | Readiness representa la capacidad real |
| Readiness exige que todo esté perfecto | Las dependencias se evalúan según la degradación posible |

Fuente: [T18, p. 16]. El caso para discutir: si el adquirente externo de pagos está caído, ¿se apagan todos los Pods o se degrada la funcionalidad? [T18, p. 16]

El Deployment de ejemplo usa `/health/startup` con `failureThreshold: 24` y `periodSeconds: 5`, `/health/ready` cada 5 segundos y `/health/live` cada 10. **Regla: el endpoint de liveness no debe fallar solo porque PostgreSQL o Redis estén caídos.** [T18, p. 17] *Inferencia:* 24 intentos cada 5 segundos dan hasta dos minutos de arranque antes de que el kubelet reinicie el contenedor.

### Rolling update y cierre coordinados

El ejemplo usa `RollingUpdate` con `maxUnavailable: 0`, `maxSurge: 1` y `terminationGracePeriodSeconds: 30`. La secuencia es: el Pod nuevo arranca y calienta; al estar `Ready` recibe tráfico; el viejo pasa a *terminating*, recibe `SIGTERM`, drena sus requests y termina antes del plazo. [T18, p. 18]

**Advertencia:** el cambio de endpoints es asíncrono, así que no garantiza que el Pod saliente deje de recibir solicitudes nuevas. [T18, p. 18] Por eso el servidor tiene que seguir atendiendo durante el drenado.

## 4. API, worker y job

| Tipo de proceso | Recurso habitual | Contrato en el código |
|---|---|---|
| API HTTP | Deployment + Service | Readiness, shutdown, sin estado |
| Consumidor continuo | Deployment (+ KEDA) | ACK, backpressure, drenaje |
| Tarea finita | Job | Exit codes, idempotencia |
| Tarea programada | CronJob → Job | Idempotencia, concurrencia |

Fuente: [T18, p. 20]. Ver los controladores en [Kubernetes — Parte 1](kubernetes-parte-1.md#elegir-un-workload). [T16, p. 37] [T16, p. 38]

### Workers y mensajes

Al recibir `SIGTERM`, el worker deja de consumir mensajes nuevos, espera los que están en vuelo, hace ACK solo de los completados, devuelve a la cola los no terminados y cierra la conexión. En la aplicación quedan el ACK/NACK, la idempotencia, la transacción y el prefetch y la concurrencia acotados; Kubernetes aporta reinicio, réplicas, scheduling, recursos y, opcionalmente, escala por cola. [T18, p. 21]

El caso de redelivery: el worker ejecuta un pago, la base confirma el commit, el Pod muere antes del ACK, el broker reentrega y un worker nuevo tiene que detectar el duplicado. **La unicidad de la operación se diseña en el dominio y la persistencia**, por ejemplo con una clave idempotente, una restricción única y la consulta del resultado previo. [T18, p. 22] Es el mismo problema de [Estado distribuido](estado-distribuido.md) y del inbox en [Kafka](comunicacion-asincrona-y-kafka.md). [T12, p. 20] [T14, p. 50]

### Quién responde cada pregunta de escala

| Pregunta | Quién responde |
|---|---|
| ¿Cuántos Pods se ejecutan? | Deployment + HPA/KEDA |
| ¿Cuánto trabajo toma cada Pod? | El código: concurrencia, prefetch, buffers |
| ¿Qué ocurre si la cola crece? | Backpressure, escala y alertas |
| ¿Qué pasa si un Pod muere? | Reejecución segura con ACK e idempotencia |

Fuente: [T18, p. 23].

### Política de reintentos

| Situación | Decisión |
|---|---|
| HTTP 503 transitorio | Retry acotado en el cliente, si es seguro |
| Deadlock local irrecuperable | Liveness y reinicio |
| Arranque incompleto | Startup o readiness |
| Configuración inválida | Fail fast, diagnóstico y alerta |
| Pago potencialmente duplicado | Idempotencia antes del retry |

Fuente: [T18, p. 24].

## 5. Patrones de apoyo y seguridad

- **Init container:** preparación finita y previa. No correr migraciones globales por réplica sin coordinación ni esperar a dependencias para siempre. [T18, p. 26]
- **Sidecar o ambassador:** función transversal común, como un proxy de transporte, pero no la semántica de los pagos. Agrega costo, recursos y modos de falla. [T18, p. 26]

| Evitar | Preferir |
|---|---|
| Ejecutar como root por costumbre | `runAsNonRoot` y privilegios mínimos |
| Modificar `/etc` o `/var/log` | `readOnlyRootFilesystem`, cuando sea viable |
| Secretos dentro de la imagen | Secret o almacén externo de secretos |
| Usar el socket de Docker o permisos del host | RBAC mínimo y sin acceso a la API si no hace falta |

Fuente: [T18, p. 27]. Ver [Contenedores sobre Linux](contenedores-linux.md) y [Secrets](kubernetes-parte-2.md). [T17, p. 52]

### Antipatrones operativos

| Señal de alerta | Mejor enfoque |
|---|---|
| Entrypoint con 200 líneas de esperas | Init finito, probes y fail fast |
| Cada servicio descubre IPs de Pods | Service + DNS |
| Liveness consulta tres sistemas externos | Liveness local y readiness degradada |
| Job implementado como `while(true)` | Job o CronJob |
| Caché local como fuente de verdad | Estado externo o sincronización explícita |
| Retries automáticos a un `POST` de pagos | Contrato idempotente antes del retry |

Fuente: [T18, p. 28].

## 6. Ejercicio: API de pagos

El flujo es `POST /payments` → la API valida y persiste → publica `PaymentRequested` en el broker → un worker procesa el pago y guarda el resultado de forma idempotente. Tiene cuatro problemas puestos a propósito: sesión en memoria, ACK antes del commit, worker que ignora `SIGTERM`, y retry infinito con logs a archivos. [T18, p. 30]

Las seis preguntas de revisión de una aplicación para Kubernetes: [T18, p. 31]

1. ¿Termina por `SIGTERM` sin perder trabajo crítico?
2. ¿Sigue siendo correcta con tres réplicas?
3. ¿Tiene readiness y liveness con semántica distinta?
4. ¿Tiene límites de concurrencia y memoria?
5. ¿Sus reintentos preservan la idempotencia?
6. ¿Logs, configuración y secretos están externalizados?

## Relación con el TP

*Inferencia de estudio:* la [tercera entrega](tercera-entrega-album-cicd-aks.md) pide probes de readiness y liveness "que verifiquen algo real" y resta puntos por sondas triviales. [E06] Las láminas 15 a 17 dan el criterio para no caer en el error opuesto, una liveness que reinicia todos los Pods cuando cae la base. [T18, p. 16] [T18, p. 17] El caso del worker que muere antes del ACK es el mismo de la apertura de sobres y la aceptación de ofertas del Álbum. [T18, p. 22] [E01, p. 6]
