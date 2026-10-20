# Kubernetes — Parte 1

## Fuente y alcance

Síntesis de `T16`, presentación de la cátedra identificada por su portada, docentes y curso Microservicios (73.40), ITBA. **Inferencia de catalogación:** el ciclo 2026 se asigna por la configuración de la materia y la fecha de creación del PDF; no figura un ciclo en la portada. Se cubren arquitectura, Pods, workloads, reconciliación y acceso a la API. [T16, p. 1] [T16]

Kubernetes automatiza despliegues, actualizaciones y mantenimiento de réplicas. Permite configurar autoscaling y comprobaciones de salud; métricas, almacenamiento de logs y otras funciones necesitan componentes adicionales. [T16, p. 5]

## Arquitectura y responsabilidades

El clúster reúne un plano de control que coordina el estado deseado y nodos que ejecutan Pods. Los componentes se coordinan mediante la API: los controladores gestionan objetos y el kubelet ejecuta los contenedores asignados a su nodo. [T16, p. 6] [T16, p. 30]

| Pieza | Responsabilidad |
|---|---|
| API server | Expone la API usada por clientes y componentes. [T16, p. 7] |
| Scheduler | Elige un nodo para un Pod sin asignación y registra la decisión mediante la API. [T16, p. 7] |
| Controller manager | Ejecuta controladores que acercan lo observado a lo deseado. [T16, p. 7] |
| etcd | Persiste los datos de la API; su disponibilidad y respaldo son críticos. [T16, p. 8] |
| kubelet | Observa los Pods asignados al nodo y mantiene sus contenedores en ejecución. [T16, p. 9] |
| Runtime CRI | Ejecuta contenedores; ejemplos: containerd y CRI-O. [T16, p. 9] |
| Plugin CNI | Configura la conectividad de los Pods. [T16, p. 9] |
| kube-proxy | Implementa reglas para Services; ciertas soluciones de red lo reemplazan. [T16, p. 9] |

`kubectl` es un cliente para consultar y modificar recursos; `kubeadm` inicializa clústeres, incorpora nodos y asiste en actualizaciones. Son herramientas, no componentes permanentes del plano de control. [T16, p. 8]

## Pod y relación con los contenedores Linux

El Pod es la unidad mínima administrada por Kubernetes: agrupa uno o más contenedores en un mismo nodo. El scheduler lo asigna y el kubelet lo ejecuta. Un Pod no migra entre nodos; si se necesita reemplazarlo, el controlador crea otro con nuevo UID. [T16, p. 10]

El caso habitual es un contenedor por Pod; agrupar varios tiene sentido cuando necesitan un contexto estrechamente compartido. El esquema distingue red, IPC y hostname compartidos de los espacios de procesos y raíces propios. [T16, p. 12] [T16, p. 14]

- Comparten IP y puertos, y se comunican por `localhost`. [T16, p. 15]
- PID está separado por defecto; compartirlo requiere `shareProcessNamespace: true`. [T16, p. 15]
- Cada contenedor conserva su filesystem raíz; pueden montar volúmenes comunes. Los UID/GID se configuran con `securityContext`. [T16, p. 15]
- Los namespaces del kernel aíslan vistas y los cgroups controlan recursos. No son los objetos Namespace de la API de Kubernetes. Ver [Contenedores sobre Linux](contenedores-linux.md). [T16, p. 13]

Por contenedor, `requests` de CPU/memoria orientan la elección del nodo y `limits` restringen la ejecución: CPU puede sufrir throttling y memoria, OOM. El almacenamiento efímero se mide en bytes; no hay campos estándar de requests/limits para velocidad de disco o red. [T16, p. 16]

## Inicialización, fases y salud

Los contenedores de aplicación pueden iniciar en paralelo. Los init comunes se ejecutan en orden y deben finalizar con éxito antes de las aplicaciones. Un sidecar nativo permanece activo y se declara en `initContainers` con `restartPolicy: Always`; un contenedor efímero se agrega para depuración y no se reinicia automáticamente. [T16, p. 17]

El ejemplo de init espera que DNS resuelva primero `myservice` y luego `mydb`; resolver sus nombres no garantiza que los servicios estén listos. [T16, p. 22] [T16, p. 23]

| Fase del Pod | Significado |
|---|---|
| `Pending` | Aceptado; espera asignación o preparación, incluida descarga de imágenes. [T16, p. 19] |
| `Running` | Asignado; contenedores creados y al menos uno ejecutándose, iniciándose o reiniciándose. [T16, p. 19] |
| `Succeeded` | Todos terminaron con éxito y no se reiniciarán. [T16, p. 19] |
| `Failed` | Todos terminaron; alguno falló y no se reiniciará. [T16, p. 19] |
| `Unknown` | No se pudo obtener el estado, normalmente por comunicación con el nodo. [T16, p. 19] |

`Running` no implica `Ready`. Además, `CrashLoopBackOff` y `Terminating`, visibles en la columna STATUS de `kubectl`, no son fases del Pod. Los estados del contenedor son otro nivel: `Waiting`, `Running` y `Terminated`. [T16, p. 18] [T16, p. 19] [T16, p. 20]

Las probes tienen propósitos distintos: startup comprueba el arranque y demora las demás hasta tener éxito; readiness determina si puede recibir tráfico; liveness detecta cuándo reiniciarlo. Ver el antecedente de [salud en Compose](docker-compose.md). [T16, p. 20]

`postStart` puede ejecutarse junto con el ENTRYPOINT. El período de gracia de terminación empieza antes de `preStop` e incluye tanto el hook como el apagado del proceso. [T16, p. 21]

## Elegir un workload

Normalmente se declaran recursos cuyos controladores crean y reemplazan Pods, en lugar de crear Pods aislados. [T16, p. 24]

| Recurso | Qué mantiene o coordina | Distinción importante |
|---|---|---|
| Deployment | ReplicaSets y actualizaciones declarativas. | Cambiar `.spec.template` inicia rollout; cambiar solo `.spec.replicas` escala. [T16, p. 26] |
| ReplicaSet | Cantidad deseada de Pods. | Crea/elimina; cambiar su template no actualiza Pods existentes. Usa selector y ownership; puede adoptar Pods coincidentes sin otro controlador propietario. [T16, p. 28] [T16, p. 29] |
| StatefulSet | Identidad estable por Pod, con ordinal y nombre. | Un Service headless da identidad de red; `volumeClaimTemplates`, si se configura, crea PVCs propios. La identidad puede mantenerse al reemplazar el Pod en otro nodo. [T16, p. 33] [T16, p. 35] |
| DaemonSet | Un Pod por nodo elegible. | La cantidad depende de selector, afinidad y tolerations, no de `replicas`; puede incluir nodos del plano de control si la configuración lo permite. [T16, p. 31] [T16, p. 32] [T16, p. 36] |
| Job | Una tarea hasta completar la cantidad prevista de finalizaciones exitosas. | `completions` fija éxitos, `parallelism` concurrencia solicitada, `backoffLimit` reintentos y `activeDeadlineSeconds` tiempo total. [T16, p. 37] |
| CronJob | Crea Jobs según cron y zona horaria. | `Allow`, `Forbid` o `Replace` regulan solapamiento para ese CronJob. Puede haber ejecuciones duplicadas u omitidas: la tarea debe ser idempotente. [T16, p. 38] |

El contraste visual es: Deployment mantiene réplicas intercambiables mediante ReplicaSet; StatefulSet conserva nombres ordinales; DaemonSet sigue la elegibilidad de cada nodo. Los diagramas no asignan una identidad ordinal estable a los Pods de Deployment. [T16, p. 34] [T16, p. 35] [T16, p. 36]

## Reconciliación y consistencia eventual

El bucle observa recursos por la API, compara con lo deseado, solicita crear/actualizar/eliminar objetos y repite mientras otros componentes actúan y reportan. La convergencia lleva tiempo y puede bloquearse por errores o falta de recursos. Un recurso puede estar gestionado por controladores aunque no tenga campos `spec` y `status`, como un ConfigMap o Secret. [T16, p. 40] [T16, p. 41] [T16, p. 42]

Para tres réplicas de nginx, la cadena es **Deployment → ReplicaSet → Pods**. Se declara el Deployment, su controlador gestiona un ReplicaSet y el de este crea los Pods; no se crea el ReplicaSet manualmente. Los comandos del ejemplo son: [T16, p. 43] [T16, p. 44] [T16, p. 45]

```sh
kubectl apply -f nginx-deployment.yaml
kubectl rollout status deployment/nginx-deployment
kubectl get pods -l app=nginx
```

`apply` guarda la intención; los controladores crean los recursos y el kubelet inicia los contenedores. `metadata.ownerReferences`, filtrando `controller == true`, permite seguir del Pod al ReplicaSet y de este al Deployment. Cambiar un ConfigMap por sí solo no inicia un rollout: debe cambiar el template del Pod. [T16, p. 42] [T16, p. 45] [T16, p. 46] [T16, p. 47]

Al crear un Namespace, el ServiceAccount `default` puede aparecer después: la API persiste el Namespace, el controlador lo observa y solicita crear la cuenta, que la API también persiste. La asincronía de estas acciones no significa que etcd sea una base inconsistente. La cuenta no necesita `spec`/`status` para estar gestionada por un controlador. [T16, p. 48] [T16, p. 49] [T16, p. 50]

## API, identidad y ámbito

El esquema de una creación/actualización aceptada muestra HTTPS → autenticación → autorización → admisión mutante → validación/admisión validante → persistencia en etcd. Es una vista simplificada que omite conversión y defaulting. `get`, `list` y `watch` no pasan por admisión ni persisten objetos. [T16, p. 52]

Un ServiceAccount identifica a una aplicación ante la API; sus permisos se conceden por separado, por ejemplo con Role y RoleBinding en RBAC. Un Pod sin cuenta explícita usa `default` de su namespace. La presentación describe tokens proyectados, de corta duración y renovados automáticamente; `SECRETS 0` es normal porque desde 1.24 no se crean automáticamente Secrets con tokens permanentes. [T16, p. 53]

La admisión ocurre después de autenticar y autorizar: la mutante puede modificar el objeto (por ejemplo, inyectar un sidecar); la validante comprueba reglas y puede rechazarlo sin modificarlo. Hay controladores integrados, webhooks y políticas declarativas. Incluye creación, modificación, eliminación y ciertas conexiones. [T16, p. 54]

Un manifiesto YAML/JSON describe un recurso: `apiVersion` indica la versión de su API y `kind` el tipo de objeto, no la versión del clúster. La presentación usa `v1` para Pod y `apps/v1` para Deployment. [T16, p. 55]

Un Namespace organiza recursos y delimita sus nombres; por sí solo no impone aislamiento de seguridad ni de red. Pods, Services y PVC son namespaced; Nodes y PV tienen alcance de clúster. Los cuatro namespaces iniciales mostrados son `default`, `kube-node-lease`, `kube-public` y `kube-system`. [T16, p. 56]

## Alcance pendiente

Esta parte menciona Services, CNI y almacenamiento para explicar otras piezas, pero no desarrolla su configuración completa ni una instalación de clúster. Ese detalle no se deduce de los diagramas ni se agrega como contenido oficial. [T16, p. 9] [T16, p. 33] [T16, p. 56] Services, red y almacenamiento se desarrollan en [Kubernetes — Parte 2](kubernetes-parte-2.md). [T17]
