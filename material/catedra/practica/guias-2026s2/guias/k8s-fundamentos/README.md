# Fundamentos de Kubernetes

## Objetivo

Desplegar una aplicación en Kubernetes usando **Minikube en tu máquina** y observar el comportamiento
del sistema ante fallas: self-healing, réplicas y service discovery por DNS.

## Pre-requisitos

- Docker funcionando (clases anteriores)
- Una terminal Linux, WSL2 en Windows, o macOS
- **2 CPUs, 2 GB de RAM libres y ~20 GB de disco** para Minikube
- Familiaridad con la terminal

---

## Parte 1: Preparar el ambiente

### kubectl

Seguir las instrucciones oficiales: https://kubernetes.io/docs/tasks/tools/

En Linux / WSL2:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

En macOS: `brew install kubectl`

### Minikube

Seguir las instrucciones oficiales: https://minikube.sigs.k8s.io/docs/start/

En Linux / WSL2:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

En macOS: `brew install minikube`

> Si tu máquina es ARM (Mac con Apple Silicon, Linux ARM), bajá los binarios `arm64` en lugar de `amd64`.

### Bajar el repositorio

```bash
git clone https://gitlab.com/itba-73-40-microservicios/alumnos/2026s2.git
cd 2026s2/guias/k8s-fundamentos
```

---

## Parte 2: Iniciar Minikube

Minikube levanta un cluster de Kubernetes de **un solo nodo** adentro de un contenedor Docker.

```bash
minikube start --driver=docker
```

> `--driver=docker` quiere decir que el **nodo** corre como un contenedor de tu Docker. Adentro del nodo,
> los pods **no** los corre Docker sino **containerd**: el mismo runtime que usan los clusters
> gestionados (AKS, EKS, GKE). Kubernetes no necesita Docker; necesita imágenes OCI, y Docker es una
> de las herramientas que las construyen.

Verificar el runtime del nodo:

```bash
kubectl get nodes -o wide
# ... CONTAINER-RUNTIME
# ... containerd://2.x.x
```

Verificar que el nodo está listo:

```bash
kubectl get nodes
# NAME       STATUS   ROLES           AGE   VERSION
# minikube   Ready    control-plane   ...
```

Ese único nodo cumple los dos roles: **control plane** (API server, scheduler, etcd, controllers) y
**worker** (kubelet, que corre los pods). Podés ver los componentes del control plane corriendo como pods:

```bash
kubectl get pods -n kube-system
```

Si Minikube se queda colgado durante el inicio:

```bash
minikube delete
minikube start --driver=docker
```

---

## Parte 3: Construir la imagen dentro del cluster

El cluster de Minikube no ve las imágenes de tu Docker local: tiene su propio almacén de imágenes,
el de containerd. Para que Kubernetes pueda usar nuestra imagen, la construimos **directamente adentro
del nodo**:

```bash
minikube image build -t my-python-app:latest app/
```

Verificar que la imagen está disponible para el cluster:

```bash
minikube image ls | grep my-python-app
# docker.io/library/my-python-app:latest
```

> Si ya tenés la imagen construida con tu Docker, también podés copiarla al cluster con
> `minikube image load my-python-app:latest`.

---

## Parte 4: Namespace

En Kubernetes, un **Namespace** agrupa recursos lógicamente dentro del cluster. Es una buena práctica
crear uno por proyecto.

```bash
kubectl create namespace my-namespace
kubectl get namespaces
```

---

## Parte 5: Deployment

El archivo `deployment.yaml` define cómo Kubernetes debe correr nuestra aplicación:

- `replicas: 1` — mantener un pod corriendo en todo momento
- `selector.matchLabels` — identifica qué pods pertenecen a este deployment
- `imagePullPolicy: IfNotPresent` — usar la imagen local sin intentar descargarla

```bash
kubectl apply -f deployment.yaml
kubectl get deployments -n my-namespace
kubectl get pods -n my-namespace
```

El pod tiene que quedar en `Running`. Si queda en `ErrImagePull` o `ImagePullBackOff`, la imagen no está
en el cluster: volvé a la Parte 3.

Para ver los logs de la aplicación:

```bash
kubectl logs -l app=my-python-app -n my-namespace
```

El Deployment no crea los pods directamente: crea un **ReplicaSet**, y es el ReplicaSet el que mantiene
la cantidad de pods.

```bash
kubectl get replicasets -n my-namespace
```

---

## Parte 6: Service

Los pods son efímeros: cada vez que se recrean cambian de nombre y de IP. Un **Service** les da una
dirección estable y reparte el tráfico entre todos los pods que matcheen su `selector`.

Usamos `NodePort`, que además expone un puerto en el nodo (en este caso, Minikube) para poder
llegar desde afuera del cluster.

```bash
kubectl apply -f service.yaml
kubectl get services -n my-namespace
```

Obtener la URL de acceso:

```bash
minikube service my-python-app-service -n my-namespace --url
```

> Con el driver Docker en macOS o WSL2, este comando abre un túnel y **se queda corriendo**.
> Dejalo abierto y usá otra terminal para los `curl`.

Probar los endpoints (si da `Connection refused` justo después del `apply`, esperá unos segundos):

```bash
curl http://<IP>:<PUERTO>/
# Hello, World!

curl http://<IP>:<PUERTO>/users
# [{"name":"Alice"},{"name":"Bob"},{"name":"Charlie"}]
```

---

## Parte 7: Service Discovery por DNS

Adentro del cluster nadie usa IPs: cada Service tiene un **nombre DNS** que resuelve CoreDNS. Para
comprobarlo necesitamos estar **adentro** del cluster, así que levantamos un pod de debug con `curl`
que se queda corriendo:

```bash
kubectl run debug --image=curlimages/curl -n my-namespace --command -- sleep infinity
kubectl wait --for=condition=Ready pod/debug -n my-namespace
```

Ahora ejecutamos `curl` **dentro** de ese pod y le pegamos al Service por nombre:

```bash
kubectl exec debug -n my-namespace -- curl -s http://my-python-app-service/users
# [{"name":"Alice"},{"name":"Bob"},{"name":"Charlie"}]
```

Notá que usamos el puerto **80** del Service (`port`), no el 5000 del contenedor (`targetPort`).

> Si falla con `Could not resolve host` y el cluster tiene menos de un minuto, CoreDNS todavía está
> arrancando: esperá unos segundos y reintentá.

¿Por qué alcanza con el nombre corto? Mirá la configuración de DNS del pod:

```bash
kubectl exec debug -n my-namespace -- cat /etc/resolv.conf
# search my-namespace.svc.cluster.local svc.cluster.local cluster.local
# nameserver 10.96.0.10
```

El `search` completa el nombre: `my-python-app-service` se prueba como
`my-python-app-service.my-namespace.svc.cluster.local`. Por eso el nombre corto solo funciona **dentro
del mismo namespace**. Desde otro namespace hay que usar el nombre completo
`<service>.<namespace>.svc.cluster.local`:

```bash
kubectl exec debug -n my-namespace -- curl -s http://my-python-app-service.my-namespace.svc.cluster.local/
# Hello, World!
```

Esto es lo que van a usar sus microservicios para hablarse entre sí cuando estén en Kubernetes.

---

## Parte 8: Self-Healing ✨

Esto es lo que hace a Kubernetes especial. El **ReplicaSet** compara continuamente el estado deseado
(`replicas`) con el estado real, y corrige la diferencia.

Abrir **dos terminales**:

**Terminal 1** — observar los pods en tiempo real:

```bash
kubectl get pods -n my-namespace -w
```

**Terminal 2** — borrar el pod a mano:

```bash
# Obtener el nombre de un pod de la app (filtrando por label)
kubectl get pods -n my-namespace -l app=my-python-app

# Borrarlo
kubectl delete pod <nombre-del-pod> -n my-namespace
```

Observar en la Terminal 1 cómo Kubernetes detecta la falla y recrea el pod automáticamente en segundos.
El pod nuevo tiene **otro nombre** — y el Service lo sigue encontrando, porque selecciona por label:

```bash
kubectl exec debug -n my-namespace -- curl -s http://my-python-app-service/
# Hello, World!
```

Ahora hacé lo mismo con el pod `debug`:

```bash
kubectl delete pod debug -n my-namespace
kubectl get pods -n my-namespace
```

No vuelve. `debug` es un pod "suelto": no hay ningún Deployment ni ReplicaSet que declare que tiene que
existir. **El self-healing no es del pod, es del controlador que lo administra.** Si querés seguir
usándolo en lo que queda, volvé a crearlo con el `kubectl run` de la Parte 7.

### Escalar el deployment

```bash
# Aumentar a 3 réplicas
kubectl scale deployment my-python-app --replicas=3 -n my-namespace
kubectl get pods -n my-namespace -w

# Ahora borrar uno de los pods y ver qué pasa
kubectl delete pod <nombre-del-pod> -n my-namespace
```

Ver las IPs de los pods que hay detrás del Service:

```bash
kubectl get endpointslices -n my-namespace
```

---

## Limpieza

```bash
kubectl delete namespace my-namespace
minikube stop
```

`minikube stop` conserva el cluster para la próxima; `minikube delete` lo borra por completo.

---

## Desafío Extra

1. **Configurar liveness probe**: agregar un `livenessProbe` en el `deployment.yaml` que haga un HTTP GET
   a `/` cada 10 segundos. Simular una falla haciendo que el endpoint devuelva un error 500 y observar
   cómo Kubernetes reinicia el contenedor (`kubectl describe pod` y la columna `RESTARTS`).

2. **Rolling update sin downtime**: escalar a 3 réplicas, cambiar el mensaje del endpoint `/` en
   `main.py`, rebuildear la imagen con un nuevo tag (`my-python-app:v2`), actualizar el deployment y
   verificar que el servicio no tuvo downtime durante la actualización (dejar un `curl` en loop
   corriendo mientras tanto). Probar después `kubectl rollout undo`.
