# Service mesh

## Fuente y alcance

Síntesis de `T25`, presentación de la cátedra "Service Mesh: el tráfico entre servicios" (22 láminas, Microservicios · 73.40); el título del archivo es "Service Mesh: el data plane". **Inferencia de catalogación:** el ciclo 2026 se asigna por la configuración de la materia y la fecha de creación del PDF (20 de noviembre de 2026); la portada no lo indica ni nombra docentes. [T25, p. 1] Es la segunda parte de [API Gateway](api-gateway.md) y usa el mismo caso, la tienda de pedidos. [T25, p. 2] La unidad de resiliencia la había anunciado como la clase siguiente, y la de migración la daba por vista con el nombre `data-plane`. [T21, p. 55] [T15, p. 5]

**Cómo leer este apunte.** Igual que en `T24`, las láminas son animadas y el PDF conserva un solo estado de cada una, así que varias preguntas quedan sin su respuesta. Lo que no figura en la lámina está marcado como *inferencia* o *conocimiento general*. Ver [Dudas y conflictos](../dudas-y-conflictos.md).

## El punto de partida: tráfico este-oeste

El gateway resolvió el tráfico norte-sur. Entre servicios (este-oeste) quedan **tres preguntas abiertas**: [T25, p. 2] [T24, p. 26]

1. **¿Quién llama?**
2. **¿Va cifrado?**
3. **¿Cuántos reintentos?**

"Y un caso sin resolver: Envíos llama a Pedidos con la identidad de Ana, y Pedidos le cree." Pedidos lee el header `X-User-Id` sin poder saber si lo escribió el gateway. [T25, p. 2]

## De la librería al proxy

**En el código: una librería en cada servicio.** Con el supuesto de que cada equipo eligió su lenguaje (Go, Java, Python, Node.js), cada servicio implementa por su cuenta la identidad, el TLS y los reintentos, con valores distintos: 3, 5, 1 y 3 intentos. "La misma política, implementada cuatro veces y con valores distintos." La lámina pregunta qué hay que hacer, y quién, para pasar de 3 intentos a 2 en toda la tienda. [T25, p. 3] *Inferencia:* cuatro equipos tienen que cambiar cuatro configuraciones en cuatro librerías y desplegar cuatro servicios. Es el límite que la unidad de resiliencia le asigna al nivel "código o librería": conoce el negocio, pero se repite por lenguaje y equipo. [T21, p. 24]

**Proxy sidecar: la red sale del código.** El Pod de Pedidos pasa a tener dos contenedores: la app, con la lógica de negocio, y un proxy que se queda con la identidad, el TLS y los reintentos. "Las tres respuestas pasan al proxy. En la app queda la lógica de negocio." [T25, p. 4]

**Intercepción.** La app de Pedidos llama a `http://facturacion` como siempre. Las reglas de red del Pod desvían la llamada a su proxy, que la manda al proxy del Pod de Facturación, y este la entrega a la app. "Malla de servicios (service mesh): cada llamada pasa por dos proxies. Juntos son el data plane." La lámina pregunta qué hubo que cambiar en el código de Pedidos. [T25, p. 5] *Inferencia:* nada; la app sigue hablando HTTP con el nombre del Service y el desvío es transparente. [T25, p. 7]

**Control plane y data plane.** El control plane tiene la política (por ejemplo, intentos: 2) y la distribuye a todos los proxies, que quedan con el mismo valor. Otro día el control plane no responde y Pedidos necesita llamar a Facturación; la lámina pregunta qué pasa con esa llamada. [T25, p. 6] *Inferencia:* la llamada funciona, porque cada proxy ya tiene la política y el control plane no está en el camino del request. Lo que se pierde mientras tanto es la capacidad de cambiar políticas y, si la caída dura, de renovar los certificados, que vencen en 24 h. [T25, p. 8]

El mismo par de términos aparece en otras tres unidades, siempre con la misma idea: el control plane decide y distribuye, lento y auditado; el data plane ejecuta en cada operación.

| Dónde | Control plane | Data plane |
|---|---|---|
| Malla de servicios | Distribuye políticas y firma certificados [T25, p. 6] [T25, p. 8] | Los proxies por los que pasa cada llamada [T25, p. 5] |
| Feature flags | Define y publica las reglas [T22, p. 8] | Evalúa la regla en cada operación [T22, p. 8] |
| GitOps | ArgoCD; si cae, las aplicaciones siguen corriendo [T20, p. 20] | — |

## ¿Va cifrado? mTLS e identidad de workload

**mTLS.** Los dos extremos presentan un certificado: el proxy de Pedidos presenta `pedidos` y el de Facturación presenta `facturacion`. Así Pedidos sabe que habla con Facturación y Facturación sabe que la llama Pedidos. "Lo hacen los proxies: cada app sigue hablando HTTP con el suyo." [T25, p. 7]

**Identidad de workload: un certificado por servicio.** Con el supuesto de un namespace `tienda` y una ServiceAccount por servicio, el control plane firma un certificado para cada proxy; vence en 24 h y se renueva solo. La identidad tiene forma de SPIFFE ID: [T25, p. 8]

```text
spiffe://cluster.local/ns/tienda/sa/pedidos
         └ dominio de confianza
                          └ namespace
                                    └ ServiceAccount
```

"Como el JWT de Ana: un emisor firma y el receptor verifica. Acá el sujeto es un servicio." [T25, p. 8] La ServiceAccount como identidad de una aplicación está en [Kubernetes — Parte 1](kubernetes-parte-1.md). [T16, p. 53]

**PeerAuthentication: PERMISSIVE y STRICT.** El estado visible de la lámina tiene `modo: STRICT` y a Envíos, que "ya tiene proxy y certificado", llamando a Pedidos por mTLS con el header `X-User-Id: ana`. La lámina pregunta qué responde Pedidos. [T25, p. 9] *Conocimiento general (Istio, no figura en el estado visible):* `PERMISSIVE` acepta tanto mTLS como texto plano y sirve para migrar de a poco; `STRICT` acepta solo mTLS. *Inferencia:* con `STRICT` solo, Pedidos le responde bien a Envíos. mTLS prueba **quién es** el que llama, no **qué puede hacer**: el certificado de Envíos es válido y nada le impide mandar el header.

## ¿Quién llama? AuthorizationPolicy

Una `AuthorizationPolicy` sobre Pedidos dice "puede llamar: gateway", con el supuesto de que el gateway también está en la malla. El gateway llega con mTLS y `X-User-Id: ana` y recibe 200; Envíos llega con mTLS y el mismo header y recibe 403 en el proxy de Pedidos. "El header `X-User-Id` vale porque solo el gateway puede llegar a Pedidos con él." [T25, p. 10]

Con esto se cierra el caso que venía de la primera parte: el gateway descarta el header si llega de Internet, y la malla impide que lo invente otro servicio. [T24, p. 17] [T25, p. 2] *Inferencia:* una NetworkPolicy también limita quién llega a un Pod, pero con reglas L3/L4 por selector de Pods; la `AuthorizationPolicy` decide por la identidad que prueba el certificado. [T17, p. 25]

## ¿Qué se registra? Métricas y trazas

**Métricas RED por llamada, sin tocar el código.** Como cada llamada pasa por un proxy, la malla mide cada tramo (valores ilustrativos): [T25, p. 11]

| Llamada | Rate | Errores | p99 |
|---|---|---|---|
| gateway → pedidos | 120 req/s | 4 % | 2060 ms |
| pedidos → facturacion | 120 req/s | 4 % | 2030 ms |
| facturacion → seguimiento | 120 req/s | 4 % | 2000 ms |

"El error aparece en toda la cadena. La llamada más profunda que lo muestra dice dónde empieza." [T25, p. 11] En el ejemplo, Seguimiento. RED está en [Observabilidad](observabilidad.md#red-y-use). [T23, p. 17]

**Trazas: el proxy emite spans, la app propaga.** A Facturación le llega un request con `traceparent: 4bf9…` y su proxy emite el span `pedidos → facturacion` de esa traza. Para responder, la app llama a Seguimiento y el proxy ve salir otro request. La lámina pregunta cómo sabe el proxy que el que sale es consecuencia del que entró. [T25, p. 12] No lo sabe: "La traza sigue necesitando que cada app copie `traceparent`." [T25, p. 21] *Inferencia:* eso es lo que hace el SDK de OpenTelemetry, que lee el header al recibir y lo escribe al llamar. [T23, p. 22] La malla no reemplaza la instrumentación.

## ¿Cuántos reintentos? Timeout, retry budget y outlier detection

**Timeout y reintentos en el proxy.** Una `HTTPRoute` para Facturación fija intentos: 3 y timeout: 2 s. En un `POST /facturas`, el primer intento (a `f2`) no responde en 2 s y el proxy reintenta en otro Pod (`f3`). La lámina pregunta qué tiene que garantizar Facturación si el proxy reintenta un `POST`. [T25, p. 13] *Inferencia:* idempotencia. Que no haya respuesta en 2 s no prueba que el primer intento no se procesó, así que sin una clave de idempotencia puede haber dos facturas. [T12, p. 19] [T11, p. 18]

**Reintentos en cadena: retry budget.** Con 3 intentos en cada salto (hacia Pedidos, hacia Facturación y hacia Seguimiento, que responde 503), 100 requests de usuarios se convertirían en 2700 sin tope. Una `DestinationRule` con `retryBudget` limita los reintentos al 20 % del tráfico, y los topes pasan a ser ≤ 120, ≤ 144 y ≤ 173. "Retry budget: un tope a los reintentos como proporción del tráfico. Pasado el tope, el proxy devuelve el error." [T25, p. 14] *Inferencia sobre la cuenta:* 100 × 3 × 3 × 3 = 2700, y 100 × 1,2 = 120, × 1,2 = 144, × 1,2 ≈ 173.

Es la *retry storm* de [Resiliencia y escalabilidad](resiliencia-y-escalabilidad.md#falla-parcial-en-cascada-y-retry-storm). Allá el ejemplo usa un budget de 10 % y acá de 20 %: son valores de ejemplo, no una regla distinta. [T21, p. 6]

**Outlier detection: expulsar al Pod que falla.** El proxy de Pedidos elige un Pod de Facturación en cada request. Una `DestinationRule` fija "expulsar tras 5 errores · 30 s" y "máximo expulsado: 1 de 3". El proxy deja de enviarle a `f2` durante 30 s y después vuelve a probar. La lámina pregunta si, cuando empiezan a fallar también `f1` y `f3`, el proxy expulsa a los tres. [T25, p. 15] *Inferencia:* no, porque la política permite expulsar como máximo 1 de 3; sin ese tope el proxy se quedaría sin destinos. En el diagrama los tres Pods figuran `Ready`: la expulsión la decide el proxy que llama, por los errores que observa, y es independiente de la readiness. La unidad de resiliencia lo nombra como la forma de tener un circuit breaker en la infraestructura. [T21, p. 16]

## El costo: dónde corre el proxy

**Sidecar: un proxy por Pod.** Con 3 réplicas por servicio repartidas en 3 nodos hay 12 proxies × 60 MB = 720 MB. "Cada proxy consume memoria y CPU, y cada llamada cruza dos: suma menos de 1 ms, según Istio." La lámina pregunta qué hay que reiniciar para actualizar el proxy por una vulnerabilidad. [T25, p. 16] *Inferencia:* los 12 Pods, porque el proxy es un contenedor de cada Pod.

**Un proxy por nodo.** Con un DaemonSet, los 12 Pods comparten 3 proxies que hacen L4 y L7. "Capa 4 (conexiones, mTLS, identidad) es simple y se puede compartir. Capa 7 (interpretar HTTP) es lo complejo." [T25, p. 17]

**Ambient: L4 por nodo, L7 donde hace falta.** En el modo ambient de Istio hay un `ztunnel` por nodo, que escucha dentro de la red de cada Pod. Con solo `ztunnel` se obtiene cifrado e identidad, autorización por identidad y métricas de TCP. "Para interpretar HTTP hace falta un waypoint: un proxy de capa 7 aparte, solo para quien lo necesita." La lámina pregunta cuáles de las políticas de la clase necesitan un waypoint. [T25, p. 18] *Inferencia:* las que miran el request HTTP —timeout y reintentos por request, retry budget, outlier detection por errores HTTP, métricas RED por llamada y spans—; mTLS, la identidad SPIFFE y la `AuthorizationPolicy` por identidad se resuelven con `ztunnel`.

Las tres variantes: [T25, p. 19]

| Dónde corre el proxy | Capa 4 | Capa 7 | Ejemplos |
|---|---|---|---|
| **Sidecar:** un proxy por Pod | El proxy del Pod | El proxy del Pod | Istio (sidecar), Linkerd |
| **Ambient:** separado por capa | `ztunnel`, uno por nodo | Waypoint, aparte y opcional | Istio (ambient) |
| **Kernel y nodo** | eBPF, en el kernel | Envoy, uno por nodo | Cilium |

"Cilium comparte por nodo el proxy de capa 7: la decisión que Istio descartó. Son dos criterios de diseño." [T25, p. 19]

## Adopción: qué se paga y cuándo conviene

Para la tienda (4 servicios, 12 Pods): [T25, p. 20]

| Qué se paga | Qué se obtiene |
|---|---|
| Un proxy en cada llamada: latencia y memoria | ¿Va cifrado? mTLS, sin tocar el código |
| Un control plane que operar y actualizar | ¿Quién llama? Identidad y política por servicio |
| Una capa más donde buscar cuando algo falla | ¿Cuántos reintentos? La misma política en todos |

"A cambio: las tres respuestas, iguales en todos los servicios y fuera del código." La lámina cierra con una pregunta sin respuesta única: con 4 servicios y 12 Pods, ¿cuál de las tres preguntas justificaría una malla? [T25, p. 20] Otras unidades dan el mismo balance: el mesh resuelve retries, timeouts y circuit breaker sin código a cambio de costo operativo, y un sidecar sirve para una función transversal de transporte, no para la semántica del negocio, y agrega costo, recursos y modos de falla. [T21, p. 24] [T18, p. 26]

## La llamada completa y el mapa de la clase

En el cierre, el gateway se identifica ante Pedidos por certificado; Envíos recibe 403 si intenta llamar con `X-User-Id: ana`; la llamada de Pedidos a Facturación va por mTLS; Facturación copia `traceparent`; y hacia Seguimiento hay 3 intentos, con tope. "Y queda registrado por llamada. La traza sigue necesitando que cada app copie `traceparent`." [T25, p. 21]

| Pregunta | Contenido (láminas) | Se conecta con |
|---|---|---|
| ¿Va cifrado? | mTLS (7); identidad y certificado (8); PERMISSIVE y STRICT (9) | TLS del gateway |
| ¿Quién llama? | SPIFFE ID (8); AuthorizationPolicy (10); `X-User-Id`, resuelto (10) | JWT e identidad |
| ¿Cuántos reintentos? | HTTPRoute y timeout (13); retry budget (14); outlier detection (15) | Resiliencia |
| ¿Qué se registra? | Métricas RED (11); spans del proxy (12); la app propaga (12) | Observabilidad |

Fuente: [T25, p. 22]. El mapa agrega el costo (dónde corre el proxy: sidecar, nodo, ambient; láminas 16–19) y cuándo conviene (20). Cierre: "La malla responde lo mismo para todos los servicios. Lo que necesita conocer el negocio queda en cada uno." [T25, p. 22]

## Relación con el programa y con el TP

- **Programa.** `T01` ubicaba en la clase de service mesh y seguridad: Istio/Linkerd, canary deployments, mirroring de tráfico, mTLS (Zero Trust) y microsegmentación. [T01, p. 13] `T25` desarrolla mTLS, identidad, autorización, reintentos y las variantes del data plane; no trata canary ni mirroring. El ruteo por cohorte con un `VirtualService` de Istio está en el apéndice de [Feature flagging](feature-flagging.md#red-routing-por-cohorte). [T22, p. 42]
- **Kubernetes.** El sidecar como contenedor auxiliar del Pod y su inyección por admisión mutante están en [Kubernetes — Parte 1](kubernetes-parte-1.md). [T16, p. 17] [T16, p. 54] En GitOps, el sidecar que inyecta el mesh es un drift esperado. [T20, p. 13]
- **TP.** La consigna general no exige service mesh: para la evidencia de observabilidad alcanza con logs estructurados o un backend de tracing. [E03, p. 11]
