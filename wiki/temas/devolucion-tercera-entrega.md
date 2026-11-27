# Devolución de la tercera entrega: qué corregir para la entrega final

## Propósito y fuente

`D03` es la corrección de la cátedra a la [tercera entrega](tercera-entrega-album-cicd-aks.md) del grupo 10. Evaluó el commit `ea67803` del repositorio del TP con la consigna `E06` v1.1. Los servicios reales declarados fueron `acceso-e-identidad` y `coleccion-y-sobres`, y los otros cuatro eran mocks. [D03]

La cátedra verificó con `glab` cada evidencia declarada: pipelines, jobs, artefactos, permisos del proyecto, deploy tokens, variables y miembros. Además dejó constancia de que no ejecutó código del grupo ni escribió en GitLab o en el cluster. [D03]

Importa para la entrega final por tres razones: los cuatro ítems parciales siguen abiertos, hay una lista de cosas que se van a preguntar en la defensa y aparecen dos credenciales con vencimiento que rompen el despliegue si no se renuevan. [D03]

## Resultado

Los tres gates de forma (G0, G1 y G2) pasaron, el veredicto global es **PASS** y no hubo hallazgos bloqueantes. [D03] El texto de `D03` no trae nota numérica. *Dato informado por el grupo, sin fuente catalogada:* la nota fue 10.

| Sección | Peso | Veredicto | Ítems parciales |
|---|---:|---|---|
| C1 Cluster AKS y ambientes | 20 % | PASS | C1.3 |
| C2 CI y testing de los servicios reales | 20 % | PASS | — |
| C3 Pipeline de CD y gates (GitOps) | 20 % | PASS | C3.6, C3.11 |
| C4 Tests de integración | 20 % | PASS | — |
| C5 Completitud del despliegue | 10 % | PASS | — |
| C6 Documentación y reproducibilidad | 10 % | PASS | C6.4 |

Fuente: [D03]. Las secciones y los pesos coinciden con los criterios de corrección de `E06`. [E06]

## Qué valoró la cátedra

Conviene no romper estos rasgos al pasar los mocks a servicios reales:

- **GitOps pull puro.** El CI solo publica en ACR con identidad federada y permiso `AcrPush`, escribe el estado deseado con el job token y consulta ArgoCD con una cuenta que solo puede leer aplicaciones. Ni `.gitlab-ci.yml` ni `ci/` usan `kubectl`, `helm` o `az aks`. [D03]
- **Demostraciones negativas verificables:**
  - Un pipeline que falla en unitarios no publica la imagen ni promueve.
  - Cuando falla IT-05 en `staging`, `promover-prod` queda omitido y prod no recibe ningún commit en ese intervalo. [D03]
- **Imagen inmutable promovida por digest.** El tag `<sha>-<job>` se bloquea en el ACR, `staging` y `prod` declaran el mismo bloque `images` y no se usa `:latest`. [D03]
- **Cobertura separada.** Unitaria y funcional, con un gate de 80 % en cuatro métricas y valores reales de entre 91,7 % y 99,8 %. Los funcionales corren contra Postgres y Kafka reales. [D03]
- **Suite de integración repetible.** Tokens reales de Google obtenidos por *refresh grant*, siembra y limpieza con Jobs pedidos por Git y una prueba de concurrencia real: tres aperturas en paralelo en IT-09. [D03]
- **Aislamiento entre ambientes.** Base, Kafka, PVC, SealedSecrets, gateway y emisor de sesión propios por namespace, `NetworkPolicy` default-deny y una prueba de que los cruces entre ambientes están bloqueados. [D03]
- **Migraciones como hooks de ArgoCD.** Corren antes de los Deployments gracias a *sync waves*, con *expand/contract* documentado. [D03]

## Ítems parciales

| Ítem | Qué observó la cátedra | Resolución sugerida | Relación con la teoría y la consigna |
|---|---|---|---|
| **C3.6** Aprobación a `prod` | `promover-prod` es un job `when: manual`. No hay environment protegido ni reglas de aprobación, y los cuatro integrantes son Maintainers. La protección de `main` impide que lo dispare un Developer, pero no exige que apruebe otra persona. [D03] | Proteger el environment `prod` con aprobadores explícitos, o que `promover-prod` abra un MR al estado deseado de prod que integre alguien distinto del autor. [D03] | `E06` acepta tres mecanismos: aprobación del pull request, *environment protection* o sincronización manual en ArgoCD. [E06] *Inferencia, conocimiento general:* en GitLab.com los protected environments con aprobaciones forman parte de planes pagos. Hay que verificar si el grupo de la cátedra los tiene habilitados. Si no, queda la variante del MR. |
| **C3.11** Liveness | `/salud/vivo` de los dos servicios reales devuelve un OK constante. La readiness sí es real, porque hace ping a la base con timeout. [D03] | Que la liveness verifique algo del proceso sin depender de servicios externos: lag del event loop bajo un umbral, heartbeat reciente del consumidor de Kafka y del relay del outbox, o el servidor gRPC sirviendo. [D03] | "Una sonda que siempre responde OK es un error." [E06] La liveness detecta un bloqueo local y no depende de PostgreSQL. [T18, p. 16] [T18, p. 17] Es la misma observación que la número 6 de la [devolución de la segunda entrega](devolucion-segunda-entrega.md), trasladada a Kubernetes. [D02] |
| **C1.3** Cliente OAuth | Los tres ambientes comparten el client ID y también el client secret de Google: comprometer el de `dev` compromete el de `prod`. Además, `docs/cicd.md` afirma que cada ambiente tiene su propio cliente. [D03] | Crear un cliente por ambiente, o al menos uno para `prod`, y sellarlo por ambiente. Si se mantiene uno solo, justificarlo y corregir el documento. [D03] | La configuración y los secretos van por ambiente. [E06] |
| **C6.4** Justificación de herramientas | La tabla de herramientas de `docs/cicd.md` no justifica el lenguaje y el framework (solo están en el ADR-0010, que no se cita) ni el framework de tests. [D03] | Agregar dos filas: "Lenguaje/framework", con referencia al ADR-0010, y "Tests", con Vitest, v8 y Compose y su motivo. [D03] | Lenguajes y framework de tests son decisiones del equipo y "debe justificarse en el README". [E06] |

## Observaciones que no bajaron la nota

- **Manifiestos comunes sin gate (C3.9).** `gitops/workloads`, `base` y `entrada` llegan a `prod` con cada merge a `main`, sin pasar por `staging` ni por la aprobación. El equipo lo había declarado como límite aceptado. La cátedra sugiere versionar la base por ambiente o fijar el `targetRevision` de prod a un commit o tag promovido. [D03]
- **Commits a mano en el overlay de prod sin MR (C3.6).** `c3d65e2`, con digests declarados a mano, y `8f5cbbd`, con configuración del operador. No bajan el ítem porque el pipeline entregado no escribe en prod sin aprobación, pero se van a preguntar en la defensa. [D03]
- **Estado de Terraform local (C1.1).** Un backend remoto permitiría que otro integrante opere el cluster. [D03]
- **Participación de identidad (C4.5).** En IT-05, IT-06, IT-08, IT-09 e IT-11 identidad solo aporta la sesión. Para el final conviene sumar casos en los que identidad aporte más, por ejemplo IT-04 con rol Operador. [D03]
- **Funcionales en el mismo proceso (C2.6).** El servicio corre dentro del proceso del test con supertest y no en un contenedor aparte. Se acepta porque la imagen instrumentada no es la de release. [D03]

## A verificar en la defensa

`D03` deja anotado qué se va a mirar en la defensa: [D03]

1. **Deploy token de ArgoCD.** `argocd-albumreadonly` venció el 2026-11-20. Hay que mostrar que ArgoCD sigue leyendo el repositorio, o rotarlo y volver a sellar `infra/argocd/gitlab-repository.sealed.json`.
2. **Refresh tokens de Google.** Vencen a los 7 días; hay que renovarlos para que la suite de `staging` no falle.
3. **Aprobación a prod.** Quién puede ejecutar `promover-prod`, por qué no se protegió el environment y una demostración de que un Developer no puede dispararlo.
4. **Commits manuales al overlay de prod.** Su contexto y cómo se evita que se repitan.
5. **Manifiestos comunes.** Qué le pasa a `prod` si se mergea en `gitops/workloads/` un cambio que rompe `staging`.
6. **Token `ci-lector`.** Una demostración de que no puede sincronizar ni leer Secrets, por ejemplo con un `POST /api/v1/applications/album-prod/sync` rechazado.

Esto se suma a lo que `E06` ya pedía mostrar: un cambio que llega a producción por Git y ArgoCD, un fallo que detiene la promoción, un rollback por Git y la justificación de las decisiones. [E06]

## Hacia la entrega final

**No hay consigna de la entrega final incorporada.** Lo que sigue sale de lo que `E06`, `E01` y `D03` ya dicen sobre ella; cuando llegue la consigna, hay que contrastarlo.

### Qué se sabe que exige

- **Todos los servicios reales y toda la funcionalidad demostrable.** [E06] [E01, p. 2] El criterio de la cátedra hace obligatorios todos los requerimientos de `E01`, sin el umbral de 70 % de la presentación. Ver [Dudas y conflictos](../dudas-y-conflictos.md). [N-2026-10-04-criterio-consignas-tp-album]
- **Los 16 escenarios de aceptación cubiertos por tests del pipeline**, con IT-01 a IT-39 como referencia. Un IT de escenario que no está implementado, o que no pasa, cuenta como escenario sin cubrir. [E06]
- **RNF-01 a RNF-06 completos.** En la entrega final son un gate: si no se cumplen, no se evalúa. [E06] [E01, p. 6]
- **La rúbrica de `E01`:** autenticación 10 %, RF core 25 %, RF ampliados 15 %, reglas e invariantes 20 %, escenarios 20 %, consistencia 5 % y reproducibilidad 5 %. [E01, p. 7]
- **Defensa sincrónica en la instancia final**, que evalúa la puesta en marcha y los pipelines. El cluster queda apagado salvo para prácticas y pruebas. *Aclaración de la cátedra por Discord del 2026-11-18, todavía sin nota procesada ni fuente catalogada.*

### Tests pendientes por servicio

*Inferencia de estudio:* el catálogo de `E06` agrupado según el servicio que tendría que volverse real para que cada test pase. Los dueños salen de la segunda entrega del TP; [E06]

| Servicio que pasa de mock a real | Tests pendientes | Notas |
|---|---|---|
| `intercambios` | IT-12 a IT-28 (ofertas e intercambios) | Es el bloque más grande y el que concentra la concurrencia: IT-22, IT-23 y la saga. Antes hay que cerrar las observaciones 7 (código de la aceptación) y 10 (presupuesto de latencia) de `D02`. [D02] |
| `retos-y-promociones` | IT-29 a IT-33 | Incluye idempotencia ante eventos duplicados (IT-30) y canje concurrente de un código de un solo uso (IT-33). [E06] |
| `panorama-del-coleccionista` | IT-07, IT-34, y la proyección de ofertas compatibles que usa IT-17 | El mock se desplegó sin Valkey; el servicio real lo reincorpora si la arquitectura lo mantiene. [D03] |
| `supervision-operativa` | IT-04, IT-35, IT-36 | IT-04 es el caso con rol Operador que sugiere C4.5. IT-36 necesita el criterio de cierre de `OBSERVACION_NO_CONCLUYENTE` (observación 9 de `D02`). [D03] [D02] |
| Varios servicios | IT-03 | Pide intentar acciones sobre recursos ajenos: abrir un sobre y publicar o aceptar una oferta. |

IT-20, IT-24, IT-25, IT-28, IT-35 e IT-36 necesitan inyectar fallas, pausar operaciones o sembrar estados inconsistentes **sin credenciales de administración del cluster desde el CI**. [E06] El mecanismo de Jobs pedidos por Git que ya usa la suite es el punto de partida. [D03]

### Orden sugerido

*Inferencia de estudio, ordenada por urgencia y dependencia:*

1. **Antes de volver a prender el cluster:** rotar el deploy token de ArgoCD, que ya venció, y los refresh tokens de Google. Sin el primero, ArgoCD no puede leer el estado deseado. [D03]
2. **Correcciones chicas e independientes:** C6.4 (documentación), C1.3 (cliente OAuth de prod o corrección del documento) y C3.11 (liveness real). La liveness hay que diseñarla una sola vez y reutilizarla en los cuatro servicios nuevos. [D03]
3. **Gobierno de prod:** C3.6 (aprobador distinto del autor) y la observación de C3.9 (manifiestos comunes sin gate). Las dos cambian el flujo de promoción, así que conviene resolverlas juntas y antes de que el ritmo de merges aumente con cuatro servicios en desarrollo. [D03]
4. **Servicios reales, de a uno:** cada servicio entra con su CI completo, su cobertura y los tests de integración de su bloque. Por tamaño y riesgo, `intercambios` primero.
5. **Ensayo de defensa:** arrancar el cluster, recorrer la lista de la sección anterior y apagarlo.

### Seguimiento de los arreglos

*Registro del grupo, no de la cátedra.* Se actualiza a medida que cada arreglo avanza. El estado real está en el repositorio del TP y en su pipeline; esta tabla dice quién toma cada ítem y qué se decidió.

| Ítem | Decisión | Responsable | Estado |
|---|---|---|---|
| C6.4 Justificación de herramientas | Dos filas nuevas en la tabla de `docs/cicd.md`: lenguaje y framework (ADR-0010 y ADR-0011) y tests (Vitest con V8 y funcionales con Compose) | Franco | En la rama `correcciones-e3` del TP; se integra por MR cuando se prenda el cluster |
| C1.3 Cliente OAuth | Separar solo el de `prod`; `dev` y `staging` siguen compartiendo el actual, que tiene solo cuentas de prueba. Así no cambian las variables del CI ni los refresh tokens, que están atados al cliente de `staging` | Francisco | Pendiente; necesita el cluster prendido y el deploy token de ArgoCD rotado |
| C3.11 Liveness | Medir el retraso del event loop con `monitorEventLoopDelay` de Node: si el p99 entre dos sondas supera `SALUD_RETRASO_MAXIMO_MS` (1 s por defecto), `/salud/vivo` responde `503 PROCESO_SATURADO`. Se descartó sumar un latido del relay del outbox, porque sus consultas a PostgreSQL no tienen timeout y la liveness pasaría a depender de la base | Franco, en los dos servicios reales; cada dueño, en su servicio nuevo | Implementado en `acceso-e-identidad` y `coleccion-y-sobres`, en la rama `correcciones-e3`, con unitarios y funcionales en verde en contenedor |
| C3.6 Aprobación a `prod` | Proteger el environment `prod` en GitLab (Settings → CI/CD → Protected environments): los cuatro integrantes como aprobadores, 1 aprobación, y sin permitir que apruebe quien disparó el pipeline, que es la opción por defecto. Es configuración de GitLab más un ADR-0035 y docs; `.gitlab-ci.yml` no cambia. Solo frena el último paso a `prod`: dev y staging siguen igual | Franco | Pendiente para el cierre de la entrega, para no frenar el desarrollo |
| C3.9 Manifiestos comunes sin gate | Que `album-prod` siga una rama `prod` que solo avanza con el job aprobado, hasta el commit que verificó staging. Así cierra también el camino de los commits a mano al overlay de prod, que el C3.6 solo no cubre | — | Pendiente para el cierre de la entrega, junto con el C3.6 |
| Deploy token de ArgoCD y refresh tokens de Google | Rotar antes de prender el cluster para trabajar | — | Pendiente |

**Liveness en cada servicio nuevo.** Al pasar un mock a servicio real, su dueño repite lo de los dos servicios actuales:
1. Copiar `src/salud/retraso-del-event-loop.ts` y su test desde `acceso-e-identidad`, registrarlo como provider del módulo raíz y usarlo en `vivo()`.
2. Agregar `salud.retrasoMaximoMs` a la configuración, leído de `SALUD_RETRASO_MAXIMO_MS`, y el error `ProcesoSaturado` (503).
3. Que la liveness no consulte nada externo: ni la base, ni Kafka, ni otro servicio. La base va solo en la readiness. [T18, p. 17]

**Cómo hacer C1.3 sin romper `prod`:**
1. Crear en Google Cloud un cliente web nuevo con el callback de `prod` y borrarle ese callback al cliente viejo.
2. Volver a sellar **solo** las claves `oidc-audiencia` y `oidc-secreto-cliente` de `gitops/secrets/prod/acceso-e-identidad.sealed.json` con `kubeseal --raw --scope strict --namespace prod --name acceso-e-identidad`. Cada clave de un SealedSecret se cifra por separado, así que se puede reemplazar una sin tocar las demás. No hay que usar `sellar_servicios.py`: regenera también la contraseña de la base, y PostgreSQL solo la lee al inicializarse.
3. Corregir el paso 1.6 del README y el párrafo de secretos de `docs/cicd.md`.
4. Subir el cambio por un MR que integre otra persona.
5. Cuando ArgoCD sincronice, borrar el Pod de identidad, porque la audiencia entra como variable de entorno y solo se lee al arrancar. Después, probar el login en los tres ambientes.

## Puntos abiertos

- **Consigna de la entrega final.** No está incorporada. Hay que confirmar si mantiene `E06` como base de la plataforma o si agrega requisitos, por ejemplo observabilidad o prueba de carga. Ver [Dudas y conflictos](../dudas-y-conflictos.md).
- **Protected environments en GitLab.com.** *Inferencia, conocimiento general:* puede que no estén disponibles en el plan del grupo. Hay que verificarlo antes de elegir entre esa opción y la del MR.
