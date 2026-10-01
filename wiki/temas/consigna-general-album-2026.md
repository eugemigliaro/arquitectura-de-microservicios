# Consigna general del trabajo práctico: Álbum 2026

## Estado y relación con las otras consignas

`E03` es la consigna general del trabajo práctico. Relata el producto completo y define qué debe implementarse, qué basta diseñar y qué queda fuera de alcance. [E03, p. 1] [E03, p. 8] El número de versión 1.2 surge solo del nombre del archivo, porque el documento no declara versión ni changelog.

La [segunda entrega](segunda-entrega-album-arquitectura.md) nombra un archivo `consigna.md` como fuente de verdad del producto, el alcance, los escenarios mínimos, CI/CD y escala. [E04] **Inferencia:** ese archivo es `E03`, porque la segunda entrega cita secciones que aparecen con ese título en `E03`, como "Consideraciones adicionales: diseño para un millón de usuarios", "Preguntas que deberá responder el diseño" y "Preguntas adicionales". [E03, p. 9] [E03, p. 12] [E04]

| ID | Documento | Rol |
|---|---|---|
| `E03` | Consigna general | Producto, alcance, preguntas de diseño, escenarios mínimos, CI/CD, escala y rúbrica final |
| `E01` | [Requerimientos funcionales v1.1](entrega-album-2026.md) | RF, reglas de negocio y 16 escenarios de aceptación |
| `E02` | [Primera entrega](primera-entrega-album-ddd.md) | Análisis de dominio DDD |
| `E04` | [Segunda entrega](segunda-entrega-album-arquitectura.md) | Documentación de arquitectura y ADRs |

`E03` y `E01` coinciden en gran parte, pero difieren en puntos que afectan el diseño: cuándo se aparta una copia, la autenticación, el alcance de implementación y la rúbrica. La [comparación](#comparación-con-la-consigna-funcional-v11) está al final de esta página y cada conflicto quedó registrado en [Dudas y conflictos](../dudas-y-conflictos.md).

## Producto y lenguaje

La plataforma acompaña al álbum oficial del Mundial 2026. Permite abrir sobres, comparar colecciones, encontrar repetidas e intercambiarlas, y agrega retos, recompensas, promociones y eventos especiales. [E03, p. 1]

El glosario mínimo define Figurita, Copia, Sobre, Colección, Repetida, Disponible, Apartada, Oferta y Operador. Los equipos pueden extenderlo, pero no usar la misma palabra para conceptos distintos. [E03, p. 1] Dos definiciones difieren de las de `E01`:

- **Disponible:** además de "poseída y no apartada", aclara que solo las copias disponibles y repetidas pueden ofrecerse. [E03, p. 1]
- **Apartada:** "copia reservada para una oferta o un intercambio en curso", que no puede usarse en otra oferta. [E03, p. 1] `E01` restringe el apartado al intento de intercambio. [E01, p. 1]

Las fórmulas de cantidades son idénticas a las de `E01`. "Dos personas reciben la misma copia" significa que la misma unidad apartada o transferida se acredita a más de un usuario; varios usuarios sí pueden poseer la misma figurita del catálogo. [E03, p. 2]

El álbum tiene secciones: selecciones, estadios, ciudades, escudos, jugadores y especiales. Las figuritas especiales solo se obtienen mediante promociones o retos y nunca vienen en un sobre ordinario. [E03, p. 2]

## Apertura de sobres

Un sobre puede venir de un regalo inicial, un código promocional, una recompensa de reto, una campaña o un evento de la comunidad. [E03, p. 2]

Para abrirlo se verifica que pertenezca al usuario y que siga sin abrir. Después se determina su contenido si todavía no estaba determinado, se acreditan las copias y el sobre queda registrado como abierto. [E03, p. 2] El contenido queda congelado a más tardar en la primera apertura exitosa, así que un reintento muestra las mismas figuritas. [E03, p. 2] El usuario debe poder ver el resultado aunque cierre la aplicación, pierda la conexión o repita la solicitud. [E03, p. 3]

El flujo conceptual de apertura es un flujo de negocio: no determina cuántos servicios existen ni cómo se comunican. [E03, p. 3]

## Intercambio

El producto admite ofertas N:M, pero la solución ejecutable usa ofertas 1:1. [E03, p. 3] La última copia de una figurita nunca puede ofrecerse. [E03, p. 3]

En `E03` la copia ofrecida se aparta **al publicar**:

- Mientras la oferta siga activa, esa copia no puede usarse en otro intercambio. [E03, p. 3]
- En el ejemplo, la plataforma verifica la copia de Carlos y "la aparta para esa oferta". Al aceptar, Lucía pasa por una nueva verificación y se aparta su copia. [E03, p. 4]
- El flujo conceptual muestra "Verifica y aparta copias" como paso de la publicación y "Libera copias" como paso de la cancelación y el vencimiento. [E03, p. 4]
- Cancelar o vencer una oferta libera sus copias apartadas. [E03, p. 5] [E03, p. 6]

Esta semántica contradice la aclaración central de `E01` v1.1, donde publicar no aparta y una misma copia puede respaldar varias ofertas. [E01, p. 1] [E01, p. 2] El conflicto está registrado en [Dudas y conflictos](../dudas-y-conflictos.md).

Otras reglas del intercambio:

- Aceptar revalida que ambos participantes conserven las copias y que la oferta siga activa. Si dos personas aceptan a la vez, solo una aceptación se confirma. [E03, p. 3]
- Si la copia de quien acepta ya no está disponible, la copia del publicador vuelve a quedar disponible. La oferta puede seguir activa o cerrarse, y el diseño debe justificar esa decisión. [E03, p. 5]
- Si las dos copias están apartadas y la operación se interrumpe, o se completa el intercambio o se liberan ambas reservas. [E03, p. 5]
- Una demora o un error de comunicación no prueba que la operación haya fallado. Tiene que poder determinarse su estado para continuar, reintentar o revertir. [E03, p. 5]
- El creador puede cancelar mientras no haya una aceptación en curso ni confirmada. El vencimiento ocurre aunque el usuario no esté conectado. Ante una carrera entre cancelación o vencimiento y aceptación, el resultado es uno solo. [E03, p. 5] [E03, p. 6]
- Un operador inspecciona ofertas activas, reservas y operaciones incompletas para encontrar copias bloqueadas. [E03, p. 5]

### Invariante de oro

Después de cualquier secuencia de aperturas, intercambios, reintentos, vencimientos, cancelaciones y eventos duplicados o fuera de orden:

- ninguna cantidad es negativa;
- ninguna copia se pierde ni se acredita dos veces;
- el contenido de un sobre abierto es estable;
- un intercambio completado no se vuelve a ejecutar;
- no quedan copias apartadas asociadas a una oferta cancelada, vencida o rechazada de forma terminal. [E03, p. 5]

## Retos, códigos y consistencia percibida

Los retos avanzan por acciones hechas en otras partes de la plataforma y otorgan sobres, insignias o acceso a promociones. Su avance puede demorar, pero una misma acción no se cuenta dos veces aunque llegue duplicada. [E03, p. 6]

Un código promocional puede otorgar sobres, una figurita especial o una recompensa. Puede ser de uso único en toda la plataforma o de un uso por usuario, y admite vigencia, límite global y restricciones por país o campaña. Si dos solicitudes compiten por el último uso, solo una se acepta. [E03, p. 6] [E03, p. 7]

La división entre lo que debe ser correcto de inmediato y lo que puede demorar coincide con `E01`. Tienen **consistencia fuerte** el estado y contenido del sobre, las cantidades de la colección, el resultado de operar una oferta y el resultado del intercambio. Tienen **consistencia eventual** el porcentaje del álbum, los retos, los rankings, las notificaciones y la actividad reciente. [E03, p. 7] [E01, p. 5]

## Alcance

La consigna para los estudiantes enumera doce tareas, que van del lenguaje ubicuo y los límites de contexto hasta la implementación y demostración. Se debe diseñar e implementar al menos un proceso distribuido con coordinación, recuperación y compensaciones: el candidato esperado es el intercambio. La apertura también se implementa con idempotencia, aunque su coordinación puede ser más simple. [E03, p. 8]

| Implementar y demostrar | Solo diseñar | Fuera de alcance |
|---|---|---|
| Colección: poseídas, faltantes, repetidas y disponibles | Retos, recompensas e insignias | Proveedor de identidad o login real: alcanza un `userId` en header, un token opaco o un parámetro |
| Apertura idempotente con contenido congelado | Códigos promocionales, incluido el último uso concurrente | App móvil o UI pulida |
| Intercambio 1:1: apartar, confirmar, rechazar, cancelar, vencer y compensar | Rankings | Pagos, compra de sobres, anti-fraude y fuerza bruta de códigos |
| Progreso del álbum con consistencia eventual | Matching de ofertas compatibles | Alta disponibilidad, autoscaling y acuerdos de latencia |
| Escenarios mínimos | Ofertas N:M | Destacar ofertas compatibles en tiempo real |
| Observabilidad de extremo a extremo de apertura e intercambio | Vista de operador; campañas, eventos y especiales como origen de sobres | |

Fuente de la tabla: [E03, p. 8].

Los criterios pedagógicos aclaran que la nota no depende de la cantidad de servicios, la latencia ni la UI, sino de las fronteras, el modelo, el tratamiento de fallos, la idempotencia y la justificación. No se acepta una base de datos escribible compartida; las proyecciones de solo lectura pueden ser eventualmente consistentes. El lenguaje, el broker y el motor de persistencia son libres, y la demostración debe poder ejecutarse en local desde el repositorio. [E03, p. 9]

## Preguntas que deberá responder el diseño

Las catorce preguntas cubren:

- qué significa que una copia esté disponible y cuándo deja de estarlo;
- quién decide si un intercambio puede realizarse;
- qué ocurre si solo una parte puede confirmarse y cómo se recupera una operación incompleta;
- cómo se evita abrir dos veces un sobre o procesar dos veces un intercambio;
- qué pasa con mensajes duplicados o desordenados;
- qué datos requieren consistencia fuerte y cuáles pueden ser eventuales;
- cómo se reconstruye una operación ante un incidente;
- cómo detecta un operador ofertas o reservas bloqueadas;
- cómo se prueba el invariante de oro. [E03, p. 9]

La sección 12.2 de la segunda entrega exige indicar qué parte del documento responde cada una. [E04]

## Escenarios mínimos

| # | Escenario en `E03` | Correspondencia con `E01` |
|---:|---|---|
| 1 | Apertura normal | Igual |
| 2 | Reintento del cliente: mismo contenido, sin duplicar | Igual |
| 3 | Intercambio exitoso 1:1 | Igual |
| 4 | Intercambio rechazado: la copia de quien acepta ya no está disponible y se libera la del publicador | **Distinto:** en `E01` el 4 son dos ofertas respaldadas por la misma copia |
| 5 | Interrupción después de apartar un lado | Equivalente |
| 6 | Ambas partes apartadas y luego interrupción: resultado binario | Igual |
| 7 | Resultado desconocido para el solicitante | Igual |
| 8 | Mensajes duplicados | Igual |
| 9 | Mensajes fuera de orden: criterio explícito (idempotencia, versión, descarte de obsoletos) | Igual |
| 10 | Actualización diferida | Igual |
| 11 | Dos aceptaciones concurrentes | Igual |
| 12 | Vencimiento o cancelación contra aceptación | Igual |
| 13 | Sobre ajeno o ya abierto | Igual |
| 14 | Oferta inválida: copia inexistente, no repetida o ya apartada | Equivalente; `E01` también menciona copia ajena o no disponible |
| 15 | Último uso de un código, **solo diseño** | En `E01` debe implementarse |
| — | No existe | `E01` 16: solicitud sin sesión válida |

Fuentes: [E03, p. 9] [E03, p. 10] [E03, p. 11] [E01, p. 6]. La segunda entrega usa la numeración de `E03`. [E04]

## Entregables y demostración

La lista de entregables abarca análisis de dominio, glosario, user stories, context map, flujos con compensación, arquitectura, responsabilidades por servicio, APIs y contratos de eventos, coordinación distribuida, compensación y recuperación, idempotencia, modelo de consistencia, pruebas funcionales y de fallos, evidencia de observabilidad, demo ejecutable y pipeline de CI/CD hacia AKS. [E03, p. 11]

La demostración debe:

- levantarse en local, por ejemplo con `docker compose up`, con al menos dos usuarios y datos suficientes para los escenarios 1–14;
- incluir un README que explique cómo reproducir cada escenario implementable;
- mostrar un mismo `correlationId` o `traceId` recorriendo los servicios en una apertura, un intercambio exitoso y un intercambio con fallo y compensación. [E03, p. 11]

Para esa evidencia alcanza con logs estructurados o con un backend de tracing: no se exige service mesh ni cluster. [E03, p. 11]

## CI/CD: condición de entrega

Si el repositorio no cumple estas condiciones, la entrega no se evalúa. [E03, p. 11] El pipeline debe:

1. construir y correr las pruebas unitarias y funcionales de cada microservicio en cada cambio;
2. empaquetar los servicios y desplegarlos automáticamente a staging;
3. correr en staging los tests de integración de los escenarios implementables;
4. si fallan, no promover a producción y dejar producción intacta;
5. desplegar a producción, un cluster de Kubernetes en AKS, solo si ese gate pasa. [E03, p. 11] [E03, p. 12]

El equipo documenta etapas, gates y estrategia de despliegue. También deja evidencia de una ejecución completa, incluido un caso en que el gate bloqueó, o habría bloqueado, un despliegue. [E03, p. 12] La unidad de [DevOps y CI/CD](devops-y-ci-cd.md) desarrolla estos conceptos.

## Diseño para un millón de usuarios

Esta sección suma a la corrección una segunda dimensión: seguir siendo correcto y responsivo con tráfico mucho mayor y no uniforme. [E03, p. 12] Sus once preguntas adicionales piden:

- identificar puntos de contención (un código viral, una figurita muy buscada, una oferta popular) y evitar que se vuelvan un cuello de botella de escritura;
- describir el perfil de tráfico y el comportamiento ante picos correlacionados con el torneo;
- proteger la plataforma de ráfagas con limitación de tasa, colas de absorción o load shedding;
- escalar la lectura agregada por separado de la escritura, decidiendo qué se cachea, qué se precalcula y qué se consulta en el momento;
- calcular rankings globales sin cuello de botella ni atraso inaceptable;
- elegir la clave de partición del backbone de eventos y explicar sus garantías de orden y paralelismo;
- definir la retención y el purgado de claves de idempotencia y registros de eventos;
- aplicar timeouts, circuit breakers, bulkheads y backpressure contra fallas en cascada;
- fijar SLOs para las operaciones críticas y explicar cómo se validan;
- evitar que cualquier instancia dependa de estado en memoria, como locks locales o sesiones pegajosas;
- desplegar cambios de esquema, contratos y funcionalidades sin cortar el servicio. [E03, p. 12]

Entregables adicionales: NFRs, hot spots, estrategia del camino de lectura, particionamiento, retención y una prueba de carga, aunque sea a escala reducida, sobre la apertura o el intercambio. [E03, p. 13]

**Inferencia de estudio:** estas preguntas se apoyan en unidades ya incorporadas.

| Tema de la pregunta | Unidad |
|---|---|
| Bulkhead, sobrecarga y destino del trabajo | [Comunicación síncrona](comunicacion-sincrona.md) |
| Estado fuera de la instancia, locks con fencing, cache e idempotencia | [Estado distribuido](estado-distribuido.md) |
| Particiones, orden por key, consumer lag, outbox, inbox y evolución de esquema | [Kafka](comunicacion-asincrona-y-kafka.md) |
| Rolling, blue/green y canary | [DevOps y CI/CD](devops-y-ci-cd.md) |

Hay una tensión interna en `E03`: excluye del alcance la alta disponibilidad, el autoscaling y los acuerdos de latencia, pero después pide SLOs, latencia p99 y disponibilidad mínima como parte del diseño. [E03, p. 8] [E03, p. 12] [E03, p. 13] **Lectura razonable:** la exclusión alcanza a lo que hay que implementar y garantizar en la demo, mientras que las consideraciones de escala se exigen en el documento. El conflicto quedó registrado.

## Rúbrica

Cumplir CI/CD y desplegar en AKS es una condición previa. No forma parte de la tabla de pesos. [E03, p. 13]

| Criterio | Qué se mira | Peso |
|---|---|---:|
| Fronteras y arquitectura | Context map, dueño de cada dato, sin base escribible compartida | 20 % |
| Modelo de dominio | Glosario, agregados e invariantes; figurita/copia/disponible/apartada | 15 % |
| Coordinación distribuida | Reserva, confirmación, rechazo, compensación y recuperación | 20 % |
| Idempotencia y consistencia | Reintentos, duplicados, desorden; fuerte vs. eventual | 20 % |
| Observabilidad y demo | Escenarios reproducibles, traces de éxito y fallo, invariante de oro | 15 % |
| Justificación | Respuestas a las preguntas de diseño y decisiones coherentes | 10 % |

Fuente: [E03, p. 13]. No suma puntos agregar rankings, retos o códigos implementados, ni más microservicios, si eso debilita el tratamiento de fallos, la idempotencia o las fronteras. [E03, p. 13]

## Comparación con la consigna funcional v1.1

| Aspecto | `E03` (general) | `E01` (funcional v1.1) |
|---|---|---|
| Apartado de la copia ofrecida | Al publicar la oferta; se libera al cancelar o vencer [E03, p. 4] [E03, p. 5] | Solo cuando una aceptación inicia el intento [E01, p. 2] |
| Una copia en varias ofertas | No: la copia apartada no puede usarse en otra oferta [E03, p. 1] | Sí, sin crear reservas [E01, p. 1] |
| Autenticación | Fuera de alcance; alcanza un `userId` [E03, p. 8] | Google OAuth 2.0/OIDC real; no se acepta un `userId` [E01, p. 2] [E01, p. 6] |
| Qué se implementa | El núcleo; el resto es solo diseño [E03, p. 8] | Todo; no se aceptan partes solo diseñadas [E01, p. 7] |
| Escenarios | 15; el 15 es solo diseño [E03, p. 11] | 16, todos demostrables [E01, p. 6] |
| Dónde se demuestra | En local, más el pipeline hacia AKS [E03, p. 9] [E03, p. 11] | Solo vale el despliegue en AKS; el local no lo reemplaza [E01, p. 6] [E01, p. 7] |
| Rúbrica | Fronteras, modelo, coordinación, idempotencia, observabilidad y justificación [E03, p. 13] | Autenticación, RF core y ampliados, invariantes, escenarios, consistencia y reproducibilidad [E01, p. 7] |

Coinciden en las fórmulas de cantidades, la división entre consistencia fuerte y eventual, la carrera entre aceptación y cierre, el resultado real tras una respuesta perdida y el pipeline con gate de staging hacia AKS. [E03, p. 2] [E03, p. 7] [E03, p. 12] [E01, p. 2] [E01, p. 5] [E01, p. 6]

Para la segunda entrega, `E04` fija la precedencia: `consigna.md` primero y la consigna funcional después. [E04] Ninguna fuente fija qué documento prevalece en la entrega final.
