# Laboratorios de la cátedra

## Fuente y alcance

La cátedra publica las guías prácticas de 73.40 en un repositorio público de GitLab (`itba-73-40-microservicios/alumnos/2026s2`). Están organizadas por tema y no por clase, porque el calendario cambia entre cuatrimestres; la tabla del README hace de mapa entre ambos. Cada guía es autocontenida, declara sus prerrequisitos y se sigue en orden, con comandos de verificación en cada paso. Las demos, en cambio, son código mostrado en clase para correr y sacar conclusiones, sin pasos. [G00]

**Organización en este repositorio**, decisión propia y no de la cátedra:

- El repositorio se clona en `laboratorio/catedra-2026s2/`, excluido de git, y se actualiza con `git pull`. Ahí se corren las guías. Ver [`laboratorio/README.md`](../../laboratorio/README.md).
- Se registran como fuentes, con IDs `G00`–`G10`, los documentos de texto de las guías tal como estaban en el commit `36b9189` (2026-10-17). Código, manifiestos, Dockerfiles, archivos de Compose y especificaciones quedan en el clon y se citan por ruta y commit.
- La copia de `G06` reemplaza una línea del original que contenía una credencial de acceso. Ver la sección de advertencias.

## Mapa de guías

| Clase | Guía | IDs | Qué se practica | Teoría |
|---|---|---|---|---|
| 3 (21/08) | `guias/linux-containers` | `G01`, `G01-S01`…`G01-S21` | Construir un contenedor a mano con namespaces, cgroups v2, capabilities, seccomp, OverlayFS y `runc`. [G01] | [Contenedores sobre Linux](contenedores-linux.md) |
| 4 (28/08) | `guias/dockerfile-compose` | `G02`, `G02-TEORIA`, `G02-SOLUCION` | Dockerfiles y un Compose para tres servicios FastAPI, cada backend con su Redis. [G02] | [Imágenes](imagenes-de-contenedores.md), [Compose](docker-compose.md) |
| 6 (11/09) | `guias/cicd-pipelines` | `G03` | Pipeline de GitLab CI/CD para tres microservicios Java. [G03] | [DevOps y CI/CD](devops-y-ci-cd.md) |
| 6 (11/09) | `guias/graphql` | `G04` | Queries, mutations y subscriptions con Apollo Server. [G04] | [Comunicación síncrona](comunicacion-sincrona.md) |
| 6 | `demos/capacidad-sync-async` | `G10` | Sobrecarga, bulkhead, retry y cola, medidos con cgroups. [G10] | [Comunicación síncrona](comunicacion-sincrona.md) |
| 7 (18/09) | `guias/laboratorio-redis-patterns` | `G05-LOCK`, `G05-CACHE`, `G05-EVAL`, `G05-MONOREPO` | Lock distribuido con Redlock, cache-aside con Valkey y un monorepo con CI por servicio. [G05-LOCK] [G05-CACHE] [G05-MONOREPO] | [Estado distribuido](estado-distribuido.md), [DevOps](devops-y-ci-cd.md) |
| 7 (18/09) | `guias/laboratorio-coreografia` | `G06`, `G06-CONTEXTO` | Saga coreografiada con Kafka y compensaciones. [G06] | [Comunicación asincrónica](comunicacion-asincrona-y-kafka.md), [Sistemas distribuidos](sistemas-distribuidos.md) |
| 7 (18/09) | `guias/laboratorio-orquestacion` | `G07` | 2PC orquestado sobre Kafka. [G07] | [Sistemas distribuidos](sistemas-distribuidos.md) |
| 10 (09/10) | `guias/k8s-fundamentos` | `G08` | Minikube, Deployment, Service, DNS y self-healing. [G08] | [Kubernetes — Parte 1](kubernetes-parte-1.md) |
| 11 (16/10) | `guias/k8s-aks` | `G09` | Cluster AKS: LoadBalancer, Secret, PVC, NetworkPolicy, Gateway API y los scripts del TP. [G09] | [Kubernetes — Parte 2](kubernetes-parte-2.md), [Kubernetes en AKS](kubernetes-en-aks.md) |

Fuente del mapa: [G00]. La numeración de clases difiere del cronograma de `T01`, que ubicaba Kubernetes en las clases 8 y 9; como indica el propio repositorio, conviene referirse a las unidades por tema. [T01, p. 12] [G00]

## Herramientas por guía

| Herramienta | Dónde se usa |
|---|---|
| Linux con root, `iproute2`, `capsh`, `runc`, cgroups v2 | Contenedores a mano; la demo de capacidad también lee `/sys/fs/cgroup`. [G01] [G10] |
| Docker y Docker Compose v2 | Todas desde la clase 4. [G00] |
| Python 3.12 | Servicios de la guía de Compose y del laboratorio de Valkey. [G02] [G05-MONOREPO] |
| Java 17, Spring Boot y Maven | Pipeline de CI/CD y saga coreografiada. [G03] [G06-CONTEXTO] |
| Node.js | GraphQL, opcional si se usa Docker. [G04] |
| Go | Demo de capacidad, que compila dentro del contenedor, y servicios del laboratorio de orquestación, que se corren solo con Docker. [G10] [G07] |
| `kubectl` y Minikube | Fundamentos de Kubernetes. [G08] |
| Azure CLI, Terraform ≥ 1.6 y Helm 3 | Cluster AKS. [G09] |

## Advertencias

- **Soluciones.** `G02-SOLUCION` y la carpeta `solucion/` de la guía de Compose contienen la resolución completa. Para estudiar con pistas, no abrirlas antes de intentar. [G02-SOLUCION]
- **Credenciales que generan las guías.** La de AKS produce `terraform.tfstate`, con el kubeconfig de administrador en texto plano, y un kubeconfig compartible; ninguno va a un repositorio. [G09] El clon está excluido de git, pero conviene correr las guías con eso presente.
- **Costo.** El cluster de AKS se cobra mientras existe, aunque no se use. Ver [Kubernetes en AKS](kubernetes-en-aks.md). [G09]
- **Credencial en el material publicado.** La última línea del README de la saga coreografiada, tal como está publicado, incluye una URL de GitLab con un token de acceso personal. La copia registrada como `G06` la reemplaza por una marca de redacción; el resto del archivo es idéntico. Ver [Dudas y conflictos](../dudas-y-conflictos.md).

## Diferencias entre documentos y código

| Laboratorio | Diferencia | Qué tomar como referencia |
|---|---|---|
| Saga coreografiada | El diagrama de falla de `G06` y la teoría llaman `PAYMENT_FAILED` al evento de pago fallido; el código del laboratorio y `G06-CONTEXTO` usan `FAILED_PAYMENT`. [G06] [G06-CONTEXTO] [T11, p. 39] | El nombre del código al leer logs o topics. |
| Lock distribuido | `G05-LOCK` describe un producto por defecto (`flash-sale-001`), mientras la especificación normativa del clon y el consumidor trabajan con varios SKUs elegidos al azar. [G05-LOCK] | La especificación, que el README declara normativa. [G05-LOCK] |
| Valkey | El README raíz de `laboratorio-redis-patterns` es la plantilla por defecto de GitLab, sin contenido del laboratorio; por eso no se registró. | Los README de cada servicio y `docs/monorepo-practices.md`. |
| Orquestación | El laboratorio se llama "orquestación", pero implementa 2PC con prepare, votos y commit/rollback, no una saga orquestada con compensaciones. [G07] | Leerlo junto con 2PC en [Sistemas distribuidos](sistemas-distribuidos.md). |
