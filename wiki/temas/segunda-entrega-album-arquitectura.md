# Segunda entrega TP1: arquitectura del Álbum 2026

## Propósito y fuentes

La segunda entrega no incluye implementación. Pide documentar la arquitectura de microservicios de la plataforma a partir del análisis DDD de la [primera entrega](primera-entrega-album-ddd.md). Tiene que quedar claro qué servicios existen, qué datos posee cada uno, cómo se comunican y coordinan, y cómo se despliegan y observan. El documento debe explicar **por qué** el sistema tiene esa forma, no solo **qué** forma tiene. [E04]

Los insumos se aplican en este orden de precedencia: [E04]

1. `consigna.md`: producto, alcance, escenarios mínimos, CI/CD y escala. Por su contenido es la [consigna general](consigna-general-album-2026.md) `E03`.
2. `consigna-funcional.md` v1.1 ([`E01`](entrega-album-2026.md)): requerimientos funcionales, reglas de negocio y escenarios de aceptación.
3. `docs/analisis-ddd.md` del equipo, con las correcciones de la devolución.

Esto cambia la jerarquía de la primera entrega, donde `E01` era la única fuente de verdad del dominio. [E02] [E04] Como `E03` y `E01` difieren en puntos centrales, conviene revisar la [comparación](consigna-general-album-2026.md#comparación-con-la-consigna-funcional-v11) antes de redactar.

La arquitectura **se deriva** del diseño entregado:

- cada servicio se rastrea hasta uno o más bounded contexts;
- cada agregado tiene un servicio dueño;
- cada evento que cruza fronteras tiene contrato;
- cada invariante tiene un mecanismo que la sostiene.

El modelo de la primera entrega puede corregirse, siempre que se registren el cambio y su motivo. [E04]

## Decisiones impuestas y libres

Vienen impuestas:

- producción en un cluster de Kubernetes en AKS;
- un proceso de CI/CD como el visto en clase: build y tests en cada cambio, imágenes de contenedor, despliegue automático a staging, tests de integración en staging como gate y promoción a producción solo si el gate pasa. [E04]

Todo lo demás es decisión del equipo y debe justificarse. Eso incluye el recorte de servicios, los lenguajes, la persistencia, el broker, el estilo de comunicación, la coordinación, la observabilidad, la herramienta de CI/CD, la estrategia de despliegue y la organización del cluster. "Porque corre en Kubernetes" no justifica ninguna otra decisión: el cluster es el entorno de ejecución, no la arquitectura. [E04]

El documento cubre lo que `E03` pide implementar y también lo que pide solo diseñar. Lo de solo diseño admite menos profundidad, pero debe tener lugar en la arquitectura: qué servicio lo aloja, qué datos posee y cómo se integra. [E04] [E03, p. 8] Quedan excluidos el código, los manifiestos y los pipelines ejecutables. Se admiten fragmentos ilustrativos, que no se evalúan por funcionar. [E04]

## Entregable y forma

```text
docs/arquitectura.md
docs/adr/0001-<titulo-corto>.md
docs/adr/0002-<titulo-corto>.md
```

Requisitos de forma: [E04]

- castellano y **los mismos términos del lenguaje ubicuo** de la primera entrega, sin renombrar conceptos ya definidos;
- secciones en el orden y con la numeración de la consigna;
- diagramas versionados como texto (Mermaid, PlantUML, Structurizr DSL o ASCII); una imagen exportada puede acompañarlos, pero no reemplazarlos;
- entre 15 y 25 páginas equivalentes sin contar los ADRs, aunque se evalúa precisión y no volumen;
- en el encabezado: integrantes, versión de la consigna funcional y commit o tag de `docs/analisis-ddd.md`.

## Estructura obligatoria

| § | Contenido exigido | Tablas o artefactos de formato |
|---:|---|---|
| 1 | **Drivers de arquitectura.** Objetivos de negocio, restricciones impuestas y autoimpuestas, atributos de calidad como escenarios medibles y NFRs derivados de la sección de un millón de usuarios, con estimaciones justificadas. "El sistema debe ser escalable" no es un atributo de calidad. | 1.1 Restricciones; 1.2 Escenarios de calidad (ID, atributo, estímulo, entorno, respuesta, medida, prioridad); 1.3 NFRs por operación |
| 2 | **Vista de contexto (C4 nivel 1).** Actores y sistemas externos, aunque estén fuera de alcance o simulados, como el proveedor de identidad, indicando cómo se los reemplaza. | Diagrama de contexto |
| 3 | **Descomposición (C4 nivel 2).** Catálogo de servicios con responsabilidad, contextos, agregados, datos y operaciones. Cada desvío de la correspondencia contexto–servicio se argumenta. Se indica qué servicios son stateless y dónde vive el estado. Cada dato tiene un único servicio que lo escribe. Se identifican las proyecciones: quién las materializa, desde qué eventos y con qué atraso. El motor de persistencia se elige por necesidad. | 3.1 Catálogo; 3.2 Trazabilidad contexto → servicio; 3.3 Propiedad de los datos; 3.4 Diagrama de contenedores |
| 4 | **Comunicación e integración.** Estilo síncrono o asíncrono de cada interacción, con la sección 4 de `E01` como criterio, y punto de entrada (gateway, ingress o BFF). APIs: operaciones de los escenarios, códigos de error de negocio, transporte de la clave de idempotencia y del `correlationId`, y versionado. Eventos: productor, consumidores, canal, clave de partición, esquema, envelope común y evolución de esquemas. Además, cómo no divergen el cambio de estado y la publicación, por ejemplo con outbox. | 4.1 Matriz de interacciones; 4.2 APIs; 4.3 Contratos de eventos |
| 5 | **Flujos y coordinación.** Diagramas de secuencia de los escenarios 1–2, 3, 4, 5–6, 7 y 12. Estilo de coordinación del intercambio justificado, con máquina de estados completa: transiciones, disparadores, timeouts y compensaciones. Recuperación de operaciones incompletas sin duplicar efectos. Para lo que es solo diseño: el flujo principal de retos y el escenario 15. | Diagramas de secuencia y máquina de estados |
| 6 | **Consistencia, idempotencia y fallos.** Fuerte o eventual por dato, con servicio y mecanismo. Cómo se sostiene y verifica el invariante de oro. Idempotencia en la API y en los consumidores, con dónde se guardan las claves, su alcance y su duración. Orden (escenario 9). Resiliencia: timeouts, backoff, circuit breakers, bulkheads y backpressure, dónde se aplican y qué pasa al activarse. Concurrencia de los escenarios 11 y 15 sin estado en memoria. | 6.1 Modelo de consistencia; 6.2 Fallos por escenario |
| 7 | **Escalabilidad.** Puntos calientes, perfil de tráfico, protección ante picos, camino de lectura del resumen y los rankings, particionamiento del backbone y de las bases, retención y purgado, y qué impide escalar horizontalmente a cada servicio. | — |
| 8 | **Despliegue en Kubernetes.** Ambientes local, staging y producción (cluster compartido por namespace o clusters separados, y por qué). Mapeo a Deployment, StatefulSet, Job/CronJob, Service e Ingress. Estado dentro del cluster o gestionado. Configuración y secretos por ambiente. Sondas liveness, readiness y startup con sentido: una readiness que siempre responde OK es un error de diseño. Requests, limits y HPA aunque no se implementen. Ejecución local y sus diferencias con el cluster. | 8.1 Ambientes; 8.2 Recursos por servicio; 8.3 Diagrama de despliegue |
| 9 | **CI/CD.** Etapas con disparador, acción y artefacto. Gates, en particular cómo la integración en staging bloquea producción y en qué estado queda. Versionado de imágenes y garantía de que lo probado en staging es lo que llega a producción. Monorepo o multirepo, y cómo evitar reconstruir servicios que no cambiaron. Estrategia de despliegue y de rollback. Migraciones y contratos con expand/contract, y su orden. Tests por etapa. Credenciales sin exponer secretos. | 9.1 Etapas; 9.2 Diagrama del pipeline |
| 10 | **Observabilidad y operación.** Propagación del `correlationId` o `traceId` por llamadas síncronas y eventos. Campos obligatorios de los logs. Métricas de negocio y técnicas que alimentan los SLOs de §1. Herramienta de tracing y dónde corre. Alertas. Cómo el operador detecta y resuelve bloqueos y reconstruye una operación. | — |
| 11 | **ADRs.** Índice con estado de cada decisión. | Un archivo por ADR en `docs/adr/` |
| 12 | **Riesgos, cambios y preguntas abiertas.** Cambios respecto de la primera entrega, matriz de preguntas de diseño de `E03`, riesgos y preguntas abiertas. Esta sección se evalúa: reconocer un riesgo o un cambio vale más que ocultarlo. | 12.1 Cambios; 12.2 Matriz de preguntas; 12.3 Riesgos |

Fuente de la tabla: [E04]. Las secciones 1, 7 y 12 dependen de las [consideraciones de escala y las preguntas de diseño de `E03`](consigna-general-album-2026.md#diseño-para-un-millón-de-usuarios). [E03, p. 9] [E03, p. 12]

## ADRs

Una decisión es significativa si cuesta caro cambiarla después o si afecta a más de un servicio. Como mínimo debe haber ADRs para: [E04]

1. el recorte de servicios a partir de los bounded contexts;
2. el estilo de comunicación entre servicios;
3. el broker y la clave de partición de los eventos;
4. el estilo de coordinación del intercambio;
5. la estrategia de idempotencia;
6. la publicación confiable de eventos;
7. el motor de persistencia, al menos del servicio que sostiene las cantidades de la colección;
8. la organización de ambientes en Kubernetes;
9. la estrategia de despliegue y promoción;
10. la estrategia de observabilidad.

Cada ADR considera **al menos dos alternativas reales** y explica por qué se descartaron: "un ADR sin alternativas es una afirmación, no una decisión". El formato es título, estado (Propuesta, Aceptada o Reemplazada por otro ADR), fecha, contexto con los drivers en juego, alternativas, decisión y consecuencias. [E04]

## Reglas de calidad

- Toda pieza se rastrea hasta un contexto, agregado o evento de la primera entrega.
- Toda tecnología tiene un ADR o una justificación explícita; "es la que conocemos" no alcanza.
- Ningún dato tiene dos servicios que lo escriban.
- Los diagramas son consistentes entre sí y con el texto.
- Los servicios, eventos, operaciones y campos usan el lenguaje ubicuo. Se rechazan nombres genéricos como `core-service`, `manager`, `handler`, `data-service` o `processor`.
- Kubernetes y CI/CD se documentan con el mismo rigor que el resto.
- Si una plantilla no aplica, se explica por qué en vez de rellenarla. [E04]

## Rúbrica

| Criterio | Qué se mira | Peso |
|---|---|---:|
| Drivers y trazabilidad | Atributos medibles, NFRs justificados, trazabilidad al modelo, cambios registrados | 10 % |
| Descomposición y datos | Recorte justificado, único dueño por dato, proyecciones, persistencia por necesidad | 20 % |
| Comunicación y coordinación | Síncrono o asíncrono según consistencia, contratos completos, saga con máquina de estados, compensación y recuperación | 20 % |
| Consistencia, idempotencia y escala | Mecanismo por garantía, invariante de oro, duplicados, orden, concurrencia, hot spots y camino de lectura | 20 % |
| Despliegue y CI/CD | Ambientes, recursos de Kubernetes, sondas con sentido, gate de staging, artefacto inmutable, rollback y migraciones | 20 % |
| Observabilidad y decisiones | `traceId` propagado, operación ante incidentes, ADRs con alternativas | 10 % |

Fuente: [E04].

Restan puntos:

- una base escribible compartida;
- servicios sin bounded context;
- tecnologías sin justificar;
- ADRs sin alternativas;
- diagramas inconsistentes;
- sondas triviales;
- un pipeline que no garantiza que producción ejecute lo probado en staging;
- funcionalidades de solo diseño sin lugar en la arquitectura.

No suma agregar servicios, tecnologías o infraestructura sin un driver documentado. [E04]

La entrega es grupal y se versiona en el repositorio del equipo. La fecha límite y el canal se informan en el Campus. [E04]

## Unidades de la materia que alimentan cada sección

**Inferencia de estudio:** la consigna no asigna unidades a cada sección. Esta tabla relaciona lo pedido con el material ya incorporado.

| § | Unidades y conceptos |
|---:|---|
| 3 | [DDD estratégico](ddd-estrategico.md): el bounded context es una frontera de modelo y no necesariamente de despliegue [T03, p. 31]. [Diseño de servicios](diseno-de-servicios.md): acoplamiento, cohesión y anatomía. |
| 4 | [Comunicación síncrona](comunicacion-sincrona.md): REST para APIs públicas y gRPC interno [T09, p. 17]. [Kafka](comunicacion-asincrona-y-kafka.md): CloudEvents, key de partición, outbox y evolución de esquema [T14, p. 11] [T14, p. 47] [T14, p. 55]. |
| 5 | [Sistemas distribuidos](sistemas-distribuidos.md): orquestación y coreografía [T11, p. 20] [T11, p. 37]. [Kafka](comunicacion-asincrona-y-kafka.md): Sagas con timeouts explícitos [T14, p. 27]. |
| 6 | [Estado distribuido](estado-distribuido.md): locks con fencing, claves de idempotencia en la misma frontera atómica que el efecto y semánticas de entrega [T12, p. 11] [T12, p. 20] [T12, p. 26]. Inbox del lado consumidor [T14, p. 50]. Bulkhead y sobrecarga [T09, p. 26]. |
| 7 | Consumer lag como medida del atraso de las proyecciones [T14, p. 43]. Valkey para estado derivado o cacheado, como rankings y vistas [T12, p. 5], con cache-aside [T12, p. 15] y colas [T12, p. 14]. |
| 8 | [Docker Compose](docker-compose.md) para la ejecución local; readiness frente a liveness [T07, p. 18]. Kubernetes todavía no está incorporado: el programa lo ubica en el tercer módulo [T01, p. 12] [T01, p. 19]. |
| 9 | [DevOps y CI/CD](devops-y-ci-cd.md): pipeline y estrategias de despliegue [T13, p. 27] [T13, p. 40]. Promoción por digest inmutable [T06, p. 36]. [Testing](testing.md): niveles y contract testing [T10, p. 10]. |
| 10 | `correlationId` y `causationId` [T14, p. 54]. La unidad de observabilidad todavía no está incorporada: el programa la ubica en el cuarto módulo [T01, p. 13] [T01, p. 20]. |

## Puntos a resolver antes de redactar

- **Cuándo se aparta una copia.** Según `E03`, al publicar; según `E01` v1.1, al aceptar. `E04` pone a `consigna.md` por encima, pero asigna las reglas de negocio a la consigna funcional. [E03, p. 4] [E01, p. 2] [E04] La decisión condiciona la máquina de estados (§5), el modelo de consistencia (§6) y los hot spots (§7). Hay que registrarla en un ADR o en §12, y consultarla con la cátedra.
- **Autenticación.** La §2 pide explicar cómo se reemplaza el proveedor de identidad. `E03` admite un `userId` en header; `E01` exige Google real. [E04] [E03, p. 8] [E01, p. 6]
- **Numeración de escenarios.** `E04` usa la de `E03`, donde el escenario 4 es el intercambio rechazado y el 15 es el código promocional. [E04] [E03, p. 10] [E03, p. 11]
- **Referencias internas de `E04`.** Dice que hay "una sola" decisión impuesta y enumera dos. También remite a la "sección 10 del punto 5" para los cambios del modelo, aunque esos cambios se registran en la sección 12.1. [E04] Ambos casos están en [Dudas y conflictos](../dudas-y-conflictos.md).
- **Solo diseño con lugar concreto.** Retos, recompensas, códigos, rankings, matching, N:M, vista de operador y campañas deben aparecer en el catálogo de servicios, en la propiedad de datos y en la matriz de interacciones. [E04] [E03, p. 8]
