# Helm

## Fuente y alcance

Síntesis de `T19`, presentación breve de la cátedra (11 láminas, una de ellas un video). Su portada lleva el título general "Arquitectura de Microservicios con Contenedores", los docentes y el curso Microservicios (73.40), ITBA. **Inferencia de catalogación:** el ciclo 2026 se asigna por la configuración de la materia y la fecha de creación del PDF; no figura en la portada. [T19, p. 1] [T19, p. 2] El cronograma de `T01` ubicaba los Helm Charts en la clase de configuración y acceso, junto con volúmenes, StatefulSets, API gateways y Secrets. [T01, p. 12]

## El problema

Desplegar una aplicación en Kubernetes implica construir y subir la imagen, y crear el Deployment, el Service, el volumen y la persistencia, los ConfigMaps, los Secrets, el Ingress y las variables de entorno. Modificar después esa estructura desplegada no es sencillo. [T19, p. 3]

## Qué es

Helm es un **gestor de paquetes para Kubernetes**: facilita instalar, actualizar y administrar aplicaciones, como `npm` o `apt-get`. Hay paquetes de todo tipo, como GitLab, Jenkins, Grafana, nginx o MySQL. [T19, p. 4]

| Concepto | Qué es |
|---|---|
| Chart | El paquete: todos los recursos necesarios para ejecutar una aplicación (Deployment, Service, etc.), configurables. [T19, p. 5] |
| Release | Una instalación concreta de un chart. Está versionada, lo que facilita el rollback. [T19, p. 5] |
| Repositorio | Fuente desde la que se descargan e instalan charts, y que se actualiza con facilidad. [T19, p. 5] |

## Uso

**Repositorios.** `helm repo add bitnami https://charts.bitnami.com/bitnami` agrega el repositorio y `helm search repo bitnami` lista sus charts con dos versiones separadas: la del chart (`CHART VERSION`) y la de la aplicación empaquetada (`APP VERSION`). Por ejemplo, `bitnami/argo-cd` 7.0.14 empaqueta Argo CD 2.12.4. [T19, p. 6]

**Instalar un chart.** `helm install my-postgresql bitnami/postgresql --version 15.5.37` crea, según la salida de `kubectl get all` y `kubectl get pvc`: [T19, p. 7]

- un StatefulSet `my-postgresql` con el Pod `my-postgresql-0`;
- un Service ClusterIP `my-postgresql` y uno headless `my-postgresql-hl` en el puerto 5432;
- un PVC `data-my-postgresql-0` de 8 Gi, `RWO`, en la StorageClass `standard`.

*Inferencia de estudio:* es el mismo patrón que se arma a mano en la guía de AKS (StatefulSet, Service headless y `volumeClaimTemplates`), resuelto por un solo comando. [G09] [T16, p. 33] Ver [Kubernetes en AKS](kubernetes-en-aks.md).

**Chart propio.** `helm create my-python-app` genera esta estructura: [T19, p. 8]

```text
my-python-app
├── charts
├── templates
│   ├── tests
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── hpa.yaml
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── service.yaml
│   └── serviceaccount.yaml
├── .helmignore
├── Chart.yaml
└── values.yaml
```

El `values.yaml` generado fija, entre otros, `replicaCount: 1`, `image.repository`, `image.pullPolicy: IfNotPresent`, `image.tag: "2.0.0"`, `imagePullSecrets` y los *overrides* de nombre. [T19, p. 8] Los templates leen esos valores, así que el mismo chart se instala con valores distintos por ambiente. *Inferencia:* la lámina no muestra la sintaxis de los templates ni un archivo de valores por ambiente.

## Beneficios

Confiabilidad de los repositorios, creación y gestión simplificadas de recursos, versionado de releases y rollback, facilidad para instalar y actualizar, y configuración personalizable. [T19, p. 9] La lámina 10 remite a un video tutorial externo. [T19, p. 10]

## Helm y GitOps

`T19` presenta Helm operado por comandos (`helm install`, y el rollback de la release). [T19, p. 5] [T19, p. 7] La unidad de [GitOps](gitops-y-entrega-continua.md) lo pone en otro rol: el `argocd-repo-server` genera los manifiestos a partir de charts, y el `helm-controller` de Flux gestiona un `HelmRelease` como recurso declarativo, "Helm operado como reconciliador, no como comando manual". [T20, p. 18] [T20, p. 24]

En la [tercera entrega](tercera-entrega-album-cicd-aks.md), Helm es una opción para la base común con valores por ambiente, pero el pipeline **nunca** ejecuta `helm upgrade` contra el cluster, y un rollback con `helm` directo no cuenta: se hace por Git. [E06] *Inferencia:* el rollback por release que destaca `T19` queda reemplazado en el TP por un `git revert` del estado deseado.
