# Resiliencia y escalabilidad en Kubernetes

## Fuente y alcance

Síntesis de `T21`, presentación de la cátedra con los docentes de la materia y el subtítulo "Patrones y buenas prácticas, de un Pod a múltiples regiones". **Inferencia de catalogación:** el ciclo 2026 se asigna por la configuración de la materia y la fecha de creación del PDF; no figura en la portada. [T21, p. 1] Tiene cuatro partes: patrones de resiliencia, alta disponibilidad y escalado en Kubernetes, multi-región, y validación y costo. [T21, p. 3] Varias láminas son demos interactivas exportadas como captura; acá se resume la conclusión que cada una enuncia. La presentación cierra anunciando Service Mesh como próxima clase. [T21, p. 55]

Se apoya en [Programar microservicios para Kubernetes](programar-microservicios-para-kubernetes.md), que da por vista, y retoma la sobrecarga de [Comunicación síncrona](comunicacion-sincrona.md). [T21, p. 26] Presenta como recapitulación las falacias de la computación distribuida, que no aparecen en el material incorporado antes; los modelos de falla de red, nodos y tiempo están en [Sistemas distribuidos](sistemas-distribuidos.md). [T21, p. 4]

**Advertencia de fuentes:** la tabla de recapitulación de la lámina 26 dice "Visto en Programar microservicios para Kubernetes", pero `T18` no trata el PodDisruptionBudget ni `preStop`. `preStop` aparece en `T16`; el PDB no figura en ninguna fuente incorporada antes de `T21`. [T21, p. 26] [T16, p. 21] Ver [Dudas y conflictos](../dudas-y-conflictos.md).

## Caso motivador y conceptos

Los grandes incidentes rara vez empiezan con una falla grande: un componente interno se degrada, los clientes reintentan de inmediato y en masa, la carga extra lo satura más, la falla se propaga a servicios sin relación directa, y recuperarse exige frenar el tráfico, no solo arreglar la causa. Los ejemplos son AWS us-east-1 (diciembre de 2021), Cloudflare (julio de 2019) y Facebook (octubre de 2021). [T21, p. 2]

| Concepto | Pregunta que responde |
|---|---|
| Resiliencia | ¿Qué pasa cuando algo falla? |
| Escalabilidad | ¿Qué pasa cuando la carga crece? |
| Disponibilidad | ¿Qué porcentaje del tiempo funciona? |

Fuente: [T21, p. 3]. Las falacias de la computación distribuida (la red es confiable, la latencia es cero, el ancho de banda es infinito, la topología no cambia, hay un solo administrador) llevan a una consecuencia: **toda llamada remota puede fallar, demorar o duplicarse, y el diseño tiene que asumirlo**. [T21, p. 4]

### Falla parcial, en cascada y retry storm

- **Una respuesta lenta es peor que un error.** Con tres slots por salto, un `POST /checkout` contra una base lenta y sin timeout deja ocupados los tres slots de cada salto, y el cuarto pedido falla en el gateway; contra una base caída que responde 503 de inmediato, los slots quedan libres. "Un 200 tardío ocupa toda la cadena. Un 503 rápido la libera." [T21, p. 5]
- **Reintentar en cada capa multiplica la carga.** Si cliente, gateway y checkout reintentan tres veces cada uno, un pedido llega 27 veces a Pagos. Con jitter el pico baja a 9, y con un *retry budget* de 10 % llegan 11 pedidos, no 270. [T21, p. 6] *Inferencia:* el 270 corresponde a una oleada de diez usuarios con tres capas reintentando.

### SLI, SLO, SLA y error budget

| Término | Qué es | Ejemplo |
|---|---|---|
| SLI | Métrica medida | % de requests con 2xx en menos de 300 ms |
| SLO | Objetivo interno | 99,9 % en 30 días |
| SLA | Compromiso contractual | Crédito si baja de 99,5 % |
| Error budget | 100 % − SLO | 43 min/mes de falla permitida |

Fuente: [T21, p. 7]. Si el budget se agota, se frenan las funcionalidades nuevas y se prioriza la confiabilidad. **El SLO decide cuánta resiliencia vale la pena construir.** [T21, p. 7]

### Disponibilidad compuesta

Las dependencias en serie multiplican su disponibilidad: con dependencias al 99,9 %, una da 99,9 %, cinco dan ≈ 99,5 % (unas 3,6 horas por mes caído) y diez ≈ 99,0 %. Las réplicas en paralelo la mejoran: dos instancias independientes al 99,9 % dan 99,9999 %. Los patrones que siguen buscan romper la dependencia en serie. [T21, p. 8] *Cálculo:* 0,999⁵ ≈ 0,995 y 1 − 0,001² = 0,999999.

## 1. Patrones de resiliencia

| Patrón | Protege contra |
|---|---|
| Timeout | Esperas infinitas |
| Retry + backoff + jitter | Fallas transitorias |
| Idempotencia | Duplicados por reintentos |
| Circuit breaker | Insistir sobre un servicio caído |
| Bulkhead | Que una falla consuma todos los recursos |
| Load shedding / rate limit | Sobrecarga |
| Fallback | Una dependencia no disponible |

Fuente: [T21, p. 9].

### Timeouts y deadline propagado

La demo compara una query que tarda 2,5 s con un cliente que espera 2 s. Con **timeouts locales** (cliente 2 s; checkout, pagos y base 3 s cada uno) el cliente se va, pero la cadena sigue y el cobro queda hecho. Con el **deadline propagado** (2 → 1,5 → 1 → 0,5 s) la cadena se cancela. "Si el deadline no baja, Pagos sigue cobrando después del timeout." [T21, p. 10]

### Retries

```python
def delay(attempt, base=0.1, cap=10.0):
    # full jitter
    return random.uniform(0, min(cap, base * 2 ** attempt))
```

- **Backoff** espacia los intentos; **jitter** desincroniza a los clientes y evita el *thundering herd*.
- **Retry budget:** como máximo, alrededor de 10 % de requests adicionales.
- Reintentar solo errores transitorios (503, timeouts), **nunca 4xx**.
- Reintentar **en un solo nivel** de la cadena.

Fuente: [T21, p. 11]. Ver cuánto reintentar según la demo de capacidad en [Comunicación síncrona](comunicacion-sincrona.md). [G10]

#### Reintentos en Jobs de Kubernetes

El Job `sync-facturas` usa `backoffLimit: 4`, `activeDeadlineSeconds: 600`, `ttlSecondsAfterFinished: 3600` y `restartPolicy: OnFailure` (`Always` no vale en un Job). Cada falla del Pod cuenta contra `backoffLimit`, cuyo valor por defecto es 6. Kubernetes aplica un backoff exponencial de 10 s, 20 s, 40 s…, con tope de 6 minutos, que **no es configurable y no lleva jitter**. [T21, p. 12]

| Campo | Qué controla |
|---|---|
| `backoffLimit` | Máximo de fallas antes de marcar el Job como `Failed` |
| `activeDeadlineSeconds` | Tiempo total; al vencer se terminan los Pods aunque queden reintentos |
| `restartPolicy: OnFailure` | Reinicia el contenedor en el mismo Pod, con el backoff del kubelet |
| `restartPolicy: Never` | Crea un Pod nuevo por intento y conserva los logs de cada falla |
| `backoffLimitPerIndex` | En Jobs indexados, límite por índice: una tarea mala no tumba al resto |
| `podFailurePolicy` | Distingue errores: `FailJob` (no reintentar), `Ignore` (no cuenta), `Count` |

Fuente: [T21, p. 13]. El ejemplo de `podFailurePolicy` falla el Job con el exit code 2 (error de validación, reintentar no sirve) e ignora la condición `DisruptionTarget` (preemption o drain), que no gasta el presupuesto. [T21, p. 13] Ver Jobs en [Kubernetes — Parte 1](kubernetes-parte-1.md#elegir-un-workload). [T16, p. 37]

### Idempotencia

En un `POST /payments` que el servidor cobra antes de que vuelva el 200, sin clave hay dos cobros; con `Idempotency-Key` queda uno y el reintento devuelve el 200 original. "Un timeout no alcanza. El reintento es seguro solo si el servidor recuerda la clave." [T21, p. 14] Ver [Estado distribuido](estado-distribuido.md). [T12, p. 20]

### Circuit breaker

Tres fallas consecutivas abren el breaker; mientras está **abierto** responde 503 localmente y Pagos no recibe el pedido. Tras esperar pasa a **semiabierto** y deja pasar una prueba: un 200 lo cierra y un 500 lo reabre. [T21, p. 15]

- **Dónde:** en el código, con Resilience4j (Java), Polly (.NET) o gobreaker (Go); en la infraestructura, con proxies o service mesh con *outlier detection*. [T21, p. 16]
- **Parámetros:** umbral de fallas, ventana, tiempo en abierto y llamadas de prueba. [T21, p. 16]
- Hay que medir y alertar cuando abre: **es un síntoma, no el problema**. [T21, p. 16]

### Bulkhead

Con los mismos cuatro slots en un pool único, recomendaciones lentas los llenan y checkout recibe 503; con un mamparo de dos slots por dependencia, checkout entra igual. En Kubernetes el mamparo se arma con ResourceQuota, node pools dedicados y Deployments por tipo de tráfico. [T21, p. 17] Ver el bulkhead medido en [Comunicación síncrona](comunicacion-sincrona.md). [T09, p. 26]

### Rate limiting y load shedding

Proteger al servicio rechazando trabajo temprano y barato: [T21, p. 18]

- **Token bucket** permite ráfagas y limita el promedio; **leaky bucket** emite a ritmo constante.
- **429 Too Many Requests** para un límite por cliente; **503 + `Retry-After`** para sobrecarga.
- **Load shedding:** descartar primero lo menos crítico, por prioridades.

"Rechazar el 10 % de los requests a tiempo es mejor que fallar el 100 % por saturación." [T21, p. 18]

### Ataques de volumen

| Capa | Ejemplo | Qué satura |
|---|---|---|
| L3/L4 | UDP flood, SYN flood, amplificación (DNS, NTP) | Ancho de banda, tabla de conexiones |
| L7 | HTTP flood, requests costosos (búsquedas, login) | CPU, threads, base |

Fuente: [T21, p. 19]. Un ataque L7 parece tráfico legítimo y pocos requests alcanzan si cada uno es caro. **El autoscaling no es una defensa:** escala el costo junto con el ataque (*denial of wallet*). Absorber el volumen es un problema del borde, no del clúster. [T21, p. 19]

La defensa es en capas, filtrando lo más lejos y lo más barato posible: CDN/Anycast con *scrubbing* (AWS Shield, Cloudflare, Cloud Armor), WAF (reglas, reputación de IP, CAPTCHA, geobloqueo), gateway o Ingress (rate limit por IP, API key o usuario, límites de conexiones y de tamaño del body) y, en el servicio, load shedding, colas acotadas y circuit breaker. [T21, p. 20] En Kubernetes: [T21, p. 21]

- no exponer NodePorts ni LoadBalancers directos: todo entra por CDN/WAF;
- límites en el Ingress (`limit-rps` y `limit-connections` en ingress-nginx, `RateLimitPolicy` en Gateway API/Envoy);
- timeouts cortos de cabecera y body contra ataques lentos (*slowloris*);
- NetworkPolicy y cuentas de servicio para limitar el movimiento lateral;
- tope en `maxReplicas` del HPA y presupuesto fijado, para evitar facturas y cascadas hacia la base;
- cachear en el CDN las respuestas costosas y proteger los endpoints caros;
- plan de respuesta: alertas por tráfico anómalo, runbook de bloqueo y contacto con el proveedor.

### Backpressure

Que el consumidor lento pueda frenar al productor: colas acotadas que rechazan o bloquean al llenarse, flow control (HTTP/2, gRPC, Reactive Streams) y, en Kafka, el ritmo que fija el consumidor con `poll` y `max.poll.records`. **Una cola sin límite solo traslada el problema a la memoria** (`OOMKilled`). [T21, p. 22]

### Fallbacks y degradación elegante

| Funcionalidad | Fallback |
|---|---|
| Recomendaciones | Lista genérica o la última calculada |
| Precio / stock | Cache con datos levemente viejos |
| Reseñas | Ocultar el bloque |
| Pago externo | Encolar y confirmar después |

Fuente: [T21, p. 23]. Hay que definir de antemano qué es crítico y qué es prescindible; los feature flags permiten degradar manualmente durante un incidente. [T21, p. 23]

### Dónde implementar cada patrón

| Nivel | Ventajas | Límites |
|---|---|---|
| Código o librería | Conoce la semántica de negocio | Se repite por lenguaje y equipo |
| API gateway | Punto central en el borde | No ve el tráfico interno |
| Plataforma (Kubernetes) | Uniforme y declarativa | Solo infraestructura |
| Service mesh | Retries, timeouts y circuit breaker sin código | Costo operativo |

Fuente: [T21, p. 24]. **Regla práctica: el negocio en el código, las políticas uniformes en la plataforma.** [T21, p. 24] Los niveles de gateway y de malla se desarrollan en [API Gateway](api-gateway.md) y [Service mesh](service-mesh.md). [T24, p. 28] [T25, p. 22]

## 2. Kubernetes: alta disponibilidad y escalado

### Lo que Kubernetes da y lo que no

El loop de reconciliación reinicia contenedores que mueren, reprograma Pods si cae un nodo, mantiene la cantidad de réplicas y saca del Service a los Pods que no están `Ready`. **No da** timeouts, retries ni circuit breakers en el código, idempotencia ni manejo de duplicados, ni capacidad ante la caída de una zona o región. [T21, p. 25]

| Mecanismo | Error común |
|---|---|
| Readiness, liveness, startup | Liveness que depende de la base: reinicios en cadena |
| Requests y limits | Sin requests, el scheduler no puede decidir |
| PodDisruptionBudget | Sin PDB, el drenado de un nodo baja todas las réplicas |
| Graceful shutdown | Sin `preStop` ni manejo de `SIGTERM`: errores en cada deploy |

Fuente: [T21, p. 26]. Como mínimo, **dos réplicas** para los servicios que deben estar siempre disponibles. [T21, p. 26]

### Distribución de réplicas

Que las réplicas no compartan el mismo punto de falla: repartirlas por nodo y por zona de disponibilidad con `topologySpreadConstraints` (`maxSkew: 1`, `topologyKey: topology.kubernetes.io/zone`), y usar `podAntiAffinity` para no ponerlas en el mismo nodo. Con `whenUnsatisfiable: DoNotSchedule`, si no hay capacidad en una zona el Pod queda `Pending`. [T21, p. 27]

### QoS, PriorityClass y preemption

| Clase QoS | Condición según la lámina | Orden de desalojo |
|---|---|---|
| BestEffort | Sin requests ni limits | Primero |
| Burstable | requests < limits | Intermedio |
| Guaranteed | requests = limits | Último |

Fuente: [T21, p. 28]. Una `PriorityClass` permite que un Pod crítico desplace (*preempt*) a otros de menor prioridad; sirve para separar servicios críticos, batch y Pods "de relleno". [T21, p. 28]

*Precisión por conocimiento general, no del repositorio:* Guaranteed exige requests iguales a limits de CPU y memoria en todos los contenedores, y Burstable es cualquier Pod que no califica en las otras dos clases (por ejemplo, con requests y sin limits). Ante presión de recursos, el kubelet también pondera la prioridad y cuánto excede cada Pod sus requests.

### Despliegues seguros

La mayoría de los incidentes empiezan con un cambio. [T21, p. 29]

| Estrategia | Idea |
|---|---|
| Rolling update | Reemplazo gradual (`maxSurge`, `maxUnavailable`) |
| Blue/green | Dos entornos completos; el cambio de tráfico es instantáneo |
| Canary | 5 % → 25 % → 100 % con análisis de métricas |

Fuente: [T21, p. 29]. Argo Rollouts o Flagger automatizan el canary y hacen rollback según el SLO. Combinado con feature flags: **desplegar no es lo mismo que liberar**. [T21, p. 29] La liberación con flags está en [Feature flagging](feature-flagging.md). [T22, p. 6] Ver [DevOps y CI/CD](devops-y-ci-cd.md#estrategias-de-despliegue) y [GitOps](gitops-y-entrega-continua.md).

**Demo: matar Pods bajo carga.** Con `k6 run --vus 20 --duration 2m` como carga constante, se borran Pods con `kubectl delete pod -l app=checkout --wait=false` y se drena un nodo con `kubectl drain`. Primero sin probes, sin PDB y sin `preStop`, para observar los 5xx; después con readiness, PDB, `preStop` y graceful shutdown, para comparar. [T21, p. 30]

### Escalar: vertical u horizontal

| | Vertical | Horizontal |
|---|---|---|
| Cómo | Más CPU o RAM por instancia | Más instancias |
| Límite | Tamaño máximo del nodo | Coordinación y estado |
| Disponibilidad | Un solo punto de falla | Mejora con réplicas |

Fuente: [T21, p. 31]. Escalar horizontalmente exige servicios sin estado, o con el estado externalizado. Pregunta de diseño: **¿qué pasa si hay 10 réplicas en vez de 1?** [T21, p. 31]

### HPA

`réplicas = ceil(réplicas_actuales × métrica_actual / métrica_objetivo)`. Con cuatro réplicas al 90 % de CPU y objetivo de 60 %, quedan seis. El ejemplo fija `minReplicas: 3`, `maxReplicas: 30` y `stabilizationWindowSeconds: 300` para bajar. `behavior` evita oscilaciones (*flapping*): subir rápido, bajar lento. **Requiere requests de CPU definidos.** [T21, p. 32]

| Tipo de métrica | Ejemplo |
|---|---|
| Resource | CPU, memoria |
| Pods | Requests por segundo por Pod |
| External | Longitud de cola, lag de Kafka |

Fuente: [T21, p. 33]. La CPU rara vez refleja la carga real. Prometheus Adapter expone métricas de Prometheus a la API del HPA; las buenas señales son RPS por réplica, latencia p95 y profundidad de cola. **La métrica tiene que moverse al agregar réplicas.** [T21, p. 33]

### VPA

El Vertical Pod Autoscaler recomienda o ajusta requests y limits. En modo `Off` solo recomienda; en `Initial` aplica al crear el Pod; en `Auto` reinicia Pods para aplicar. No combinar HPA y VPA sobre la misma métrica (CPU o memoria); el uso recomendado es el modo `Off` como guía para dimensionar. Requests bien ajustados bajan el costo y mejoran el scheduling. [T21, p. 34]

### KEDA

Escala según la demanda real de trabajo, **incluso hasta cero**. El `ScaledObject` de ejemplo apunta a `order-worker` con `minReplicaCount: 0`, `maxReplicaCount: 20` y un trigger de Kafka sobre el topic `orders` con `lagThreshold: "100"`. Soporta Kafka, colas, Prometheus, cron y otros, e internamente crea y gestiona un HPA. [T21, p. 35] `T18` lo propone para los consumidores continuos. [T18, p. 20]

### Nodos

Escalar Pods requiere nodos disponibles. Cluster Autoscaler ajusta los grupos de nodos existentes; Karpenter elige el tipo de instancia según los Pods pendientes. Aprovisionar un nodo toma de uno a varios minutos, así que los picos rápidos necesitan capacidad ya caliente: el *overprovisioning* usa Pods "de relleno" de baja prioridad que se desalojan cuando llega la carga real. [T21, p. 36]

### Escalar el estado

Los Pods escalan fácil; los datos no. Réplicas de lectura para repartir consultas, sharding para repartir escrituras, cache (Valkey/Redis) delante de la base y, en Kafka, particiones como unidad de paralelismo: **más consumidores que particiones no aumenta el throughput**. [T21, p. 37] Ver [Kafka](comunicacion-asincrona-y-kafka.md).

**Cuellos de botella aguas abajo.** 50 réplicas × 20 conexiones son 1000 conexiones a la base. Hace falta *connection pooling* (PgBouncer, pools acotados), un máximo de réplicas según la capacidad de la base, y rate limit y colas delante de los recursos no elásticos. "Escalar es una decisión de toda la cadena, no de un solo servicio." [T21, p. 38]

**Cache.** Cache-aside, TTL con tolerancia a datos viejos y CDN para contenido estático. Riesgo: *cache stampede*, muchas peticiones que recalculan el mismo valor al expirar; se mitiga con *request coalescing*, TTL con jitter y refresco anticipado. Invalidar es difícil: hay que decidir qué obsolescencia es aceptable. [T21, p. 39] Ver [Estado distribuido](estado-distribuido.md).

**Escalado asincrónico.** Absorber los picos en vez de resistirlos: *queue-based load leveling*, *competing consumers*, DLQ para mensajes que fallan N veces, y KEDA para escalar los workers según el lag. El esquema es API → cola o topic → workers con autoscaling → base. [T21, p. 40]

## 3. Multi-región

| Motivo | Detalle |
|---|---|
| Disponibilidad | Sobrevivir a la caída de una región entera |
| Latencia | Acercar el servicio a los usuarios |
| Compliance | Residencia de datos por país |

Fuente: [T21, p. 41]. Cuesta infraestructura duplicada, replicación y operación más compleja. Multi-AZ ya cubre las fallas de datacenter y las regiones completas caen muy poco: **decidir con SLO y RTO/RPO, no por moda**. [T21, p. 41]

- **Topologías.** No estirar un clúster entre regiones, porque etcd y el control plane esperan latencias de pocos milisegundos: un clúster por región, en *active-active* (ambas sirven) o *active-passive* (una espera), o en celdas independientes que limitan el *blast radius*. [T21, p. 42]
- **Tráfico global.** DNS con health checks para failover y routing por latencia, geografía o pesos; ExternalDNS para publicar registros desde recursos de Kubernetes; Anycast o un balanceador global para una sola IP. El TTL bajo acelera el failover pero multiplica las consultas, y algunos clientes lo ignoran. Health checks profundos y afinidad por región cuando hay datos locales. [T21, p. 43]
- **GitOps multi-clúster.** ApplicationSet o Flux generan una Application por clúster, con base común y overlays por región; Cluster API crea y mantiene clústeres de forma declarativa. Desplegar por olas: una región, validar con métricas, después las demás. "Un cambio malo desplegado en todas las regiones a la vez derrota el propósito del multi-región." [T21, p. 44] Ver [GitOps](gitops-y-entrega-continua.md).
- **Conectividad entre clústeres.** Multi-Cluster Services API, Cilium Cluster Mesh, Submariner o un service mesh multi-clúster. Preferir el tráfico local y cruzar regiones solo en failover o para datos que lo requieren. [T21, p. 45]

### Datos multi-región

| Estrategia | Trade-off |
|---|---|
| Replicación síncrona | Sin pérdida (RPO ≈ 0), pero latencia en cada escritura |
| Replicación asíncrona | Rápida, pero puede perder las últimas escrituras |
| Escritor único + réplicas | Simple; el failover requiere promoción |
| Multi-escritor | Requiere resolver conflictos (LWW, CRDT) |

Fuente: [T21, p. 46]. Bases globales: Spanner, CockroachDB, Aurora Global, DynamoDB Global Tables. Kafka: MirrorMaker 2 o Cluster Linking. **PACELC:** aun sin fallas hay que elegir entre latencia y consistencia. [T21, p. 46]

### Failover, RTO y RPO

El failover necesita detección con señales de varias fuentes. Uno automático erróneo puede ser peor que la falla; el riesgo es el *split-brain*, dos regiones que se creen principales y aceptan escrituras. La región pasiva debe estar caliente, con capacidad reservada, y la sobreviviente recibe el doble de carga. El failback se planifica resincronizando datos, y los runbooks se ensayan en *game days*. [T21, p. 47]

**RTO** es cuánto tiempo puede estar caído el sistema; **RPO**, cuántos datos puede perder. [T21, p. 48]

| Estrategia | RTO / RPO | Costo |
|---|---|---|
| Backup y restore | Horas / horas | Bajo |
| Pilot light | Decenas de minutos | Bajo-medio |
| Warm standby | Minutos | Medio |
| Active-active | Segundos / ≈ 0 | Alto |

Fuente: [T21, p. 48]. Velero respalda recursos y volúmenes de Kubernetes. **Un backup no probado no es un backup.** [T21, p. 48]

## 4. Validación y costo

- **Observabilidad.** Golden signals (latencia, tráfico, errores, saturación), RED para servicios y USE para recursos, alertas por *burn rate* del SLO en lugar de umbrales de CPU, trazas distribuidas para ubicar el origen de una cascada y paneles por región. [T21, p. 49] Ver [Observabilidad](observabilidad.md). [T23, p. 17]
- **Chaos engineering.** Definir el estado estable, formular una hipótesis ("si cae un nodo, no hay errores visibles"), inyectar la falla con *blast radius* acotado, observar, aprender y corregir. Herramientas: Chaos Mesh y Litmus; un *game day* simula la caída de una región con todo el equipo. [T21, p. 50]

| Prueba | Pregunta |
|---|---|
| Load | ¿Soporta la carga esperada? |
| Stress | ¿Dónde y cómo se rompe? |
| Spike | ¿Resiste un pico repentino? |
| Soak | ¿Se degrada con las horas (leaks)? |

Fuente: [T21, p. 51]. Herramientas: k6, Gatling y Locust. Probar en un entorno representativo, medir p95/p99 y no solo el promedio, y verificar que el autoscaling reacciona a tiempo. [T21, p. 51] Ver la performance como dimensión no funcional en [Testing](testing.md). [T10, p. 3]

**Cada "nueve" extra cuesta más:** 99 % permite ≈ 7,3 horas de falla por mes, 99,9 % ≈ 43 minutos y 99,99 % ≈ 4,4 minutos. Capacidad N+1 por zona y por región; overprovisioning frente a autoscaling es costo contra tiempo de reacción. El nivel de resiliencia se elige según el costo de la indisponibilidad para el negocio. [T21, p. 52]

### Ejercicio y checklist

**Ejercicio grupal:** el checkout de un e-commerce en Black Friday, con 10 veces el tráfico habitual, tiene que sobrevivir a la caída de una región. Definir SLOs y qué es crítico o prescindible, patrones entre servicios, métrica y límites del autoscaling, topología multi-región y estrategia de datos, y cómo validar el diseño antes del evento. [T21, p. 53]

| Resiliencia | Escalabilidad | Multi-región |
|---|---|---|
| Timeouts en toda llamada | Servicios sin estado | RTO/RPO definidos |
| Retries con jitter e idempotencia | HPA con métrica real | Failover ensayado |
| Circuit breaker y bulkhead | Límites en la base | Datos replicados y probados |
| Degradación definida | Autoscaling de Pods y nodos | Despliegue por olas |

Fuente: [T21, p. 54]. Antipatrones: retries en cada capa, liveness que depende de la base, colas sin límite, una sola zona y DR nunca probado. [T21, p. 54] Lecturas sugeridas: *Release It!* (M. Nygard), *Site Reliability Engineering* (Google) y el pilar de confiabilidad de AWS Well-Architected. [T21, p. 55]

## Relación con el TP

*Inferencia de estudio:* la [tercera entrega](tercera-entrega-album-cicd-aks.md) deja fuera de alcance la alta disponibilidad, el autoscaling y los acuerdos de latencia, y aclara que no implementarlos no resta puntos pero no mencionarlos sí. [E06] Esta unidad da el vocabulario para justificarlo: requests y limits por servicio (que `E06` sí exige), probes con sentido, rolling update con drenado, idempotencia en la apertura y la aceptación, y una sola capa de reintentos. [T21, p. 26] [T21, p. 11] [E06] El escenario 7 de `E01` (respuesta perdida, consulta del resultado real) es el caso de la lámina de idempotencia. [E01, p. 6] [T21, p. 14]
