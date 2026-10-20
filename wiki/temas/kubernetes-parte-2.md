# Kubernetes — Parte 2

## Fuente y alcance

Síntesis de `T17`, presentación de la cátedra con la misma portada, docentes y curso que `T16`. **Inferencia de catalogación:** el ciclo 2026 se asigna por la configuración de la materia y la fecha de creación del PDF; no figura un ciclo en la portada. Continúa [Kubernetes — Parte 1](kubernetes-parte-1.md) con red, Services, entrada al clúster, NetworkPolicy, almacenamiento, Operators, ConfigMaps y Secrets. [T17, p. 1] [T17]

## Modelo de red del clúster

Hay tres rangos de direcciones: los nodos usan IPs de la red de infraestructura (*underlay*); los Pods, un rango asignado por el plugin de red; los Services, un rango virtual en el que una ClusterIP ofrece un destino estable que reenvía a los Pods. [T17, p. 2]

Kubernetes distingue cuatro necesidades de comunicación: [T17, p. 3]

| Comunicación | Cómo se resuelve |
|---|---|
| Contenedores del mismo Pod | Comparten el namespace de red, IP y puertos, y se hablan por `localhost`. Pods distintos no comparten `localhost`, y la IP puede cambiar al recrear el Pod. [T17, p. 4] |
| Pod a Pod | Cada Pod recibe IPs únicas (IPv4, IPv6 o dual-stack). El runtime invoca un plugin mediante CNI, que conecta el Pod y le asigna IPs; el modelo permite comunicación Pod–Pod sin NAT y no exige una subred por nodo. [T17, p. 5] |
| Pod a Service | El Service ofrece un acceso estable a uno o más Pods sin depender de sus IPs ni de su ciclo de vida. kube-proxy u otra implementación programa el reenvío desde la ClusterIP. [T17, p. 7] |
| Desde afuera hacia un Service | NodePort o LoadBalancer exponen puertos de transporte; Ingress enruta HTTP/S por host y path; Gateway API es la alternativa recomendada para nuevos despliegues; `kubectl port-forward` sirve para desarrollo y diagnóstico. [T17, p. 8] |

kube-proxy no conecta Pods entre sí: solo configura el reenvío de los Services. [T17, p. 5] El Service da un destino estable, no permisos de acceso, y su balanceo depende de la implementación: no garantiza round-robin ni reparto por solicitud HTTP. [T17, p. 7]

### Overlay o ruteo nativo

El CNI decide cómo viaja el paquete Pod–Pod entre nodos. La overlay es una implementación posible, no un requisito de Kubernetes. [T17, p. 2] [T17, p. 6]

| | Ruteo nativo | Overlay (VXLAN) |
|---|---|---|
| Mecanismo | La red conoce la ruta `10.244.2.0/24 → Nodo B` y el paquete conserva las IPs de los Pods. [T17, p. 6] | El paquete Pod A → Pod B viaja encapsulado dentro de uno Nodo A → Nodo B. [T17, p. 6] |
| Mantenibilidad | — | El underlay no necesita rutas de Pods, pero hay que operar túneles y MTU. [T17, p. 6] |
| Extensibilidad | — | Funciona sobre redes que desconocen las IPs de Pods; acceder a ellas desde afuera suele requerir un gateway. [T17, p. 6] |
| Seguridad | — | Suele dejar las IPs de Pods fuera del ruteo del underlay; no cifra ni reemplaza NetworkPolicy. [T17, p. 6] |
| Performance | Suele ser más eficiente. [T17, p. 6] | La encapsulación agrega encabezados y trabajo. [T17, p. 6] |

Es el mismo costo de encapsulación que la red `overlay` de Docker. Ver [Docker CLI, redes y volúmenes](docker-cli-redes-y-volumenes.md). [T08, p. 29]

## Services y descubrimiento

El selector del Service determina qué Pods lo integran, y Kubernetes mantiene sus direcciones y puertos en EndpointSlices. El cliente usa el nombre DNS, como `mi-svc.mi-ns.svc.cluster.local`, o la ClusterIP, sin conocer las IPs de los Pods. [T17, p. 10]

- **EndpointSlice:** lista direcciones y puertos de los backends. Usa la API `discovery.k8s.io/v1`, estable desde v1.21, agrupa por Service, familia IP y puertos (hasta 100 endpoints por slice por defecto) y soporta IPv4/IPv6, readiness y topología. El controlador lo actualiza cuando cambian los Pods o su disponibilidad. La API Endpoints está deprecada desde v1.33: no soporta dual-stack y trunca listas de más de 1000 endpoints. [T17, p. 9]
- **Ejemplo:** un Deployment de tres réplicas de nginx con `app: nginx` y un Service `my-nginx-svc` con ese selector. El controlador deriva los slices del selector y del estado de los Pods, y se consultan con `kubectl get endpointslices -l kubernetes.io/service-name=my-nginx-svc`. [T17, p. 11] [T17, p. 12]
- **Acceso:** desde un Pod, `wget` llega a nginx tanto por la IP del Service como por `my-nginx-svc.default` (`servicio.namespace`). [T17, p. 13]

### kube-proxy

kube-proxy observa Services y EndpointSlices mediante el API server y programa en cada nodo las reglas que llevan `ClusterIP:puerto` a un endpoint disponible. Implementa IP virtual, NAT y balanceo, pero no la conectividad base Pod–Pod. No hay un proxy central: el reenvío se resuelve en los nodos. [T17, p. 14]

| Modo o alternativa | Estado según la presentación |
|---|---|
| `iptables` | Vigente y predeterminado en Kubernetes 1.37. [T17, p. 15] |
| `nftables` | Alternativa estable; requiere kernel 5.13 o superior. [T17, p. 15] |
| `IPVS` | Deprecado desde v1.35. [T17, p. 15] |
| `kernelspace` | Modo usado en Windows. [T17, p. 15] |
| Reemplazo por la red | Por ejemplo, Cilium con *kube-proxy replacement* balancea con eBPF; la función de encaminar hacia los backends actuales se mantiene. [T17, p. 15] |

### DNS del clúster

El DNS del clúster, habitualmente CoreDNS, resuelve el nombre de un Service a su ClusterIP, a las IPs de los Pods si es headless o a un CNAME si es ExternalName. El `resolv.conf` de un Pod en `default` muestra `nameserver 10.96.0.10`, la búsqueda `default.svc.cluster.local svc.cluster.local cluster.local` y `ndots:5`. [T17, p. 16] Esa lista de búsqueda explica por qué el nombre corto `my-nginx-svc.default` resuelve. [T17, p. 13] [T17, p. 16]

### ClusterIP o headless

| ClusterIP | Headless (`clusterIP: None`) |
|---|---|
| IP virtual estable delante de los Pods; el DNS resuelve a ella. [T17, p. 17] | Sin IP virtual ni proxy de Service. [T17, p. 17] |
| kube-proxy u otra implementación programa el reenvío. [T17, p. 17] | Con selector, el DNS devuelve registros A/AAAA de los Pods elegibles. [T17, p. 17] |
| No garantiza round-robin: `iptables` y `nftables` eligen por conexión, no por solicitud HTTP. [T17, p. 17] | El cliente elige el Pod; sirve para descubrir réplicas o balancear en la aplicación. [T17, p. 17] |

Los YAML de ejemplo publican `port: 80` hacia `targetPort: 8080`. [T17, p. 18] El Service headless es el que da identidad de red a un StatefulSet. Ver [Kubernetes — Parte 1](kubernetes-parte-1.md). [T16, p. 33]

### Tipos de Service

| Tipo | Qué hace |
|---|---|
| ClusterIP | IP interna para acceder desde el clúster. [T17, p. 19] |
| NodePort | Publica un puerto en los nodos y conserva una ClusterIP. El rango predeterminado es 30000–32767 y es configurable; la accesibilidad depende de las IPs habilitadas, la red y los firewalls. [T17, p. 19] |
| LoadBalancer | Requiere integración con un proveedor o controlador, aunque no exige nube. Normalmente asigna NodePorts y no prohíbe el acceso directo a ellos; puede omitirlos con `allocateLoadBalancerNodePorts: false` si la implementación lo soporta. [T17, p. 19] |
| ExternalName | CNAME hacia un nombre externo, sin proxy ni IP virtual. [T17, p. 19] |

En la salida de `kubectl get svc`, un LoadBalancer muestra una EXTERNAL-IP y puertos como `3200:30010/TCP`; un headless figura con CLUSTER-IP `None`. [T17, p. 20]

## Entrada al clúster: Ingress y Gateway API

Ingress y Gateway declaran reglas de entrada, pero el tráfico llega al Service solo si un controlador las implementa. [T17, p. 8]

**Ingress.** Declara rutas HTTP/S por host y path hacia Services internos y requiere un controlador instalado. La API `networking.k8s.io/v1` es estable desde v1.19 y está congelada, no deprecada ni en retirada. El controlador comunitario ingress-nginx fue retirado en marzo de 2026 y ya no recibe correcciones de seguridad; no hay que confundirlo con otros controladores NGINX. [T17, p. 21] `ingressClassName` referencia una IngressClass asociada a un controlador, y cada regla combina `host`, `path`, `pathType` y un backend (Service y puerto). La clase del ejemplo es ilustrativa y debe reemplazarse por una implementación instalada y mantenida. [T17, p. 22]

**Gateway API.** Amplía el modelo de entrada con recursos L4/L7 y separa responsabilidades: el operador administra la infraestructura y las aplicaciones declaran rutas. [T17, p. 23]

| Recurso | Responsabilidad | Referencia |
|---|---|---|
| GatewayClass | Tipo de gateway y controlador que lo implementa (`controllerName`). [T17, p. 23] [T17, p. 24] | — |
| Gateway | Infraestructura de entrada y sus listeners, por ejemplo HTTP en el puerto 80. [T17, p. 23] [T17, p. 24] | `gatewayClassName` |
| HTTPRoute | Reglas HTTP, como `PathPrefix /` para `nginx.example.com`, y sus `backendRefs`. [T17, p. 23] [T17, p. 24] | `parentRefs` hacia el Gateway |

Requiere instalar aparte los CRDs y una implementación compatible: los objetos solos no habilitan tráfico. [T17, p. 23] [T17, p. 24]

## NetworkPolicy

Publicar una ruta no implica permitir cualquier comunicación. NetworkPolicy restringe el tráfico de los Pods con reglas L3/L4 de la API `networking.k8s.io/v1`: `podSelector` elige Pods del namespace y `policyTypes` indica Ingress, Egress o ambas. [T17, p. 25]

- Un Pod que ninguna política selecciona en una dirección no está aislado en esa dirección. [T17, p. 25]
- Las políticas son aditivas: se permite la unión de sus reglas, sin prioridad por orden. [T17, p. 25]
- Si ambos extremos están aislados, deben permitir la conexión el egress del origen y el ingress del destino. [T17, p. 25]
- Sin una implementación de red que las aplique, los objetos no tienen efecto. No autentican, no cifran y no filtran HTTP por URL. [T17, p. 25]

El patrón de la presentación es *default deny + allow*: `default-deny-ingress` selecciona todos los Pods de `default` con `podSelector: {}` y aísla solo su tráfico entrante; `allow-frontend-ingress` permite a `app=frontend` del mismo namespace llegar a `app=nginx` en TCP 80. Si el tráfico entra por un Gateway, hace falta una regla que permita ese origen. [T17, p. 26]

## Almacenamiento

Kubernetes no proporciona por sí mismo un backend persistente; el ciclo de vida de los datos depende de dónde se guardan: [T17, p. 28]

| Almacenamiento | Ciclo de vida | Qué sobrevive |
|---|---|---|
| Capa escribible del contenedor | Contenedor | No sobrevive al reinicio o reemplazo del contenedor. [T17, p. 28] |
| `emptyDir` | Pod | Sobrevive a reinicios del contenedor, no a la eliminación del Pod. [T17, p. 28] |
| Volumen persistente (PV/PVC) | Independiente del Pod | Sobrevive al reemplazo del Pod, según el backend y la retención. [T17, p. 28] |

Un volumen no tiene por qué ser accesible desde todos los nodos: un disco zonal está disponible en su zona y un volumen local queda ligado a un nodo, así que el scheduling debe respetarlo, por ejemplo con `nodeAffinity`. Tampoco es garantía de Kubernetes que los datos sobrevivan a la destrucción del clúster: el PV es un objeto de la API y los datos viven en el backend. Persistencia no significa disponibilidad ilimitada. [T17, p. 29]

### PV, PVC y StorageClass

| Objeto | Qué es | Campos centrales |
|---|---|---|
| PersistentVolume (PV) | Recurso de almacenamiento (disco local, volumen de red u otro backend), con ciclo de vida independiente de los Pods. [T17, p. 30] | `capacity`, `accessModes`, `persistentVolumeReclaimPolicy`. [T17, p. 30] |
| PersistentVolumeClaim (PVC) | Solicitud de almacenamiento. El Pod referencia el PVC sin conocer el backend, y Kubernetes lo vincula con un PV compatible. [T17, p. 33] | Tamaño, modos de acceso y, opcionalmente, clase. [T17, p. 33] |
| StorageClass | Clase de almacenamiento disponible (rendimiento, backend o políticas); no se limita a la nube. [T17, p. 35] | `provisioner`, `parameters`, `reclaimPolicy`, `volumeBindingMode`. [T17, p. 35] |

**Modos de acceso.** No son permisos ni reemplazan la protección de escritura. [T17, p. 31]

| Modo | Alcance |
|---|---|
| RWO, ReadWriteOnce | Lectura/escritura desde un nodo; puede haber varios Pods en él. [T17, p. 31] |
| ROX, ReadOnlyMany | Lectura desde varios nodos. [T17, p. 31] |
| RWX, ReadWriteMany | Lectura/escritura desde varios nodos. [T17, p. 31] |
| RWOP, ReadWriteOncePod | Un único Pod; requiere CSI y es estable desde v1.29. [T17, p. 31] |

**Reclaim policy.** Rige qué pasa al eliminarse el PVC: `Retain` conserva el volumen y sus datos para recuperarlos a mano; `Delete` elimina el PV y el almacenamiento subyacente, si el backend lo soporta; `Recycle` está deprecado. Los PV dinámicos heredan la política de su StorageClass, cuyo valor predeterminado es `Delete`. [T17, p. 31]

**Aprovisionamiento estático.** El administrador crea el PV, como el de NFS del ejemplo, que requiere un servidor con el recurso exportado. El PVC lo elige con `volumeName: nfs` y pide almacenamiento sin clase con `storageClassName: ""`. [T17, p. 32] [T17, p. 34]

**Aprovisionamiento dinámico.** El PVC pide una clase, por ejemplo `fast`, sin fijar `volumeName`, y el provisioner de esa StorageClass crea un PV si no existe uno compatible. Un driver CSI implementa la interfaz estándar de almacenamiento sin modificar Kubernetes y se instala aparte. Si el PVC omite `storageClassName`, puede usarse la clase predeterminada. [T17, p. 35] [T17, p. 36] [T17, p. 37] Con `WaitForFirstConsumer`, la vinculación espera un Pod consumidor y tiene en cuenta su scheduling y la topología del volumen; la alternativa es `Immediate`. [T17, p. 35] [T17, p. 37]

En resumen: la StorageClass define cómo aprovisionar, el provisioner crea el PV, Kubernetes lo vincula al PVC y el Pod monta el almacenamiento a través del PVC. En modo estático el PV lo crea el administrador y también puede pertenecer a una clase. [T17, p. 38]

## Operators

Un Operator combina recursos personalizados con un controlador para automatizar tareas propias de una aplicación. El CRD registra el tipo de recurso, el recurso declara el estado deseado y el controlador reconcilia, como los controladores nativos de [Parte 1](kubernetes-parte-1.md). No se limita a aplicaciones stateful ni a cambios de Deployment, y puede coordinar cómputo, red y almacenamiento. OperatorHub.io es un catálogo; el mantenimiento depende de cada proyecto. [T17, p. 39]

En el ejemplo, un `PostgresCluster` ilustrativo que no pertenece a la API nativa pasa de `instances: 3` a `5`. El controlador crea las instancias nuevas, ajusta replicación y configuración, informa el avance en `status` y repite hasta converger. La lógica de PostgreSQL vive en el controlador, no en el manifiesto. [T17, p. 40]

## ConfigMaps

Un ConfigMap guarda datos no confidenciales en pares clave-valor, hasta 1 MiB, para separar la configuración del entorno de la imagen. No ofrece confidencialidad ni cifrado: para datos sensibles corresponde un Secret o una herramienta adicional. Admite claves simples y claves con forma de archivo. [T17, p. 41]

Un Pod lo consume en `command` o sus argumentos, como variables de entorno, como archivos de solo lectura en un volumen o leyéndolo desde la aplicación con un cliente de la API. [T17, p. 42] Con `env[].valueFrom.configMapKeyRef`, el nombre de la variable puede diferir de la clave. [T17, p. 43] [T17, p. 44] Para archivos, el volumen se declara a nivel Pod con `configMap.name` y `items` (clave → path) y cada contenedor lo monta con `volumeMounts`, en el ejemplo como `/config` read-only. [T17, p. 45] [T17, p. 46] [T17, p. 47]

Cómo llega una actualización depende del mecanismo de consumo: [T17, p. 48]

| Consumo | Efecto de cambiar el ConfigMap |
|---|---|
| Variables de entorno y argumentos derivados | No cambian; hay que recrear el Pod. [T17, p. 48] |
| Volumen proyectado | El kubelet actualiza los archivos de forma eventual; la demora depende de su sincronización y detección de cambios. [T17, p. 48] |
| Montaje con `subPath` | No recibe actualizaciones. [T17, p. 48] |
| Aplicación | Debe releer o recargar el archivo: la actualización no reinicia el proceso. [T17, p. 48] |
| Cliente de la API | Puede leer u observar cambios sin depender del kubelet. [T17, p. 48] |

Esto completa lo de Parte 1: cambiar un ConfigMap no inicia un rollout del Deployment. [T16, p. 47]

## Secrets

Un Secret contiene datos confidenciales, como contraseñas, tokens o claves, separados de la imagen y del código. Se crea independientemente del Pod y se consume como archivo o variable de entorno. Se parece a un ConfigMap, pero su acceso se trata distinto, y por sí solo no garantiza confidencialidad: hay que proteger acceso, almacenamiento y consumo. [T17, p. 49]

- **Acceso:** leer o modificar Secrets requiere autorización, habitualmente RBAC. `get`, `list` y `watch` pueden revelar los datos; acceder a etcd saltea los controles de la API; y quien puede crear Pods o Deployments puede montar Secrets del mismo namespace salvo controles adicionales. El namespace delimita referencias, pero no reemplaza una política de seguridad. [T17, p. 50]
- **Almacenamiento:** en una instalación estándar se guardan sin cifrar en etcd salvo que se configure cifrado *at rest* con `EncryptionConfiguration` en el API server, con claves o un proveedor KMS. Base64 no es cifrado. En servicios gestionados hay que verificar las garantías del proveedor. [T17, p. 51]
- **Creación:** en `data` los valores van en Base64; `stringData` acepta texto plano que Kubernetes incorpora a `data` al persistir. No hay que guardar manifiestos con credenciales reales en el repositorio. [T17, p. 52]
- **Buenas prácticas:** habilitar cifrado *at rest* y proteger sus claves, aplicar RBAC con mínimo privilegio y controlar quién crea workloads, limitar el acceso a los contenedores que lo necesitan, rotar credenciales asegurando que la aplicación recargue, y evaluar proveedores externos. External Secrets Operator sincroniza hacia Secrets de Kubernetes (etcd); Secrets Store CSI Driver monta desde un proveedor externo sin sincronizar. [T17, p. 53]

El antecedente en Compose es el secreto montado en `/run/secrets/<nombre>`. Ver [Docker Compose](docker-compose.md). [T07, p. 11]

## De Compose a Kubernetes

**Inferencia de estudio**, no tabla de la cátedra: equivalencias aproximadas para pasar del despliegue con Compose de la [segunda entrega](segunda-entrega-album-arquitectura.md) a un clúster. Los conceptos de cada columna están en las fuentes citadas; la correspondencia es interpretación.

| En Compose | En Kubernetes | Diferencia que importa |
|---|---|---|
| Nombre del servicio dentro de una red compartida [T07, p. 9] | Service y DNS del clúster [T17, p. 10] [T17, p. 16] | El Service tiene IP virtual y selector; el balanceo es por conexión. [T17, p. 17] |
| `ports` publicados en el host [T07, p. 8] | NodePort, LoadBalancer, Ingress o Gateway [T17, p. 8] [T17, p. 19] | Ingress y Gateway necesitan un controlador instalado. [T17, p. 21] [T17, p. 23] |
| Volumen nombrado [T08, p. 37] | PVC con StorageClass [T17, p. 33] [T17, p. 35] | Topología y retención pasan a ser decisiones explícitas. [T17, p. 29] [T17, p. 31] |
| `environment` / `env_file` [T07, p. 10] | ConfigMap [T17, p. 41] | Las variables de entorno no se actualizan sin recrear el Pod. [T17, p. 48] |
| `secrets` montados en `/run/secrets` [T07, p. 11] | Secret [T17, p. 49] | Sin cifrado *at rest* ni RBAC, quedan expuestos en etcd o a quien cree workloads. [T17, p. 50] [T17, p. 51] |

## Alcance pendiente

`T01` ubicaba en la clase de configuración y acceso volúmenes, StatefulSets, Helm Charts, API gateways (Nginx/Traefik) y Secrets/ConfigMaps. `T17` cubre volúmenes, entrada con Ingress y Gateway API, ConfigMaps y Secrets, pero no desarrolla Helm ni HPA. [T01, p. 12] [T17] `T15` da por visto un deck `api-gateway` usado para interceptar y redirigir tráfico en un Strangler Fig. [T15, p. 5] *Inferencia:* `T17` presenta los recursos de entrada del clúster, pero no ese uso, así que no reemplaza aquel deck. Ver [Dudas y conflictos](../dudas-y-conflictos.md).
