# Laboratorio de Pipelines CI/CD con GitLab

**Introducción**

Este repositorio ha sido concebido como un material de estudio y laboratorio práctico. Su propósito es ilustrar la implementación de un pipeline de Integración Continua y Despliegue Continuo (CI/CD) mediante la plataforma GitLab CI/CD. A través de este ejemplo, se abordará la gestión automatizada del ciclo de vida de software para una arquitectura basada en microservicios, utilizando tres componentes: `microservice-sale`, `microservice-inventory` y `microservice-payment`.

El objetivo es que puedan analizar, comprender y, eventualmente, replicar o adaptar estas prácticas en sus propios proyectos.

---

## 📜 Descripción del Pipeline (`.gitlab-ci.yml`)

El núcleo de este laboratorio es el archivo de configuración `.gitlab-ci.yml`. Este archivo define la secuencia de pasos automatizados que abarcan desde la validación del código hasta su eventual despliegue. A continuación, se procederá a desglosar su estructura y los elementos que lo componen.

---

## 🏗️ Imagen Base e Inclusiones

* **Imagen Base:** Se utiliza `openjdk:17-jdk-slim` como imagen Docker base para la ejecución de todos los trabajos (jobs). Esto asegura un entorno de ejecución consistente basado en Java 17.
* **Inclusiones de GitLab:** Se aprovechan plantillas proporcionadas por GitLab para integrar funcionalidades estándar, específicamente en el ámbito de la seguridad:
    * `Security/SAST.gitlab-ci.yml`: Integra el Análisis Estático de Seguridad de Aplicaciones (SAST).
    * `Security/Dependency-Scanning.gitlab-ci.yml`: Incorpora el escaneo de dependencias para identificar vulnerabilidades conocidas.

---

## 🌐 Variables Globales

Se han definido variables de entorno a nivel global para facilitar la configuración y la reutilización dentro del pipeline:

* `GITLAB_ADVANCED_SAST_ENABLED`: Activa características extendidas del análisis SAST.
* `SEARCH_MAX_DEPTH`: Define la profundidad de búsqueda para ciertas herramientas de SAST.
* `SAST_JAVA_VERSION`: Especifica la versión de Java a considerar por las herramientas SAST.
* `MAVEN_OPTS`: Establece opciones para Maven, principalmente para definir un directorio de repositorio local (`.m2/repository`), lo cual es fundamental para el funcionamiento de la caché.
* `MICROSERVICE_SALE`, `MICROSERVICE_INVENTORY`, `MICROSERVICE_PAYMENT`: Identificadores de los microservicios gestionados.
* `ENVIRONMENT_NAME`: Define el nombre del entorno de destino para el despliegue (ej. `produccion`).

---

## 🔄 Etapas (Stages)

El flujo de trabajo se organiza en las siguientes etapas, ejecutadas de forma secuencial:

1.  **`test`**: Ejecución de pruebas unitarias y análisis de seguridad estático.
2.  **`build`**: Compilación del código fuente y generación de los artefactos ejecutables (archivos `.jar`).
3.  **`package`**: Creación de imágenes Docker para cada microservicio y almacenamiento en el registro de contenedores de GitLab.
4.  **`release`**: Etapa *simulada* que representa la preparación de las imágenes Docker para su lanzamiento oficial.
5.  **`deploy`**: Etapa *simulada* que representa el despliegue de los microservicios en el entorno definido.

---

## 🛠️ Plantillas de Jobs (Templates)

Con el fin de promover la reutilización de código y mantener la consistencia, se emplean plantillas (`.job_name`). Estas definen la configuración base para jobs recurrentes:

* **`.maven-job`**: Configura la caché de dependencias de Maven.
* **`.unit-test`**:
    * Ejecuta las pruebas unitarias (`./mvnw test`) para el servicio indicado por `SERVICE_NAME`.
    * Genera y almacena artefactos con los reportes de pruebas (formato JUnit).
    * **Reglas**: Se ejecuta en todos los commits, excepto en la creación de tags de versión.
* **`.build`**:
    * Compila y empaqueta el código (`./mvnw package -DskipTests`), omitiendo la ejecución de tests.
    * Guarda los archivos `.jar` como artefactos, disponibles por 30 días.
    * **Reglas**: Se ejecuta en todos los commits.
* **`.package`**:
    * Utiliza una imagen Docker con `docker:dind` (Docker-in-Docker) para construir imágenes.
    * Construye la imagen Docker a partir del `Dockerfile` correspondiente.
    * Etiqueta la imagen con el hash del commit (`CI_COMMIT_SHA`).
    * Autentica y sube la imagen al registro de GitLab (`CI_REGISTRY`).
    * **Reglas**: Se ejecuta únicamente en la rama principal, excluyendo Merge Requests y tags.
* **`.release`**:
    * *Simula* el proceso de etiquetado (`latest` o tag de versión) y publicación de la imagen Docker final.
    * **Reglas**: Se ejecuta tras un éxito en la rama principal o al crear un tag. Utiliza el tag de versión si está presente.
* **`.deploy`**:
    * *Simula* el despliegue utilizando `kubectl`.
    * Determina la etiqueta de la imagen a desplegar (`latest` o tag).
    * Simula la actualización de un *deployment* en Kubernetes.
    * **Reglas**: Se ejecuta tras un éxito en la rama principal o al crear un tag.

---

## 🛡️ Seguridad (SAST)

* El job `semgrep-sast`, provisto por la plantilla, ejecuta el análisis SAST.
* **Reglas**: Se ejecuta en todos los commits, a excepción de los tags de versión, para centrar el análisis en el código en desarrollo y no en los releases ya validados.

---

## ⚙️ Jobs Específicos por Microservicio

Para cada uno de los tres microservicios, se crean jobs concretos que **heredan** (`extends`) de las plantillas definidas. La variable `SERVICE_NAME` se utiliza para parametrizar estos jobs y dirigir la acción al microservicio correcto.

Se definen jobs para cada etapa (`test`, `build`, `package`, `release`, `deploy`) y para cada microservicio.

Notablemente, los jobs de `release` y `deploy` utilizan la directiva `needs`. Esto establece una dependencia explícita, asegurando que un job (ej. `inventory-release`) solo se ejecute si su prerrequisito (ej. `inventory-package`) ha finalizado satisfactoriamente. Esto permite un cierto grado de paralelismo controlado entre los microservicios en estas etapas tardías.

---

## 💡 Flujo Operativo

El pipeline responde a diferentes eventos de la siguiente manera:

1.  **Commit en la rama principal (`main`)**: Se ejecuta el ciclo completo: test, build, package, release (etiquetado como `latest`) y deploy (usando `latest`).
2.  **Commit en otra rama**: Se ejecutan las etapas de `test` y `build`. No se generan paquetes Docker ni se realizan despliegues.
3.  **Creación de un Tag (`vX.Y.Z`)**: Se ejecutan `build` (sin tests), `release` (etiquetado con el tag de versión) y `deploy` (usando el tag de versión).

Este laboratorio busca ofrecer una visión estructurada de cómo implementar pipelines CI/CD en GitLab, aplicando principios de reutilización y gestionando dependencias para un sistema distribuido. Se recomienda analizar detenidamente el archivo `.gitlab-ci.yml` y observar su ejecución en GitLab para una comprensión más profunda.