# Tercera entrega TP1: CI/CD y despliegue en AKS del Álbum

## Propósito y fuentes

La tercera entrega es **ejecutable**: pide un cluster AKS funcionando y un proceso de CI/CD que lleve los cambios de código hasta producción, sobre la arquitectura documentada en la [segunda entrega](segunda-entrega-album-arquitectura.md). El equipo tiene que demostrar que un cambio en un servicio recorre **commit → build → tests → imagen → dev → staging → prod** sin intervención manual, y que ese camino se detiene cuando algo falla. [E06]

Rige la versión 1.1 de la consigna (`E06`). La versión anterior no está incorporada; su changelog se resume en [Versiones](#versiones-de-la-consigna). [E06]

Los insumos son: [E06]

1. `consigna-funcional.md` v1.1 ([`E01`](entrega-album-2026.md)): requerimientos, los 16 escenarios de aceptación de la sección 7 y RNF-01 a RNF-06. [E01, p. 6]
2. `docs/arquitectura.md` y `docs/adr/` de la segunda entrega, con las correcciones de la devolución. Son la fuente de verdad de **qué servicios existen**, cómo se comunican y qué contratos tienen. [E06] Las correcciones del grupo y su efecto en esta entrega están en [Devolución de la segunda entrega](devolucion-segunda-entrega.md). [D02]
3. Las guías de AKS, contenedores y CI/CD de las clases. Ver [Kubernetes en AKS](kubernetes-en-aks.md), [Imágenes](imagenes-de-contenedores.md) y [DevOps y CI/CD](devops-y-ci-cd.md). [G09] [G02] [G03]

**Ambientes.** `E01` habla de *staging* y *producción*; esta entrega usa tres: `dev`, `staging` y `prod`. `staging` es el ambiente de RNF-02 y RNF-03 (despliegue previo y gate de integración), `dev` es un ambiente previo adicional y `prod` es la producción de `E01`. [E06] [E01, p. 6]

**Qué prevalece.** Las decisiones ya tomadas de esta entrega (un único AKS con namespaces y GitOps *pull* con ArgoCD) prevalecen sobre las secciones 8 y 9 de la segunda entrega, que trataban la topología de ambientes y el acceso del pipeline al cluster. El equipo registra en `docs/cicd.md` qué cambió y por qué. [E06] Con esto queda confirmado que el TP pasa de Docker Compose a AKS, como se había inferido. Ver [Dudas y conflictos](../dudas-y-conflictos.md).

## Alcance

### Decisiones ya tomadas

- Un **único cluster AKS**, con tres ambientes separados por **namespace**: `dev`, `staging` y `prod`.
- **Despliegue GitOps de tipo pull con ArgoCD.** El pipeline **nunca** ejecuta `kubectl apply`, `helm upgrade` ni equivalentes contra el cluster: solo publica imágenes y modifica el estado deseado en Git. ArgoCD, dentro del cluster, detecta el cambio y lo aplica.
- Registro de imágenes y credenciales resueltos sin exponer secretos en el repositorio.
- Los tests unitarios y funcionales corren en **contenedores Docker**, igual en la máquina del desarrollador y en el pipeline.

Fuente: [E06]. Todo lo demás (herramienta de CI/CD, IaC, empaquetado de manifiestos, estrategia de despliegue, lenguajes, framework de tests) es decisión del equipo y se justifica en el README. [E06] Ver [GitOps y entrega continua](gitops-y-entrega-continua.md).

### Servicios reales y mocks

El pipeline despliega **todos** los servicios de la arquitectura de la segunda entrega, pero solo dos tienen que estar implementados de verdad: [E06]

| | Servicios reales (exactamente dos) | Mocks (todos los demás) |
|---|---|---|
| Implementación | Lógica de negocio, persistencia propia si la arquitectura la define y tests reales | Contenedor que respeta el contrato (rutas, esquemas y códigos de la segunda entrega), devuelve respuestas fijas o simples y expone sondas de salud |
| Pipeline | CI completo de la sección 5.2 | Build, verificación de arranque y sondas, publicación de la imagen |
| Despliegue | Los tres ambientes, vía ArgoCD | Igual: se versionan, empaquetan y despliegan a los tres ambientes |

Si la segunda entrega no fijó el contrato de un servicio (por ejemplo, solo lo diseñó con un diagrama de secuencia), el equipo lo define ahora y lo registra en `docs/cicd.md`. [E06]

Criterios para elegir los dos servicios reales: [E06]

- **interactúan entre sí** (una llamada síncrona o un evento), para que haya pruebas de integración significativas;
- al menos uno **tiene estado persistente**, con base propia desplegada o conectada en el cluster;
- permiten verificar al menos cinco criterios funcionales; se recomienda incluir el servicio que sostiene las cantidades de la colección o el que coordina el intercambio, porque ahí viven los invariantes más valiosos;
- la elección se justifica en el README y es coherente con la segunda entrega.

Implementar más de dos servicios reales no suma, y resta si debilita el pipeline de los dos declarados. [E06]

**Autenticación.** RNF-06 aplica al sistema desplegado, sea cual sea el recorte: el punto de entrada de `staging` y `prod` valida tokens reales de Google (OAuth 2.0/OIDC). No se acepta un `userId` por parámetro o header, ni un servicio de autenticación mockeado que simule el login. Los servicios reales que toquen identidad cumplen RF-A01 a RF-A05 en lo que les corresponda. [E06] [E01, p. 6]

### Excluido

- La implementación completa de los servicios mockeados.
- Alta disponibilidad, autoscaling y acuerdos de latencia, fuera de alcance según la sección 6 de `E01`. No implementarlos no resta puntos; **no mencionarlo, sí**. [E06] [E01, p. 5]
- Observabilidad avanzada: alcanzan los logs accesibles por `kubectl` y las sondas de salud; implementarla suma, pero no es requisito. [E06]

## Entregable y forma

```text
README.md                  # cómo reproducir todo
services/<real-1>/         # código, Dockerfile, tests unitarios y funcionales
services/<real-2>/
services/<mock-N>/         # mock + Dockerfile
tests/integration/         # tests que corren contra staging
infra/cluster/             # IaC de AKS, ACR, red e identidades (Terraform, Bicep u otro)
infra/k8s/                 # manifiestos, Helm o Kustomize, con valores por ambiente
gitops/                    # estado deseado por ambiente (puede ser un repo aparte)
gitops/argocd/             # Applications/ApplicationSet (dev, staging, prod)
.github/workflows/         # o equivalente: pipelines de CI y promoción
docs/cicd.md               # diseño del pipeline
docs/evidencias/           # capturas, logs y enlaces de corridas
```

Fuente: [E06]. Los nombres de directorios pueden variar si el README los documenta. [E06] Requisitos de forma: [E06]

- **todo versionado como código**: lo hecho a mano en el portal de Azure o con `kubectl` que no quede en el repositorio no cuenta;
- castellano y diagramas como texto (Mermaid, PlantUML o ASCII);
- nombres de servicios, endpoints y eventos iguales a los de la segunda entrega, o con los cambios registrados en `docs/cicd.md`;
- en el encabezado del README: integrantes, versión de `E01` y commit o tag de `docs/arquitectura.md`.

## Contenido obligatorio

### 5.1 Cluster y ambientes

- Cluster, grupo de recursos, ACR, identidades y red **definidos como código** y recreables desde cero con un procedimiento documentado, siguiendo las guías. [E06]
- Tres namespaces con **aislamiento efectivo** (configuración, secretos y datos propios; un recurso de un ambiente no puede apuntar a otro por accidente) y con **ResourceQuota/LimitRange** o `requests/limits` por servicio, con un criterio explicado. [E06]
- **ArgoCD** instalado en el cluster (la instalación también es código), con una `Application` o `ApplicationSet` por ambiente apuntando a la ruta o rama correspondiente. Su acceso al repositorio es **de solo lectura**. [E06]
- **El pipeline no tiene credenciales de administración del cluster**: solo permiso de push al ACR y de escritura en el repositorio de estado deseado. Se espera identidad federada (OIDC o workload identity), un service principal con permisos mínimos o equivalente, y los secretos del pipeline en el almacén del sistema de CI o en Key Vault. Si necesita consultar ArgoCD (por ejemplo, para esperar que `staging` esté *Healthy*), usa un token de solo lectura. [E06]
- Los `Secret` llegan al cluster por Sealed Secrets, SOPS o External Secrets, **nunca con `kubectl create secret`**. [E06]
- Bases de datos y broker en el cluster o como servicios gestionados, por ambiente: **los ambientes no comparten datos**. [E06]
- Costo y forma de apagar o destruir el cluster y los recursos asociados. [E06] Ver costos en [Kubernetes en AKS](kubernetes-en-aks.md). [G09]

### 5.2 CI de cada servicio real

Ante cada push y cada pull request: [E06]

1. **Build** reproducible en contenedor: Dockerfile multi-stage, usuario no root, imagen base fijada por versión y sin `latest`. **La imagen que se publica y se promueve no contiene código, dependencias, herramientas, datos ni endpoints de testing**: sale de una etapa final de release que solo copia lo necesario. Los tests corren en etapas o imágenes de test separadas, que no se publican en el registro de despliegue.
2. **Análisis estático:** formato o lint y al menos un análisis de seguridad de dependencias o de imagen.
3. **Tests unitarios** en contenedor, con reporte de resultados y de cobertura como artefactos. La cobertura tiene que ser **mayor al 80 %** (líneas, o líneas y ramas si la herramienta lo permite) y el pipeline falla si no se supera. El README declara qué se mide y qué archivos se excluyen, y justifica cada exclusión.
4. **Tests funcionales** con Docker (`docker compose` o Testcontainers): el servicio con sus dependencias reales de infraestructura (base, broker) y mocks para los otros servicios. Cubren la API pública, incluidos errores de negocio e idempotencia. La cobertura se mide con el servicio instrumentado en una imagen de test aparte (nunca la de release) y también tiene que superar el 80 %, con gate. **Las dos coberturas se miden por separado: una no compensa a la otra.**
5. **Empaquetado:** construir y publicar la imagen solo si todo lo anterior pasó.

Los pipelines evitan reconstruir y redesplegar lo que no cambió, por ejemplo con filtros por ruta en un monorepo o un pipeline por servicio. [E06] Ver el child pipeline del monorepo en [DevOps y CI/CD](devops-y-ci-cd.md#monorepo-con-ci-por-servicio). [G05-MONOREPO]

### 5.3 Artefactos y versionado

- Cada imagen se etiqueta de forma **inmutable y trazable** al commit (SHA o versión semántica generada); no se despliega con `latest`. [E06]
- **Lo que se prueba en `staging` es exactamente lo que se despliega en `prod`:** la misma imagen, idealmente por digest (si se usa un tag, el registro debe impedir reescribirlo), sin reconstruir. Los ambientes difieren solo en la configuración. [E06] [T20, p. 9]
- Manifiestos con base común y valores por ambiente; en cada ambiente, la versión de cada servicio queda **declarada en Git**, de modo que el historial de Git es el historial de despliegues. [E06]

### 5.4 Continuous Delivery

| # | Etapa | Disparador | Qué hace | Gate |
|---|---|---|---|---|
| 1 | CI de servicio | Push o pull request | Build, lint, tests unitarios y funcionales en Docker, publicación de la imagen | Todos los checks en verde |
| 2 | Promoción a `dev` | Merge a la rama principal | El pipeline commitea el nuevo tag o digest en el estado deseado de `dev`; ArgoCD sincroniza | *Synced* + *Healthy* + smoke test |
| 3 | Promoción a `staging` | Éxito en `dev` | Commitea **las mismas imágenes** en `staging`; ArgoCD sincroniza | *Synced* + *Healthy* |
| 4 | Integración en `staging` | Sincronización de `staging` exitosa | Corre `tests/integration` contra `staging` | **Todos los tests pasan** |
| 5 | Promoción a `prod` | Gate de `staging` en verde + aprobación manual | Propone el cambio (PR o commit protegido); tras aprobarlo, ArgoCD sincroniza | *Synced* + *Healthy* + smoke test |

Fuente: [E06]. Requisitos: [E06]

- **Pull, no push:** se demuestra que el pipeline no puede modificar recursos del cluster y que un cambio manual con `kubectl` es revertido por *self-heal* o, al menos, detectado como *OutOfSync*.
- **ArgoCD:** sincronización automática en `dev` y `staging`; controlada en `prod` (manual o condicionada a la aprobación en Git). Se recomienda `prune` y `selfHeal` donde corresponda, justificando donde no.
- **Si el gate de integración falla, `prod` no se toca:** no hay ningún cambio en su estado deseado, y debe demostrarse con una corrida fallida.
- **Aprobación manual explícita** para `prod` (revisión del PR, *environment protection* o sync manual). Sin ella se pierde ese ítem del criterio de CD, aunque se justifique.
- **Estrategia de despliegue** elegida y justificada, con readiness y liveness que verifiquen algo real: una sonda que siempre responde OK es un error.
- **Rollback por Git** (`git revert` o fijar la versión anterior), documentado y probado al menos una vez, mostrando el commit, la sincronización y el resultado. Un rollback con `kubectl` o `helm` contra el cluster no cuenta.
- Los mocks se promueven por el mismo flujo, así que los tres namespaces tienen siempre el conjunto completo.
- Si hay bases con esquema, explicar cómo y en qué orden se aplican las **migraciones** respecto del despliegue, por ejemplo con *expand/contract*.

### 5.5 Dónde corre cada nivel de test

| Nivel | Dónde corre | Contra qué |
|---|---|---|
| Unitarios | Contenedor del pipeline | El código del servicio, sin infraestructura |
| Funcionales | Docker en el runner (Compose o Testcontainers) | El servicio real, su infraestructura real y mocks de los otros servicios |
| Integración | Namespace `staging` en AKS | Los servicios desplegados (reales y mocks), por su punto de entrada |

Fuente: [E06]. Unitarios y funcionales se corren **localmente con un único comando** documentado, como `make test` o `docker compose run tests`. [E06]

*Inferencia terminológica:* con las categorías de [Testing](testing.md), los "funcionales" de `E06` tienen el alcance de un test de componente (servicio entero aislado, base real, vecinos reemplazados) y los "de integración" son end-to-end; el Anexo A los llama así: "caso end to end a través del punto de entrada". [T10, p. 9] [T10, p. 12] [E06] Ver [Dudas y conflictos](../dudas-y-conflictos.md).

### 5.6 Tests de integración

Al menos **cinco**, automatizados y ejecutados por el pipeline en `staging`. Cada uno: [E06]

- corresponde a un RF o escenario identificado en su nombre o documentación;
- parte de **datos conocidos** y se puede repetir sin romper la corrida siguiente (siembra o limpia sus datos);
- verifica un **resultado de negocio observable** (estado, cantidades, códigos), no solo un 200;
- afirma un resultado **producido por los dos servicios reales**: puede atravesar mocks, pero la aserción no depende de su lógica ni de su estado. Un test que ejercita un solo servicio real no cuenta.

Al menos uno cubre un caso de **fallo o concurrencia**. Los resultados se publican como artefacto (por ejemplo, JUnit o HTML). [E06]

### 5.7 a 5.10 Configuración, README, evidencia y diseño

- **Configuración y secretos.** Configuración separada del código e inyectada por ambiente (ConfigMaps, valores del chart en Git). Los secretos (cadenas de conexión, credenciales OAuth, claves del registro) nunca están en texto plano en el repo ni en las imágenes; como ArgoCD solo lee de Git, el equipo elige y explica Sealed Secrets, SOPS, External Secrets con Key Vault u otro. Se demuestra que los tres ambientes usan configuración y datos distintos. [E06] Ver [Secrets](kubernetes-parte-2.md) y el drift de secretos externos en [GitOps](gitops-y-entrega-continua.md#reconciliación-y-drift). [T17, p. 50] [T20, p. 13]
- **README.** Un tercero, sin ayuda, tiene que poder: crear o conectarse al cluster desde `infra/` e instalar ArgoCD y sus Applications; configurar las credenciales del pipeline; disparar un cambio y seguirlo hasta producción, incluida la UI o CLI de ArgoCD; correr localmente los tests unitarios y funcionales; verificar cada ambiente (`kubectl get pods -n <ns>`, estado de las Applications y puntos de entrada); hacer un rollback por Git; y destruir los recursos de Azure. Incluye una tabla de **todos** los servicios, reales o mocks, con su estado en cada ambiente. [E06]
- **Evidencia** en `docs/evidencias/`: una corrida completa de commit a producción; una con un check de CI fallido y sin imagen publicada; una con integración fallida en `staging` y `prod` sin cambios; un rollback por Git con su reconciliación; ArgoCD con las tres Applications *Synced* y *Healthy* más `kubectl get pods,svc` por ambiente; y deriva corregida por *self-heal* o marcada *OutOfSync*. [E06]
- **`docs/cicd.md`**, de 3 a 6 páginas: diagrama del pipeline, etapas y gates, versionado de imágenes, estructura del estado deseado y flujo de promoción por Git, política de sync por ambiente, estrategia de despliegue y rollback, secretos, justificación de los dos servicios reales y de las herramientas, y cambios respecto de la segunda entrega. [E06]

## Reglas de calidad

**Todo es código. Git es la fuente de verdad del cluster. Una imagen, muchos ambientes. Sin testing en producción** (ni en `staging`, porque es la misma imagen). **El pipeline falla de verdad:** un gate que no puede fallar no es un gate. Sin secretos en el repositorio, el historial ni las imágenes. Sondas con sentido. Reproducibilidad local igual que en CI. Trazabilidad de cada test de integración a un RF o escenario. Coherencia con la arquitectura de la segunda entrega. Y nada de boilerplate: si una práctica no aplica, se explica por qué en lugar de simularla. [E06]

## Criterios de corrección

| Criterio | Qué se mira | Peso |
|---|---|---:|
| Cluster AKS y ambientes | IaC reproducible, tres namespaces aislados, ArgoCD por ambiente, acceso seguro del pipeline, recursos y políticas por ambiente | 20 % |
| CI y testing de los servicios reales | Build en contenedor, análisis estático, unitarios y funcionales en Docker, cobertura > 80 % de cada tipo con gate, imágenes publicadas y versionadas | 20 % |
| Pipeline de CD y gates (GitOps) | Pull con ArgoCD, promoción `dev → staging → prod` con la misma imagen por commits, gate de integración que detiene la promoción, aprobación a `prod`, rollback por Git | 20 % |
| Tests de integración | Al menos cinco, trazables, con verificación de negocio, repetibles, con un caso de fallo o concurrencia, en `staging` | 20 % |
| Completitud del despliegue | Todos los servicios en los tres ambientes, mocks conformes al contrato, configuración y secretos por ambiente | 10 % |
| Documentación y reproducibilidad | README ejecutable por un tercero, `docs/cicd.md`, evidencias exitosas y fallidas | 10 % |

Fuente: [E06]. Esta tabla es la nota de la tercera entrega; `criterios-evaluacion.md` (fronteras, dominio, coordinación), que no está incorporado, no se aplica. [E06]

**Restan puntos:** desplegar por push (`kubectl apply` o `helm upgrade` desde el CI), desplegar con `latest`, reconstruir para producción, imágenes con código o dependencias de testing, secretos en el repositorio, gates que no bloquean, sondas triviales, mocks que contradicen un contrato definido, namespaces que comparten datos, cambios manuales no reflejados en el código e implementar más de dos servicios reales. [E06]

**No se evalúa** una entrega que no pueda reproducir un despliegue en AKS vía pipeline con promoción `staging` → `prod` (RNF-04 y RNF-05). La falta de gate de integración, el incumplimiento de RNF-06 u otros defectos no la vuelven no evaluable en esta entrega intermedia: se reflejan en los criterios. En la entrega final rigen RNF-01 a RNF-06 completos. [E06] [E01, p. 6]

## Modalidad y defensa

La entrega es grupal, versionada en el repositorio del equipo, con el pipeline ejecutable y el cluster **desplegado y accesible durante la defensa**; después se destruye y se adjunta la evidencia. Fecha, canal, modelo de acceso a Azure (suscripción o créditos, región y tope de gasto) y ventana de defensa se informan en el Campus. [E06]

En la defensa el equipo tiene que: hacer un cambio en un servicio real y mostrar cómo llega a producción por Git y ArgoCD; provocar un fallo y mostrar que la promoción se detiene y producción queda intacta; ejecutar un rollback por Git; y justificar los dos servicios reales, la estrategia de despliegue, los secretos y el aislamiento entre ambientes. [E06]

## Anexo A: catálogo de tests end-to-end

El Anexo A define IT-01 a IT-39, derivados de los RF y de los 16 escenarios de `E01`, cada uno a través del punto de entrada de `staging` con usuarios de prueba autenticados (A y B coleccionistas, Op operador). [E06]

**Alcance.** En esta entrega el equipo elige al menos cinco que su recorte pueda satisfacer. En la **entrega final todos los escenarios de aceptación tienen que estar cubiertos por tests del pipeline**, porque todos los servicios serán reales; un IT de escenario sin implementar o que no pasa equivale a un escenario sin cubrir. Conviene elegir, nombrar y estructurar la suite pensando en completarla, y registrar los pendientes como deuda en `docs/cicd.md`. [E06]

| Grupo | IDs | Requerimientos |
|---|---|---|
| A.1 Autenticación y acceso | IT-01 a IT-04 | RF-A01 a RF-A05, RF-05, RF-33, RF-36 |
| A.2 Colección y álbum | IT-05 a IT-07 | RF-01 a RF-04, RF-30, RF-31 |
| A.3 Sobres | IT-08 a IT-11 | RF-06 a RF-08, RF-22 |
| A.4 Ofertas | IT-12 a IT-18 | RF-10 a RF-13, RF-18, RF-19, RF-23, RF-24, RF-37 |
| A.5 Intercambios | IT-19 a IT-28 | RF-04, RF-12, RF-14 a RF-17, RF-20 a RF-22, RF-39 |
| A.6 Retos, recompensas, códigos y rankings | IT-29 a IT-34 | RF-09, RF-25 a RF-29, RF-31, RF-32, RF-38 |
| A.7 Operador | IT-35, IT-36 | RF-33 a RF-35 |
| A.8 Despliegue y salud (smoke) | IT-37 a IT-39 | Sondas, punto de entrada y versión desplegada igual a la de Git |

Fuente: [E06]. Los de A.8 se ejecutan en cada ambiente, también en esta entrega, y no cuentan para el mínimo de cinco. [E06]

**Tests de fallo, concurrencia o duplicación (F):** IT-01, IT-09, IT-10, IT-11, IT-13, IT-22, IT-23, IT-24, IT-25, IT-26, IT-27, IT-28, IT-30 e IT-33. [E06]

### Escenario de aceptación → test

| Escenario de `E01` | IT |
|---|---|
| 1. Apertura normal | IT-08 |
| 2. Reintento de apertura | IT-09 |
| 3. Intercambio 1:1 exitoso | IT-19 |
| 4. Dos ofertas con la misma copia | IT-23 |
| 5 y 6. Interrupción con una o ambas copias apartadas | IT-24 |
| 7. Respuesta perdida | IT-11 (apertura), IT-26 (aceptación) |
| 8. Evento duplicado | IT-27; IT-30 en su variante de evento duplicado |
| 9. Progreso fuera de orden | IT-30 |
| 10. Actualización diferida | IT-06 |
| 11. Dos aceptaciones concurrentes | IT-22 |
| 12. Vencimiento o cancelación durante la aceptación | IT-25 |
| 13. Sobre ajeno o ya abierto | IT-10 |
| 14. Oferta inválida | IT-13 |
| 15. Doble canje del último uso | IT-33 |
| 16. Sin sesión válida | IT-01 |

Elaboración propia a partir de las columnas "Req. / escenario" del Anexo A. [E06] [E01, p. 6]

### Datos, autenticación y control del escenario

- Cada test declara y siembra o limpia sus datos (usuarios, sobres, copias, ofertas). [E06]
- **Tokens de prueba:** cuentas de Google del equipo (A, B, C, Op). Una sola vez, en un navegador, se obtiene un *refresh token* por cuenta y se guarda como secreto de CI; el job solo hace el *refresh grant*. Un token inválido es una cadena arbitraria y uno expirado es un *access token* real ya vencido, guardado como secreto. El README documenta el procedimiento y la rotación. [E06]
- **Sin credenciales de cluster en el CI:** los tests que inyectan fallas (IT-24, IT-28), pausan una operación (IT-20, IT-25, IT-35) o siembran un estado inconsistente (IT-36) usan un mecanismo versionado y acotado a `staging`, **sin agregar código ni endpoints de testing a la imagen**. Ejemplos: un mock cuyo paso de confirmación el test puede retener y liberar, un proxy de fallas desplegado por ArgoCD, un Job de ArgoCD con permiso solo para borrar Pods de ese namespace, o un Job que siembra la base de `staging` con una credencial de test de ese ambiente. IT-36 usa ese último, con identificadores `it-36-*` y limpieza al terminar. [E06]

### Recorte recomendado para esta entrega

| Servicios reales | Tests sugeridos |
|---|---|
| Colección/sobres + autenticación | IT-01, IT-02, IT-05, IT-06, IT-08, IT-09, IT-10, IT-11 |
| Colección/sobres + ofertas/intercambio | IT-12, IT-13, IT-14, IT-19 a IT-24, IT-28 (los de sobres cuentan solo si abren sobres a través de ofertas) |
| Ofertas/intercambio + coordinación (saga) | IT-19 a IT-27, si la coordinación y las cantidades son de los servicios reales |
| Códigos promocionales + retos/recompensas | IT-29, IT-31, IT-33; IT-30 e IT-32 solo si el segundo servicio participa de la aserción |

Fuente: [E06]. Un test sugerido cuenta solo si cumple la sección 5.6 con el recorte elegido. [E06]

## Versiones de la consigna

Solo está incorporada la versión 1.1. Su changelog registra: [E06]

- RNF-06 aplica cualquiera sea el recorte, y se documenta el origen de los tokens de prueba;
- los cinco tests tienen que tener una aserción producida por los dos servicios reales, y se rehízo la tabla A.9;
- se aclara qué prevalece sobre la segunda entrega, qué hacer si falta un contrato y qué hace que una entrega no se evalúe;
- se define cómo inyectar fallas, pausar operaciones y sembrar estados sin credenciales de cluster;
- se corrigen referencias del catálogo y la cobertura exigida en la entrega final;
- las imágenes desplegadas no pueden incluir código ni dependencias de testing;
- se fija el umbral de cobertura mayor al 80 % para unitarios y funcionales, medidos por separado y con gate;
- se unifica "dos servicios reales", la aprobación a `prod` y los secretos del cluster, y se agregan el acceso a Azure y la ventana de defensa.

## Unidades que alimentan cada parte

**Inferencia de estudio:** la consigna no asigna unidades; esta tabla relaciona lo pedido con el material incorporado.

| Parte | Unidades y conceptos |
|---|---|
| 5.1 Cluster | [Kubernetes en AKS](kubernetes-en-aks.md): Terraform, costo, `down.sh`, Gateway API y NetworkPolicy [G09]. Namespaces sin aislamiento por sí solos [T16, p. 56]; NetworkPolicy [T17, p. 25]. |
| 5.2 CI | [Imágenes](imagenes-de-contenedores.md): multi-stage y usuario no root [T06, p. 27] [T06, p. 17]. [Testing](testing.md): niveles y límites de la cobertura [T10, p. 3] [T10, p. 18]. [DevOps](devops-y-ci-cd.md): pipeline y monorepo [T13, p. 27] [G05-MONOREPO]. |
| 5.3 Versionado | Tags mutables frente a digest [T20, p. 9] [T06, p. 36]. |
| 5.4 CD | [GitOps](gitops-y-entrega-continua.md): pull, `Application`, `prune`, `selfHeal` y drift [T20, p. 11] [T20, p. 17] [T20, p. 13]. [Helm](helm.md) o Kustomize para la base común [T19, p. 8] [T20, p. 18]. Estrategias de despliegue [T13, p. 40] [T21, p. 29]. Probes y rolling update con drenado [T18, p. 16] [T18, p. 18]. |
| 5.6 Integración | Idempotencia y respuesta perdida [T21, p. 14] [T18, p. 22]; concurrencia y locks [T12, p. 11]. |
| 5.7 Secretos | Secrets en Base64 y prácticas [T17, p. 51] [T17, p. 52]; secretos externos como drift esperado [T20, p. 13]. |
| Excluido | HA, autoscaling y latencia, con el vocabulario de [Resiliencia y escalabilidad](resiliencia-y-escalabilidad.md) para justificarlo [T21, p. 7] [T21, p. 32]. |

## Devolución recibida

La cátedra corrigió la entrega del grupo con veredicto PASS en las seis secciones y sin hallazgos bloqueantes. Quedaron cuatro ítems parciales: la aprobación a `prod` sin un aprobador distinto del autor (C3.6), una liveness que responde OK siempre (C3.11), un cliente OAuth compartido entre ambientes (C1.3) y la falta de justificación del lenguaje y del framework de tests (C6.4). [D03] El detalle, lo que se revisa en la defensa y el camino hacia la entrega final están en [Devolución de la tercera entrega](devolucion-tercera-entrega.md).

## Puntos aclarados y abiertos

- **AKS en lugar de Compose.** Resuelto: esta entrega se despliega en AKS y sus decisiones prevalecen sobre las secciones 8 y 9 de la segunda entrega. [E06]
- **La guía de AKS frente a esta consigna.** `G09` usa el registry de GitLab con un deploy token, crea el Secret de Postgres con `kubectl create secret` y comparte un kubeconfig de administrador entre integrantes. `E06` pide ACR, prohíbe `kubectl create secret` y deja al pipeline sin credenciales de administración. [G09] [E06] *Inferencia:* la guía sirve como base del cluster (Terraform, Gateway, NetworkPolicy, costos), pero para la entrega rige `E06`. Ver [Dudas y conflictos](../dudas-y-conflictos.md).
- **`criterios-evaluacion.md`.** `E06` lo menciona para decir que no se aplica; no está incorporado. [E06] La devolución de la segunda entrega dice que corrigió con ese archivo. [D02]
- **Correcciones de la devolución.** Sondas, estrategia de despliegue, PostgreSQL por ambiente, códigos de la aceptación, presupuesto de latencia y extensión de `docs/cicd.md`: ver [Devolución de la segunda entrega](devolucion-segunda-entrega.md#cómo-aplicarla-en-la-tercera-entrega). [D02]
- **Testing terminológico.** "Funcional" e "integración" son niveles distintos en `E06`. Ver la fila de [Dudas y conflictos](../dudas-y-conflictos.md) sobre la equivalencia oral. [E06] [N-2026-09-11-integration-test-functional-testing]
