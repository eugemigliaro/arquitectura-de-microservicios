# Entrega 3: CI/CD y despliegue en AKS del Álbum de Figuritas Mundial 2026

**Materia:** 73.40 Microservicios (ITBA)
**Versión:** 1.1

Esta entrega es **ejecutable**: el trabajo pedido es un cluster AKS funcionando y un proceso de CI/CD que lleve cambios de código hasta producción, construido sobre la arquitectura que el equipo documentó en la Entrega 2.

---

## 1. Objetivo

Materializar la arquitectura en un cluster de Kubernetes en Azure (AKS) y construir el pipeline que la entrega de forma automatizada, repetible y segura, aplicando las prácticas de DevOps vistas en clase: integración continua, testing automatizado en varios niveles, artefactos inmutables, promoción entre ambientes con *gates*, y operación reproducible.

El propósito es que el equipo demuestre que un cambio en el código de un servicio puede recorrer el camino **commit → build → tests → imagen → dev → staging → prod** sin intervención manual, y que ese camino se detiene cuando algo falla.

## 2. Insumo

1. `consigna-funcional.md` (versión 1.1): requerimientos funcionales, escenarios de aceptación (sección 7) y requerimientos no funcionales RNF-01 a RNF-06.
2. `docs/arquitectura.md` y `docs/adr/` (Entrega 2), con las correcciones que surjan de la devolución de la cátedra: es la fuente de verdad de **qué servicios existen**, cómo se comunican y qué contratos tienen.
3. Las guías de AKS, contenedores y CI/CD entregadas durante las clases.

Nota de terminología: `consigna-funcional.md` habla de ambientes *staging* y *producción*. En esta entrega se usan tres ambientes: `dev`, `staging` y `prod`. **`staging` es el ambiente de RNF-02 y RNF-03** (despliegue previo a producción y gate de integración), y `dev` es un ambiente previo adicional. `prod` es el ambiente de producción de `consigna-funcional.md`.

Las decisiones de la sección 3 ("Decisiones ya tomadas": un único AKS con namespaces, GitOps *pull* con ArgoCD) prevalecen sobre las secciones 8 y 9 de la Entrega 2 (topología de ambientes y acceso del pipeline al cluster). El equipo registra en `docs/cicd.md` qué decisiones de la Entrega 2 cambiaron y por qué.

## 3. Alcance

### Decisiones ya tomadas

- Cluster **AKS** único, con tres ambientes separados por **namespace**: `dev`, `staging` y `prod`.
- **El despliegue en Kubernetes es GitOps de tipo *pull*, con ArgoCD.** El pipeline **nunca ejecuta `kubectl apply`, `helm upgrade` ni equivalentes contra el cluster** (no hay *push* de manifiestos). El pipeline solo publica imágenes y modifica el **estado deseado en Git**; ArgoCD, que corre dentro del cluster, es quien detecta el cambio y lo aplica.
- Registro de imágenes y credenciales resueltos sin exponer secretos en el repositorio.
- Los tests unitarios y funcionales corren en **contenedores Docker**, de modo que se ejecutan igual en la máquina del desarrollador y en el pipeline.

Todo lo demás (herramienta de CI/CD, IaC, empaquetado de manifiestos, estrategia de despliegue, lenguajes, framework de tests) es decisión del equipo y debe justificarse en el README.

### Qué se debe implementar y qué se puede simular

El pipeline debe poder desplegar **todos los servicios comprometidos en la arquitectura de la Entrega 2**. Sin embargo:

- **Al menos dos servicios** (los dos que el README declare como reales, que son los que se evalúan) deben estar **implementados de verdad** (lógica de negocio, persistencia propia si la arquitectura la define, tests reales) y gestionados por el pipeline completo descripto en esta consigna. Los llamamos **servicios reales**.
- **Todos los demás servicios** pueden ser **mocks**: un contenedor que respeta el contrato (rutas, esquemas y códigos de respuesta de la Entrega 2) y devuelve respuestas fijas o simples, y que expone sus sondas de salud. Los mocks **igual se construyen, versionan, empaquetan y despliegan a los tres ambientes mediante el pipeline**; lo único que se simplifica es su lógica interna y sus tests. Donde la Entrega 2 no fijó el contrato de un servicio (por ejemplo, porque solo lo diseñó con un diagrama de secuencia), el equipo lo define en esta entrega y lo registra en `docs/cicd.md`.

Criterios para elegir los dos servicios reales:

- Deben **interactuar entre sí** (una llamada síncrona o un evento), de modo que existan pruebas de integración significativas entre ambos.
- Al menos uno debe **poseer estado persistente** (base de datos propia desplegada o conectada en el cluster).
- Deben permitir verificar, como mínimo, cinco criterios funcionales (ver sección 5.6). Se recomienda que el recorte incluya el servicio que sostiene las cantidades de la colección o el que coordina el intercambio, ya que ahí viven los invariantes más valiosos.
- La elección debe justificarse en el README, y debe ser coherente con la Entrega 2.

`RNF-06` aplica al sistema desplegado, cualquiera sea el recorte. El punto de entrada de `staging` y `prod` valida tokens reales de Google (OAuth 2.0 / OIDC); no se acepta un `userId` por parámetro o header ni un servicio de autenticación mockeado que simule el login. Los servicios reales que toquen identidad cumplen RF-A01 a RF-A05 en lo que les corresponda.

### Excluido de esta entrega

- La implementación completa de los requerimientos funcionales de los servicios mockeados.
- Alta disponibilidad, autoscaling y acuerdos de latencia (fuera de alcance según la sección 6 de `consigna-funcional.md`). No resta puntos no implementarlos; sí resta no mencionarlo.
- Observabilidad avanzada: alcanza con logs accesibles por `kubectl` y las sondas de salud. Quien la implemente suma, pero no es requisito.

## 4. Entregable

Un repositorio (puede ser el mismo del equipo) con, como mínimo, la siguiente estructura. Los nombres exactos de directorios pueden variar si el README los documenta.

```text
README.md                      # punto de entrada: cómo reproducir todo (ver sección 5.8)
services/
  <servicio-real-1>/           # código + Dockerfile + tests unitarios y funcionales
  <servicio-real-2>/
  <servicio-mock-N>/           # contenedor mock + Dockerfile
tests/
  integration/                 # tests de integración que corren contra staging
infra/
  cluster/                     # IaC del cluster AKS, ACR y red/identidades (Terraform, Bicep u otro)
  k8s/                         # manifiestos, Helm charts o Kustomize, con valores por ambiente
gitops/                        # estado deseado por ambiente (puede ser un repo aparte) y
  argocd/                      # Applications/ApplicationSet de ArgoCD (dev, staging, prod)
.github/workflows/ (o equivalente)   # definición de los pipelines (CI y promoción)
docs/
  cicd.md                      # diseño del pipeline (diagrama, etapas, gates, decisiones)
  evidencias/                  # capturas, logs y enlaces de corridas (ver sección 5.9)
```

Requisitos de forma:

- Todo (cluster, ambientes, pipelines, manifiestos) debe estar **versionado como código** en el repositorio. Cambios hechos a mano en el portal de Azure o con `kubectl` que no queden reflejados en el repositorio no cuentan como entrega.
- Redactado en castellano. Los diagramas versionados como texto (Mermaid, PlantUML o ASCII).
- Los nombres de servicios, endpoints y eventos deben coincidir con los de la Entrega 2. Si hubo cambios, registrarlos en `docs/cicd.md`.
- Debe incluir en el encabezado del README los integrantes del equipo, la versión de `consigna-funcional.md` y el commit o tag de `docs/arquitectura.md` sobre los que se trabajó.

## 5. Contenido obligatorio

### 5.1 Cluster AKS y ambientes

- El cluster y sus recursos asociados (grupo de recursos, registro de contenedores ACR, identidades, red) se **definen como código** y se pueden recrear desde cero con un procedimiento documentado, siguiendo las guías de clase.
- Tres namespaces: `dev`, `staging` y `prod`. Cada uno con:
  - **aislamiento** efectivo: configuración, secretos y datos propios; un recurso de un ambiente no puede apuntar a otro por accidente;
  - **ResourceQuota/LimitRange** o `requests/limits` por servicio, con un criterio explicado;
- **ArgoCD** se instala en el cluster (la instalación misma también es código) y gestiona los tres ambientes mediante `Application`/`ApplicationSet`, uno por ambiente, apuntando a la ruta o rama correspondiente del repositorio de estado deseado. El acceso de ArgoCD al repositorio es de solo lectura.
- **El pipeline de CI no tiene credenciales de administración del cluster**: solo necesita permiso de *push* al registro (ACR) y de escritura sobre el repositorio de estado deseado. Se espera identidad federada (OIDC / workload identity), service principal con permisos mínimos o equivalente, y secretos del pipeline en el almacén del sistema de CI o en Key Vault. Los `Secret` del cluster llegan por el mecanismo de la sección 5.7 (Sealed Secrets, SOPS o External Secrets), nunca con `kubectl create secret`. Si el pipeline necesita consultar el estado de ArgoCD (por ejemplo para esperar que `staging` esté *Healthy*), usa un token de solo lectura.
- Las bases de datos y el broker de mensajes (si la arquitectura los define) se despliegan en el cluster o se conectan como servicios gestionados, por ambiente. Los ambientes **no comparten datos**.
- Documentar el **costo y la forma de apagar o destruir** el cluster y los recursos asociados.

### 5.2 Integración continua (CI) de cada servicio real

Por cada servicio real, ante cada push y cada pull request:

1. **Build** reproducible dentro de un contenedor (Dockerfile multi-stage, usuario no root, imagen base fijada por versión y sin `latest`). La imagen que se publica y se promueve (la de `dev`, `staging` y `prod`) **no contiene código, dependencias, herramientas, datos ni endpoints de testing** (frameworks de test, herramientas de cobertura, fixtures, rutas de prueba): se construye desde una etapa final de release que solo copia lo necesario para ejecutar el servicio. Los tests unitarios y funcionales corren en etapas o imágenes de test separadas, que no se publican en el registro de despliegue ni se referencian desde el estado deseado.
2. **Análisis estático**: formato/lint y, al menos, un análisis de seguridad (de dependencias y/o de imagen).
3. **Tests unitarios**, ejecutados en contenedor, con reporte de resultados y de cobertura publicados como artefactos del pipeline. La cobertura de código de los tests unitarios debe ser **mayor al 80 %** (cobertura de líneas, o de líneas y ramas si la herramienta lo permite), y el pipeline debe fallar si no se supera ese umbral. El equipo declara en el README qué mide y qué archivos excluye (por ejemplo, código generado) y justifica cada exclusión.
4. **Tests funcionales** del servicio, ejecutados con **Docker** (por ejemplo `docker compose` o Testcontainers) levantando el servicio junto a sus dependencias reales de infraestructura (base de datos, broker) y, para sus dependencias de otros servicios, usando los mocks. Deben cubrir la API pública del servicio incluyendo errores de negocio e idempotencia. Se mide la cobertura de código del servicio, instrumentado en una imagen de test aparte (nunca la de release), mientras corre la suite funcional; debe ser **mayor al 80 %**, medida con el mismo criterio que la unitaria, y el pipeline falla si no se supera. El reporte se publica como artefacto. Las dos coberturas se miden por separado: la funcional no compensa a la unitaria ni al revés.
5. **Empaquetado**: construcción y publicación de la imagen en el registro, solo si todo lo anterior pasó.

Los servicios mock pasan por una versión reducida de este flujo: build, verificación de que el contenedor arranca y responde sus sondas de salud, y publicación de la imagen.

Los pipelines deben evitar reconstruir y redesplegar lo que no cambió (por ejemplo, filtros por ruta en un monorepo, o un pipeline por servicio).

### 5.3 Artefactos y versionado

- Cada imagen se etiqueta de forma **inmutable y trazable** al commit (SHA o versión semántica generada), y no se usa `latest` para desplegar.
- **Lo que se prueba en `staging` es exactamente lo que se despliega en `prod`**: la misma imagen (idealmente referenciada por *digest*; si se usa un tag, el registro debe impedir reescribirlo) se promueve entre ambientes; **no se reconstruye** para producción. La diferencia entre ambientes está únicamente en la configuración.
- Los manifiestos (Helm, Kustomize u otro) tienen una base común y valores por ambiente. En cada ambiente, la versión desplegada de cada servicio queda **declarada en Git** (tag o digest de la imagen), de modo que el historial de Git es el historial de despliegues.

### 5.4 Continuous Delivery: ambientes, promoción y gates

El flujo mínimo esperado es:

| # | Etapa | Disparador | Qué hace | Gate para avanzar |
|---|---|---|---|---|
| 1 | CI de servicio | Push / pull request | Build, lint, tests unitarios y funcionales en Docker, publicación de imagen | Todos los checks en verde |
| 2 | Promoción a `dev` | Merge a la rama principal | El pipeline **commitea** al repositorio de estado deseado el nuevo tag/digest de las imágenes en el ambiente `dev`. **ArgoCD** detecta el cambio y sincroniza el namespace `dev` | Application de `dev` en *Synced* + *Healthy* + *smoke test* |
| 3 | Promoción a `staging` | Éxito en `dev` | El pipeline commitea las **mismas imágenes** en el ambiente `staging`; ArgoCD sincroniza `staging` | Application de `staging` en *Synced* + *Healthy* |
| 4 | **Tests de integración en `staging`** | Éxito de la sincronización de `staging` | Ejecuta la suite de `tests/integration` contra los servicios desplegados en `staging` | **Todos los tests pasan** |
| 5 | Promoción a `prod` | Gate de `staging` en verde (+ aprobación manual configurada) | El pipeline propone (pull request o commit protegido) el cambio de versión en el ambiente `prod`; tras la aprobación, ArgoCD sincroniza `prod` | Application de `prod` en *Synced* + *Healthy* + *smoke test* |

Requisitos:

- **Pull, no push**: la única vía por la que un cambio llega al cluster es que ArgoCD lo lea desde Git. Debe poder demostrarse que el pipeline no tiene credenciales para modificar recursos del cluster y que un cambio manual con `kubectl` sobre un recurso gestionado es revertido por ArgoCD (*self-heal*) o al menos detectado como *OutOfSync*.
- **Configuración de ArgoCD**: `dev` y `staging` con sincronización automática; `prod` con sincronización controlada (manual o condicionada a la aprobación del cambio en Git). Se recomienda habilitar `prune` y `selfHeal` donde corresponda y justificar donde no.
- **Si el gate de integración en `staging` falla, `prod` no se toca**: no se genera ningún cambio en el estado deseado de `prod`, y esa Application permanece en el estado anterior. Esto debe poder demostrarse con evidencia de una corrida que falle (ver 5.9).
- La promoción a `prod` requiere una **aprobación manual explícita** (revisión/aprobación del pull request, environment protection o sincronización manual en ArgoCD). Sin aprobación manual, el criterio de CD de la sección 7 pierde ese ítem, aun si se documenta una justificación.
- Estrategia de despliegue en Kubernetes (rolling update, blue-green, canary u otra) elegida y justificada, con *readiness* y *liveness probes* que verifiquen algo real del servicio (una sonda que siempre responde OK es un error).
- **Rollback**: documentado y probado al menos una vez, realizado **a través de Git** (`git revert` del cambio de versión en el repositorio de estado deseado, o fijar la versión anterior) para que ArgoCD converja a ella. Se debe poder mostrar el commit, la sincronización de ArgoCD y el resultado. Un rollback hecho con `kubectl`/`helm` directamente contra el cluster no cuenta.
- Los **mocks** también se promueven por el mismo flujo (y son gestionados por ArgoCD), de modo que los tres namespaces contienen siempre el conjunto completo de servicios de la arquitectura.
- Si la arquitectura incluye bases de datos con esquema, el pipeline explica cómo se aplican las **migraciones** y en qué orden respecto del despliegue del servicio (por ejemplo, expand/contract).

### 5.5 Testing sobre contenedores

Los distintos niveles de test corren en estos lugares:

| Nivel | Dónde corre | Contra qué |
|---|---|---|
| Unitarios | Contenedor del pipeline | El código del servicio, sin infraestructura |
| Funcionales | Docker en el runner (compose/Testcontainers) | El servicio real + sus dependencias de infraestructura reales + mocks de otros servicios |
| Integración | Contra el namespace `staging` en AKS | Los servicios realmente desplegados (reales y mocks) a través de su punto de entrada |

Los tests unitarios y funcionales deben poder correrse **localmente con un único comando** documentado en el README (por ejemplo `make test` o `docker compose run tests`).

### 5.6 Tests de integración (mínimo cinco)

Escribir una suite de integración, **automatizada y ejecutada por el pipeline en `staging`**, que verifique **al menos cinco** criterios funcionales importantes de `consigna-funcional.md`. Cada test:

- Corresponde a un requerimiento (RF) y/o escenario de aceptación (sección 7) identificado explícitamente en su nombre o documentación.
- Parte de **datos de prueba conocidos** y se puede repetir sin dejar el ambiente en un estado que haga fallar la siguiente corrida (datos de prueba sembrados o limpiados por el propio test o por el pipeline).
- Verifica el **resultado de negocio observable** (estado, cantidades, códigos de respuesta), no solo que el servicio "respondió 200".
- Afirma un resultado de negocio **producido por los dos servicios reales**; puede atravesar mocks, pero la aserción no puede depender de la lógica ni del estado de un mock. Un test que ejercita un solo servicio real no cuenta para el mínimo de cinco.

El **Anexo A** contiene el catálogo completo de tests (IT-01 a IT-39), derivado de los requerimientos funcionales y de los 16 escenarios de aceptación. En esta entrega el equipo elige de allí los que su recorte permite verificar (mínimo cinco), o agrega los propios; **en la entrega final debe estar cubierto cada escenario de aceptación** (ver Anexo A). Ejemplos de criterios aptos:

- Apertura de sobre propio y disponible acredita las copias esperadas (escenario 1).
- Reintento de apertura del mismo sobre devuelve el mismo contenido sin duplicar copias (RF-07, escenario 2).
- Intento de abrir un sobre ajeno o ya abierto se rechaza y no acredita nada (RF-08, escenario 13).
- Oferta inválida (copia inexistente, ajena, no ofrecible) no se publica (RF-11, escenario 14).
- Intercambio 1:1 exitoso transfiere ambas copias una sola vez (escenario 3).
- Dos aceptaciones concurrentes sobre la misma oferta: solo una se completa y no hay duplicación (RF-17, escenario 11).
- Solicitud sin sesión válida se rechaza sin efectos de negocio (RF-A05, escenario 16).
- Invariantes de cantidades: tras una secuencia de operaciones con reintentos y duplicados no hay negativos ni copias perdidas (RF-39).

Se espera que al menos **uno de los cinco** cubra un caso de **fallo o concurrencia** (no solo el camino feliz).

Los resultados de la suite deben quedar publicados como artefacto del pipeline (por ejemplo, reporte JUnit/HTML).

### 5.7 Configuración y secretos

- La configuración se separa del código y se inyecta por ambiente (ConfigMaps, valores del chart declarados en Git).
- Los secretos (cadenas de conexión, credenciales OAuth de Google, claves del registro) **nunca** están en el repositorio en texto plano ni en las imágenes. Como ArgoCD solo lee de Git, el equipo debe resolver cómo llegan al cluster sin exponerlos (por ejemplo, Sealed Secrets, SOPS, External Secrets Operator con Azure Key Vault) y explicar su elección.
- Debe poder demostrarse que `dev`, `staging` y `prod` usan configuración y datos distintos.

### 5.8 README de reproducción

El README debe permitir a un tercero, sin ayuda del equipo:

1. Crear (o conectarse a) el cluster y los ambientes a partir del código de `infra/`, e instalar ArgoCD y sus Applications.
2. Configurar las credenciales y variables necesarias del pipeline.
3. Disparar el pipeline con un cambio y seguirlo hasta producción, incluyendo la sincronización en ArgoCD (cómo acceder a su UI/CLI).
4. Ejecutar localmente los tests unitarios y funcionales.
5. Verificar el estado de cada ambiente (`kubectl get pods -n dev|staging|prod`, estado de las Applications en ArgoCD, URLs o puntos de entrada de cada uno).
6. Ejecutar un rollback vía Git.
7. Destruir los recursos de Azure para no incurrir en costos.

Además debe incluir una tabla con **todos los servicios de la arquitectura**, indicando cuáles son reales y cuáles mocks, y su estado de despliegue en cada ambiente.

### 5.9 Evidencia

En `docs/evidencias/` (o en el README, con enlaces estables) incluir:

- Una corrida **completa y exitosa** del pipeline, de commit a producción.
- Una corrida donde **falla un check de CI** (unitario o funcional) y no se publica imagen.
- Una corrida donde **fallan los tests de integración en `staging`** y se demuestra que `prod` no cambió.
- La ejecución de un **rollback vía Git**, con el commit y la reconciliación de ArgoCD.
- Captura de ArgoCD con las tres Applications (`dev`, `staging`, `prod`) en *Synced* y *Healthy*, y la salida de `kubectl get pods,svc -n <ns>` para cada ambiente, con todos los servicios de la arquitectura presentes.
- Evidencia de **deriva corregida o detectada**: un cambio manual en el cluster revertido por *self-heal* o marcado *OutOfSync*.

### 5.10 Documento de diseño del CI/CD

`docs/cicd.md` (entre 3 y 6 páginas equivalentes) con: diagrama del pipeline, tabla de etapas y gates, estrategia de versionado de imágenes, estructura del repositorio de estado deseado y flujo de promoción vía Git, configuración de ArgoCD (política de sync por ambiente), estrategia de despliegue y rollback, manejo de secretos, justificación de la elección de los dos servicios reales y las decisiones de herramientas, y cambios respecto de la Entrega 2 (si los hubo).

## 6. Reglas de calidad

- **Todo es código**: cluster, pipelines, manifiestos y tests están versionados; nada se despliega a mano.
- **Git es la fuente de verdad del cluster**: el pipeline no accede al cluster para desplegar; ArgoCD hace *pull* del estado deseado.
- **Una imagen, muchos ambientes**: nunca se reconstruye para promover.
- **Sin testing en producción**: ninguna imagen desplegada contiene código o dependencias de testing; la imagen de `prod` es la misma que se probó en `staging`, y por eso el código de test tampoco está en `staging`.
- **El pipeline falla de verdad**: un gate que no puede fallar no es un gate.
- **Sin secretos en el repositorio**, en el historial de git ni en las imágenes.
- **Sondas con sentido**: liveness y readiness verifican algo real.
- **Reproducibilidad**: los tests unitarios y funcionales corren igual localmente y en CI.
- **Trazabilidad**: cada test de integración se rastrea a un RF o escenario de la consigna funcional.
- **Coherencia con la arquitectura**: los servicios desplegados son los de la Entrega 2, con sus mismos nombres y contratos.
- Nada de boilerplate: si una práctica pedida no aplica a tu solución, explicar por qué en lugar de simularla.

## 7. Criterios de corrección

| Criterio | Qué se mira | Peso |
|---|---|---|
| Cluster AKS y ambientes | IaC reproducible, tres namespaces aislados, ArgoCD instalado y configurado por ambiente, acceso seguro del pipeline, recursos y políticas por ambiente | 20% |
| CI y testing de los servicios reales | Build en contenedor, análisis estático, tests unitarios y funcionales en Docker, cobertura unitaria y funcional mayor al 80 % con gate, imágenes publicadas y versionadas | 20% |
| Pipeline de CD y gates (GitOps) | Despliegue por *pull* con ArgoCD (sin `kubectl`/`helm` desde el pipeline), promoción `dev → staging → prod` con la misma imagen vía commits al estado deseado, gate de integración que realmente detiene la promoción, aprobación a prod, rollback vía Git demostrado | 20% |
| Tests de integración | Al menos cinco, trazables a RF/escenarios, con verificación de negocio, repetibles, incluyendo un caso de fallo/concurrencia, ejecutados por el pipeline en `staging` | 20% |
| Completitud del despliegue | Todos los servicios de la arquitectura desplegados en los tres ambientes (reales y mocks conformes al contrato); configuración y secretos por ambiente | 10% |
| Documentación y reproducibilidad | README ejecutable por un tercero, `docs/cicd.md`, evidencias de corridas exitosas y fallidas | 10% |

Restan puntos: que el pipeline despliegue haciendo *push* al cluster (`kubectl apply`, `helm upgrade` desde el CI), desplegar con `latest`, reconstruir imágenes para producción, desplegar imágenes con código o dependencias de testing, secretos en el repositorio, gates que no bloquean, sondas triviales, mocks que contradicen una ruta, un esquema o un código de respuesta que la Entrega 2 sí definió, namespaces que comparten datos, cambios manuales en el cluster no reflejados en el código, e implementar más de dos servicios "reales": no suma puntos y resta si debilita el pipeline de los dos declarados.

Esta tabla es la nota de la Entrega 3; `criterios-evaluacion.md` (fronteras, dominio, coordinación) no se aplica a esta entrega.

Una entrega en la que **no se pueda reproducir un despliegue en AKS vía pipeline, con promoción `staging` → `prod`, no se evalúa** (RNF-04 y RNF-05 de `consigna-funcional.md`). La ausencia de gate de integración, el incumplimiento de RNF-06 u otros defectos no vuelven la entrega no evaluable en esta entrega intermedia: se reflejan en los criterios de la sección 7. En la entrega final rigen RNF-01 a RNF-06 de `consigna-funcional.md`, sección 8.

## 8. Formato y modalidad de entrega

- Todo el trabajo se entrega versionado en el repositorio del equipo, con el pipeline en estado ejecutable al momento de la entrega y el cluster **desplegado y accesible durante la defensa**.
- La entrega es **grupal**; se espera que todos los integrantes hayan participado en el diseño del pipeline y comprendan cada etapa.
- Fecha límite, canal de entrega, modelo de acceso a Azure (suscripción o créditos, región y tope de gasto) y ventana de defensa: según lo indicado por la cátedra en el campus. El cluster debe permanecer encendido y accesible para la cátedra desde la entrega hasta el fin de la defensa; el equipo lo destruye después y adjunta la evidencia (sección 5.1).

## 9. Defensa

El equipo debe poder, sobre su propio repositorio y el cluster desplegado:

- Realizar un cambio en uno de los servicios reales, hacer push y mostrar cómo el pipeline lo lleva hasta producción a través de Git y ArgoCD.
- Provocar un fallo (un test roto o un test de integración que falla) y mostrar que la promoción se detiene y producción permanece intacta.
- Ejecutar un rollback vía Git y mostrar la reconciliación en ArgoCD.
- Justificar la elección de los dos servicios reales, la estrategia de despliegue, el manejo de secretos y las decisiones de aislamiento entre ambientes.

---

## Anexo A. Catálogo de tests de integración end to end

Este catálogo deriva de los requerimientos (RF/RNF) y de los 16 escenarios de aceptación de `consigna-funcional.md`. Cada test es un caso **end to end** a través del punto de entrada del sistema desplegado en `staging`, con usuarios de prueba autenticados.

Cómo usarlo:

- **Alcance por entrega.** En **esta entrega** el equipo elige **como mínimo cinco** tests (sección 5.6) según los dos servicios reales que implementó, e implementa solo los que su recorte puede satisfacer. No cuenta un test cuya aserción de negocio la produce un servicio mockeado, porque no verifica lógica de negocio real.
- **En la entrega final del trabajo, todos los escenarios de aceptación de `consigna-funcional.md` (sección 7) deben poder demostrarse y estar cubiertos por tests del pipeline**, ya que en ese momento todos los servicios deben ser reales y toda la funcionalidad debe ser demostrable (sección 9, punto 6, de la consigna funcional). El catálogo IT-01 a IT-39 es la referencia esperada de esa cobertura (los IT de A.8 son *smoke tests*, no escenarios). Un IT de escenario que no está implementado o no pasa equivale a un escenario de aceptación sin cubrir. Por eso conviene que, desde ahora, el equipo elija, nombre y estructure su suite pensando en completarla: los tests elegidos hoy son la base de la suite final, y los que no se implementen en esta entrega quedan como deuda explícita que el equipo debe registrar (por ejemplo, en `docs/cicd.md`).
- Los tests de A.8 (despliegue y salud) se ejecutan siempre, también en esta entrega.
- Los tests marcados con **(F)** cubren fallo, concurrencia o duplicación; al menos uno de los elegidos debe ser de este tipo.
- Los IDs (`IT-xx`) se usan en el nombre del test y en el reporte del pipeline. El equipo puede agregar tests propios fuera del catálogo, justificando el RF o escenario que cubren.
- Cada test declara sus datos de prueba (usuarios, sobres, copias, ofertas) y los siembra o limpia por sí mismo, de forma que pueda repetirse.
- Autenticación: los tests usan cuentas de Google de prueba del equipo; los tokens se obtienen con las credenciales OAuth reales y se inyectan como secretos del pipeline. No se acepta un `userId` por parámetro o header (RNF-06). El pipeline no ejecuta el flujo interactivo de login: el equipo obtiene una sola vez, en un navegador, un *refresh token* por cuenta de prueba (A, B, C, Op), lo guarda como secreto de CI y el job solo hace el *refresh grant*. Un token inválido es una cadena arbitraria; uno expirado es un *access token* real ya vencido, guardado como secreto. El README documenta el procedimiento y la rotación.
- Control del escenario sin credenciales de cluster: los tests que requieren inyectar fallas (IT-24, IT-28), pausar una operación a mitad de camino (IT-20, IT-25, IT-35) o sembrar un estado inconsistente (IT-36) **no usan credenciales de administración del cluster desde CI** (sección 5.1). Usan un mecanismo versionado y acotado al namespace `staging`, **sin agregar código ni endpoints de testing a la imagen del servicio** (la imagen de `staging` es la de `prod`): por ejemplo, un mock cuyo paso de confirmación el test puede retener y liberar, un proxy de fallas desplegado por ArgoCD, un Job de ArgoCD con permiso solo para borrar pods de ese namespace, o un Job que siembra datos directamente en la base de `staging` con una credencial de test de ese ambiente. El equipo documenta el mecanismo y su limpieza en `docs/cicd.md`.

Notación: **A**, **B** son coleccionistas de prueba; **Op** es el operador.

### A.1 Autenticación y acceso (RF-A01 a RF-A05)

| ID | Req. / escenario | Dado | Cuando | Entonces |
|---|---|---|---|---|
| IT-01 | RF-A05, esc. 16 (F) | Sin token, con token inválido y con token expirado | Se invoca una operación de negocio (abrir sobre, publicar oferta, consultar colección) | Todas se rechazan como no autenticadas y no se ejecuta ningún efecto de negocio (cantidades y sobres sin cambios) |
| IT-02 | RF-A01, RF-A02, RF-05 | Cuenta de Google de prueba nunca vista | Primer acceso autenticado | Se crea el álbum automáticamente y se acreditan los sobres iniciales; un segundo acceso no vuelve a acreditarlos |
| IT-03 | RF-A03 | A autenticado, B con sobre y oferta propios | A intenta abrir el sobre de B, publicar una oferta con la copia de B y cancelar la oferta de B | Las tres operaciones se rechazan; el estado de B no cambia |
| IT-04 | RF-A04, RF-33, RF-36 | A (coleccionista) y Op (operador con rol asignado) | A invoca la vista de inspección; Op la invoca; Op intenta modificar la colección de A | A es rechazado; Op puede inspeccionar; la modificación se rechaza y la colección de A no cambia |

### A.2 Colección y álbum (RF-01 a RF-04, RF-30)

| ID | Req. / escenario | Dado | Cuando | Entonces |
|---|---|---|---|---|
| IT-05 | RF-01, RF-02, RF-03 | A con una colección sembrada conocida (algunas figuritas repetidas, otras faltantes) | A consulta su colección, su porcentaje y sus faltantes | Por figurita: poseídas, repetidas y disponibles coinciden con lo sembrado; el porcentaje y los faltantes son coherentes con la colección |
| IT-06 | RF-04, esc. 10 | A con colección conocida y un sobre cerrado | A abre el sobre y consulta inmediatamente la colección | Las copias nuevas se ven al instante y las cantidades cumplen `poseídas = disponibles + apartadas`; el porcentaje puede actualizarse con demora y converge |
| IT-07 | RF-30, RF-31 | A con actividad previa (apertura, intercambio) | A consulta su resumen | Muestra porcentaje, cantidad de figuritas, repetidas, faltantes y resultado de operaciones recientes; incluye además retos activos, ofertas compatibles, actividad reciente y la posición en ambos rankings |

### A.3 Sobres (RF-05 a RF-09)

| ID | Req. / escenario | Dado | Cuando | Entonces |
|---|---|---|---|---|
| IT-08 | RF-06, esc. 1 | A con un sobre propio cerrado | A abre el sobre | Se devuelve el contenido revelado y las copias se acreditan en la colección una sola vez; el sobre pasa a abierto |
| IT-09 | RF-07, esc. 2 (F) | A abre un sobre y repite la misma solicitud con la misma clave de idempotencia (y también sin clave, reintento posterior) | Se envía la apertura 3 veces, incluso en paralelo | Todas devuelven el mismo contenido; se acreditan copias una sola vez; no se vuelve a abrir el sobre |
| IT-10 | RF-08, esc. 13 (F) | Un sobre de B y un sobre de A ya abierto | A intenta abrir el de B y el ya abierto | Ambos se rechazan; no se acredita nada |
| IT-11 | RF-22, esc. 7 (F) | A envía una apertura y se pierde la respuesta (el test descarta la respuesta) | A consulta el resultado de la operación | Obtiene el resultado real sin que se acrediten copias adicionales |

### A.4 Ofertas (RF-10 a RF-13, RF-18, RF-19, RF-23, RF-24)

| ID | Req. / escenario | Dado | Cuando | Entonces |
|---|---|---|---|---|
| IT-12 | RF-10, RF-13 | A con una copia ofrecible | A publica una oferta 1:1; B explora las ofertas | La oferta aparece activa para B; las cantidades de A no cambian (la copia no se aparta por publicar) |
| IT-13 | RF-11, esc. 14 (F) | A con: copia inexistente, copia ajena, última copia (no repetida) y copia ya apartada | A intenta publicar una oferta con cada una | Todas se rechazan y ninguna oferta se publica |
| IT-14 | RF-11 (segunda oración), RF-12 (precondición del esc. 4) | A con una copia ofrecible | A publica tres ofertas respaldadas por la misma copia | Las tres se publican y coexisten; las cantidades no cambian |
| IT-15 | RF-18 | A con una oferta activa sin aceptaciones | A la cancela | La oferta queda cancelada, no puede aceptarse y la colección de A no cambia |
| IT-16 | RF-19, RF-37 | A con una oferta creada con vencimiento de pocos segundos | Pasa la fecha límite sin intervención | La oferta deja de poder aceptarse; la colección de A no cambia |
| IT-17 | RF-24 | A y B con ofertas cruzadas; B posee lo que pide la oferta de A | B consulta las ofertas | La oferta de A aparece destacada como compatible con B |
| IT-18 | RF-23 | A y B con copias para un intercambio N:M | A publica una oferta N:M y B la acepta | Se transfieren todas las copias involucradas una sola vez o ninguna |

### A.5 Intercambios (RF-12, RF-14 a RF-17, RF-20, RF-21, RF-39)

| ID | Req. / escenario | Dado | Cuando | Entonces |
|---|---|---|---|---|
| IT-19 | RF-14, RF-21, esc. 3 | A publica una oferta 1:1; B posee la copia solicitada en condición ofrecible | B acepta | Ambas copias se transfieren una sola vez; ambas partes reciben su copia y el resultado; si ya la tenían, la nueva copia queda como repetida y disponible |
| IT-20 | RF-12, RF-14 | Intercambio en curso (aceptación iniciada) | Se consulta la colección de A y B durante el intento | Las copias involucradas figuran como apartadas y no pueden ser usadas por otra operación; `poseídas = disponibles + apartadas` |
| IT-21 | RF-14 | B no posee la copia solicitada, o solo tiene la última | B intenta aceptar | Se rechaza; no se aparta ni transfiere nada |
| IT-22 | RF-17, esc. 11 (F) | Una oferta activa | B y C aceptan simultáneamente (peticiones paralelas) | Solo una aceptación completa el intercambio; la otra se rechaza y libera toda reserva que hizo; no hay copias duplicadas |
| IT-23 | esc. 4 (F) | Dos ofertas distintas respaldadas por la misma copia de A | B y C aceptan al mismo tiempo, una aceptación concurrente sobre cada oferta | Solo una logra apartar la copia y completar; la otra se rechaza sin pérdida, duplicación ni reservas indefinidas |
| IT-24 | RF-16, esc. 5 y 6 (F) | Intercambio en curso con una o ambas copias apartadas | Se fuerza una falla del servicio (por ejemplo, se elimina un pod o se corta una dependencia) antes de confirmar | El intercambio no se confirma o se completa exactamente una vez; toda reserva se libera; no queda ninguna copia apartada indefinidamente |
| IT-25 | RF-15, RF-20, esc. 12 (F) | Oferta con aceptación en curso | Coinciden una cancelación o un vencimiento con la aceptación | El resultado es único: o se completa el intercambio, o la oferta queda cerrada sin intercambio y se liberan las reservas |
| IT-26 | RF-22, esc. 7 (F) | B acepta una oferta y no recibe la respuesta, pero la operación se completó | B consulta el resultado de la operación | Obtiene el resultado real; las copias no se transfieren de nuevo |
| IT-27 | RF-39, esc. 8 (F) | Un intercambio completado | Se reenvía el mismo evento/solicitud de aceptación (duplicado) | No se re-completa el intercambio ni se crean o liberan reservas adicionales |
| IT-28 | RF-39, RF-04 (F) | Secuencia de aperturas, ofertas, aceptaciones, cancelaciones, reintentos y duplicados con varios usuarios | Se ejecuta la secuencia (con fallas inyectadas) y se verifican los invariantes | Sin cantidades negativas, sin copias perdidas ni duplicadas; la suma de copias del sistema se conserva |

### A.6 Retos, recompensas, códigos promocionales y rankings (RF-09, RF-25 a RF-32, RF-38)

| ID | Req. / escenario | Dado | Cuando | Entonces |
|---|---|---|---|---|
| IT-29 | RF-25, RF-26, RF-38 | Un reto activo (por ejemplo, abrir cinco sobres) | A completa la acción requerida | Con demora aceptable, el reto se completa, A recibe la recompensa y es notificado |
| IT-30 | RF-27, esc. 9 (y esc. 8 en su variante de evento duplicado) (F) | Un reto en progreso | El mismo evento de acción se informa duplicado y fuera de orden | El progreso se incrementa una sola vez; no se aplica un progreso imposible ni se corrompen cantidades |
| IT-31 | RF-28, RF-09 | Un código promocional vigente con uso disponible | A lo canjea | Recibe lo prometido (sobres o recompensa) una sola vez; un segundo canje del mismo código por A respeta el límite por usuario |
| IT-32 | RF-28 | Códigos vencido, fuera de campaña o con restricción no cumplida | A intenta canjear cada uno | Todos se rechazan sin acreditar nada |
| IT-33 | RF-29, esc. 15 (F) | Un código con un único uso disponible | A y B lo canjean simultáneamente | Solo uno es aceptado; el otro se rechaza |
| IT-34 | RF-32 (y la posición de RF-31) | A y B con distinto porcentaje y cantidad de intercambios completados | Se consultan ambos rankings | Cada uno se ordena de mayor a menor con desempate determinístico; la posición puede actualizarse con demora y converge |

### A.7 Operador (RF-33 a RF-36)

| ID | Req. / escenario | Dado | Cuando | Entonces |
|---|---|---|---|---|
| IT-35 | RF-33, RF-34 | Ofertas activas y un intercambio en curso con copias apartadas | Op consulta ofertas activas y copias apartadas | Ve las ofertas y las copias apartadas, cada una asociada a su operación en curso |
| IT-36 | RF-35 | Un estado inconsistente sembrado por un fixture de test (por ejemplo, una reserva sin operación en curso), aplicado solo en `staging` mediante un Job versionado que escribe en la base del servicio dueño de la reserva con una credencial de test de ese ambiente (no un endpoint de test dentro de la imagen del servicio), con identificador reconocible (`it-36-*`) y limpieza al terminar; no se usa la API del operador para crearlo | Op consulta las operaciones incompletas o inconsistentes | La inconsistencia queda identificada |

### A.8 Despliegue y salud (smoke tests de esta consigna)

Estos tests no cubren funcionalidad de negocio ni se corresponden uno a uno con RNF-01 a RNF-05, que se evalúan con la evidencia de la sección 5.9. Verifican que el despliegue en el ambiente es operable. El pipeline el pipeline debe ejecutarlos como *smoke tests* en cada ambiente. No cuentan para el mínimo de cinco de la sección 5.6.

| ID | Req. | Entonces |
|---|---|---|
| IT-37 | Esta consigna, 5.4 y 6 (sondas con sentido) | Todos los servicios de la arquitectura (reales y mocks) responden sus sondas de salud en el ambiente, a través del punto de entrada o de endpoints de salud expuestos (el pipeline no accede al cluster) |
| IT-38 | Esta consigna, 5.1 y 5.5 (punto de entrada) | El punto de entrada del ambiente responde y enruta a cada servicio esperado |
| IT-39 | Esta consigna, 5.4 | La versión (tag/digest) de cada imagen desplegada en el ambiente coincide con la declarada en Git |

### A.9 Recomendación de recorte (solo para esta entrega)

Como en la entrega final se implementan todos los tests, esta tabla solo orienta qué tests priorizar ahora según los dos servicios reales elegidos; el resto queda pendiente para la entrega final. Un test sugerido cuenta para el mínimo de cinco **solo si cumple la sección 5.6 con el recorte elegido** (la aserción de negocio la producen los dos servicios reales):

| Servicios reales | Tests de integración habilitados (sugeridos) |
|---|---|
| Colección/sobres + autenticación | IT-01, IT-02, IT-05, IT-06, IT-08, IT-09, IT-10, IT-11 |
| Colección/sobres + ofertas/intercambio | IT-12, IT-13, IT-14, IT-19, IT-20, IT-21, IT-22, IT-23, IT-24, IT-28 (los IT de sobres cuentan solo si abren sobres a través de ofertas, por ejemplo como precondición de IT-12) |
| Ofertas/intercambio + coordinación de intercambio (saga) | IT-19 a IT-27, si la coordinación y las cantidades de las copias son de los servicios reales elegidos |
| Códigos promocionales + retos/recompensas | IT-29, IT-31 y IT-33 si el reto y la recompensa los resuelven esos dos servicios; IT-30 y IT-32 solo si el segundo servicio real participa de la aserción |

---

## Changelog

### 1.1

- RNF-06 aplica al sistema desplegado cualquiera sea el recorte; se documenta el origen de los tokens de prueba (sección 3 y Anexo A).
- La regla de conteo de los cinco tests de integración exige que la aserción de negocio la produzcan los dos servicios reales; se rehízo la tabla A.9.
- Se aclara qué prevalece sobre la Entrega 2 (topología, GitOps), qué hacer donde el contrato de la Entrega 2 no existe y qué hace que una entrega no se evalúe.
- Se define cómo se inyectan fallas, se pausan operaciones y se siembran estados inconsistentes sin credenciales de cluster en el pipeline (Anexo A).
- Correcciones de referencias del catálogo (IT-02, IT-07, IT-14, IT-23, IT-25, IT-30, IT-34, IT-37, IT-38, A.8) y de la cobertura exigida en la entrega final.
- Las imágenes desplegadas (`dev`, `staging`, `prod`) no pueden incluir código ni dependencias de testing; los mecanismos de control de IT-24, IT-28, IT-20, IT-25, IT-35 e IT-36 no agregan endpoints de test al servicio.
- Se fija un umbral de cobertura mayor al 80 % para los tests unitarios y para los funcionales, medidos por separado y con gate en el pipeline.
- Se unifica "dos servicios reales", la aprobación a `prod` y los secretos del cluster; se agrega el modelo de acceso a Azure y la ventana de defensa.
