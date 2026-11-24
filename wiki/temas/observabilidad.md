# Observabilidad

## Fuente y alcance

Síntesis de `T23`, presentación de la cátedra (36 láminas). La portada lleva el título general "Arquitectura de Microservicios con Contenedores" y los docentes; el título de la clase está en la lámina 2. **Inferencia de catalogación:** el ciclo 2026 se asigna por la configuración de la materia y la fecha de creación del PDF (13 de noviembre de 2026). [T23, p. 1] [T23, p. 2] Varias láminas son gráficos o capturas —resolución, promedio, percentiles, el *waterfall* de una traza y una captura de Uptrace—, verificadas sobre el original.

`T01` ubicaba en esta clase "Observabilidad integral: logs, métricas y trazas. OpenTelemetry, Jaeger, Prometheus y Grafana". [T01, p. 13] Las unidades de Kafka y de migración remitían a ella para los *correlation IDs* y el tracing distribuido. [T14, p. 54] [T15, p. 45]

La clase abre con "You can't manage what you don't measure", atribuida a Peter Drucker, "que nunca la dijo". [T23, p. 3]

## El problema del monitoreo en microservicios

Cuatro problemas: [T23, p. 4]

- **Silos de datos.** Varias herramientas para infraestructura, logs y APM crean datos aislados; correlacionar un incidente pasa a ser manual, lento y propenso a errores.
- **Falta de contexto.** Una métrica de infraestructura, como un pico de CPU, no alcanza si no se vincula con el microservicio, la transacción o el usuario que la causó.
- **Rastrear requests distribuidas.** Una solicitud puede generar docenas de llamadas entre servicios; sin tracing distribuido es casi imposible encontrar el origen de la latencia o de la falla.
- **Infraestructura efímera.** Los contenedores viven segundos; acceder a sus logs y métricas locales para un post-mortem es muy difícil. El monitoreo tiene que ser centralizado y continuo.

## Qué es observabilidad

"Una medida de qué tan bien pueden inferirse los estados internos de un sistema a partir del conocimiento de sus salidas externas" (Rudolf Kálmán, teoría de control, años 60). Nunca se ve el estado interno, solo sus salidas; en microservicios esas salidas son **logs, métricas y trazas**. [T23, p. 5]

## Los tres pilares

| Pilar | Qué responde | Analogía |
|---|---|---|
| Logs | Qué pasó | La historia clínica |
| Métricas | Cuándo y cuánto | Los signos vitales |
| Tracing | Dónde | La tomografía |

Fuente: [T23, p. 6]. Más adelante la clase describe el papel de cada uno en un diagnóstico: las métricas detectan el **síntoma** (qué), los traces revelan la **cadena causal** (dónde) y los logs explican el **detalle** (por qué); "sin los 3: diagnóstico siempre parcial". [T23, p. 28] *Observación de estudio:* el "qué" cambia de dueño entre las dos láminas (logs en la 6, métricas en la 28) porque la primera describe qué guarda cada pilar y la segunda, qué aporta en una investigación.

## Logs

**Event log.** El registro de un evento es un elemento inmutable, con timestamp, asociado a un evento discreto. Puede ser texto plano (Syslog, CEF y otros estándares) o estructurado (JSON, XML), más simple de interpretar entre sistemas. [T23, p. 7]

**El problema de los logs.** Sirven para depurar, pero su abuso trae: volumen alto, difícil de usar; reconstruir acciones entre servicios, muy costoso; sin correlación, silos de información por servicio; y logs de texto, donde "grep no escala en producción". [T23, p. 8]

**Structured logging.** Campos mínimos de un log estructurado: [T23, p. 9]

| Campo | Contenido |
|---|---|
| `timestamp` | Con zona horaria (UTC) |
| `level` | DEBUG, INFO, WARN o ERROR |
| `service` | De qué servicio viene |
| `traceId` | Conecta el log con la traza |
| `message` | Texto fijo, sin variables |
| `data` | Las variables de negocio (`orderId`, `userId`, `amount`) |

Desde el lado del contenedor, los logs van estructurados a `stdout`/`stderr`. [T18, p. 8] Ver [Programar microservicios para Kubernetes](programar-microservicios-para-kubernetes.md).

**Wide events.** El log tradicional (*narrow*) emite muchos eventos chicos por request: en el ejemplo, cinco logs para una operación. Un *wide event* es un solo evento rico con todo el contexto: `traceId`, `userId`, `endpoint`, `duration_ms`, `dbQueries`, `cacheHits`, `kafkaMessages`, `status`, `tenant` y `featureFlag.newCheckout`. La idea es de Charity Majors (Honeycomb). [T23, p. 10]

El beneficio: con narrow logs, reconstruir una operación exige *joins* entre servicios; con wide events hay un evento por operación, filtrable por cualquier dimensión, con `tenant`, `user_type` y `feature_flag` siempre presentes, y se puede depurar sin reproducir el problema. "Está lento para usuarios premium en mobile" pasa a ser un filtro sobre los campos. En OpenTelemetry, los atributos de un span son un wide event. [T23, p. 11]

## Métricas

Una métrica es una medición numérica que se **agrega en el tiempo**: no guarda cada request, guarda contadores. El nombre más los labels identifican la serie, y cada muestra es un timestamp más un valor. [T23, p. 12]

| Tipo | Comportamiento | Ejemplos |
|---|---|---|
| Counter | Solo sube; se mira su ritmo con `rate()` | Requests, errores |
| Gauge | Sube y baja | Memoria, conexiones abiertas, mensajes en cola |
| Histogram | Cuenta valores por rangos; de ahí salen los percentiles | Latencia |

Fuente: [T23, p. 12].

### Cardinalidad

Cada combinación de valores de labels es una serie nueva en Prometheus. `http_requests_total{service="orders", status="500"}` genera decenas de series; `http_requests_total{service="orders", userId="u-456"}`, una por usuario. [T23, p. 13]

- Labels con pocos valores posibles: servicio, endpoint, status, región.
- `userId`, `orderId` y `traceId` producen millones de series: "Prometheus se queda sin memoria".
- Ese detalle va en el wide event o en la traza, no en la métrica.
- Por eso conviven: la métrica es barata y agregada; el evento tiene todo el detalle. [T23, p. 13]

La misma regla aparece para las flags: no usar tenant, usuario ni trace ID como labels. [T22, p. 33]

### Resolución

Medir con alta frecuencia captura todos los picos, pero cuesta más; con baja frecuencia se pierde precisión y se escapan anomalías. Prometheus scrapea cada `scrape_interval`: 1 minuto por defecto y, en la práctica, entre 15 y 30 s. Alcanza para tendencias y alertas; "los picos de milisegundos se ven en las trazas". [T23, p. 14]

### Promedio y percentiles

**La mentira del promedio.** Una CPU que oscila entre 0 % y 100 % promedia 50 %: "el dashboard en verde y el servicio saturado la mitad del tiempo". El gráfico compara dos series con el mismo promedio, una que oscila de punta a punta y otra casi plana. [T23, p. 15]

**Percentiles.** [T23, p. 16]

- **P50** es la mediana: la mitad de los requests tarda menos que eso.
- **P95:** el 95 % de los requests tarda X ms o menos.
- **P99:** el peor 1 % de las experiencias; con un millón de usuarios diarios son 10.000 personas.
- "El SLI que vieron en Resiliencia es un percentil." [T21, p. 7]

`T22` muestra la misma trampa con una tasa de error: el promedio global esconde a la variante que falla. [T22, p. 34] Ver [Feature flagging](feature-flagging.md#el-promedio-global-esconde-el-incidente).

### RED y USE

| | RED Model | USE Metrics |
|---|---|---|
| Se aplica a | Cada servicio: cómo lo ve el usuario | Cada recurso: CPU, memoria, disco, red, pools |
| Primera letra | **Rate:** requests por unidad de tiempo | **Utilization:** porcentaje del tiempo que el recurso estuvo ocupado |
| Segunda letra | **Errors:** requests con error | **Saturation:** trabajo que el recurso no da abasto a atender y queda en cola (threads, conexiones del pool) |
| Tercera letra | **Duration:** duración de cada request | **Errors:** errores del propio recurso (disco, red, contenedores reiniciados por memoria) |

"La alerta salta por RED; se investiga con USE." [T23, p. 17] La unidad de resiliencia ya los nombraba junto con las *golden signals*. [T21, p. 49] El gateway y la malla producen métricas RED sin tocar el código. [T24, p. 28] [T25, p. 11]

## Prometheus y Grafana

**Prometheus: modelo pull.** Prometheus va a buscar `/metrics` a cada servicio cada `scrape_interval` (15 s en el diagrama); "el servicio no sabe que existe". Con PromQL alimenta los dashboards de Grafana, y Alertmanager manda las alertas a Slack o PagerDuty, hacia quien está de guardia. [T23, p. 18]

```promql
# Tasa de errores HTTP en 5 minutos
rate(http_requests_total{status=~"5.."}[5m])

# Latencia P99
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

Fuente: [T23, p. 18].

**Grafana.** Se conecta a Prometheus, Loki, Tempo, Jaeger e InfluxDB; no guarda datos, los visualiza desde cada backend. Los dashboards son código (JSON exportable), tiene alertas con notificaciones y es "un solo *pane of glass* para los 3 pilares". Stack típico: Prometheus + Loki + Grafana, y Jaeger o Tempo para las trazas. [T23, p. 19]

## Tracing distribuido

El tracing es la capacidad de rastrear una solicitud a través de varios servicios. [T23, p. 20]

- **Trace:** la vista de punta a punta de una solicitud a medida que pasa por los servicios.
- **Span:** una unidad individual de trabajo dentro de un solo servicio. Varios spans forman un trace.
- Cada span tiene un *parent*: el conjunto arma el **árbol causal** del request. [T23, p. 20]

Cómo leer el *waterfall*: el span raíz es el request del usuario; cada servicio agrega su span; el **ancho es la duración** y la **profundidad, las dependencias**; el span más ancho indica dónde está el cuello de botella. [T23, p. 21]

### Propagación del contexto

Cada servicio recibe y envía el trace ID en un header HTTP: [T23, p. 22]

```text
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  └──────────── trace-id ────────┘ └─ parent-id ──┘ └ flags
             └ versión
```

- Es el estándar **W3C Trace Context** (W3C Recommendation, 2020).
- En colas y Kafka viaja igual, en los headers del mensaje.
- **Un servicio sin instrumentar corta la traza.**
- El SDK lee el header al recibir y lo escribe al llamar: el código de negocio no se entera. [T23, p. 22]

**Tracing efectivo:** "Event Log + Trace ID = Tracing". El trace ID se propaga en cada request, se incluye en cada log estructurado y el backend correlaciona trazas y logs automáticamente. [T23, p. 23]

*Inferencia:* la malla no hace esta propagación por la app. El proxy emite spans, pero no puede saber que un request que sale es consecuencia de uno que entró; la app, o su SDK, tiene que copiar `traceparent`. [T25, p. 12] [T25, p. 21] Ver [Service mesh](service-mesh.md#qué-se-registra-métricas-y-trazas).

## OpenTelemetry

Cuatro ideas: [T23, p. 24]

1. Estandariza la observabilidad y la telemetría: se instrumenta una vez y se cambia de backend por configuración.
2. Propagación automática del trace context.
3. Instrumentación con SDK más auto-agente.
4. **OTLP**, un protocolo de transporte universal.

**Instrumentación automática o manual.** La automática no cambia el código: en Java, `java -javaagent:opentelemetry-javaagent.jar -jar app.jar` intercepta HTTP, JDBC y Kafka. La manual agrega contexto de negocio, por ejemplo un span `process-order` con los atributos `order.id`, `order.amount` y `user.tier`. "Auto da la estructura; manual agrega el contexto de negocio. Lo ideal: combinar las dos." [T23, p. 25]

**OTel Collector: el router de la telemetría.** Los servicios, con el SDK, le hablan OTLP al Collector, que tiene tres etapas: [T23, p. 26]

| Etapa | Qué hace |
|---|---|
| Receive | Recibe OTLP (gRPC o HTTP), Jaeger, Zipkin, StatsD y otros |
| Process | Sampling (no se guarda el 100 % de las trazas), enriquecimiento, filtrado y batching |
| Export | Envía a varios backends a la vez: Jaeger (trazas), Prometheus (métricas), Loki (logs) |

Desacopla: los servicios solo hablan OTLP con el Collector, y cambiar de backend no los toca. [T23, p. 26]

**Backends para tracing.** [T23, p. 27]

| Backend | Fortaleza | Nota |
|---|---|---|
| Jaeger | Tracing puro, liviano | Ideal para empezar |
| SigNoz | Los tres pilares integrados | Alternativa open source a Datadog |
| ELK | Logs y búsqueda full-text | Ecosistema establecido y maduro |

## Del síntoma a la causa

La lámina toma una tabla del libro de SRE de Google: [T23, p. 28]

| Síntoma | Causa |
|---|---|
| Estoy respondiendo HTTP 500 o 404 | Los servidores de base de datos rechazan conexiones |
| Mis respuestas son lentas | La CPU está sobrecargada, o hay un cable dañado que produce pérdida parcial de paquetes |
| Usuarios de una región no reciben cierto contenido | La CDN bloqueó algunas IP de clientes |
| Contenido privado quedó público | Un despliegue nuevo dejó de aplicar las ACL |

**Los tres pilares conectados.** La alerta de Grafana (pico de `error_rate` a las 10:23:41) trae el trace ID del request culpable gracias a un **exemplar**: "el trace ID que Prometheus guarda junto a una muestra de la métrica". Con ese trace ID, Jaeger muestra qué span falló y en qué servicio (`db-lookup`, 3.200 ms), y los logs filtrados por el mismo trace ID dan el detalle (`Connection pool exhausted: max=10`). Causa raíz: un pool de conexiones insuficiente. [T23, p. 29]

**El flujo de investigación.** [T23, p. 30]

| Paso | Pregunta |
|---|---|
| Alerta | Grafana: pico de `error_rate` |
| Métrica | ¿Qué servicio? |
| Trace | ¿Qué request? |
| Span | ¿Qué operación? |
| Log | ¿Qué error? |

"Métricas → Traces → Spans → Logs → Causa." [T23, p. 30] Como ejemplo de producción real, la clase muestra `play.uptrace.dev`, un e-commerce con más de diez servicios y tráfico en vivo. [T23, p. 31]

## Una traza que cruza la cola: el caso del TP

El ejemplo es del Álbum: "Beto acepta una oferta de Ana". El request va del cliente al gateway y de ahí al servicio `intercambio` por HTTP; `intercambio` usa su Postgres y Valkey, y publica eventos en NATS; `coleccion` los consume y usa su propio Postgres y Valkey. [T23, p. 32]

- El cliente recibe un **202**; la traza sigue después.
- "Un `traceId` para la aceptación, la confirmación y la compensación: la evidencia del TP (27/11)."
- El SQL y los comandos de Valkey quedan en los spans. [T23, p. 32]

**Propagar a mano en NATS.** HTTP, Postgres y Valkey se instrumentan solos; NATS en Node no tiene auto-instrumentación. Al publicar se inyecta el contexto en los headers del mensaje (`propagation.inject`) y al consumir se extrae (`propagation.extract`) para que el span nuevo, de tipo `CONSUMER`, sea hijo del que publicó. Es el mismo `traceparent`, en headers del mensaje. "Sin estas líneas: 3 trazas sueltas." En Kafka o RabbitMQ lo hace el agente. [T23, p. 33]

**`traceId` y `correlationId`.** [T23, p. 34]

| | `traceId` | `correlationid` |
|---|---|---|
| Qué identifica | Una ejecución técnica | Una operación de negocio: el intercambio |
| Vida | Nace con el request | Vive días, sobrevive reintentos |
| Dónde viaja | En `traceparent` | En el CloudEvent, como extensión |
| Quién lo genera | OpenTelemetry | Tu dominio |

Si la compensación la dispara un vencimiento, el job no tiene un request de origen y arranca otra traza. Se une a la original por el `correlationid` como atributo del span, o con un *span link*, un span que apunta a otra traza. [T23, p. 34] El `correlationId` y el `causationId` como extensiones de CloudEvents están en [Comunicación asincrónica y Kafka](comunicacion-asincrona-y-kafka.md). [T14, p. 54]

**Demo.** `cd demo-intercambio-async && ./demo.sh up` levanta Jaeger en `http://localhost:16686`. El recorrido: [T23, p. 35]

1. Con *Min Duration* 100 ms, el span ancho es la reputación.
2. El 202 sale antes; la traza cruza NATS.
3. Los atributos `app.*` son el wide event.
4. En la compensada se ven el `ROLLBACK` y el `del lock:oferta:N`.
5. `./demo.sh logs`: un `traceId` en tres servicios.
6. `./demo.sh cortar`: la misma operación, tres trazas.
7. "¿El lock aguanta si Valkey se cae? No es la reserva del TP: esa va durable en Colección."

La demo `demo-intercambio-async` no está incorporada: el clon local de las guías es del 17 de octubre de 2026 y solo trae `capacidad-sync-async`. Ver [Laboratorios de la cátedra](laboratorios-de-la-catedra.md).

## Relación con el TP

- **Evidencia pedida.** La consigna general pide mostrar un mismo `correlationId` o `traceId` recorriendo los servicios en una apertura, un intercambio exitoso y un intercambio con fallo y compensación; alcanza con logs estructurados o con un backend de tracing, sin service mesh ni cluster. [E03, p. 11] `T23` muestra cómo se ve esa evidencia para el intercambio. [T23, p. 32]
- **Tercera entrega.** `E06` deja la observabilidad avanzada fuera de lo exigido: alcanzan los logs accesibles por `kubectl` y las sondas de salud. [E06]
- **Fecha.** `T23` menciona "la evidencia del TP (27/11)" sin decir a qué entrega corresponde. [T23, p. 32] Queda registrado en [Dudas y conflictos](../dudas-y-conflictos.md).
- **Contratos.** La segunda entrega pide transportar el identificador de correlación en las APIs y en el envelope de los eventos. [E05]
