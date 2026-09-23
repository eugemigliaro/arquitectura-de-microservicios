# Wiki de la materia

Índice del conocimiento canónico compilado para Arquitectura de Microservicios. Las afirmaciones específicas se vinculan con el material oficial mediante citas `[ID, p. N]`; ante una diferencia con el Campus u otra comunicación oficial más reciente, prevalece esa fuente y debe registrarse el conflicto.

## Navegación

- [Glosario](glosario.md)
- [Dudas y conflictos](dudas-y-conflictos.md)
- [Preguntas de repaso](repaso/preguntas.md)

## Temas

- [Programa y organización](temas/programa-y-organizacion.md): objetivos, recorrido de la materia, bibliografía y versiones del régimen de evaluación.
- [Fundamentos de microservicios](temas/fundamentos-de-microservicios.md): definición, comparación con monolitos y SOA, propiedades, costos y criterios de adopción.
- [Diseño de servicios](temas/diseno-de-servicios.md): acoplamiento, cohesión, capacidades de negocio, contratos y anatomía interna.
- [DDD estratégico](temas/ddd-estrategico.md): lenguaje ubicuo, subdominios, bounded contexts, context mapping y fronteras de servicio.
- [DDD táctico y EventStorming](temas/ddd-tactico-y-eventstorming.md): entidades, value objects, aggregates, comandos, eventos y taller de e-commerce.
- [Práctica de DDD](temas/practica-ddd.md): resolución 2025 de Pedidos y ejercicio abierto 2026 de Huella.
- [Contenedores sobre Linux](temas/contenedores-linux.md): namespaces, cgroups, OverlayFS, OCI, runtime stack, seguridad y demo manual.
- [Imágenes de contenedores y Dockerfile](temas/imagenes-de-contenedores.md): capas, instrucciones, cache, build context, multi-stage, seguridad y trazabilidad.
- [Docker CLI, redes y volúmenes](temas/docker-cli-redes-y-volumenes.md): ciclo de vida, diagnóstico, modos de red, publicación de puertos, DNS y persistencia.
- [Docker Compose](temas/docker-compose.md): servicios, redes, volúmenes, configuración, secretos, builds y orden de arranque.
- [Comunicación síncrona](temas/comunicacion-sincrona.md): REST, gRPC, GraphQL, ConnectRPC, capacidad, sobrecarga y arquitectura multi-protocolo.
- [Testing de microservicios](temas/testing.md): niveles y dimensiones de prueba, contract testing, dobles, estructura, cobertura y seguridad.
- [Sistemas distribuidos](temas/sistemas-distribuidos.md): modelos de falla de red, nodos y tiempo, RPC, orquestación, serializabilidad, locking, 2PC y coreografía.
- [Estado distribuido y patrones con Valkey](temas/estado-distribuido.md): estado entre instancias, locks con fencing, semáforos, colas, cache, idempotencia, inbox, semánticas de entrega y streams.
- [DevOps y CI/CD](temas/devops-y-ci-cd.md): ciclo DevOps, CI y CD, pull requests, pipelines en GitLab, estrategias de despliegue y shift-left.
- [Entrega: Álbum de Figuritas Mundial 2026](temas/entrega-album-2026.md): versión 1.1, invariantes, RF/RNF, escenarios y rúbrica.
- [Primera entrega TP1: análisis DDD del Álbum](temas/primera-entrega-album-ddd.md): alcance, estructura obligatoria, trazabilidad y rúbrica del primer avance.
- [DDD moderno: lectura complementaria](temas/ddd-complementario.md): Event Modeling, arquitectura hexagonal, outbox, idempotencia y Data Mesh desde una fuente externa.

## Fuentes incorporadas

| ID | Fuente y autoridad | Páginas | Cobertura principal |
|---|---|---:|---|
| `T01` | *Presentación de la materia* — oficial | 25 | Objetivos, cronograma, recursos, bibliografía y evaluación |
| `T02` | *Conceptos acerca de microservicios* — oficial | 44 | Fundamentos, criterios de diseño, adopción y anatomía |
| `T03` | *Domain-Driven Design* — oficial | 58 | DDD estratégico y táctico, context mapping y EventStorming |
| `T04` | *Linux y contenedores* — oficial | 41 | Primitivas de kernel, capas, runtimes y seguridad |
| `T05` | *Demo: construyendo un container a mano* — oficial | 59 | Laboratorio paso a paso y ciclo de vida manual |
| `T06` | *Imágenes de contenedores* — oficial | 42 | Dockerfile, cache, build context, multi-stage, seguridad, tags y digests |
| `T07` | *Docker Compose* — oficial | 25 | Orquestación local, configuración, secretos, redes, volúmenes y healthchecks |
| `T08` | *Docker CLI, redes y volúmenes* — oficial | 47 | Operación y diagnóstico, drivers de red, NAT, DNS, mounts y persistencia |
| `T09` | *Comunicación síncrona* — oficial | 37 | REST, gRPC, GraphQL, ConnectRPC, sobrecarga y arquitectura multi-protocolo |
| `T10` | *Testing* — oficial | 24 | Pirámide, niveles y dimensiones de prueba, contratos, cobertura y seguridad |
| `T11` | *Sistemas distribuidos* — oficial | 40 | Modelos de falla, RPC, transacciones concurrentes, 2PC, orquestación y coreografía |
| `T12` | *Estado distribuido: patrones con Valkey* — oficial | 33 | Coordinación entre instancias, locks, cache, idempotencia, entrega de mensajes y streams |
| `T13` | *DevOps* — oficial | 47 | Ciclo DevOps, CI/CD, repositorios y PR, pipeline, despliegues y shift-left |
| `P01` | *Ejercicio DDD: Pedidos — resolución* — oficial, ciclo 2025 | 16 | Fronteras, EventStorming y detalle por dominio |
| `P02` | *Ejercicio de DDD: Huella* — oficial | 10 | Taller sin solución, entregables y rúbrica |
| `E01` | *Requerimientos funcionales: Álbum de Figuritas Mundial 2026* — oficial, v1.1 | 7 | Consigna evaluada, invariantes, escenarios y condiciones de entrega |
| `E02` | *Primera entrega TP1: análisis de dominio DDD del Álbum 2026* — oficial | — | Primer avance grupal, estructura de `docs/analisis-ddd.md`, alcance y rúbrica |
| `B01` | *Modern Domain-Driven Design Guide* — externa | 17 | Extensiones y patrones complementarios de DDD |
