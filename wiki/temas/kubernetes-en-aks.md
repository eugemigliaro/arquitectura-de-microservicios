# Kubernetes en AKS: el cluster del TP

## Fuente y alcance

Síntesis de `G09`, la guía de la clase 11 del [repositorio de laboratorios](laboratorios-de-la-catedra.md). Pasa de Minikube a AKS: el mismo Kubernetes, pero Azure opera el control plane y crea en la nube lo que un manifiesto pide y no existe dentro del clúster, como una IP pública o un disco. La guía termina en dos scripts, `up.sh` y `down.sh`, y declara que **ese es el cluster que se usa para el TP**: se levanta para trabajar y se destruye al terminar. [G09]

Los manifiestos, scripts y archivos Terraform que se mencionan acá están en el clon local `laboratorio/catedra-2026s2/guias/k8s-aks/`, en el commit `36b9189`; no se versionan en este repositorio.

**Relación con el TP.** `E01` exige producción en AKS y solo acepta la entrega desplegada ahí; la segunda entrega usa Docker Compose según `E05`. [E01, p. 6] [E05] `G09` refuerza la inferencia de que la entrega final del Álbum se despliega en este cluster. Ver [Dudas y conflictos](../dudas-y-conflictos.md) y [Segunda entrega](segunda-entrega-album-arquitectura.md).

## Costo y hábito de trabajo

- Azure for Students da US$100 de crédito. Mientras el cluster exista se cobran nodos, IPs públicas, balanceador y discos aunque no se usen; el control plane en el tier Free no se cobra. [G09]
- El cluster de la guía cuesta unos US$0,30 por hora: hacer la guía completa sale menos de US$1, olvidarlo prendido un fin de semana unos US$18 y un mes entero consume todo el crédito. [G09]
- El hábito es **levantar, trabajar, destruir**. Todo lo necesario tiene que estar en el repo (manifiestos, scripts), porque el cluster no sobrevive de un día para el otro. El consumo se ve en el portal con un día de demora. [G09]

## Preparación

| Requisito | Detalle |
|---|---|
| Entorno | Linux, macOS o WSL2. En WSL hay que trabajar dentro de `/home/...` e instalar todas las herramientas adentro, porque si `az` o `kubectl` son los de Windows el kubeconfig queda fuera de WSL. [G09] |
| Herramientas | Azure CLI, Terraform ≥ 1.6, `kubectl` ≥ 1.35, Helm 3, git, curl y openssl. El cluster sale con Kubernetes 1.36 y `kubectl` se soporta hasta una versión menor de distancia. [G09] |
| Cuenta | Azure for Students activa con el mail del ITBA y los providers `Microsoft.ContainerService`, `Compute`, `Network` y `Storage` registrados. [G09] |
| Región y VM | Una policy permite solo cinco regiones, distintas para cada suscripción; no todas ofrecen los mismos tamaños; la cuota es de 6 vCPU por región para las familias `Bsv2` y `Basv2`. `elegir-region.sh` busca una región permitida con `Standard_B2s_v2` o `Standard_B2as_v2` y la guarda en `terraform/terraform.tfvars`. La lejanía de la región no importa, porque el tráfico entre servicios queda dentro del cluster. [G09] |

## El cluster y la teoría

La creación a mano con `az aks create` usa estas opciones: [G09]

| Opción | Qué pide | Relación con la teoría |
|---|---|---|
| `--tier free` | Control plane sin costo y sin SLA; en producción se usa `standard`. [G09] | — |
| `--node-count 2 --node-vm-size` | Dos VMs como nodos, que son las que consumen crédito. [G09] | Nodos que ejecutan kubelet y runtime. [T16, p. 9] |
| `--network-plugin azure --network-plugin-mode overlay` | CNI con IPs de Pods en una red propia, separada de la de las VMs. [G09] | La guía lo vincula con "Overlay o ruteo nativo". [T17, p. 6] |
| `--network-dataplane cilium` | Cilium, con eBPF, reemplaza a kube-proxy y hace cumplir las NetworkPolicy. [G09] | Reemplazo de kube-proxy y enforcement de políticas. [T17, p. 15] [T17, p. 25] |

Azure crea dos resource groups: el propio (`rg-figuritas`) y uno `MC_...` que administra AKS, con el VM Scale Set de los nodos, la red, el balanceador `kubernetes` y la IP pública de salida. Ahí aparecen después las IPs y los discos que pidan los manifiestos. [G09]

**Dónde está el control plane.** La API responde en una URL `*.azmk8s.io`, fuera de los nodos. En `kube-system` no hay `etcd`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager` ni `kube-proxy`; sí hay `konnectivity-agent` (el túnel del control plane hacia los nodos), `cloud-node-manager` y los drivers CSI de Azure Disk y Azure Files. En los nodos siguen kubelet y containerd, que se ven con `kubectl debug node/...`, `chroot /host` y `crictl ps`. [G09] En Minikube, en cambio, un solo nodo cumple ambos roles y el control plane se ve como Pods en `kube-system`. [G08]

En 2026S1 el cluster se armaba sobre VMs con `kubeadm`: preparar kernel y swap, instalar runtime, `kubeadm init`, Calico, `kubeadm join` y copiar el kubeconfig. AKS resuelve todo eso con opciones y `az aks get-credentials`; kubelet y runtime siguen ahí, solo cambia quién los instala y mantiene. [G09]

## Lo que la nube crea por vos

| Se declara | Azure crea | Quién lo hace |
|---|---|---|
| Service `LoadBalancer` | Una IP pública `kubernetes-<id>` y una regla en el balanceador; `EXTERNAL-IP` aparece en unos 30 segundos, mientras que en Minikube queda en `<pending>`. [G09] | El cloud controller de AKS. [G09] |
| PVC sin `storageClassName` | Un Azure Disk `pvc-...` en estado `Attached`, con el mismo nombre que el PV. [G09] | El driver CSI de la StorageClass default. [G09] |
| Gateway de clase `eg` | Una IP pública para el proxy de Envoy; la del Service `web` desaparece al pasarlo a ClusterIP. [G09] | Envoy Gateway, instalado con Helm. [G09] |

- **Balanceo.** El header `X-Servido-Por` muestra los nombres de los dos Pods de `web`, no necesariamente alternados, porque el balanceo es por conexión y al azar. [G09] Coincide con que un Service no garantiza round-robin. [T17, p. 17]
- **StorageClasses de AKS.** Las `managed*` y `default` crean Azure Disks, uno por PVC, enganchados a un solo nodo (`ReadWriteOnce`). Las `azurefile*` crean shares de Azure Files que varios Pods montan a la vez (`ReadWriteMany`). Con `WaitForFirstConsumer` el disco nace en la zona del nodo donde cae el Pod. [G09] [T17, p. 35]
- **Secret.** La contraseña de Postgres se crea desde la línea de comandos con `kubectl create secret generic ... --from-literal` y `openssl rand`, nunca en un archivo del repo. Decodificarla con `base64 --decode` muestra que es codificación, no cifrado: cualquiera con permiso de leer Secrets del namespace la ve. [G09] [T17, p. 51]
- **Postgres como StatefulSet.** El manifiesto `20-postgres.yaml` combina un ConfigMap (base, usuario y `PGDATA`), el Secret `postgres-cred` con `envFrom`, un Service headless y `volumeClaimTemplates` de 4 GiB `ReadWriteOnce`. `PGDATA` va en una subcarpeta porque el disco recién formateado trae `lost+found` y Postgres no inicializa en un directorio no vacío. [G09]
- **Los datos sobreviven al Pod.** Al borrar `postgres-0`, el StatefulSet crea un Pod nuevo que monta el mismo PVC `data-postgres-0`, y la tabla sigue. [G09] [T16, p. 33]

**Inferencia de estudio:** el manifiesto `10-web.yaml` monta el HTML y la configuración de nginx desde un ConfigMap con `subPath`, y un montaje con `subPath` no recibe actualizaciones del ConfigMap. [T17, p. 48] Cambiar ese ConfigMap obliga a recrear los Pods.

## NetworkPolicy

Sin políticas, un Pod "intruso" llega a Postgres (`accepting connections`). El manifiesto `30-networkpolicies.yaml` aplica tres reglas: [G09]

1. *Default deny* de ingress para todo el namespace.
2. La web acepta el puerto 80 desde cualquier origen.
3. Postgres solo acepta Pods con la label `acceso-db=true`.

Después, el intruso da `no response`, un Pod con la label conecta y la web sigue respondiendo `200`. Las reglas las hace cumplir Cilium; con un CNI sin soporte, Kubernetes acepta el objeto y no pasa nada. [G09] [T17, p. 25] [T17, p. 26]

**Inferencia de estudio:** como la regla de Postgres usa solo `podSelector`, autoriza Pods del mismo namespace. En el TP, cada servicio que acceda a su base necesita esa label o una política propia.

## Un solo punto de entrada con Gateway API

Un Service `LoadBalancer` por microservicio implica una IP pública, con su costo, por servicio. Lo habitual es un único punto de entrada que rutea por path o host, y para eso hay que instalar una implementación de Gateway API: la guía usa **Envoy Gateway** (Helm, versión `v1.9.2`, namespace `envoy-gateway-system`). [G09] [T17, p. 23]

- La `GatewayClass` `eg` es infraestructura del cluster: indica que los Gateway de esa clase los implementa Envoy Gateway. [G09]
- `40-gateway.yaml` declara el Gateway `figuritas` con un listener HTTP en el puerto 80, una `HTTPRoute` con `PathPrefix /` hacia `web:80` y vuelve a dejar el Service `web` como ClusterIP. [G09]
- **En el TP, cada microservicio es una `HTTPRoute` más contra el mismo Gateway.** [G09]

## Dejarlo listo para el TP

| Tema | Qué indica la guía |
|---|---|
| Imágenes privadas de GitLab | Crear un deploy token con scope `read_registry`, un Secret `docker-registry` llamado `gitlab-registry` y agregarlo como `imagePullSecrets` al ServiceAccount `default` del namespace. El token nunca va al repo; como el cluster se destruye, conviene un script del grupo que lo lea de una variable de entorno. [G09] |
| Compartir el cluster | Un integrante lo levanta y genera un kubeconfig aparte, que da acceso total: se comparte por un canal privado y se agrega `kubeconfig*` al `.gitignore` del TP. La alternativa es que cada integrante levante su cluster con su crédito. [G09] |
| Apagar | `az group delete` borra el cluster y el grupo `MC_...` en unos 8 minutos. El disco de Postgres se borra con él: los datos de prueba deben poder recrearse con un seed o una migración. [G09] |
| Scripts | `up.sh` tarda unos 10 minutos: `terraform apply` (resource group y AKS), credenciales y espera de nodos `Ready`, y Envoy Gateway con la `GatewayClass` `eg`. Los nombres llevan un sufijo aleatorio y el grupo de nodos termina en `-nodos`. `down.sh` hace `terraform destroy`, limpia el contexto de `kubectl` y lista los grupos que quedan. [G09] |
| Estado de Terraform | `terraform.tfstate` vive en la máquina de quien corrió `up.sh` y contiene el kubeconfig de administrador en texto plano: no se comparte ni se commitea. Si se pierde, se borra el grupo con `az group delete`. [G09] |

El ServiceAccount `default` es la identidad que usan los Pods sin cuenta explícita. Ver [Kubernetes — Parte 1](kubernetes-parte-1.md). [T16, p. 53]

## Desafío: un nodo que se va

Con Postgres desplegado, `kubectl drain` sobre su nodo obliga a reprogramarlo en el otro. La guía pide medir cuánto tarda en volver a estar `Ready`, por qué tarda más que un Pod de `web`, buscar el evento `FailedAttachVolume ... Volume is already exclusively attached to one node`, comprobar si siguen los datos y ver si vuelve solo al nodo original después de `uncordon`. [G09]

*Inferencia:* el Pod de `web` no tiene estado y arranca en cualquier nodo. El de Postgres depende de un Azure Disk `ReadWriteOnce`, que primero debe desengancharse del nodo viejo para engancharse en el nuevo. [T17, p. 29] [T17, p. 31]

## Problemas frecuentes

| Síntoma | Causa probable según la guía |
|---|---|
| `RequestDisallowedByPolicy` | Región no permitida: correr `elegir-region.sh`. [G09] |
| `NotAvailableForSubscription` / `SkuNotAvailable` | Tamaño de VM no ofrecido en esa región. [G09] |
| Cuota de vCPU excedida | Quedó prendido otro cluster; borrarlo o usar `-var="node_count=1"`. [G09] |
| `EXTERNAL-IP <pending>` por más de 3 minutos | Contexto equivocado, por ejemplo Minikube. [G09] |
| Postgres en `CrashLoopBackOff` con "directory not empty" | Falta `PGDATA` en subcarpeta. [G09] |
| `kubectl` responde `no such host` | El cluster ya fue destruido. [G09] |
