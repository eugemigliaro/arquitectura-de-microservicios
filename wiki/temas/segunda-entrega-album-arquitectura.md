# Segunda entrega TP1: arquitectura del Álbum 2026

## Propósito y fuentes

La segunda entrega no incluye implementación. Pide documentar la arquitectura de microservicios de la plataforma a partir del análisis DDD de la [primera entrega](primera-entrega-album-ddd.md). Tiene que quedar claro qué servicios existen, qué datos posee cada uno, cómo se comunican y coordinan, y cómo se empaquetan y despliegan en contenedores. El documento debe explicar **por qué** el sistema tiene esa forma, no solo **qué** forma tiene. [E05]

Rige la versión 1.2 de la consigna (`E05`), que reemplaza a la versión sin número `E04`. Las diferencias se resumen en [Versiones de la consigna](#versiones-de-la-consigna).

Los insumos se aplican en este orden de precedencia: [E05]

1. `consigna-funcional.md` v1.1 ([`E01`](entrega-album-2026.md)): fuente de verdad del producto. Incluye requerimientos funcionales, reglas de negocio, escenarios de aceptación y requerimientos no funcionales de CI/CD y despliegue en contenedores.
2. `docs/analisis-ddd.md` del equipo, con las correcciones de la devolución.

`E05` ya no menciona la [consigna general](consigna-general-album-2026.md) `consigna.md` (`E03`), que `E04` ponía primera. [E04] [E05] Así, la segunda entrega vuelve a la jerarquía de la primera, donde `E01` era la fuente de verdad del dominio. [E02]

**Conflicto abierto.** `E05` atribuye a `E01` requerimientos no funcionales de "despliegue en contenedores", pero la `E01` v1.1 incorporada exige producción en un cluster de Kubernetes en AKS y no acepta la ejecución local con docker compose como entrega. [E05] [E01, p. 6] [E01, p. 7] Ver [Dudas y conflictos](../dudas-y-conflictos.md).

La arquitectura **se deriva** del diseño entregado:

- cada servicio se rastrea hasta uno o más bounded contexts;
- cada agregado tiene un servicio dueño;
- cada evento que cruza fronteras tiene contrato;
- cada invariante tiene un mecanismo que la sostiene.

El modelo de la primera entrega puede corregirse, siempre que se registren el cambio y su motivo en la sección 11.1. [E05]

## Decisiones impuestas y libres

Vienen impuestas: [E05]

- el sistema corre en contenedores: cada servicio se empaqueta con un Dockerfile, y los ambientes se levantan y despliegan con Docker y docker compose;
- un proceso de CI/CD como el visto en clase: build y tests en cada cambio, imágenes de contenedor, despliegue automático a staging, tests de integración en staging como gate y promoción a producción solo si el gate pasa.

Todo lo demás es decisión del equipo y debe justificarse. Eso incluye el recorte de servicios, los lenguajes, los frameworks, la persistencia, el broker, el estilo de comunicación, la coordinación, la herramienta de CI/CD, la estrategia de despliegue y la organización de los ambientes. "Porque corre en contenedores" no justifica ninguna otra decisión: el contenedor es el entorno de ejecución, no la arquitectura. [E05]

**Heredado como fuera de alcance.** La sección 6 de `E01` excluye la alta disponibilidad, el autoscaling y los acuerdos de latencia más allá de que el progreso secundario puede demorar. [E01, p. 5] El equipo no los implementa ni los garantiza, pero documenta cómo escalaría el diseño y qué se lo impediría (secciones 1.3, 7 y 8). No implementarlos no resta puntos; no justificar por qué, sí. [E05]

**Cobertura.** Deben cubrirse todos los requerimientos RF-01 a RF-39 de `E01`: colección, apertura de sobres, intercambios 1:1 y N:M, ofertas compatibles, retos, recompensas, códigos promocionales, campañas, rankings, vista de operador y comportamiento automático. Cada uno necesita un lugar en la arquitectura: qué servicio lo aloja, qué datos posee y cómo se integra. [E05]

Quedan excluidos el código, los Dockerfiles, los archivos de docker compose y los pipelines ejecutables. Se admiten fragmentos ilustrativos, como un esquema de evento, un extracto de Dockerfile o una etapa del pipeline, que no se evalúan por funcionar. [E05]

## Entregable y forma

```text
docs/arquitectura.md
docs/adr/0001-<titulo-corto>.md
docs/adr/0002-<titulo-corto>.md
```

Requisitos de forma: [E05]

- castellano y **los mismos términos del lenguaje ubicuo** de la primera entrega, sin renombrar conceptos ya definidos;
- secciones en el orden y con la numeración de la consigna;
- diagramas versionados como texto (Mermaid, PlantUML, Structurizr DSL o ASCII); una imagen exportada puede acompañarlos, pero no reemplazarlos;
- entre 15 y 25 páginas equivalentes sin contar los ADRs, aunque se evalúa precisión y no volumen;
- en el encabezado: integrantes, versión de la consigna funcional y commit o tag de `docs/analisis-ddd.md`.

## Estructura obligatoria

| § | Contenido exigido | Tablas o artefactos de formato |
|---:|---|---|
| 1 | **Drivers de arquitectura.** Objetivos de negocio. Restricciones impuestas (contenedores con Docker, CI/CD con gate de staging, cada servicio dueño de sus datos, ejecución local a partir del repositorio) y autoimpuestas. Atributos de calidad como escenarios medibles; "el sistema debe ser escalable" no es un atributo de calidad. NFRs: throughput, latencia p99 por operación crítica y disponibilidad objetivo, como supuestos de diseño justificados. Una disponibilidad "best effort, sin HA" es válida si se fundamenta en la sección 6 de `E01`; si la disponibilidad o el autoscaling no se diseñan, se indica en la celda con esa referencia. | 1.1 Restricciones; 1.2 Escenarios de calidad (ID, atributo, estímulo, entorno, respuesta, medida, prioridad); 1.3 NFRs por operación |
| 2 | **Vista de contexto (C4 nivel 1).** Actores de `E01` y sistemas externos, aunque estén fuera de alcance o simulados, como el proveedor de identidad, indicando cómo se los reemplaza. | Diagrama de contexto |
| 3 | **Descomposición (C4 nivel 2).** Catálogo de servicios con responsabilidad, contextos, agregados, datos y operaciones. Cada desvío de la correspondencia contexto–servicio se argumenta. Se indica qué servicios son stateless y dónde vive el estado. Cada dato tiene un único servicio que lo escribe. Se identifican las proyecciones: quién las materializa, desde qué eventos y con qué atraso. El motor de persistencia se elige por necesidad. | 3.1 Catálogo; 3.2 Trazabilidad contexto → servicio; 3.3 Propiedad de los datos; 3.4 Diagrama de contenedores |
| 4 | **Comunicación e integración.** Estilo síncrono o asíncrono de cada interacción, con la sección 4 de `E01` como criterio, y punto de entrada (API gateway, proxy inverso, BFF u otro). APIs: operaciones de los escenarios, códigos de error de negocio, transporte de la clave de idempotencia y del identificador de correlación, y versionado. Eventos: productor, consumidores, canal, clave de partición, esquema, envelope común y evolución de esquemas. Además, cómo no divergen el cambio de estado y la publicación, por ejemplo con outbox. | 4.1 Matriz de interacciones; 4.2 APIs; 4.3 Contratos de eventos |
| 5 | **Flujos y coordinación.** Diagramas de secuencia de los escenarios 1–2, 3, 4, 5–6, 7 y 12. Estilo de coordinación del intercambio justificado, con máquina de estados completa: transiciones, disparadores, timeouts y compensaciones. Recuperación de operaciones incompletas sin duplicar efectos. Para lo que es solo diseño: el flujo principal de retos y el escenario 15. | Diagramas de secuencia y máquina de estados |
| 6 | **Consistencia, idempotencia y fallos.** Fuerte o eventual para cada dato de la sección 4 de `E01`, con servicio y mecanismo. Cómo se sostiene y verifica el invariante de oro. Idempotencia en la API y en los consumidores, con dónde se guardan las claves, su alcance y su duración. Orden (escenario 9). Resiliencia: timeouts, backoff, circuit breakers, bulkheads y backpressure, dónde se aplican y qué pasa al activarse. Concurrencia de los escenarios 11 y 15 sin estado en memoria. | 6.1 Modelo de consistencia; 6.2 Fallos por escenario |
| 7 | **Escalabilidad.** Puntos calientes, perfil de tráfico, protección ante picos, camino de lectura del resumen y los rankings, particionamiento del backbone y de las bases, y retención y purgado. Qué servicios podrían escalar con réplicas y qué impide a los demás; es un análisis de diseño, sin exigir escalado automático ni orquestador funcionando. | — |
| 8 | **Despliegue en contenedores.** Ambientes local, staging y producción: cómo se ejecutan los contenedores en cada uno y qué aislamiento hay. Empaquetado: contenido del Dockerfile de cada servicio (imagen base, dependencias, comando de arranque, usuario) y cómo se construyen, versionan y etiquetan las imágenes. Orquestación con Compose: servicios, redes, volúmenes, conexiones, políticas de reinicio y `depends_on`. Componentes con estado como contenedores con volúmenes o servicios externos, con sus implicancias de persistencia, respaldo y costo. Configuración y secretos por ambiente. Healthcheck de cada contenedor y por qué: uno que siempre responde OK es un error de diseño. Límites de CPU y memoria y política de réplicas, aunque no se implemente. Ejecución local con `docker compose up` y sus diferencias con producción. | 8.1 Ambientes; 8.2 Recursos por servicio, con réplicas justificadas si son una; 8.3 Diagrama de despliegue |
| 9 | **CI/CD.** Etapas con disparador, acción y artefacto. Gates, en particular cómo la integración en staging bloquea producción y en qué estado queda. Versionado de imágenes y garantía de que lo probado en staging es lo que llega a producción. Monorepo o multirepo, y cómo evitar reconstruir servicios que no cambiaron. Estrategia de despliegue de los contenedores (recrear, rolling con varias réplicas, blue-green, canary u otra) y de rollback. Migraciones y contratos con expand/contract, y su orden. Tests por etapa. Acceso del pipeline al host de despliegue y al registro sin exponer secretos. | 9.1 Etapas; 9.2 Diagrama del pipeline |
| 10 | **ADRs.** Índice con el estado de cada decisión. | Un archivo por ADR en `docs/adr/` |
| 11 | **Riesgos, cambios y preguntas abiertas.** Cambios respecto de la primera entrega, matriz de trazabilidad de cada RF, RNF y escenario de `E01` hacia la sección que lo resuelve, riesgos y preguntas abiertas con la parte de la arquitectura que depende de cada una. Esta sección se evalúa: reconocer un riesgo o un cambio vale más que ocultarlo. | 11.1 Cambios; 11.2 Matriz de trazabilidad; 11.3 Riesgos |

Fuente de la tabla: [E05].

## ADRs

Una decisión es significativa si cuesta caro cambiarla después o si afecta a más de un servicio. Como mínimo debe haber ADRs para: [E05]

1. el recorte de servicios a partir de los bounded contexts;
2. el estilo de comunicación entre servicios;
3. el broker y la clave de partición de los eventos;
4. el estilo de coordinación del intercambio;
5. la estrategia de idempotencia;
6. la publicación confiable de eventos;
7. el motor de persistencia, al menos del servicio que sostiene las cantidades de la colección;
8. la organización de ambientes y el empaquetado en contenedores;
9. la estrategia de despliegue y promoción del pipeline.

Cada ADR considera **al menos dos alternativas reales** y explica por qué se descartaron: "un ADR sin alternativas es una afirmación, no una decisión". El formato es título, estado (Propuesta, Aceptada o Reemplazada por otro ADR), fecha, contexto con los drivers en juego, alternativas, decisión y consecuencias. [E05]

## Reglas de calidad

- Toda pieza se rastrea hasta un contexto, agregado o evento de la primera entrega.
- Toda tecnología tiene un ADR o una justificación explícita; "es la que conocemos" no alcanza.
- Ningún dato tiene dos servicios que lo escriban.
- Los diagramas son consistentes entre sí y con el texto.
- Los servicios, eventos, operaciones y campos usan el lenguaje ubicuo. Se rechazan nombres genéricos como `core-service`, `manager`, `handler`, `data-service` o `processor`.
- Los contenedores y el CI/CD se documentan con el mismo rigor que el resto.
- Si una plantilla no aplica, se explica por qué en vez de rellenarla. [E05]

## Rúbrica

| Criterio | Qué se mira | Peso |
|---|---|---:|
| Drivers y trazabilidad | Atributos medibles, NFRs justificados, trazabilidad al modelo, cambios registrados | 10 % |
| Descomposición y datos | Recorte justificado, único dueño por dato, proyecciones, persistencia por necesidad | 20 % |
| Comunicación y coordinación | Síncrono o asíncrono según consistencia, contratos completos, saga con máquina de estados, compensación y recuperación | 20 % |
| Consistencia, idempotencia y escala | Mecanismo por garantía, invariante de oro, duplicados, orden, concurrencia, hot spots y camino de lectura | 20 % |
| Despliegue y CI/CD | Ambientes, imágenes y docker compose con sentido, healthchecks útiles, gate de staging, artefacto inmutable, rollback y migraciones | 20 % |
| Decisiones de arquitectura | ADRs con alternativas reales y consecuencias | 10 % |

Fuente: [E05].

Restan puntos:

- una base escribible compartida;
- servicios sin bounded context;
- tecnologías sin justificar;
- ADRs sin alternativas;
- diagramas inconsistentes;
- healthchecks triviales;
- decisiones de escalado o disponibilidad sin justificar;
- un pipeline que no garantiza que producción ejecute lo probado en staging;
- funcionalidades de solo diseño sin lugar en la arquitectura.

No implementar alta disponibilidad ni autoscaling no resta puntos; no justificar por qué, sí. No suma agregar servicios, tecnologías o infraestructura sin un driver documentado. [E05]

La entrega es grupal y se versiona en el repositorio del equipo; se espera que todo el equipo haya discutido las decisiones. La fecha límite y el canal se informan en el Campus. [E05]

## Unidades de la materia que alimentan cada sección

**Inferencia de estudio:** la consigna no asigna unidades a cada sección. Esta tabla relaciona lo pedido con el material ya incorporado.

| § | Unidades y conceptos |
|---:|---|
| 3 | [DDD estratégico](ddd-estrategico.md): el bounded context es una frontera de modelo y no necesariamente de despliegue [T03, p. 31]. [Diseño de servicios](diseno-de-servicios.md): acoplamiento, cohesión y anatomía. |
| 4 | [Comunicación síncrona](comunicacion-sincrona.md): REST para APIs públicas y gRPC interno [T09, p. 17]. [Kafka](comunicacion-asincrona-y-kafka.md): CloudEvents, key de partición, outbox y evolución de esquema [T14, p. 11] [T14, p. 47] [T14, p. 55]. `correlationId` y `causationId` en el envelope [T14, p. 54]. |
| 5 | [Sistemas distribuidos](sistemas-distribuidos.md): orquestación y coreografía [T11, p. 20] [T11, p. 37]. [Kafka](comunicacion-asincrona-y-kafka.md): Sagas con timeouts explícitos [T14, p. 27]. |
| 6 | [Estado distribuido](estado-distribuido.md): locks con fencing, claves de idempotencia en la misma frontera atómica que el efecto y semánticas de entrega [T12, p. 11] [T12, p. 20] [T12, p. 26]. Inbox del lado consumidor [T14, p. 50]. Bulkhead y sobrecarga [T09, p. 26]. |
| 7 | Consumer lag como medida del atraso de las proyecciones [T14, p. 43]. Valkey para estado derivado o cacheado, como rankings y vistas [T12, p. 5], con cache-aside [T12, p. 15] y colas [T12, p. 14]. |
| 8 | [Imágenes de contenedores](imagenes-de-contenedores.md): multi-stage [T06, p. 27], ejecución sin root [T06, p. 17] y errores comunes [T06, p. 37]. [Docker Compose](docker-compose.md): healthcheck, `depends_on` con `service_healthy`, readiness frente a liveness y dónde definir la prueba [T07, p. 15] [T07, p. 17] [T07, p. 18] [T07, p. 19]; secretos [T07, p. 11]. [Docker CLI, redes y volúmenes](docker-cli-redes-y-volumenes.md): redes, publicación de puertos, DNS y persistencia. |
| 9 | [DevOps y CI/CD](devops-y-ci-cd.md): pipeline y estrategias de despliegue [T13, p. 27] [T13, p. 40]. Promoción por digest inmutable [T06, p. 36]. [Testing](testing.md): niveles y contract testing [T10, p. 10]. |

## Versiones de la consigna

`E04` no tiene número de versión. *Inferencia:* corresponde a la versión 1.0 del changelog de `E05`, porque conserva la referencia cruzada a la "sección 10 del punto 5" que la versión 1.1 corrige. [E04] [E05]

| Tema | `E04` (sin número, reemplazada) | `E05` (v1.2, vigente) |
|---|---|---|
| Insumos | `consigna.md` primero, la consigna funcional después y el análisis DDD | La consigna funcional primero y el análisis DDD. El changelog no registra este cambio |
| Cobertura | Lo que `consigna.md` pide implementar y lo que pide solo diseñar | Todos los RF-01 a RF-39 de la consigna funcional, cada uno con un lugar en la arquitectura |
| Decisión impuesta de ejecución | Cluster de Kubernetes, con producción en AKS | Contenedores con Dockerfile, Docker y docker compose |
| NFRs | Desde la sección de un millón de usuarios de `consigna.md`, con disponibilidad mínima | Throughput, latencia p99 y disponibilidad objetivo como supuestos justificados; "best effort, sin HA" admitido. Alta disponibilidad, autoscaling y acuerdos de latencia, fuera de alcance |
| Sección 8 | Despliegue en Kubernetes: namespaces o clusters, Deployments, StatefulSets, Jobs, Services, Ingress, sondas y HPA | Despliegue en contenedores: Dockerfiles, Compose, volúmenes, healthchecks, límites y réplicas justificadas |
| Observabilidad | Sección 10 propia, criterio de corrección y ADR obligatorio | Eliminada. El identificador de correlación sigue pedido en APIs y envelope (sección 4) |
| Numeración final | ADRs en §11; riesgos en §12, con matriz de preguntas de `consigna.md` | ADRs en §10; riesgos en §11, con matriz de trazabilidad de RF, RNF y escenarios de la consigna funcional |
| ADRs mínimos | Diez, con ambientes en Kubernetes y observabilidad | Nueve, con ambientes y empaquetado en contenedores |
| Rúbrica | 10 % a observabilidad y decisiones | 10 % a decisiones de arquitectura; restan healthchecks triviales y decisiones de escala o disponibilidad sin justificar |

Fuentes: [E04] [E05].

## Puntos a resolver antes de redactar

- **Contenedores o AKS.** `E05` impone contenedores con Docker y docker compose y los atribuye a la consigna funcional, pero `E01` v1.1 exige AKS como única forma de entrega. [E05] [E01, p. 6] [E01, p. 7] Puede existir una versión de la consigna funcional no incorporada. Hay que confirmarlo en el Campus o con la cátedra.
- **Cuándo se aparta una copia.** Con `E01` como fuente de verdad rige su semántica: publicar no aparta, y el apartado empieza con la aceptación. [E01, p. 1] [E01, p. 2] [E05] La diferencia con `E03` deja de afectar a esta entrega, pero sigue abierta para la entrega final.
- **Autenticación.** La §2 sigue pidiendo explicar cómo se reemplazan los sistemas externos simulados, pero `E01` exige Google real (RF-A01 a RF-A05, RNF-06). [E05] [E01, p. 2] [E01, p. 6]
- **Numeración de escenarios.** `E05` nombra los escenarios con los títulos de `E03`, como "intercambio rechazado (escenario 4)" y "último uso concurrente de un código promocional (escenario 15)", pero toma a `E01` como fuente. Los números 1 a 15 coinciden en tema, salvo el 4: en `E01` son dos ofertas respaldadas por la misma copia, y una aceptación concurrente no la obtiene. `E01` agrega el 16, sin sesión válida. [E05] [E01, p. 6] [E03, p. 10] [E03, p. 11]
- **"Solo diseño" sin lista.** `E05` sigue hablando de funcionalidades de solo diseño en la §5 y en los criterios, pero ya no dice cuáles son, porque esa lista venía de `consigna.md`. [E05] [E03, p. 8]
- **"Una sola" decisión impuesta.** `E05` conserva esa frase y enumera dos: contenedores y CI/CD. La referencia a la sección 11.1 para los cambios del modelo ya está corregida. [E05] Ambos casos están en [Dudas y conflictos](../dudas-y-conflictos.md).
