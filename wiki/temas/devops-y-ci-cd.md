# DevOps y CI/CD

## DevOps

DevOps es un conjunto de prácticas, herramientas y filosofía de trabajo que busca automatizar todos los procesos entre el desarrollo y la puesta en producción. El nombre une Development y Operations; DevSecOps agrega la seguridad como prioridad. [T13, p. 3]

Sus beneficios presentados son despliegues más rápidos, frecuentes y con menos errores; colaboración entre equipos interdisciplinarios; cambios más chicos y seguidos; confianza para incorporar funcionalidades sin afectar producción; y seguridad incorporada desde el desarrollo. [T13, p. 4]

El ciclo en forma de infinito, verificado sobre la lámina, recorre del lado Dev plan, code, build y test, pasa por release y sigue del lado Ops con deploy, operate y monitor. [T13, p. 5]

| Etapa | Qué involucra |
|---|---|
| Plan | Planificar la siguiente iteración, incluidas funcionalidades nuevas y correcciones de métricas observadas; define alcance y objetivo [T13, p. 5] |
| Code | Todo CI/CD empieza con un commit en un branch, que puede disparar un webhook hacia la herramienta de CI/CD [T13, p. 6] |
| Build | Transforma el código en el artefacto a desplegar; las imágenes de contenedores son un ejemplo [T13, p. 7] |
| Test | Testeo continuo de los artefactos contra sus especificaciones; se desarrolla en [Testing](testing.md) [T13, p. 8] |
| Release | Versión habilitada para producción, identificable por ejemplo con el tag de imagen; su delivery equivale a subirla a la registry [T13, p. 9] |
| Deploy | Poner en producción un nuevo release; la forma depende de la arquitectura y la plataforma [T13, p. 10] |
| Operate | Controlar el uso de recursos y ajustar parámetros para maximizar capacidad al menor costo [T13, p. 11] |
| Monitor | Registrar métricas para evaluar la calidad del servicio e identificar problemas operativos y de seguridad [T13, p. 12] |

El **Leftover Principle** de Erik Hollnagel advierte que, al automatizar, el operador humano queda con las tareas dispersas que no fue factible o económico automatizar. [T13, p. 13]

## CI, Continuous Delivery y Continuous Deployment

Un proceso de CI/CD pone en producción automáticamente una nueva versión cada vez que se detecta un cambio en el código. [T13, p. 15]

| Práctica | Qué agrega | Límite |
|---|---|---|
| Continuous Integration | Build y test automáticos con cada commit y PR, sobre ramas de feature de vida corta; nadie tiene que acordarse de lanzarlo y los errores aparecen rápido | Solo integra y valida el cambio [T13, p. 16] |
| Continuous Delivery | CI más tests de integración, de performance y UAT; todo lo que pasa queda listo para desplegarse en otros ambientes | No incluye el despliegue [T13, p. 17] |
| Continuous Deployment | Todo release que pasa las pruebas se despliega automáticamente a producción | Es la meta salvo regulaciones u otras restricciones; es una decisión de negocio [T13, p. 18] |

## Repositorio, branches y pull requests

Un repositorio de código (VCS) guarda toda la historia de cambios, incluidas versiones que nunca llegaron a implementarse, y permite que varias personas integren cambios sobre los mismos archivos. [T13, p. 19] La lámina compara un monolito en un solo repositorio, un multi-repo con un Git por proyecto y un monorepo con varios proyectos en un único Git. [T13, p. 20]

El diagrama de branches muestra `master` con tags de versión, `hotfix` que sale de master y vuelve a master y develop, `develop` como integración, ramas `feature` que salen y vuelven a develop, y `release` que pasa de develop a master. [T13, p. 22] *Inferencia:* corresponde al modelo conocido como Git Flow; la lámina no lo nombra.

Un pull request abre una discusión para incorporar un cambio a un branch principal: otras personas evalúan el impacto, sugieren, aprueban o rechazan, y tras la aprobación se hace merge. Los repositorios modernos agregan controles automáticos a esa aprobación. [T13, p. 23] En el workflow, el PR dispara el CI server, luego viene la revisión de pares, y se actualiza o aprueba antes del merge. [T13, p. 24]

Los SCM como GitHub o GitLab se integran con otros sistemas mediante webhooks y su API. En cada estado de un PR pueden disparar procesos de CI/CD, típicamente build o testing, cuyo resultado se refleja en el mismo PR para ayudar a decidir la integración. [T13, p. 25] [T13, p. 26]

## Pipeline

Un pipeline es una secuencia de pasos del proceso de CI/CD; ante una falla se vuelve al inicio para corregirla, y los principios fail fast y fail first son críticos. [T13, p. 27] Los ejemplos muestran un Jenkinsfile en Groovy y un `.gitlab-ci.yml` con jobs asignados a stages `build`, `test` y `deploy`. [T13, p. 28]

- **Stage:** etapa con función propia, compuesta por uno o más jobs.
- **Job o step:** paso concreto dentro de una stage.
- **Runner o agente:** ambiente donde se ejecutan los steps, sea local, remoto o SaaS del proveedor. [T13, p. 29]

La cátedra toma GitLab como referencia: cada paso corre en un contenedor aislado, y los pasos intercambian archivos mediante artefactos exportados o un cache compartido por el pipeline para reutilizar librerías o imágenes. [T13, p. 30]

| Etapa típica | Contenido |
|---|---|
| Preparation | Versiones de software, credenciales y secretos, otros repositorios; la configuración como código del ambiente asegura el mismo estado inicial y, por lo tanto, el mismo resultado [T13, p. 32] |
| Build | Compilación, resolución de dependencias y paquetes; genera los artefactos que usan las demás etapas, y cada uno es candidato a release que debe preservarse hasta ser rechazado [T13, p. 33] |
| Unit Test | Pruebas incrementales que confirman que no se rompe comportamiento previo; en microservicios puede enfocarse en cada endpoint, los cambios en la base y que el servicio siga corriendo [T13, p. 34] |
| Deliver | Disponibilizar los artefactos sin modificarlos, aunque cambie la configuración que los elige, como un `docker-compose.yaml`; por ejemplo, subir una imagen a una registry o publicar en npm o PyPI —la lámina escribe “PyPy”— [T13, p. 35] |
| Integration Test | Ambiente que simula producción con las nuevas versiones y operaciones de punta a punta; es costoso, suele ejecutarse por partes y solo en releases con muchos cambios [T13, p. 36] |
| Deploy | Poner el cambio en producción, desde cambiar una imagen en la registry hasta migrar una base, reiniciar máquinas o abrir una ventana sin servicio; plataformas como Kubernetes lo simplifican [T13, p. 37] |

Fuente del orden: [T13, p. 31]. El ejemplo real de GitLab encadena `preparation`, builds paralelos por servicio, tests por servicio, `test-integration`, `deliver-dockerhub` y `deploy-prod`. [T13, p. 46]

La descripción de unit test de esta unidad no coincide con la de [Testing](testing.md), que lo reduce a una función o método; ver [Dudas y conflictos](../dudas-y-conflictos.md). [T13, p. 34] [T10, p. 5]

## Estrategias de despliegue

| Estrategia | Funcionamiento | Ventajas y riesgos |
|---|---|---|
| Basic | Se bajan, actualizan y levantan todos los nodos | La más simple y la más riesgosa: no hay servicio durante el proceso y el rollback revierte todo [T13, p. 38] |
| Multi-service | Varios servicios se despliegan juntos, cada uno con su nueva versión | A veces necesario por alto acoplamiento; complica las pruebas de integración y aumenta los fallos [T13, p. 39] |
| Rolling | Los nodos se actualizan progresivamente hasta reemplazarlos todos | Automático, fácil de implementar y revertir, pero no evalúa el éxito hasta el final y exige retrocompatibilidad para que convivan versiones [T13, p. 40] |
| Blue/Green | Se arma un ambiente paralelo y un distribuidor de tráfico decide a cuál enviar | Muy usado en ambientes de contenedores simples [T13, p. 41] |
| Canary | La nueva versión recibe primero un grupo de usuarios y se amplía progresivamente | Progresivo, de bajo impacto ante fallas y con rollback rápido [T13, p. 42] |

La estrategia depende del riesgo aceptable y del impacto de negocio de un error —no se despliega igual el firmware de un respirador que un videojuego—, y como cada una tiene costos, forma parte del diseño de la arquitectura y del CI/CD. [T13, p. 43]

## Seguridad y shift-left

La seguridad es tan crítica como la disponibilidad: sus controles pueden integrarse al pipeline y bloquear el despliegue de módulos, imágenes, configuraciones o código inseguro. [T13, p. 44] Las técnicas concretas —linter, SAST, SCA y DAST— están en [Testing](testing.md). [T10, p. 20]

**Shift-left** significa devolverle el error al desarrollador cuanto antes: un commit o merge dispara el pipeline con pruebas funcionales, de performance y de seguridad, y cualquier falla se avisa de inmediato. Hace responsable al desarrollador de resolver apenas surge el problema y reduce costos de operación al atacarlo antes de producción. [T13, p. 45]
