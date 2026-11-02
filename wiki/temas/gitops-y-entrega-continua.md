# GitOps y entrega continua

## Fuente y alcance

Síntesis de `T20`, presentación de la cátedra con los docentes de la materia. **Inferencia de catalogación:** el ciclo 2026 se asigna por la configuración de la materia y la fecha de creación del PDF; no figura en la portada. [T20, p. 1] Tiene tres partes: principios de GitOps, ArgoCD y Flux. Continúa [DevOps y CI/CD](devops-y-ci-cd.md): el pipeline sigue construyendo y probando, pero el despliegue pasa a ser la reconciliación de un estado declarado en Git. El cronograma de `T01` ubicaba esta clase después de la de configuración y acceso, con "principios de GitOps, herramientas (ArgoCD/Flux) y automatización de despliegues vía repositorios Git". [T01, p. 12]

## Qué es GitOps

Un **modelo de operación para sistemas declarativos**: el estado deseado se expresa en archivos versionados, un agente lo obtiene automáticamente y un reconciliador hace converger el estado real. Los cuatro principios de OpenGitOps son: [T20, p. 3]

1. **Declarativo.**
2. **Versionado e inmutable.**
3. **Obtenido automáticamente** (*pulled automatically*).
4. **Reconciliado continuamente.**

**El cambio de paradigma.** La unidad de operación deja de ser "ejecutar pasos": se declara un estado objetivo y el sistema de reconciliación observa la realidad y corrige la divergencia. El cambio parece menor, pero modifica la seguridad, la auditoría, el rollback, la propiedad del cambio y la respuesta a incidentes. [T20, p. 4]

**Describir el resultado, no el procedimiento.** En un pipeline imperativo el despliegue es una secuencia de acciones; en GitOps el manifiesto describe cómo debe quedar el sistema, y Kubernetes y el controlador GitOps calculan y ejecutan la convergencia. Eso alinea GitOps con el modelo de controladores de Kubernetes. [T20, p. 5] El diagrama de la lámina pone Infrastructure as Code, Configuration as Code y "X as Code" en Git, y de ahí CI/CD hacia la nube. [T20, p. 5] Ver la reconciliación en [Kubernetes — Parte 1](kubernetes-parte-1.md#reconciliación-y-consistencia-eventual). [T16, p. 40]

| Imperativo | Declarativo |
|---|---|
| `kubectl set image deployment/payments-api api=registry/payments-api:v2` y `kubectl rollout status` | Un Deployment `payments-api` con `replicas: 4` e `image: registry/payments-api@sha256:...` |

Fuente: [T20, p. 6].

### Trade-offs de lo declarativo

| Ventaja | Costo | Criterio |
|---|---|---|
| Estado deseado auditable y reproducible | Los pasos intermedios quedan menos explícitos | Hooks, *sync waves* o Jobs cuando hay un orden operacional estricto |
| Rollback por revert del estado deseado | No siempre revierte datos, migraciones o efectos externos | Separar el rollback de aplicación, el de datos y el de tráfico |
| Menos scripts ad hoc | Puede ocultar una operación destructiva detrás de un YAML inocente | Revisión de PR y *policy-as-code* para cambios sensibles |

Fuente: [T20, p. 7].

## Versionado e inmutabilidad

Git funciona como registro de decisiones: cada cambio debería responder quién lo propuso y quién lo aprobó, qué se modificó y cuándo se aplicó, y cómo volver a la versión previa. **Pero Git solo versiona lo que está declarado:** si el registry permite sobrescribir tags, Git puede seguir diciendo lo mismo mientras el artefacto real cambió. [T20, p. 8]

| Referencia | Evaluación |
|---|---|
| Tag mutable | Cómodo para humanos y riesgoso para auditoría: `v1.4.2` puede apuntar a otra imagen si el registry lo permite. [T20, p. 9] |
| Digest | Más incómodo de leer, pero identifica el artefacto exacto; es la opción sólida en producción. [T20, p. 9] |
| SemVer + digest | Compromiso práctico: semántica para humanos y reproducibilidad para máquinas. [T20, p. 9] |

Ver tags y digests en [Imágenes de contenedores](imagenes-de-contenedores.md). [T06, p. 36]

## Push o pull

La diferencia no es solo de implementación: cambia el modelo de seguridad y de red. En **push**, el sistema de CI entra al clúster y aplica los cambios; en **pull**, un agente dentro del clúster sale a buscar el estado deseado y lo reconcilia. [T20, p. 10]

| | Push | Pull |
|---|---|---|
| A favor | Simple, familiar, baja latencia, fácil de integrar con el CI existente | Menor exposición del API server, credenciales del clúster dentro del clúster, reconciliación continua, mejor postura de seguridad |
| Riesgos | Kubeconfigs en los runners, API server expuesto, mayor *blast radius* si se compromete el CI | Más componentes, consistencia eventual, depurar el controlador, dependencia del agente GitOps |
| Tiene sentido en | Laboratorios, ambientes efímeros, organizaciones chicas, despliegues no críticos | Producción, multi-clúster, sectores regulados, redes cerradas, separación fuerte entre CI y CD |

Fuente: [T20, p. 11].

## Reconciliación y drift

El loop de GitOps extiende el de Kubernetes: leer el estado deseado, leer el real, comparar, aplicar la diferencia, reportar y repetir. **Drift** es la divergencia entre lo declarado y lo observado, y no todo drift es igual: puede ser peligroso, esperado o producido por otro controlador. [T20, p. 12]

| Fuente de drift | Ejemplo | Cómo tratarlo |
|---|---|---|
| Cambio manual | `kubectl edit` en producción | Que quede auditado y después vuelva a Git o se revierta |
| Controlador legítimo | El HPA cambia `replicas` | Que no se pelee con GitOps: ignorar el campo o delegar su propiedad |
| Webhook o mutación | El service mesh inyecta un sidecar | Diferencia esperada si la gobierna la plataforma |
| Defaulting de Kubernetes | Campos que completa el API server | No necesariamente es un problema |
| Secreto externo | External Secrets rota un Secret | Git no contiene el valor, sí la intención de consumirlo |

Fuente: [T20, p. 13].

### Antipatrones

- **Git como basurero de YAML:** manifiestos sin dueño, revisión ni contrato de promoción.
- **GitOps sin inmutabilidad:** tags mutables, creyendo que el commit alcanza para reproducir.
- **Self-heal sin política de incidentes:** el controlador revierte cambios manuales necesarios durante una emergencia.
- **Secretos en claro:** confundir versionado con exponer datos sensibles.
- **CI con poder total y GitOps decorativo:** kubeconfigs en el CI y ArgoCD instalado solo para mirar dashboards.

Fuente: [T20, p. 14].

## ArgoCD

ArgoCD es un **control plane GitOps orientado a aplicaciones**. Modela el despliegue con el recurso `Application`, que une una fuente declarativa (repo, revisión y path) con un destino en Kubernetes (clúster y namespace). [T20, p. 16]

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payments-api
  namespace: argocd
spec:
  source:
    repoURL: https://git.example.com/platform/manifests
    targetRevision: main
    path: apps/payments-api/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: payments
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Fuente: [T20, p. 17]. *Inferencia, por conocimiento general sobre ArgoCD y no por la lámina:* `prune` borra del clúster los recursos que ya no están en Git, y `selfHeal` revierte los cambios hechos fuera de Git. El path `overlays/prod` sugiere una base común con overlays de Kustomize por ambiente.

El diagrama de arquitectura muestra al desarrollador haciendo commit en Git, y a ArgoCD trayendo los cambios de Git y sincronizando el estado del clúster de Kubernetes. [T20, p. 18]

| Componente | Función | Riesgo |
|---|---|---|
| `argocd-server` | API gRPC/REST, UI, CLI, SSO, RBAC, sync y rollback, credenciales | Superficie de administración: mal expuesto, es una entrada privilegiada a producción |
| `argocd-repo-server` | Clona repositorios, cachea y genera manifiestos con Helm, Kustomize o Jsonnet | Puede saturarse por generación concurrente, repos grandes o templates pesados |
| `application-controller` | Compara el estado deseado con el vivo y ejecuta la reconciliación | Es el corazón: una mala configuración produce syncs peligrosos o lentos |
| Redis | Caché para reducir llamadas al API server y a Git | No es fuente de verdad, pero afecta la performance |
| ApplicationSet Controller | Genera Applications a escala para muchos clústeres o namespaces | Multiplica los errores si el template o el generador están mal |

Fuentes: [T20, p. 18] [T20, p. 19]. La lámina 16 también menciona SSO/Dex y las notificaciones. [T20, p. 16]

**Si ArgoCD cae**, las aplicaciones siguen corriendo, pero se pierden la sincronización, la visibilidad, la detección de drift y la operación declarativa. No es "un Pod que hace pull de Git", sino un control plane de entrega continua. [T20, p. 20]

| Ventajas | Precauciones |
|---|---|
| Visibilidad muy buena para los equipos | La UI puede tentar a operar a mano si no se gobierna con RBAC y proceso |
| Modelo `Application` fácil de explicar | Control plane centralizado: hay que diseñar el *blast radius* |
| Soporte para Helm, Kustomize, multi-clúster, SSO y RBAC | Hay que operar sus componentes: repo-server, controller, Redis, credenciales y HA |
| Útil para entender reconciliación, drift y sync | Puede volverse un "sistema central mágico" si el equipo no entiende Kubernetes |

Fuente: [T20, p. 21].

## Flux

Flux es una CLI más una serie de controladores de Kubernetes. Su **GitOps Toolkit** separa responsabilidades en controladores independientes, cada uno reconciliando sus CRDs: un diseño más granular y nativo de Kubernetes, más natural para equipos de plataforma maduros. [T20, p. 23]

| Controlador | Función |
|---|---|
| `source-controller` | Obtiene fuentes desde Git, repositorios de Helm u OCI y produce artefactos |
| `kustomize-controller` | Reconcilia Kustomizations y aplica los manifiestos ensamblados |
| `helm-controller` | Gestiona `HelmRelease` como recurso declarativo: Helm operado como reconciliador, no como comando manual |
| `notification-controller` | Emite eventos e integra webhooks; la operación se vuelve observable |
| Image reflector / automation | Detecta imágenes nuevas y propone cambios declarativos; requiere gobierno para no automatizar promociones inseguras |

Fuente: [T20, p. 24]. Ver [Helm](helm.md).

## ArgoCD o Flux

| | ArgoCD | Flux |
|---|---|---|
| Experiencia | UI fuerte, modelo `Application` claro, incorporación rápida | CLI, eventos y CRDs |
| Arquitectura | Control plane GitOps visible y centralizado | Controladores componibles dentro del modelo de Kubernetes |
| Multi-clúster | Muy práctico con UI, clústeres registrados y ApplicationSets | Muy sólido con el GitOps Toolkit, pero menos visual |
| Riesgo | Centralización y credenciales amplias si no se segmenta | Complejidad distribuida: el troubleshooting exige entender varios controladores |

Fuente: [T20, p. 25]. Las fuentes citadas por la presentación son OpenGitOps y la documentación de arquitectura de ArgoCD y de Flux. [T20, p. 26]

## Relación con otras unidades y con el TP

- **Estrategias de despliegue.** GitOps no reemplaza a rolling, blue/green o canary: decide cómo llega el estado deseado al clúster. Argo Rollouts y Flagger automatizan el canary y el rollback según SLO. [T21, p. 29] Ver [DevOps y CI/CD](devops-y-ci-cd.md#estrategias-de-despliegue).
- **Multi-clúster.** ApplicationSet o Flux generan una Application por clúster, con base común y overlays por región, y conviene desplegar por olas. [T21, p. 44] Ver [Resiliencia y escalabilidad](resiliencia-y-escalabilidad.md).
- **Tercera entrega.** `E06` impone GitOps *pull* con ArgoCD: una `Application` por ambiente (`dev`, `staging`, `prod`), el pipeline sin credenciales del clúster, promoción por commits al estado deseado, sincronización automática en `dev` y `staging` y controlada en `prod`, y rollback por `git revert`. [E06] Ver [Tercera entrega](tercera-entrega-album-cicd-aks.md).
- **Del laboratorio al TP.** El pipeline de `G03` simula un `kubectl set image` desde el CI, es decir, el modelo push. [G03] *Inferencia:* para la tercera entrega ese paso se reemplaza por un commit del nuevo digest en el repositorio de estado deseado.
