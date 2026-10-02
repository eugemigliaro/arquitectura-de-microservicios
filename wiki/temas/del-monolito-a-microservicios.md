# Del monolito a microservicios

## Qué ordena esta unidad

La clase no introduce herramientas nuevas. Toma lo visto en DDD, comunicación, Kafka, API gateway, estado distribuido, sistemas distribuidos, testing, DevOps y data plane, y lo ordena en el tiempo de una migración. [T15, p. 5] Sus objetivos son cinco:

1. juzgar si una migración tiene sentido y reconocer anti-patterns como el monolito distribuido o los "microservicios porque Netflix";
2. encontrar servicios candidatos con DDD y con datos del sistema en producción;
3. aplicar patrones incrementales en lugar de un rewrite big-bang;
4. planificar cómo partir una base de datos compartida, "la parte más difícil";
5. armar un roadmap por fases, con riesgos y métricas de éxito. [T15, p. 4]

La columna de la clase son los capítulos 3 y 4 de *Monolith to Microservices*, de Sam Newman. [T15, p. 57]

## ¿Por qué (y si) migrar?

| Empresa | Qué hizo | Por qué |
|---|---|---|
| Amazon (~2001–2006) | Pasó del monolito Obidos a servicios con APIs, con equipos chicos dueños de cada uno | El monolito frenaba a cientos de desarrolladores que necesitaban desplegar por separado |
| Segment (2018) | Volvió de ~140 microservicios a un monolito para sus destinations | Tres ingenieros pasaban la mayor parte del tiempo manteniendo servicios, no agregando valor |
| Prime Video (2023) | Pasó su monitoreo de calidad de video de funciones serverless distribuidas a un solo proceso | Costo de orquestación y transferencia de datos entre pasos; bajó ~90 % el costo |

Fuente de la tabla: [T15, p. 7]. La moraleja es que migrar no es "progreso": es una decisión con costos, y a veces la correcta es la inversa. [T15, p. 7]

**Razones que alcanzan:** despliegue independiente sin coordinar con otros equipos, autonomía de equipos dueños de punta a punta, escalado selectivo (el catálogo recibe 100× más tráfico que la facturación), aislamiento de fallas (si cae el recomendador, la gente sigue comprando) y un stack distinto donde hace falta, como un motor de búsqueda o un modelo de ML. La pregunta de control es qué problema del negocio resuelve la migración y cómo se va a medir que se resolvió. [T15, p. 8]

**Razones que no alcanzan:**

- el hype: Netflix tiene miles de ingenieros y un problema de escala que casi nadie tiene;
- "el código es feo": un rewrite no arregla la falta de tests ni de diseño, y distribuirlo lo vuelve más difícil de arreglar;
- la performance: una llamada en memoria cuesta nanosegundos y una por red, milisegundos;
- "para escalar": un monolito escala horizontalmente detrás de un load balancer, así que hay que mostrar que una parte escala distinto que el resto;
- el CV-driven development: querer usar Kubernetes y Kafka no es un requerimiento del negocio. [T15, p. 9]

Estos criterios amplían los de [Fundamentos de microservicios](fundamentos-de-microservicios.md): valor para el negocio, frecuencia de releases, escalado desigual y autonomía de equipos. [T02, p. 29] [T02, p. 30]

### El monolito modular

Es un solo deployable con módulos de fronteras explícitas. Cada módulo es un bounded context con su API pública, está prohibido llamar internals de otro módulo (se valida en CI) e idealmente cada módulo tiene su propio schema en la base. Da gran parte de los beneficios de diseño sin pagar la red, el deploy distribuido ni la consistencia eventual. Es un paso intermedio excelente y muchas veces el destino final; Shopify es el ejemplo más conocido. [T15, p. 10] Coincide con la distinción de [DDD estratégico](ddd-estrategico.md) entre frontera de modelo y frontera de despliegue. [T03, p. 31]

### Prerrequisitos: "you must be this tall"

Antes de extraer el primer servicio tienen que existir CI/CD que haga trivial y repetible desplegar un servicio, observabilidad básica (logs centralizados, métricas y un `correlationId` que cruce servicios), tests automatizados —sobre todo de caracterización del monolito y contract tests— y provisioning rápido, para levantar un servicio en horas y no en semanas. Sin eso, cada servicio nuevo multiplica el trabajo manual y los incidentes. [T15, p. 11] Ver [DevOps y CI/CD](devops-y-ci-cd.md) y [Testing](testing.md).

## Encontrar los seams

En un sistema existente, DDD funciona como mapa. El bounded context es el mejor candidato a frontera de servicio; una misma palabra con significados distintos en dos partes del código marca una frontera; y el context mapping anticipa qué integración hará falta al separar. La diferencia con un proyecto nuevo es que el código tiene fronteras implícitas, casi siempre violadas, que hay que descubrir en el código y no solo en la pizarra. [T15, p. 14]

| Seam | Qué se mira | Señales |
|---|---|---|
| 1. Dependencias en el código | Grafo de dependencias entre paquetes o módulos (jdeps, dependency-cruiser, Structure101, Sonargraph o el IDE) | Alta cohesión adentro y bajo acoplamiento afuera es un buen candidato; los ciclos se rompen antes de extraer; los "god modules" (`utils`, `common`, `core`) suelen esconder lógica de dominio |
| 2. Change coupling en git | Qué archivos aparecen juntos en muchos commits (code-maat o CodeScene) | Si dos archivos cambian juntos en el 70 % de los commits, separarlos en servicios distintos significa deploys coordinados para siempre |
| 3. Datos de runtime | Carga, requisitos de disponibilidad y latencia, tablas "calientes" compartidas (APM, slow query log) | El catálogo en Black Friday frente a la facturación |
| 4. Organización | Dueño de cada área, frecuencia de cambio, motivo del cambio (regulación, marketing, operaciones) | Lo que cambia seguido es lo que más se beneficia de un deploy independiente |

Fuente de la tabla: [T15, p. 15] [T15, p. 16] [T15, p. 17]. El código dice qué depende de qué; el historial dice qué cambia junto, aunque los archivos no se importen entre sí. La lámina muestra cómo obtener con `git log --name-only` los pares por commit y el top de archivos más cambiados (hotspots). [T15, p. 16]

### El primer servicio

Se busca bajo acoplamiento, valor visible y riesgo bajo. El cuadrante de la lámina cruza facilidad de extracción con beneficio. [T15, p. 18]

| | Poco acoplado | Muy acoplado |
|---|---|---|
| **Alto valor** | Empezar por acá | Más adelante, con experiencia |
| **Bajo valor** | Buen "piloto" de práctica | Dejarlo en el monolito |

Fuente de la tabla: [T15, p. 18]. Buenos primeros candidatos son notificaciones, catálogo (mucha lectura), búsqueda y auth, si ya es un módulo aparte; malos primeros candidatos son pagos y órdenes, centrales, muy acoplados y caros si fallan. El primer servicio no es para ganar mucho, sino para aprender el camino —pipeline, observabilidad, rollback— con poco en juego. [T15, p. 18]

### Granularidad

Un servicio demasiado chico es tan problemático como uno demasiado grande. Los síntomas de nano-servicios son necesitar seis llamadas por red para un caso de uso, dos servicios que siempre se despliegan juntos y un servicio por entidad (`UserService`, `AddressService`, `PhoneService`). La heurística es un servicio por bounded context, o por subdominio dentro de uno grande. Empezar más grande y partir después es mucho más barato que juntar: juntar implica migrar datos y reescribir contratos, y partir un módulo bien encapsulado no. [T15, p. 19] Es coherente con el rechazo a cortar por tabla o entidad de [DDD estratégico](ddd-estrategico.md). [T03, p. 33]

### Tres puertas antes de extraer

Encontrar el acoplamiento no es lo mismo que tener un seam que se pueda cortar. Antes de sacar la *tajada* —la funcionalidad a extraer— del proceso:

1. **Tests de caracterización** sobre el borde público de la tajada; el corte se permite cuando siguen verdes.
2. **Reorganizar adentro del monolito:** el código pasa detrás de una interfaz local y el resto del sistema llama a esa interfaz, sin cambiar el deployable. Es el primer paso de Branch by Abstraction.
3. **Inventario de efectos laterales** que el grafo de paquetes no muestra: crons, listeners, mailers, archivos y procesos batch enganchados al módulo. Si un job nocturno o un listener se queda en el monolito, el servicio nuevo es media capacidad, porque el monolito sigue escribiendo "sus" datos por atrás. [T15, p. 20]

## Patrones de migración incremental

| Big-bang rewrite | Incremental |
|---|---|
| El sistema viejo sigue cambiando mientras se reescribe: target móvil | Se extrae de a una funcionalidad, en producción y con posibilidad de volver atrás |
| Efecto segundo sistema: el rewrite quiere arreglar todo y crece sin límite | Se entrega valor en cada paso |
| Cero valor hasta el día del corte, que casi siempre se atrasa | Cada paso es chico, reversible y se puede frenar |
| Se pierde el conocimiento escondido en el código viejo (bugs que son features) | Monolito y servicios conviven meses o años |

Fuente de la tabla: [T15, p. 22]. Regla: si el paso no se puede revertir, es demasiado grande. [T15, p. 22]

### Strangler Fig

El nombre viene de una planta que crece alrededor de un árbol hasta reemplazarlo. Se pone una capa de interceptación —proxy o API Gateway— delante del monolito, se implementa una funcionalidad en un servicio nuevo, se redirige esa ruta y se repite; el monolito se achica y el cliente no se entera, porque la URL es la misma. [T15, p. 23]

La interceptación puede ser tan simple como un `location` de nginx que manda `/api/notifications/` al servicio nuevo y todo lo demás al monolito. El rollback es volver a apuntar la ruta al monolito: un cambio de configuración, no de código. Con un gateway (Traefik, Kong, Envoy) se puede mandar primero el 5 % del tráfico, y lo mismo vale para la UI mediante UI composition o micro-frontends. La fachada es una tabla de rutas temporal que reenvía, registra y permite volver; si empieza a acumular reglas de negocio, es un segundo monolito. [T15, p. 24]

### Branch by Abstraction

Sirve para funcionalidad que se usa desde adentro del monolito, donde no hay ruta que interceptar:

1. crear una abstracción delante de la funcionalidad actual;
2. hacer que todo el monolito use la abstracción;
3. escribir una segunda implementación que llama al servicio nuevo;
4. cambiar de implementación con un toggle;
5. borrar la implementación vieja y, si no aporta, la abstracción. [T15, p. 25]

El ejemplo define un protocolo `Notifier` con una implementación en proceso (el código viejo, movido) y otra que hace un POST al servicio nuevo, elegidas por un feature flag. Todo vive en `main`, sin branches largos: el código nuevo está desplegado pero apagado. [T15, p. 25]

### Parallel Run y Dark Launching

Cuando equivocarse es caro —precios, impuestos, cálculo de riesgo— no basta con cambiar y rezar. En **Parallel Run** cada request va a la implementación vieja y a la nueva, se devuelve el resultado de la vieja y se registran las diferencias. En **dark launching** el servicio nuevo recibe tráfico real espejado, pero su respuesta no llega al usuario. Se cambia de implementación recién cuando las diferencias son cero o están explicadas. [T15, p. 26]

El shadowing de una lectura (catálogo, una cotización) es seguro. El de una escritura (mails, pagos, reservas de stock) manda, cobra o reserva dos veces: hay que apagar el efecto en la sombra con stubs o usar una clave de idempotencia. [T15, p. 26] El ciclo de vida de esa clave está en [Estado distribuido](estado-distribuido.md). [T12, p. 19]

### Decorating Collaborator y CDC

Ambos agregan comportamiento nuevo sin modificar el monolito: lo nuevo se engancha desde afuera. Un **Decorating Collaborator** es un proxy que deja pasar la llamada y, con la respuesta, dispara algo en un servicio nuevo; por ejemplo, tras cada `POST /orders` exitoso el servicio de loyalty suma puntos. Con **Change Data Capture** se escucha el log de cambios de la base del monolito (Debezium → Kafka) y cada `INSERT` en `orders` se vuelve un evento que consume el servicio nuevo. [T15, p. 27] CDC ya apareció como forma de vaciar un outbox en [Comunicación asincrónica y Kafka](comunicacion-asincrona-y-kafka.md). [T14, p. 49]

### Feature toggles y canary releases

La idea es separar el deploy del release. Con un feature toggle el código nuevo se despliega apagado y se prende por configuración; con un canary el servicio nuevo recibe primero el 1 %, después el 10 %, el 50 %… y si empeoran errores o latencia se vuelve al 0 %, con rollback en segundos y sin deploy. [T15, p. 28] El canary es la estrategia de despliegue de [DevOps y CI/CD](devops-y-ci-cd.md), aplicada aquí a la migración. [T13, p. 42]

El trabajo en curso no se migra a mitad de camino: un pedido que empezó en el monolito termina en el monolito, solo el tráfico nuevo va al servicio y se drena. [T15, p. 28]

### Anti-Corruption Layer

Conviviendo con el monolito, el servicio nuevo hereda su modelo si no se cuida: flags de 2009, tablas de 80 columnas. El ACL traduce en el borde el modelo del monolito al lenguaje del nuevo contexto; el ejemplo convierte una fila de `CUST_MSTR` (`FLG_VIP`, `STATUS_CD`) en un `Customer` con `Tier` y `active`. Cuando el monolito desaparece se borra el ACL y el modelo interno queda intacto. Si el servicio llama sincrónicamente al monolito en cada request, el acoplamiento pasó de la base a HTTP: se está formando un monolito distribuido. [T15, p. 29] Es el mismo patrón de context mapping de [DDD estratégico](ddd-estrategico.md). [T03, p. 26]

### ¿Qué patrón uso?

| Situación | Patrón |
|---|---|
| La funcionalidad se accede por una ruta HTTP desde afuera | Strangler Fig |
| La funcionalidad se usa desde adentro del monolito | Branch by Abstraction |
| Equivocarse es caro y hay que comparar resultados | Parallel Run, dark launching |
| Agregar algo sin tocar el monolito | Decorating Collaborator, CDC |
| Pasar tráfico de a poco y poder volver | Feature toggles, canary |
| El servicio nuevo tiene que hablar con el modelo viejo | Anti-Corruption Layer |

Fuente de la tabla: [T15, p. 30]. No son excluyentes: una migración real usa varios a la vez. [T15, p. 30]

## Ejemplo conductor: extraer Notificaciones

El deck recorre una misma extracción en siete láminas:

1. **Seam:** `notifications` solo depende de `common`, así que es buen candidato; `orders` e `inventory` tienen un ciclo que hay que romper primero. [T15, p. 15]
2. **Elección:** poco acoplado, valor visible y riesgo bajo. [T15, p. 18]
3. **Puertas:** tests sobre `send()` y los templates, todo detrás de una interfaz `Notifier`, y un inventario que encuentra el cron de reintentos y el mail del resumen semanal. [T15, p. 20]
4. **Strangler Fig:** `/notifications/*` va al servicio nuevo, con base propia; todo lo demás, al monolito. [T15, p. 23]
5. **En la práctica:** un `location` de nginx con rollback por configuración. [T15, p. 24]
6. **Branch by Abstraction:** cubre lo que el checkout y el cron llaman desde adentro, porque la ruta HTTP ya la movió el Strangler. [T15, p. 25]
7. **Canary:** día 1, 1 % de los mails, solo usuarios internos; día 3, 10 %, mirando errores, p99 y rebotes; día 7, 50 %; día 10, 100 % y se borra `InProcessNotifier`; en cualquier momento, 0 % si algo se rompe. [T15, p. 28]

## Separar los datos

### La base compartida

Si el servicio nuevo lee y escribe las mismas tablas que el monolito, nadie puede cambiar un schema sin coordinar con todos, no se sabe quién es dueño de un dato, las reglas de negocio se duplican o se saltean al escribir directo en la tabla, y el deploy independiente no sirve si el schema no es independiente. La base pasa a ser la API real del sistema, aunque nadie la diseñó como API. El resultado es un monolito distribuido: todos los costos de la red y ninguno de los beneficios. [T15, p. 33] Ver la misma advertencia en [Fundamentos](fundamentos-de-microservicios.md). [T02, p. 32]

### ¿Código o datos primero?

| Código primero | Datos primero |
|---|---|
| Se extrae el servicio, que sigue usando la base del monolito por un tiempo | Se separa el schema dentro del monolito; el código sigue ahí |
| Da valor rápido y enseña el pipeline | Descubre temprano los joins y transacciones que se rompen, y todavía es reversible y barato |
| Riesgo de que "por un tiempo" sea para siempre; los problemas de datos aparecen al final | Más tiempo sin beneficio visible |

Fuente de la tabla: [T15, p. 34]. Si hay dudas sobre la frontera (joins, transacciones), datos primero; si hace falta aprender el camino, código primero, con fecha para los datos. [T15, p. 34]

### Patrones para partir la base

- **Database View:** se expone una vista de solo lectura con lo que el consumidor necesita, y el monolito puede cambiar sus tablas mientras la mantenga. [T15, p. 35]
- **Database Wrapping Service:** un servicio delante de las tablas por cuya API pasan todos; aunque el schema sea horrible, al menos tiene dueño e interfaz. Ambos son puentes temporales para dejar de acceder a las tablas directamente. [T15, p. 35]
- **Split Table:** una tabla usada por dos contextos (por ejemplo `items` con precio y stock) se parte en `catalog.items` e `inventory.stock`. Se identifica qué columnas pertenecen a quién, se crea la tabla nueva y se copian los datos, se mueven primero las lecturas y después las escrituras, y se borran las columnas viejas. Si dos contextos escriben la misma columna no hay split posible: primero se decide el dueño. Es un caso de expand/contract, nunca un `RENAME` in-place, que rompe al lado que todavía no desplegó. [T15, p. 36]
- **Move Foreign-Key Relationship to Code:** una FK no puede cruzar bases de datos. La relación se guarda como un ID sin constraint y la integridad se valida en código, consultando la API del otro servicio o con eventos como `CustomerDeleted`. Los joins pasan a la aplicación o a una copia local de lo necesario (nombre, email). Se pierde la integridad referencial garantizada por la base y hay que decidir qué pasa cuando se borra el cliente: prohibirlo, anonimizar o dejar huérfana la orden. [T15, p. 37]

### Sincronizar durante la transición

Mientras conviven, monolito y servicio necesitan los datos, y la tentación es escribir en las dos bases. Ninguna transacción cubre ambas: si algo falla en el medio, divergen y nadie se entera. Es el mismo dual write entre base y broker visto con Kafka. [T15, p. 38] [T14, p. 46] Las soluciones son CDC con Debezium y Kafka, donde la base que es fuente de verdad publica sus cambios; Transactional Outbox, que escribe el evento en la misma transacción que el cambio; y una sola fuente de verdad en cada momento, con un dueño explícito y el otro lado como réplica de solo lectura. [T15, p. 38]

El camino típico con CDC va de la base del monolito, por el WAL, a Debezium, a un topic `orders.cdc` y, a través de un ACL o traductor, al servicio nuevo con su base propia. [T15, p. 39]

| Fase | Escribe | Rollback |
|---|---|---|
| 1 | El monolito es dueño; el servicio nuevo tiene una réplica vía CDC y solo lee | Dar vuelta la ruta |
| 2 | El servicio nuevo es dueño; el monolito recibe los cambios de vuelta (CDC u Outbox) mientras alguien lo use | Difícil: requiere reconciliar |
| 3 | El servicio nuevo, sin sincronización; nadie lee del monolito y se borran las tablas viejas | Ya no hay vuelta barata |

Fuente de la tabla: [T15, p. 39] [T15, p. 40]. Un feature flag no revierte datos: si la sincronización divergió, volver al monolito es leer datos viejos, por eso la reconciliación se define antes de entrar en la fase 2. No hay que meter 2PC para "salvar" el corte. [T15, p. 40]

**Terminado** quiere decir que la ruta y el módulo viejos están borrados, hay un solo escritor por tabla, el usuario de base del monolito ya no tiene grant de escritura sobre esa tabla y el rollback se ensayó, no solo se escribió. Una convención de "dejamos de escribir" no es un corte; el `REVOKE` sí. [T15, p. 40]

### Adiós ACID

En el monolito, crear la orden, reservar stock y cobrar era una transacción; separado, son tres bases. La respuesta es una Saga, una secuencia de transacciones locales con compensaciones, coordinada por coreografía o por orquestación, y el negocio tiene que aceptar estados intermedios como "orden pendiente de pago". [T15, p. 41] El detalle está en [Sistemas distribuidos](sistemas-distribuidos.md) y en [Comunicación asincrónica y Kafka](comunicacion-asincrona-y-kafka.md). Antes de separar hay que preguntarle al negocio:

- ¿Cuánto tiempo puede estar inconsistente este dato?
- ¿Qué pasa si se reservó el stock pero falló el pago?
- ¿Esta invariante tiene que ser inmediata? Si sí, quizás esos datos van en el mismo servicio. [T15, p. 41]

`T11` incluye "transacciones entre bases de datos" y "sistemas legacy" entre los casos de 2PC; la diferencia de énfasis está registrada en [Dudas y conflictos](../dudas-y-conflictos.md). [T11, p. 35]

### Reporting y joins entre servicios

El reporte que hacía un `JOIN` de seis tablas ahora cruza cuatro servicios. Se puede componer por API, que es simple pero lento y frágil con volumen; mantener read models con CQRS, donde un servicio de reporting materializa vistas a partir de eventos; o llevar todo por CDC a un data warehouse o lake, para que el reporting deje de ser responsabilidad de los servicios operativos. Lo que no se hace es abrir una conexión SQL "de solo lectura" a la base de otro servicio: es la base compartida otra vez, por la puerta de atrás. [T15, p. 42] Ver CQRS light en [Comunicación asincrónica y Kafka](comunicacion-asincrona-y-kafka.md). [T14, p. 19]

## Organización y operación

**Conway inverso:** la ley de Conway dice que la arquitectura copia la organización; el *Inverse Conway Maneuver* la usa a favor y diseña los equipos con la forma que se quiere para la arquitectura. Team Topologies distingue *stream-aligned teams*, dueños de un flujo de negocio de punta a punta; *platform team*, que provee CI/CD, observabilidad e infraestructura como producto interno; y *enabling teams*, que ayudan a adoptar prácticas nuevas. Si nueve servicios los mantiene un equipo que coordina todos los deploys, la arquitectura dice "microservicios" pero la organización dice "monolito", y gana la organización. [T15, p. 44] Ver la ley de Conway en [Diseño de servicios](diseno-de-servicios.md). [T02, p. 24]

**Lo que se complica en operación:** un bug que antes era un stack trace ahora cruza cuatro servicios y dos brokers. Hacen falta tracing distribuido y correlation IDs, que se retoman en la clase de observabilidad; contract testing, porque monolito y servicios cambian a ritmos distintos y el monolito es el primer consumidor del contrato; service mesh para mTLS, retries, timeouts y mirroring sin tocar el código; manejo de fallas parciales con timeouts, circuit breakers y degradación elegante; y on-call, porque cada equipo es dueño de su servicio también a las 3 de la mañana. [T15, p. 45]

### ¿Cómo sé que la migración funciona?

Las métricas DORA se miden antes de empezar y en cada fase: deploy frequency, lead time for changes, change failure rate y time to restore (MTTR). [T15, p. 46] Las métricas propias de la migración son:

- deploys que requieren coordinar a más de un equipo, que deberían tender a cero;
- porcentaje del tráfico que ya no pasa por el monolito;
- tablas del monolito que todavía lee otro servicio;
- por tajada, el delta de error rate frente al monolito, el lag del CDC y si el rollback se ensayó. [T15, p. 46]

Si después de seis meses el lead time es el mismo y los deploys siguen coordinados, la migración no logra su objetivo aunque haya diez servicios nuevos. [T15, p. 46]

### Anti-patterns de migración

- **Monolito distribuido:** servicios que se despliegan juntos y comparten base; lo peor de los dos mundos.
- **Base compartida "por ahora":** el "por ahora" más largo de la industria.
- **Librería compartida con lógica de dominio:** un `common-domain.jar` que todos importan obliga a redesplegar a todos al cambiar una regla.
- **Llamadas sincrónicas encadenadas (chatty):** una request dispara doce llamadas en serie y la disponibilidad total es el producto de todas.
- **Big-bang disfrazado:** una "migración incremental" que no pone nada en producción durante ocho meses.
- **Extraer sin sacar:** el servicio nuevo existe, pero el código viejo sigue vivo en el monolito "por las dudas", y ahora hay dos que mantener. [T15, p. 47]

## Taller: MegaShop

El taller se hace en grupos de tres o cuatro personas, con una sola hoja por grupo: 3 minutos para leer el caso, 10 para discutir las preguntas en orden y 7 para la puesta en común, en la que cada grupo defiende un bloque en 90 segundos. El producto es un plan de migración en una hoja: qué se extrae primero, cómo, qué pasa con los datos y cómo se vuelve atrás. Vale más una respuesta defendida que cinco a medias. [T15, p. 49]

**Situación.** E-commerce de 7 años, un solo deployable Java contra PostgreSQL, 40 desarrolladores en 5 equipos y un release cada 3 semanas, un sábado a la mañana, con un deploy manual de 4 horas. En el último Black Friday el catálogo recibió 30× el tráfico normal, el proceso se quedó sin memoria y cayó el checkout con todo lo demás: 3 horas sin ventas. Además, el equipo de notificaciones espera 3 semanas para cambiar un template de mail y un cambio en catálogo rompió pagos dos veces en el año. El diagrama muestra Catálogo, Carrito, Órdenes, Pagos, Inventario, Usuarios y Notificaciones con un cron nocturno en un solo proceso, una base con `orders`, `order_items`, `inventory`, `users`, `products` y `payments`, y el equipo de BI con SQL directo. [T15, p. 50]

**Pedido y restricciones.** El directorio pide "pasar todo a microservicios en 6 meses, antes del próximo Black Friday". Al mirar código y base aparece que el checkout es una transacción que crea la orden, descuenta stock y registra el pago; `orders` tiene FK a `users` y a `inventory`; un cron nocturno recalcula el stock y manda los mails de "volvió a entrar"; BI arma reportes con SQL directo sobre producción; y hay tests unitarios, casi ningún end-to-end, y logs en cada servidor. Lo que no se ve en el diagrama es el cron, el SQL de BI, la transacción de checkout y el deploy manual. [T15, p. 51]

| Bloque | Preguntas |
|---|---|
| A. ¿Hay que migrar? ¿Así? | Qué problema de negocio resolvería y si es el que pidió el directorio; si "todo en 6 meses" es incremental o un big-bang con otro nombre; qué prerrequisitos faltan |
| B. ¿Dónde cortar? | Qué bounded contexts hay y si coinciden con los módulos; cuál es el primer servicio y por qué; si el Black Friday se resuelve extrayendo un servicio o alcanza con otra cosa |
| C. ¿Cómo extraer? | Qué patrón y dónde va la capa de interceptación; qué pasa con el cron nocturno y a qué servicio le pertenece cada parte |
| D. ¿Qué pasa con los datos? | Quién es dueño de `orders`, `inventory` y `users` y cómo se sincronizan; qué estados intermedios ve el usuario sin la transacción de checkout y qué se compensa si falla el pago; qué hacer con el SQL de BI |
| E. Roadmap | Tres fases, cada una con su riesgo principal, cómo se vuelve atrás y una métrica que diga si funcionó |

Fuente de la tabla: [T15, p. 52] [T15, p. 53] [T15, p. 54]. El deck no incluye una resolución.

*Pistas de estudio (inferencia, no resolución oficial):* cada bloque se apoya en una sección de esta página. A usa las razones que alcanzan y no alcanzan, y los prerrequisitos; B, los seams, el cuadrante del primer servicio y la advertencia de que un monolito también escala horizontalmente [T15, p. 9]; C, la tabla "¿Qué patrón uso?" y la tercera puerta, el inventario de efectos laterales; D, Move FK to Code, las fases de sincronización, las Sagas y las opciones de reporting; y E, las fases con su rollback y las métricas DORA y de migración.

## Ideas clave

- Migrar es una decisión, no un destino: primero el problema de negocio, después la arquitectura; el monolito modular es una respuesta válida.
- Las fronteras se descubren: DDD para el mapa, y dependencias, historial de git y datos de producción para contrastarlo.
- Incremental y reversible: si el paso no se puede revertir, es demasiado grande.
- Los datos son la parte difícil: un dueño por dato, nada de base compartida, CDC u Outbox para sincronizar y Sagas donde se pierde ACID.
- La organización manda: Conway inverso, equipos dueños de punta a punta y métricas DORA.
- Terminado es ruta vieja borrada, un solo escritor, grant revocado y rollback ensayado. [T15, p. 56]

Para seguir leyendo, la cátedra recomienda *Monolith to Microservices* de Newman; los artículos StranglerFigApplication, MonolithFirst y MicroservicePrerequisites de Martin Fowler; microservices.io de Chris Richardson; *Team Topologies* de Skelton y Pais; y *Your Code as a Crime Scene* de Adam Tornhill. [T15, p. 57]

## Material referenciado no incorporado

`T15` da por vistos los decks `api-gateway` y `data-plane`, y remite a la clase 11 de observabilidad. [T15, p. 5] [T15, p. 45] Ninguno de esos materiales está en este repositorio; el programa ubica los API gateways en el módulo de Kubernetes y el service mesh y la observabilidad en el de operaciones. [T01, p. 12] [T01, p. 13] El caso queda registrado en [Dudas y conflictos](../dudas-y-conflictos.md).

## Relación con el Álbum 2026

*Inferencia:* el Álbum es un sistema nuevo, así que los patrones de extracción (Strangler Fig, Branch by Abstraction, Parallel Run) no se aplican directamente. Sí se aplican los criterios de datos y de granularidad. La [segunda entrega](segunda-entrega-album-arquitectura.md) pide que cada agregado tenga un servicio dueño y que el estado y su publicación no diverjan. [E04] Eso corresponde a "un dueño por dato", a Move FK to Code entre servicios y a la regla contra el dual write. [T15, p. 37] [T15, p. 38] El rechazo a los nano-servicios ayuda a justificar el recorte de servicios en su ADR. [T15, p. 19] Las vistas del Álbum que pueden demorar admiten read models en lugar de joins entre servicios. [T15, p. 42] [E01, p. 5]
