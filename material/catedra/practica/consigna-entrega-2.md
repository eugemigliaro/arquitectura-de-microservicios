# Entrega 2: Documentación de arquitectura del Álbum de Figuritas Mundial 2026

**Materia:** 73.40 Microservicios (ITBA)

Esta entrega **no incluye implementación**. El trabajo pedido es la **documentación de la arquitectura** de la plataforma, construida a partir del análisis de dominio (DDD) que el equipo realizó y entregó en la Entrega 1.

---

## 1. Objetivo

Traducir el modelo de dominio en una arquitectura de microservicios concreta, justificada y documentada: qué servicios existen, qué datos posee cada uno, cómo se comunican, cómo se coordinan las operaciones distribuidas, cómo se despliegan y cómo se observan.

El propósito es que el equipo llegue a la implementación con las decisiones técnicas tomadas, discutidas y registradas, y que cualquier persona que lea el documento pueda entender **por qué** el sistema tiene la forma que tiene, y no solo **qué** forma tiene.

## 2. Insumo

La arquitectura se apoya en tres documentos, en este orden de precedencia:

1. `consigna.md`: fuente de verdad del producto, del alcance, de los escenarios mínimos, de las condiciones de CI/CD y de las consideraciones de escala.
2. `consigna-funcional.md` (versión 1.1): requerimientos funcionales, reglas de negocio y escenarios de aceptación.
3. `docs/analisis-ddd.md`: el análisis de dominio entregado por el equipo en la Entrega 1, con las correcciones que surjan de la devolución de la cátedra.

La arquitectura **debe derivarse del diseño entregado**. Cada servicio debe poder rastrearse hasta uno o más contextos acotados; cada agregado debe tener un servicio dueño; cada evento de dominio que cruza fronteras debe tener un contrato; cada invariante debe tener un mecanismo que lo sostiene. Si al diseñar la arquitectura el equipo descubre que el modelo de la Entrega 1 era incorrecto o incompleto, puede modificarlo, pero debe registrar el cambio y su motivo (ver sección 10 del punto 5).

## 3. Alcance

### Decisiones ya tomadas

Hay **una sola** decisión de arquitectura que viene impuesta y no se discute:

- **El sistema corre en un cluster de Kubernetes**. El ambiente de producción es un cluster de Kubernetes en AKS (Azure Kubernetes Service), según lo establecido en `consigna.md`.
- **El sistema se construye y despliega con un proceso de CI/CD** como el visto en clase: build y tests automatizados en cada cambio, empaquetado en imágenes de contenedor, despliegue automático a staging, tests de integración en staging como gate, y promoción a producción solo si ese gate es exitoso.

Todo lo demás es decisión del equipo y debe justificarse: cantidad y recorte de servicios, lenguajes, frameworks, motores de persistencia, broker de mensajes, estilo de comunicación, estrategia de coordinación, herramientas de observabilidad, herramienta de CI/CD, estrategia de despliegue, y cómo se organiza el cluster.

"Porque corre en Kubernetes" no es una justificación de ninguna otra decisión: el cluster es el entorno de ejecución, no la arquitectura.

### Incluido

Las secciones descriptas en el punto 5, el registro de decisiones de arquitectura (ADRs) y los diagramas asociados.

Deben cubrirse tanto lo que `consigna.md` pide **implementar** (colección, apertura de sobres, intercambio 1:1, progreso del álbum, observabilidad) como lo que pide **solo diseñar** (retos, recompensas, códigos promocionales, rankings, matching de ofertas, ofertas N:M, vista de operador, campañas). Lo que es solo de diseño puede tratarse con menos profundidad, pero debe tener un lugar en la arquitectura: qué servicio lo aloja, qué datos posee y cómo se integra.

### Excluido de esta entrega

Código de los servicios, manifiestos de Kubernetes o definiciones de pipeline ejecutables. Se pueden incluir fragmentos ilustrativos (un esquema de evento, un extracto de manifiesto, una etapa del pipeline) cuando ayuden a explicar una decisión, pero no se evalúa que funcionen.

## 4. Entregable

Un documento principal y un directorio de decisiones en el repositorio del equipo:

```text
docs/arquitectura.md
docs/adr/0001-<titulo-corto>.md
docs/adr/0002-<titulo-corto>.md
...
```

Requisitos de forma:

- Escrito en castellano, usando **los mismos términos del lenguaje ubicuo** definidos en la Entrega 1. Un servicio, un evento o un endpoint no puede nombrar con otra palabra un concepto que el glosario ya definió.
- Estructurado con las secciones y los formatos indicados en el punto 5, en ese orden y con esa numeración.
- Los diagramas deben estar versionados como texto en el repositorio (Mermaid, PlantUML, Structurizr DSL o ASCII). Una imagen exportada puede acompañarlos, pero no reemplazarlos.
- Extensión sugerida: entre 15 y 25 páginas equivalentes, sin contar los ADRs. No se evalúa el volumen sino la precisión.
- Debe incluir en el encabezado los integrantes del equipo, la versión de `consigna-funcional.md` y la versión (commit o tag) de `docs/analisis-ddd.md` sobre las que se trabajó.

## 5. Contenido obligatorio del documento

### Sección 1 — Drivers de arquitectura

Explicitar qué fuerzas le dan forma a la arquitectura antes de mostrar la arquitectura.

- **Objetivos de negocio** que la arquitectura debe soportar, en pocas líneas.
- **Restricciones**: las impuestas (Kubernetes/AKS, CI/CD con gate de staging, cada servicio dueño de sus datos, ejecución local a partir del repositorio) y las que el equipo se autoimpone.
- **Atributos de calidad** priorizados, expresados como **escenarios medibles** y no como adjetivos. "El sistema debe ser escalable" no es un atributo de calidad; "ante un pico de 20.000 aperturas de sobre por minuto durante 5 minutos, el p99 de apertura se mantiene por debajo de 500 ms y ninguna apertura se duplica" sí lo es.
- **Requisitos no funcionales** pedidos en "Consideraciones adicionales: diseño para un millón de usuarios" de `consigna.md`: throughput esperado, latencia objetivo por operación crítica y disponibilidad mínima. Los números son supuestos del equipo y deben estar justificados (cómo se estimaron).

Formato:

```markdown
## 1. Drivers de arquitectura

### 1.1 Restricciones

| Restricción | Origen | Impacto en el diseño |
|---|---|---|
| Producción en Kubernetes (AKS) | consigna.md | [...] |

### 1.2 Escenarios de atributos de calidad

| ID | Atributo | Estímulo | Entorno | Respuesta esperada | Medida | Prioridad |
|---|---|---|---|---|---|---|
| QA-01 | [Ej.: Consistencia] | [Qué ocurre] | [En qué condiciones] | [Qué hace el sistema] | [Cómo se mide] | Alta / Media / Baja |

### 1.3 Requisitos no funcionales

| Operación | Throughput esperado | Latencia objetivo (p99) | Disponibilidad | Justificación |
|---|---|---|---|---|
```

### Sección 2 — Vista de contexto

Mostrar el sistema como una caja, rodeado de sus actores y sistemas externos (nivel 1 del modelo C4).

- Actores de `consigna-funcional.md` (coleccionista, operador, etc.) y qué hace cada uno con el sistema.
- Sistemas externos, aun cuando estén fuera de alcance o simulados (por ejemplo, proveedor de identidad), indicando cómo se los reemplaza en esta solución.
- Diagrama de contexto.

### Sección 3 — Descomposición en servicios

Mostrar los servicios, los almacenes de datos, el broker y demás piezas desplegables (nivel 2 del modelo C4), y justificar el recorte.

**Parte A: catálogo de servicios**

- Para cada servicio: responsabilidad, contextos acotados que implementa, agregados que aloja, datos que posee y operaciones que expone.
- Justificar explícitamente toda diferencia entre contextos acotados y servicios: si un contexto se dividió en varios servicios, o varios contextos se agruparon en uno, explicar por qué. La correspondencia uno a uno no es obligatoria, pero cualquier desvío debe estar argumentado.
- Indicar qué servicios son **stateless** y cuáles necesitan estado, y dónde vive ese estado.

**Parte B: propiedad de los datos**

- Cada dato del sistema tiene exactamente un servicio dueño que lo escribe. No se acepta una base de datos escribible compartida entre servicios.
- Identificar las proyecciones o réplicas de solo lectura (por ejemplo, el porcentaje del álbum), quién las materializa, a partir de qué eventos y con qué atraso aceptable.
- Justificar la elección del motor de persistencia de cada servicio en función de sus necesidades (consistencia, patrón de acceso, volumen), no por costumbre.

Formato:

```markdown
## 3. Descomposición en servicios

### 3.1 Catálogo de servicios

| Servicio | Contexto(s) acotado(s) | Agregados | Datos que posee | Responsabilidad | Stateless |
|---|---|---|---|---|---|
| [nombre-servicio] | [Contexto de la Entrega 1] | [Agregado] | [Datos] | [Qué hace] | Sí / No |

### 3.2 Trazabilidad contexto → servicio

| Contexto acotado (Entrega 1) | Servicio(s) | Motivo del recorte |
|---|---|---|

### 3.3 Propiedad de los datos

| Dato | Servicio dueño | Almacén | Lectores | Proyecciones derivadas |
|---|---|---|---|---|

### 3.4 Diagrama de contenedores

[Diagrama C4 nivel 2]
```

### Sección 4 — Comunicación e integración

Definir cómo hablan los servicios entre sí y con el exterior.

**Parte A: estilo de comunicación**

- Para cada interacción entre servicios, indicar si es síncrona o asíncrona, y por qué. La sección 4 de `consigna-funcional.md` —qué debe ser correcto al instante y qué puede demorar— es el criterio principal.
- Indicar el punto de entrada del sistema (API gateway, ingress, BFF u otro) y qué responsabilidades tiene.

**Parte B: APIs**

- Para cada servicio que expone una API: recursos u operaciones principales, códigos de respuesta relevantes (incluidos los de error de negocio) y cómo se transporta la clave de idempotencia y el identificador de correlación.
- Estrategia de versionado de APIs.
- No hace falta una especificación OpenAPI completa: alcanza con las operaciones que participan en los escenarios mínimos, con el detalle necesario para entender el contrato.

**Parte C: contratos de eventos**

- Para cada evento de dominio de la Entrega 1 que cruza fronteras: productor, consumidores, tópico o canal, clave de partición y esquema.
- Un sobre (envelope) común para todos los eventos: identificador único del evento, tipo, versión del esquema, momento de ocurrencia, identificador de correlación y demás metadatos que el equipo considere necesarios.
- Estrategia de evolución de esquemas: qué cambios son compatibles, cómo se introduce un cambio incompatible sin interrumpir a los consumidores.
- Cómo se garantiza que un cambio de estado y la publicación de su evento no diverjan (por ejemplo, outbox transaccional) o, si se acepta que puedan divergir, qué lo repara.

Formato:

```markdown
## 4. Comunicación e integración

### 4.1 Matriz de interacciones

| Origen | Destino | Estilo | Mecanismo | Motivo |
|---|---|---|---|---|
| [servicio-a] | [servicio-b] | Síncrono / Asíncrono | [HTTP, gRPC, evento, comando] | [Por qué] |

### 4.2 APIs

**[servicio] — [operación]**
- Método y ruta: [...]
- Entrada: [...]
- Respuestas: [código → significado de negocio]
- Idempotencia: [cómo se identifica y qué pasa en un reintento]

### 4.3 Contratos de eventos

**[NombreDelEvento]** (v[n])
- Productor: [servicio]
- Consumidores: [servicios]
- Canal: [tópico/cola]
- Clave de partición: [campo] — [qué orden garantiza]
- Payload: [campos]
```

### Sección 5 — Flujos y coordinación distribuida

Mostrar cómo colaboran los servicios en el tiempo (vista dinámica).

- Diagramas de secuencia de, como mínimo:
  - apertura de sobre, incluido el reintento del cliente (escenarios 1 y 2);
  - intercambio exitoso (escenario 3);
  - intercambio rechazado (escenario 4);
  - intercambio con interrupción y compensación (escenarios 5 y 6);
  - resultado desconocido para el solicitante (escenario 7);
  - vencimiento o cancelación contra aceptación (escenario 12).
- Para el intercambio: elegir y justificar el estilo de coordinación (saga orquestada, coreografiada u otro), con la **máquina de estados** completa de la operación, sus transiciones, quién dispara cada una, los timeouts y las compensaciones.
- Explicar cómo se **recupera** una operación que quedó incompleta: quién la detecta, cómo se reanuda o se compensa, y cómo se garantiza que la recuperación no duplique efectos.
- Para las funcionalidades de solo diseño, alcanza con un diagrama de secuencia del flujo principal de retos y del último uso concurrente de un código promocional (escenario 15).

### Sección 6 — Consistencia, idempotencia y tolerancia a fallos

Explicar qué mecanismo concreto sostiene cada garantía.

- **Modelo de consistencia**: para cada dato listado en la sección de consistencia de `consigna.md`, indicar si es fuerte o eventual, qué servicio lo garantiza y con qué mecanismo (transacción local, bloqueo optimista, reserva, proyección).
- **Invariante de oro**: cómo garantiza la arquitectura que ninguna copia se pierda, se duplique ni quede apartada indefinidamente, y cómo se verifica.
- **Idempotencia**: en la API (reintentos del cliente) y en los consumidores (entregas duplicadas). Dónde se almacenan las claves, con qué alcance y durante cuánto tiempo.
- **Orden**: qué pasa si los mensajes llegan fuera de orden (escenario 9) y qué garantías de orden da la clave de partición elegida.
- **Resiliencia entre servicios**: timeouts, reintentos con backoff, circuit breakers, bulkheads y backpressure. Indicar dónde se aplica cada uno y qué pasa cuando se activa.
- **Concurrencia**: cómo se resuelven las dos aceptaciones concurrentes (escenario 11) y el último uso de un código (escenario 15) sin depender de estado en memoria de una instancia.

Formato:

```markdown
## 6. Consistencia, idempotencia y tolerancia a fallos

### 6.1 Modelo de consistencia

| Dato | Consistencia | Servicio responsable | Mecanismo |
|---|---|---|---|

### 6.2 Tratamiento de fallos por escenario

| Escenario | Qué falla | Cómo se detecta | Cómo se resuelve | Estado final garantizado |
|---|---|---|---|---|
```

### Sección 7 — Escalabilidad

Responder las preguntas de "Consideraciones adicionales: diseño para un millón de usuarios" de `consigna.md` desde la arquitectura.

- **Puntos calientes** del dominio (código viral, figurita muy buscada, oferta popular) y cómo se evita que se conviertan en un cuello de botella de escritura.
- **Perfil de tráfico** y comportamiento ante picos correlacionados con eventos del torneo; mecanismos de protección (limitación de tasa, colas de absorción, degradación controlada).
- **Camino de lectura**: qué se cachea, qué se precalcula y qué se consulta en el momento para el resumen del coleccionista y los rankings.
- **Particionamiento** del backbone de eventos y de las bases de datos por servicio.
- **Retención y purgado** de claves de idempotencia y eventos históricos.
- **Escalado horizontal**: qué servicios escalan con réplicas y qué impide a los demás hacerlo.

### Sección 8 — Despliegue en Kubernetes

Describir cómo se materializa la arquitectura en el cluster (vista de despliegue).

- **Topología de ambientes**: local, staging y producción. Si staging y producción comparten cluster (separados por namespace) o son clusters distintos, y por qué.
- **Mapeo de piezas a recursos de Kubernetes**: qué es un Deployment, qué es un StatefulSet, qué es un Job o CronJob (por ejemplo, un proceso de recuperación o de purgado), qué Services e Ingress existen.
- **Componentes con estado** (bases de datos, broker): si corren dentro del cluster o como servicios gestionados, con qué implicancias de operación, persistencia y costo.
- **Configuración y secretos**: cómo llegan a cada servicio y cómo difieren entre ambientes.
- **Salud del servicio**: qué verifican las sondas de liveness, readiness y startup de cada servicio, y por qué. Una readiness que siempre responde OK es un error de diseño.
- **Recursos**: criterio para requests y limits, y política de escalado (HPA u otra) aunque no se implemente.
- **Ejecución local**: cómo se levanta la misma arquitectura fuera del cluster (por ejemplo, `docker compose`) y qué diferencias hay con el cluster.
- Diagrama de despliegue.

Formato:

```markdown
## 8. Despliegue en Kubernetes

### 8.1 Ambientes

| Ambiente | Dónde corre | Aislamiento | Datos | Quién despliega |
|---|---|---|---|---|
| Local | [...] | [...] | [...] | [...] |
| Staging | [...] | [...] | [...] | Pipeline |
| Producción | AKS | [...] | [...] | Pipeline |

### 8.2 Recursos por servicio

| Servicio | Tipo de recurso | Réplicas | Liveness | Readiness | Config / Secretos |
|---|---|---|---|---|---|

### 8.3 Diagrama de despliegue

[Diagrama]
```

### Sección 9 — CI/CD

Documentar el pipeline que lleva un cambio desde el repositorio hasta producción. El pipeline es una **condición de entrega** del trabajo final; en esta entrega se evalúa su diseño.

- **Etapas** del pipeline, en orden, con lo que hace cada una, qué la dispara y qué produce.
- **Gates**: qué condición debe cumplirse para avanzar de una etapa a la siguiente. En particular, cómo los tests de integración en staging bloquean la promoción a producción y en qué estado queda producción cuando el gate falla.
- **Artefactos**: cómo se versionan y etiquetan las imágenes, y cómo se garantiza que lo que se prueba en staging es exactamente lo que se despliega en producción.
- **Organización del repositorio y del pipeline**: monorepo o multirepo, un pipeline por servicio o uno global, y cómo se evita reconstruir y redesplegar servicios que no cambiaron.
- **Estrategia de despliegue** en Kubernetes (rolling update, blue-green, canary u otra) y **estrategia de rollback**.
- **Cambios que no son código**: cómo se despliegan migraciones de esquema de base de datos y cambios en contratos de eventos o APIs sin interrumpir el servicio (por ejemplo, expand/contract), y en qué orden respecto del despliegue de los servicios.
- **Tests por etapa**: qué tipo de prueba corre en cada etapa (unitarias, funcionales, de contrato, de integración, de carga) y cuáles corresponden a los escenarios mínimos.
- **Credenciales**: cómo accede el pipeline al cluster y al registro de imágenes sin exponer secretos.
- Diagrama del pipeline.

Formato:

```markdown
## 9. CI/CD

### 9.1 Etapas

| # | Etapa | Disparador | Qué hace | Artefacto producido | Gate para avanzar |
|---|---|---|---|---|---|
| 1 | Build + tests | Push / merge request | [...] | [...] | [...] |
| ... | Despliegue a staging | [...] | [...] | [...] | [...] |
| ... | Tests de integración | [...] | [...] | [...] | Todos los escenarios pasan |
| ... | Despliegue a producción | [...] | [...] | [...] | — |

### 9.2 Diagrama del pipeline

[Diagrama]
```

### Sección 10 — Observabilidad y operación

- Cómo se propaga el **identificador de correlación / traceId** a través de llamadas síncronas y eventos, de modo que se pueda seguir una apertura de sobre y un intercambio (con éxito y con fallo) de extremo a extremo.
- **Logs estructurados**: campos obligatorios.
- **Métricas** de negocio y técnicas por servicio, y cuáles alimentan los SLOs definidos en la sección 1.
- **Tracing**: herramienta y dónde corre (dentro del cluster o fuera).
- **Alertas**: qué situaciones requieren intervención y cómo se detectan.
- **Operador**: cómo detecta y resuelve reservas u ofertas bloqueadas, y cómo reconstruye el estado de una operación ante un incidente.

### Sección 11 — Decisiones de arquitectura (ADRs)

Registrar cada decisión significativa en un archivo propio dentro de `docs/adr/`, y listar en esta sección un índice con su estado.

Una decisión es significativa si cambiarla después cuesta caro o si afecta a más de un servicio. Como mínimo, debe haber ADRs para:

- el recorte de servicios a partir de los contextos acotados;
- el estilo de comunicación entre servicios;
- el broker de mensajes y la clave de partición de los eventos;
- el estilo de coordinación del intercambio;
- la estrategia de idempotencia;
- la estrategia para publicar eventos de forma confiable;
- el motor de persistencia de al menos el servicio que sostiene las cantidades de la colección;
- la organización de ambientes en Kubernetes;
- la estrategia de despliegue y promoción del pipeline;
- la estrategia de observabilidad.

Cada ADR debe considerar **al menos dos alternativas reales** y explicar por qué se descartaron. Un ADR sin alternativas es una afirmación, no una decisión.

Formato de cada ADR:

```markdown
# ADR-NNNN: [Título de la decisión]

- Estado: Propuesta / Aceptada / Reemplazada por ADR-XXXX
- Fecha: [AAAA-MM-DD]

## Contexto
[Qué problema se resuelve, qué fuerzas y restricciones intervienen, qué drivers de la sección 1 están en juego]

## Alternativas consideradas
1. [Alternativa A] — [ventajas / desventajas]
2. [Alternativa B] — [ventajas / desventajas]

## Decisión
[Qué se decidió]

## Consecuencias
[Qué se gana, qué se pierde, qué riesgos se aceptan, qué otras decisiones condiciona]
```

### Sección 12 — Riesgos, cambios y preguntas abiertas

Cerrar el documento con:

- **Cambios respecto de la Entrega 1**: qué se modificó del modelo de dominio al diseñar la arquitectura y por qué.
- **Matriz de preguntas de diseño**: para cada pregunta de "Preguntas que deberá responder el diseño" y de "Preguntas adicionales" de `consigna.md`, la sección del documento que la responde.
- **Riesgos** técnicos identificados, su impacto y cómo se mitigan o por qué se aceptan.
- **Preguntas abiertas** que quedaron sin resolver y qué parte de la arquitectura depende de cada respuesta.

Formato:

```markdown
## 12. Riesgos, cambios y preguntas abiertas

### 12.1 Cambios respecto de la Entrega 1

| Elemento | Antes | Ahora | Motivo |
|---|---|---|---|

### 12.2 Matriz de preguntas de diseño

| Pregunta de consigna.md | Sección que la responde |
|---|---|

### 12.3 Riesgos

| Riesgo | Impacto | Probabilidad | Mitigación / aceptación |
|---|---|---|---|
```

Esta sección se evalúa: reconocer un riesgo o un cambio de modelo vale más que ocultarlo.

## 6. Reglas de calidad

- Toda pieza de la arquitectura se rastrea hasta el dominio: se debe poder responder "¿qué contexto, agregado o evento de la Entrega 1 motiva esto?".
- Toda tecnología elegida tiene un ADR o una justificación explícita. No se acepta "es la que conocemos" como único argumento.
- Ningún dato tiene dos servicios que lo escriban.
- Los diagramas son consistentes entre sí y con el texto: un servicio, un evento o una flecha que aparece en un diagrama y no en otro es un error.
- Se usa el lenguaje ubicuo de la Entrega 1 en nombres de servicios, eventos, operaciones y campos. No se aceptan nombres genéricos (`core-service`, `manager`, `handler`, `data-service`, `processor`).
- Kubernetes y CI/CD se documentan con el mismo rigor que el resto: no alcanza con nombrarlos.
- Nada de boilerplate: si una plantilla del punto 5 no aplica, explicar por qué en lugar de rellenarla.

## 7. Criterios de corrección


| Criterio                            | Qué se mira                                                                                                                                                                 | Peso |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---- |
| Drivers y trazabilidad              | Atributos de calidad medibles, NFRs justificados, servicios y contratos rastreables al modelo de la Entrega 1, cambios de modelo registrados                                | 10%  |
| Descomposición y datos              | Recorte de servicios justificado, un único dueño por dato, proyecciones identificadas, persistencia elegida por necesidad                                                   | 20%  |
| Comunicación y coordinación         | Estilo síncrono/asíncrono coherente con la consistencia requerida, contratos de eventos completos, saga del intercambio con máquina de estados, compensación y recuperación | 20%  |
| Consistencia, idempotencia y escala | Mecanismo concreto para cada garantía, invariante de oro sostenido, tratamiento de duplicados, orden y concurrencia, puntos calientes y camino de lectura                   | 20%  |
| Despliegue y CI/CD                  | Topología de ambientes, mapeo a recursos de Kubernetes, sondas con sentido, pipeline con gate de staging, artefacto inmutable promovido, rollback y migraciones             | 20%  |
| Observabilidad y decisiones         | Propagación del traceId, operación ante incidentes, ADRs con alternativas reales y consecuencias                                                                            | 10%  |


Restan puntos: una base de datos escribible compartida, servicios que no se rastrean a ningún contexto acotado, tecnologías sin justificar, ADRs sin alternativas, diagramas inconsistentes entre sí, sondas de salud triviales, un pipeline que no garantiza que lo desplegado en producción sea lo probado en staging, y funcionalidades de solo diseño sin lugar en la arquitectura.

No suma puntos agregar más servicios, más tecnologías o más componentes de infraestructura si eso no responde a un driver documentado.

## 8. Formato y modalidad de entrega

- El documento y los ADRs se entregan versionados en el repositorio del equipo, en las rutas indicadas en el punto 4.
- La entrega es **grupal**; se espera que las decisiones de arquitectura hayan sido discutidas por todo el equipo.
- Fecha límite y canal de entrega: según lo indicado por la cátedra en el campus.

