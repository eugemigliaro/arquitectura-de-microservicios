# API Gateway

## Fuente y alcance

Síntesis de `T24`, presentación de la cátedra "API Gateway: el borde de un sistema de microservicios" (28 láminas, Microservicios · 73.40). **Inferencia de catalogación:** el ciclo 2026 se asigna por la configuración de la materia y la fecha de creación del PDF (20 de noviembre de 2026); la portada no lo indica ni nombra docentes. [T24, p. 1] Es la primera de dos partes: [Service mesh](service-mesh.md) arranca "donde quedó la primera parte". [T25, p. 2]

**Cómo leer este apunte.** Las láminas son animadas y el PDF conserva un solo estado de cada una. Por eso varias preguntas quedan planteadas sin su respuesta, y de las láminas rotuladas "Caso 3 de 3" solo se ve el último caso. [T24, p. 8] [T24, p. 22] Toda respuesta que no figura en la lámina está marcada como *inferencia* o *conocimiento general*; el caso está registrado en [Dudas y conflictos](../dudas-y-conflictos.md).

**Caso conductor: la tienda de pedidos.** Ana abre su pedido en la app y ve tres datos: el pedido 42 (2 productos, en preparación), la factura A-0001 por $ 84.000 y el envío (en camino, llega el jueves). Cada dato tiene un servicio dueño: Pedidos, Facturación y Envíos. La pregunta de partida es qué tiene que saber la app para mostrar esa pantalla. [T24, p. 2]

## Del acceso directo al gateway

**Acceso directo.** La app conoce tres direcciones (`pedidos.tienda.example`, `facturas.tienda.example`, `envios.tienda.example`), manda el token a cada una y cada servicio lo verifica por su cuenta. Seis meses después el equipo de Envíos separa Seguimiento en un servicio nuevo, pero la app instalada es la misma. La lámina pregunta qué ve Ana y quién lo puede arreglar. [T24, p. 3] *Inferencia:* la app sigue pidiendo el seguimiento a la dirección vieja, así que esa parte de la pantalla se rompe, y solo se arregla publicando una versión nueva de la app y esperando a que cada usuario la actualice. El cliente quedó acoplado a la forma interna del sistema.

**API gateway.** Un único punto de entrada: la app conoce una sola dirección (`api.tienda.example`) y el gateway decide a qué servicio va cada request, incluido el nuevo Seguimiento. Sobre cada request toma **cuatro decisiones**: [T24, p. 4]

1. **A dónde va.**
2. **Quién es.**
3. **Cuánto puede pedir.**
4. **Qué queda registrado.**

**En Kubernetes.** El tráfico que entra desde Internet es **norte-sur**. Un `Gateway` (HTTPS, puerto 443) recibe y una `HTTPRoute` manda `/pedidos`, `/facturas` y `/envios` a los Services correspondientes: eso resuelve la decisión 1. "Identidad, límites y registro son políticas que agrega cada implementación", es decir, las decisiones 2 a 4 no vienen dadas por el recurso estándar. [T24, p. 5] Los recursos están en [Kubernetes — Parte 2](kubernetes-parte-2.md#entrada-al-clúster-ingress-y-gateway-api) y la implementación del cluster del TP, Envoy Gateway, en [Kubernetes en AKS](kubernetes-en-aks.md). [T17, p. 23] [G09]

## 1. A dónde va el request

La lámina de apertura enumera las preguntas del bloque: qué regla manda el request a qué servicio, qué pasa si hay dos versiones del mismo servicio, si el servicio habla otro protocolo, si la pantalla necesita tres servicios y si cada tipo de cliente necesita otra respuesta. [T24, p. 6]

### Ruteo por path

La `HTTPRoute` de `api.tienda.example` tiene reglas por prefijo de path: `/pedidos` → Pedidos, `/facturas` → Facturación, `/envios` → Envíos y `/envios/estado` → Seguimiento. Un `GET /envios/estado/42` coincide con dos reglas, `/envios` y `/envios/estado`, y la lámina pregunta cuál se aplica. [T24, p. 7] *Conocimiento general (especificación de Gateway API, no figura en la lámina):* gana la coincidencia más específica, o sea el prefijo más largo, así que el request va a Seguimiento. *Inferencia:* esa regla es la que absorbe la separación de Seguimiento sin tocar la app.

### Canary y Strangler Fig

El título de la lámina es "Ruteo por header y por peso: canary y Strangler Fig", pero el PDF muestra solo el caso 3 de 3: `/pedidos` va al servicio Pedidos y `/` (el resto) va al Monolito. "Strangler Fig: lo ya extraído va al servicio nuevo; el resto, al monolito." [T24, p. 8] *Inferencia:* los casos 1 y 2, que no se ven, corresponden al ruteo por header y por peso (canary) que anuncia el título.

Lo que sí está en otras fuentes: el Strangler Fig como capa de interceptación delante del monolito está en [Del monolito a microservicios](del-monolito-a-microservicios.md); el canary como estrategia de despliegue, en [DevOps y CI/CD](devops-y-ci-cd.md); y el ruteo por cohorte con un header interno, en [Feature flagging](feature-flagging.md#red-routing-por-cohorte). [T15, p. 23] [T13, p. 42] [T22, p. 15]

### Traducción de protocolo

La app habla HTTP + JSON y pide `GET /pedidos/42`; Pedidos habla gRPC y recibe `ObtenerPedido(42)`. "El gateway traduce en el borde: ninguno de los dos cambia de protocolo." [T24, p. 9] Coincide con el criterio de [Comunicación síncrona](comunicacion-sincrona.md): REST hacia afuera y gRPC interno. [T09, p. 17]

### Agregación

Con `GET /mi-pedido/42` el gateway arma una respuesta a partir de tres llamadas. Otro día Seguimiento está lento: Pedidos respondió 200 en 40 ms, Facturación 200 en 60 ms y Seguimiento no responde, así que la pantalla entera espera (hace 2 s). La lámina pregunta qué responde el gateway y cuándo. [T24, p. 10] *Inferencia:* la respuesta de diseño es un timeout por llamada y una respuesta parcial (pedido y factura, con el envío marcado como no disponible) en lugar de esperar o fallar todo; es la degradación controlada de [Resiliencia y escalabilidad](resiliencia-y-escalabilidad.md). [T21, p. 23]

### BFF: un gateway por tipo de cliente

La tienda suma una web y una API para socios, y cada cliente necesita otra cosa: [T24, p. 11]

| Cliente | Necesita | Gateway | Dueño |
|---|---|---|---|
| App móvil | Pocos campos | BFF móvil | Equipo móvil |
| Web | Todo el detalle | BFF web | Equipo web |
| Socios | Un contrato versionado | API de socios | Equipo de integraciones |

"Cada BFF es del equipo que hace ese cliente." [T24, p. 11] En otras unidades el BFF aparece como caso de uso de GraphQL y como el lugar donde evaluar flags para el browser. [T09, p. 14] [T22, p. 4]

## 2. Quién hace el request

Preguntas del bloque: quién es y si puede hacer esto, de dónde sale el token, qué tiene adentro para poder confiar en él, cómo se valida (solo o preguntando al proveedor), cómo se entera el servicio de quién es y quién decide si puede ver ese pedido. [T24, p. 12]

### Autenticación y autorización en el borde

Tres requests llegan al borde: [T24, p. 13]

| Request | Resultado |
|---|---|
| `GET /pedidos/42` sin token | 401 |
| `DELETE /pedidos/42` con el token de Ana | 403 |
| `GET /pedidos/42` con el token de Ana | 200, llega a Pedidos |

"Autorización: ¿puede hacer esto? Si no puede, 403." [T24, p. 13] *Inferencia:* el 401 es la falla de autenticación (no se sabe quién es) y el 403 la de autorización (se sabe quién es, pero no puede); el `DELETE` se rechaza porque la política de la ruta pide el scope `pedidos:write` y Ana tiene `pedidos:read`. [T24, p. 18]

### OAuth 2.0 y OIDC: de dónde sale el token

Ana inicia sesión en la app con OAuth 2.0 + OIDC contra un proveedor de identidad (IdP), por ejemplo Google, y la app manda `GET /pedidos/42` con el access token. "El gateway solo ve el resultado: un request con un token." [T24, p. 14]

### JWT: un token que se verifica con una firma

El token tiene tres partes separadas por puntos: encabezado, contenido y firma. El contenido lleva los *claims*: [T24, p. 15]

| Claim | Valor del ejemplo | Significado |
|---|---|---|
| `iss` | `idp.tienda.example` | Quién lo emitió |
| `sub` | `ana` | Quién es |
| `scope` | `pedidos:write` (modificado en el camino) | Qué puede pedir |
| `exp` | 18:15 | Hasta cuándo vale |

El proveedor de identidad firma el token y el gateway lo verifica con la clave pública del proveedor (JWKS). "Si alguien cambia un claim, la firma deja de coincidir: 401" (firma inválida). [T24, p. 15]

### Validación local o por introspección

A las 18:05 el proveedor revoca el token de Ana, que vencía a las 18:15. La lámina pregunta hasta cuándo lo acepta cada variante. [T24, p. 16] *Conocimiento general (no figura en la lámina):*

| Variante | Cómo valida | Token revocado a las 18:05 |
|---|---|---|
| Local | Firma y `exp`, con la clave pública; no consulta al proveedor | Lo sigue aceptando hasta las 18:15 |
| Introspección | Le pregunta al proveedor en cada request | Lo rechaza desde las 18:05 |

*Inferencia:* la validación local es rápida y no depende de la disponibilidad del proveedor, pero no se entera de una revocación; la introspección se entera, a cambio de una llamada de red por request. Los tokens de vida corta acotan la ventana de la primera.

### Propagación de identidad: del token al servicio

El gateway valida el token de Ana y le pasa a Pedidos el header `X-User-Id: ana`; Pedidos lee ese header. Beto, con un token válido (el suyo), agrega por su cuenta `X-User-Id: ana`. La lámina pregunta qué debe hacer el gateway con ese header. [T24, p. 17] La respuesta está en otras láminas: el gateway **descarta los headers externos** y escribe el propio a partir del token verificado ("token → X-User-Id"). [T24, p. 27] [T22, p. 15] [T22, p. 27]

Queda un caso sin resolver adentro del sistema: otro servicio puede mandar ese mismo header a Pedidos, y Pedidos le cree. Lo resuelve la malla con una `AuthorizationPolicy`. [T25, p. 2] [T25, p. 10] Ver [Service mesh](service-mesh.md#quién-llama-authorizationpolicy).

### Autorización por scope y por recurso

La política de la ruta asocia método y scope: `GET` → `pedidos:read`, `DELETE` → `pedidos:write`. Ana, con un token válido y scope `pedidos:read`, pide el pedido 43: el scope alcanza y el gateway la deja pasar. Pero el pedido 43 es de Beto, y ese dato está en la base de Pedidos. La lámina pregunta quién rechaza el request y con qué dato. [T24, p. 18] *Inferencia:* lo rechaza Pedidos, comparando el dueño del pedido con el `X-User-Id` que recibió. El gateway autoriza por scope (lo que dice el token); la autorización por recurso necesita datos del negocio y por eso queda en el servicio. [T24, p. 25]

## 3. Cuánto puede pedir

Las decisiones 3 y 4 comparten bloque, con tres preguntas: cuánto puede pedir cada cliente, qué queda anotado de cada request y qué más repetiría cada servicio si no estuviera el gateway. [T24, p. 19]

**Rate limiting en el borde.** El límite es de 100 requests por minuto por cliente: el request 101 de Ana recibe 429 y el request 13 de Beto pasa con 200. El gateway corre en tres réplicas y "cada una lleva su propia cuenta" (réplica 1: Ana 100; réplica 3: Beto 12). La lámina pregunta cuánto puede pedir Ana con tres réplicas y un contador local de 100 por minuto en cada una. [T24, p. 20] *Inferencia:* hasta 300 por minuto si el balanceador reparte parejo, porque cada réplica solo ve su parte. Por eso el contador tiene que ser compartido y vivir fuera del gateway. [T24, p. 24]

El 429 y los algoritmos (token bucket, leaky bucket) están en [Resiliencia y escalabilidad](resiliencia-y-escalabilidad.md#rate-limiting-y-load-shedding), y el estado compartido entre instancias, en [Estado distribuido](estado-distribuido.md). [T21, p. 18] [T12, p. 3]

## 4. Qué queda registrado

Del request de Ana, el borde registra (valores ilustrativos): [T24, p. 21]

| Señal | Ejemplo |
|---|---|
| Access log | `ana · GET /pedidos/42 · 200 · 43 ms` |
| Métricas | `/pedidos · 120 req/s · p99 180 ms` |
| Traza | `4bf9… · gateway → pedidos 40 ms` |

"Traza: si el request llega sin `traceparent`, el gateway la inicia y la propaga." [T24, p. 21] El mapa de la clase llama a esas métricas "métricas RED". [T24, p. 28] Ver [Observabilidad](observabilidad.md).

**Otras políticas del borde.** El título de la lámina nombra TLS, CORS y esquemas; el PDF muestra solo el caso 3 de 3, la validación de esquema: un `POST /pedidos` con `{ "cantidad": "muchas" }` recibe 400 sin llegar al servicio. [T24, p. 22] Los casos de TLS y CORS no se ven en el PDF.

## Riesgos y límites del gateway

Tres preguntas: qué pasa si el gateway se cae, qué pasa si le agregamos una regla más y qué tráfico no ve. [T24, p. 23]

**Disponibilidad: punto único de falla.** Un balanceador reparte entre tres réplicas del gateway; cae la réplica 2 y las otras siguen atendiendo. "Se puede replicar porque no guarda estado propio: el contador vive afuera, compartido." La lámina deja abierta la pregunta siguiente: si el contador compartido deja de responder, ¿el gateway deja pasar todo o rechaza todo? [T24, p. 24] *Inferencia:* es la elección entre *fail-open* (se pierde el límite, se conserva la disponibilidad) y *fail-closed* (se conserva el límite, se cae la entrada). No hay una respuesta única; el criterio es el mismo que la cátedra da para las flags: "seguro no siempre significa `false`", depende del negocio. [T22, p. 29]

**Lógica de negocio: lo que no va en el gateway.** Alguien propone agregar tres reglas: (a) rechazar bodies de más de 1 MB, (b) no enviar a direcciones bloqueadas por fraude y (c) envío gratis desde $ 50.000. El criterio de la lámina: "Si necesita datos del negocio, va en un servicio." [T24, p. 25] *Inferencia:* solo (a) va en el gateway, porque se decide mirando el request; (b) y (c) necesitan datos de fraude y de precios. La unidad de resiliencia ubica el límite de tamaño del body en el gateway o Ingress. [T21, p. 20]

**Tráfico que no ve.** Pedidos necesita a Facturación para responder, y esa llamada es **este-oeste**: no pasa por el gateway. Entre servicios quedan tres preguntas —¿quién llama?, ¿va cifrado?, ¿cuántos reintentos?— y "las responde la malla de servicios (service mesh)". [T24, p. 26] Es el mismo límite que señala la unidad de resiliencia: el gateway es un punto central en el borde, pero no ve el tráfico interno. [T21, p. 24]

## El request completo

Con el gateway, la app conoce una dirección y manda un token. Para `GET /mi-pedido/42`: (1) una regla de ruteo elige los servicios, (2) el token se convierte en `X-User-Id`, (3) se descuenta del límite ("ana: 14 de 100") y (4) quedan el access log, las métricas y el inicio de la traza. [T24, p. 27]

El mapa de la clase conecta cada decisión con otra unidad: [T24, p. 28]

| Decisión | Contenido (láminas) | Se conecta con |
|---|---|---|
| 1. A dónde va | HTTPRoute (7); canary y Strangler (8); agregación y BFF (10–11) | Kubernetes y Migración |
| 2. Quién es | OAuth/OIDC y token (14); JWT y firma (15–16); scope y recurso (18) | Sigue en Data plane |
| 3. Cuánto puede pedir | Límite por cliente (20); réplicas y contador (24) | Resiliencia |
| 4. Qué se registra | Access log, métricas RED y traza (21) | Observabilidad |

Cierre: "El gateway decide cuatro cosas sobre cada request; lo que necesita datos del negocio queda en los servicios." [T24, p. 28]

## Relación con el programa y con el TP

- **Programa.** `T01` ubicaba "API Gateways (Nginx/Traefik)" en la clase de configuración y acceso. [T01, p. 12] `T24` no nombra productos: trabaja con los recursos de Gateway API. La unidad de migración ya daba por visto un deck `api-gateway`. [T15, p. 5]
- **Segunda entrega.** La sección 4 del documento de arquitectura pide declarar el punto de entrada (API gateway, proxy inverso, BFF u otro). [E05]
- **Tercera entrega.** El punto de entrada de `staging` y `prod` valida tokens reales de Google (OAuth 2.0/OIDC) y no se acepta un `userId` por parámetro o header. [E06] *Inferencia:* eso no contradice el `X-User-Id` de `T24`, que es un header interno que escribe el gateway después de validar el token y que descarta si viene de afuera. [T24, p. 17] [T24, p. 27]
- **Cluster del TP.** Con Envoy Gateway, cada microservicio es una `HTTPRoute` más contra el mismo Gateway. [G09]
