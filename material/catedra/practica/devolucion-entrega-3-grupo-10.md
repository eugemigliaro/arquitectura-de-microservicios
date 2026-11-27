<!-- Incorporación: texto pegado por el grupo el 2026-11-27. Se reconstruyeron en Markdown las tablas y se restauraron los marcadores <host>, <sha>, <job> y <s> que el copiado había reemplazado. El contenido no se modificó. -->

# Corrección Entrega 3 — CI/CD y despliegue en AKS (GitOps)

**Equipo / repo:** grupo-10 — https://gitlab.com/itba-73-40-microservicios/alumnos/2026-s2/grupo-10/tp1-album-figuritas
**Commit evaluado:** ea67803433b473cbe7fcec02decef4ef7f550974 (2026-11-14 10:25 -03:00, "docs: corregir la documentación según la revisión final contra E06") · **Repos/commits adicionales:** ninguno (el estado deseado vive en el mismo repo, gitops/, rama main)
**Versión consigna-funcional declarada:** 1.1 · **Arquitectura (E2) declarada:** 8b2d8f9 (existe; docs/arquitectura.md difiere del commit evaluado en 2 líneas)
**Servicios reales declarados:** acceso-e-identidad y coleccion-y-sobres · **Mocks:** intercambios, retos-y-promociones, panorama-del-coleccionista, supervision-operativa
**Gates:** G0 PASS · G1 PASS · G2 PASS (pipeline #3068115841 sobre 901c401, ancestro del entregado: promover-dev → promover-staging (con suite) → promover-prod → verificar-prod en verde; commits de promoción 152f3b7 (dev), 46cc340 (staging), a3688e2 (prod) con los mismos digests)
**Veredicto global:** PASS

## Resumen ejecutivo

Entrega muy sólida y con evidencia verificable en GitLab. El flujo commit → build → tests → ACR por digest → dev → staging → integración → aprobación → prod está implementado como GitOps pull puro: el CI solo publica en ACR con identidad federada (AcrPush), escribe gitops/environments/<env> con el job token y lee ArgoCD con una cuenta ci-lector que solo tiene applications, get. Las demostraciones negativas están: el pipeline #3067087620 falla en unitarios y no publica ni promueve, y el #3067723027 falla IT-05 en staging, promover-prod queda skipped y no hay ningún commit en gitops/environments/prod entre d7dd465 y b2f35bf. El rollback por git revert (b49fe99) y la deriva corregida por self-heal también están documentados. Cobertura unitaria y funcional por separado, con gate de 80 % en cuatro métricas y valores reales entre 91,7 % y 99,8 %. Los ocho tests de integración cuentan, usan tokens reales de Google por refresh grant y se repiten porque cada corrida los siembra y los limpia. Hay tres puntos a mejorar: (1) la aprobación a prod es un job when: manual sobre main protegida, sin environment protegido ni lista de aprobadores, y los cuatro integrantes son Maintainers; (2) las sondas de liveness de los servicios reales devuelven un OK constante; (3) los tres ambientes comparten un mismo cliente OAuth (con su secreto), lo que contradice docs/cicd.md.

## Veredictos por sección

| Sección | Peso | Veredicto |
|---|---|---|
| C1 Cluster AKS y ambientes | 20% | PASS |
| C2 CI y testing de los servicios reales | 20% | PASS |
| C3 Pipeline de CD y gates (GitOps) | 20% | PASS |
| C4 Tests de integración | 20% | PASS |
| C5 Completitud del despliegue | 10% | PASS |
| C6 Documentación y reproducibilidad | 10% | PASS |

## Grafo del pipeline y promoción

```
workflow: MR | push a rama (sin MR abierto) | push a main protegida | web

verificar:  verificar-integracion (rules:changes integration/**, ci/…)  ──► comprobar-runner (unittest de ci/)
                                                                              │ needs
servicios:  acceso-e-identidad ─┐  coleccion-y-sobres ─┐  (rules:changes por ruta; web = todos)
            build compilacion → lint (oxlint) + prettier → unitarios en contenedor (gate 80 %)
            → funcionales con compose (Postgres/Kafka reales, gate 80 %) → npm audit --audit-level=high
            → build --target release + chequeo (no root, sin test/src/vitest)
            → [solo main + PUBLICAR_ACR] push ACR, tag <sha>-<job> bloqueado (write/delete disabled), digest
            mocks ×4: conformidad → audit → release → arranque + /salud/listo + 401 sin sesión → push ACR
            preparacion-integracion: auxiliar de siembra (no se promueve)
                                │ needs (optional, artifacts)
dev:        promover-dev  (commit ci-gitops en gitops/environments/dev, resource_group gitops-main)
            └► verificar-dev  [GATE] Synced+Healthy en la revisión exacta + IT-37/38/39 + RF-A05 (≤10 min)
staging:    promover-staging [GATE] (needs verificar-dev) un solo job (ADR-0032):
            copia digests de dev → Synced/Healthy + smoke → Job de siembra por Git → suite IT (8)
            → verificación → Job de limpieza por Git           allow_failure: false
prod:       promover-prod  when: manual, environment: prod, needs promover-staging [APROBACIÓN]
            (commit de los digests de staging en gitops/environments/prod)
            └► verificar-prod [GATE] Synced+Healthy + smoke

Estado deseado (mismo repo, rama main):
  gitops/base, gitops/workloads/*, gitops/entrada  → comunes (Kustomize)
  gitops/environments/{dev,staging,prod}/kustomization.yaml (images: …@sha256) + versiones.json
ArgoCD: album-dev / album-staging / album-prod → path gitops/environments/<env>, automated + prune + selfHeal
```

## Detalle de ítems

### C1 — Cluster AKS y ambientes

- **C1.1 — PASS** (observación: estado de Terraform local, documentado en README.md:238; conviene un backend remoto para que otro integrante pueda operar).
- **C1.2 — PASS**
- **C1.3 — PARTIAL**
  - Evidencia: "Un cliente OAuth de Google con los callbacks https://<host>:8441/…, …:8442/… y https://<host>/…" y el loop que sella el mismo --client-id (y el mismo client secret) para dev, staging y prod — README.md:117-123; en contradicción con "Cada ambiente tiene … su cliente OAuth" — docs/cicd.md:134. El resto del aislamiento es muy bueno: PostgreSQL, Kafka, PVC, SealedSecrets y gateway por namespace, NetworkPolicy default-deny + allow-same-namespace (gitops/base/network-policies.yaml), emisor de sesión distinto por ambiente (gitops/environments/prod/kustomization.yaml:58-83) y la prueba de cruces bloqueados (infra/pruebas/aislamiento.py).
  - Por qué no es PASS: compartir el client ID solo sería una observación, pero acá se comparte también el client secret entre los tres ambientes (un valor compartido no crítico: comprometer el de dev compromete el de prod), y la documentación afirma lo contrario (§5.1, §5.7).
  - Resolución sugerida: crear un cliente OAuth por ambiente (o al menos uno para prod separado de dev/staging) y sellarlo por ambiente; o, si se mantiene uno solo, justificarlo y corregir docs/cicd.md:134.
- **C1.4 — PASS**
- **C1.5 — PASS** (ArgoCD por Helm con chart fijado 10.14.0, infra/argocd/install.sh:13,39-40; tres Applications versionadas en gitops/argocd/).
- **C1.6 — PASS** (verificado con glab api projects/:id/deploy_tokens: argocd-albumreadonly, scopes ["read_repository"]. Atención: vence el 2026-11-20 03:00 UTC; ver "A verificar en la defensa").
- **C1.7 — PASS** (identidad federada gitlab_main limitada a ref:main con AcrPush sobre el ACR, infra/cluster/ci.tf:11-28; token ArgoCD ci-lector con p, role:ci-lectura, applications, get, album/*, infra/argocd/values.yaml:19-26; ningún kubectl/helm/az aks en .gitlab-ci.yml ni en ci/).
- **C1.8 — PASS**
- **C1.9 — PASS**
- **C1.10 — PASS**

### C2 — CI y testing de los servicios reales

- **C2.1 — PASS** (pipelines push, merge_request_event (p. ej. #3065876807) y web en GitLab).
- **C2.2 — PASS** (multi-stage, USER node, node:24.22.2-bookworm-slim@sha256:01d5… en ambos).
- **C2.3 — PASS** (release solo con dist, node_modules de producción y proto; el CI verifica test ! -d /app/test, /app/src, vitest y \*.spec.\* sobre la imagen de release, ci/validar-servicio.sh:98-105; solo la release se publica).
- **C2.4 — PASS** (oxlint --deny-warnings + prettier --check + npm audit --omit=dev --audit-level=high, todos bloqueantes; log del job #18031211513: "Found 0 warnings and 0 errors").
- **C2.5 — PASS** (pipeline #3068115841: acceso-e-identidad 97,56 % de líneas y 91,91 % de ramas; coleccion-y-sobres 99,78 % / 97,04 %; umbrales de 80 en vitest.config.ts:25; JUnit y Cobertura en los artefactos; exclusiones justificadas en los README de los servicios).
- **C2.6 — PASS** (funcionales en la imagen test-funcionales, instrumentada con v8, contra Postgres 18.6 y Kafka 4.3.2 reales del compose y un JWKS/OIDC simulado para los otros servicios; 96,10 % y 91,69 % de líneas, con gate separado; incluyen 404/409/422 de dominio, reintentos e idempotencia concurrente, test/funcionales/apertura.spec.ts:157-261). Observación: el servicio corre en el proceso del test (supertest), no en un contenedor separado. Es aceptable porque la imagen instrumentada no es la de release.
- **C2.7 — PASS** (publicación al final del mismo script con set -e; demostrado en #3067087620: job #18025147505 "Fallaron unitarios … no se continúa", promover-\* skipped, sin imagen-publicada.json).
- **C2.8 — PASS**
- **C2.9 — PASS** (rules:changes por servicio; #3067087620 solo ejecutó acceso-e-identidad; #3068326255 (docs) no construyó servicios).
- **C2.10 — PASS** (make -C services/<s> test; el CI usa el mismo target test-funcionales y, para unitarios, ci/Dockerfile.unitarios sobre la misma etapa compilacion y el mismo npm run test:cov).

### C3 — Pipeline de CD y gates (GitOps)

- **C3.1 — PASS**
- **C3.2 — PASS** (commit 152f3b7 de ci-gitops con trailers Commit-origen: 901c401…, Pipeline: 3068115841).
- **C3.3 — PASS** (ORIGEN = {"staging": "dev", "prod": "staging"}, ci/promover_gitops.py:16; needs: verificar-dev).
- **C3.4 — PASS** (espera Synced/Healthy en la revisión exacta + IT-37/38/39 por el gateway, con timeout; log #18031211521 "PASS gate dev: Synced/Healthy en 152f3b7…, 16 comprobaciones"; smoke también en staging y prod).
- **C3.5 — PASS** (gate dentro de promover-staging, del que depende promover-prod; #3067723027: IT-05 ×, job #18029090985 failed, promover-prod skipped; git log -- gitops/environments/prod no muestra commits entre d7dd465 (06:00 UTC) y b2f35bf (10:27 UTC)).
- **C3.6 — PARTIAL**
  - Evidencia: promover-prod es when: manual con environment: name: prod — .gitlab-ci.yml:470-493; glab api projects/:id/protected_environments → []; los cuatro integrantes tienen rol Maintainer (40) y main admite push/merge de Maintainers; "Puede ejecutar el job quien tenga permiso de escritura sobre main protegida. No hay una lista de aprobadores aparte" — docs/adr/0033-aprobacion-a-prod-en-el-pipeline.md:29. Se usó en promociones reales (#3068115841, job #18031211523, ejecutado por eugemigliaro).
  - Por qué no es PASS: la aprobación es un job manual. La protección de main impide que lo dispare un Developer, pero todo el equipo es Maintainer y no hay environment protegido con aprobadores ni un MR aprobado (§5.4: "revisión/aprobación del pull request, environment protection o sincronización manual en ArgoCD").
  - Resolución sugerida: proteger el environment prod con aprobadores explícitos (Settings → CI/CD → Protected environments, con required approvals). Alternativa: que promover-prod abra un MR al estado deseado de prod que integre una persona distinta de quien hizo el cambio.
  - Observación (cambios a prod sin MR): con merge_method: merge, estos commits tocan gitops/environments/prod sin MR asociado (.../commits/<sha>/merge_requests → 0): c3d65e2 (2026-11-11, FrancoBrandaa, digests declarados a mano en kustomization.yaml/versiones.json), 2e0c391 (2026-11-13, alta del job manual), 8f5cbbd (2026-11-13, configuración del operador en el overlay de prod). Los commits b49fe99/53ea36f corresponden al rollback documentado. No bajan el ítem: el pipeline entregado no escribe en prod sin la aprobación.
- **C3.7 — PASS** (digest @sha256: en los overlays; tag <sha>-<job> bloqueado con az acr repository update --write-enabled false --delete-enabled false, ci/publicar_acr.py:116-117; bloques images de staging y prod idénticos en el commit evaluado; sin :latest).
- **C3.8 — PASS**
- **C3.9 — PASS** (observación: album-prod también tiene automated y queda condicionada al commit aprobado (ADR-0033). Pero los manifiestos comunes (gitops/workloads, base, entrada) llegan a prod con cada merge a main, sin gate de staging ni aprobación. El equipo lo declara como límite en docs/cicd.md:119; conviene cerrarlo).
- **C3.10 — PASS** (kubectl -n prod scale deployment/gateway --replicas=2 → OutOfSync → sync automático → 1 réplica; docs/evidencias/datos/deriva-prod-2026-11-14/resumen.json).
- **C3.11 — PARTIAL**
  - Evidencia: `@Get('vivo') vivo() { return { estado: 'ok', servicio: 'acceso-e-identidad' }; }` — services/acceso-e-identidad/src/salud/salud.controller.ts:19-22; ídem services/coleccion-y-sobres/src/salud/salud.controller.ts:19-22, usada como livenessProbe en gitops/workloads/pareja-real/servicios.yaml:76-78,180-182. La readiness sí es real (ping a la base con timeout, :24-30), y la estrategia Recreate está justificada (ADR-0029, docs/cicd.md:127).
  - Por qué no es PASS: la liveness de los dos servicios reales es una respuesta constante, que no refleja el estado del proceso (§5.4, §6: "una sonda que siempre responde OK es un error").
  - Resolución sugerida: que vivo verifique algo del proceso. Por ejemplo, que el event loop no esté bloqueado (lag bajo un umbral), que el consumidor de Kafka y el relay del outbox sigan con heartbeat reciente, o que el servidor gRPC esté sirviendo, sin depender de dependencias externas.
- **C3.12 — PASS** (b49fe99 revert de 6f6030f, sincronización registrada en docs/evidencias/datos/rollback-2026-11-14/rollback-prod.json, smoke posteriores).
- **C3.13 — PASS**
- **C3.14 — PASS** (Jobs migrar-acceso/migrar-coleccion como hook Sync con sync-waves antes de los Deployments, gitops/workloads/pareja-real/preparacion.yaml:60-64,125; expand/contract en docs/cicd.md:129).

### C4 — Tests de integración

- **C4.1 — PASS** (8 tests que cuentan).
- **C4.2 — PASS** (la interpretación de IT-09 "sin clave" = otra solicitudId → 409 SOBRE_YA_ABIERTO está justificada contra el contrato de la E2, docs/cicd.md:170).
- **C4.3 — PASS** (Jobs de siembra y limpieza pedidos por Git, allowlist sellada y roles mínimos; dos pipelines consecutivos con la suite en verde: #3067870121 y #3068115841).
- **C4.4 — PASS**
- **C4.5 — PASS** (recorte "Colección/sobres + autenticación" del Anexo A.9: identidad emite, a partir del token real de Google, la sesión con la que colección resuelve la propiedad de álbum y sobres; IT-02 además afirma el efecto del evento PrimerAccesoDeColeccionistaReconocido). Observación: en IT-05, IT-06, IT-08, IT-09 e IT-11 la participación de identidad se limita a la sesión. Cuentan por la tabla A.9, pero para la entrega final conviene sumar casos donde identidad aporte más al resultado (p. ej. IT-04 con rol Operador).
- **C4.6 — PASS** (IT-09 con tres aperturas reales en paralelo, Promise.all, integration/casos/it09.test.ts:12; IT-01 e IT-10 verifican que no haya efectos).
- **C4.7 — PASS** (job #18031211522: 8/8 contra URL_STAGING = …:8442, integration/src/configuracion.ts:1, que se exige en src/autenticacion.ts:18; integracion.xml como artefacto y JUnit en la pestaña Tests).
- **C4.8 — PASS** (grant_type: "refresh_token" contra https://oauth2.googleapis.com/token, integration/src/google.ts:37-42; las variables GOOGLE_REFRESH_TOKEN_\*, INTEGRACION_SESION_VENCIDA y GOOGLE_OAUTH_CLIENT_SECRET están masked, protected y hidden; la rotación cada 7 días está documentada en README.md:157).
- **C4.9 — PASS** (no implementan IT-20/24/25/28/35/36; la siembra usa un Job versionado solo en staging, con credencial de staging y limpieza documentada en docs/cicd.md:166).
- **C4.10 — PASS**

### C5 — Completitud del despliegue

- **C5.1 — PASS** (evento acceso-e-identidad.PrimerAccesoDeColeccionistaReconocido.v1, services/acceso-e-identidad/src/eventos/eventos.ts:35, consumido en services/coleccion-y-sobres/src/mensajeria/consumidor-de-eventos.ts:22,66,165; validación de la sesión con el JWKS de identidad, services/coleccion-y-sobres/src/configuracion.ts:110).
- **C5.2 — PASS**
- **C5.3 — PASS** (se revisaron por muestreo las rutas del §4.2 de la E2: PUT /v1/publicaciones/{id}, PUT /v1/ofertas/{id}/operaciones/{id}, GET /v1/operaciones/{id}, DELETE /v1/ofertas/{id}, GET /v1/ofertas, PUT|GET /v1/canjes/{id}, GET /v1/retos, las consultas de panorama (incluido 422 CRITERIO_DESCONOCIDO) y supervision/{ofertas-activas,reservas,inconsistencias}; cada mock tiene tests de conformidad en el CI).
- **C5.4 — PASS**
- **C5.5 — PASS** (gitleaks no estaba disponible: se usó git log --all -p con los patrones de §8, sin hallazgos. Observación: aparecen credenciales triviales autodocumentadas (solo-pruebas-locales, solo-fixture) en integration/preparacion/compose.yaml:4,8 y en el test del auxiliar; no se usan en el cluster).
- **C5.6 — PASS** (identidad verifica el ID token con el JWKS de Google, issuer y audience, services/acceso-e-identidad/src/sesion/verificador-de-identidad-externa.ts:39-41; el gateway Envoy valida la sesión con jwt_authn y el JWKS de identidad, gitops/entrada/base/envoy.yaml:181-188; sin x-user-id).
- **C5.7 — PASS**

### C6 — Documentación y reproducibilidad

- **C6.1 — PASS**
- **C6.2 — PASS**
- **C6.3 — PASS**
- **C6.4 — PARTIAL**
  - Evidencia: la tabla de herramientas de docs/cicd.md:16-24 (enlazada desde README.md:24) justifica CI, IaC, registro, CD, Kustomize, secretos y gateway, y la estrategia Recreate está en docs/cicd.md:127. No justifica el lenguaje y framework (solo en ADR-0010 de la E2, no citado) ni el framework de tests (Vitest, Testcontainers/compose).
  - Por qué no es PASS: §3 pide justificar en el README los lenguajes y el framework de tests; esas dos decisiones no aparecen.
  - Resolución sugerida: agregar a la tabla de docs/cicd.md dos filas, "Lenguaje/framework" (con referencia a ADR-0010) y "Tests" (Vitest + v8 + compose, y por qué).
- **C6.5 — PASS**
- **C6.6 — PASS**
- **C6.7 — PASS**
- **C6.8 — PASS** (docs/cicd.md:128).

## Tests de integración

| Test (archivo:línea) | ID declarado (IT/RF/esc.) | ¿ID correcto? | Datos/limpieza | Aserción de negocio | ¿Producida por ambos reales? | (F) | Corrió en staging y pasó (pipeline/job) | ¿Cuenta? |
|---|---|---|---|---|---|---|---|---|
| integration/casos/it01.test.ts:7 | IT-01, RF-A05, esc. 16 | Sí | Siembra/limpieza por Job (escenarios.json) | 401 SESION_INVALIDA (sin, inválida, vencida real) y foto de la sección/sobre idéntica | Sí (sesión de identidad, validación y estado de colección) | Sí | #3068115841 / #18031211522 | Sí |
| integration/casos/it02.test.ts:8 | IT-02, RF-A01/A02, RF-05 | Sí | Cuenta it02 reseteada a "nunca vista" | Álbum creado, 5 sobres inicial DISPONIBLE, segundo acceso sin nuevos | Sí (evento de identidad → álbum en colección) | No | ídem | Sí |
| integration/casos/it05.test.ts:4 | IT-05, RF-01/02/03 | Sí | Colección sembrada conocida | Poseídas, repetidas, apartadas, faltantes y avance | Sí (A.9; identidad aporta la sesión/propiedad) | No | ídem | Sí |
| integration/casos/it06.test.ts:5 | IT-06, RF-04, esc. 10 | Sí | Sembrada | Copias visibles al instante, poseídas = disponibles + apartadas | Sí (A.9) | No | ídem | Sí |
| integration/casos/it08.test.ts:5 | IT-08, RF-06, esc. 1 | Sí | Sembrada | 201, contenido acreditado una vez, sobre abierto, resultado consultable | Sí (A.9) | No | ídem | Sí |
| integration/casos/it09.test.ts:7 | IT-09, RF-07, esc. 2 | Sí (interpretación documentada) | Sembrada | 3 aperturas paralelas → [200,200,201], mismo cuerpo, acreditación única; otra clave → 409 | Sí (A.9) | Sí | ídem | Sí |
| integration/casos/it10.test.ts:5 | IT-10, RF-08, esc. 13 | Sí | Sembrada (A y B) | Ajeno → 404, ya abierto → 409, fotos de A y B sin cambios | Sí (propiedad por sesión de identidad) | Sí | ídem | Sí |
| integration/casos/it11.test.ts:5 | IT-11, RF-22, esc. 7 | Sí | Sembrada | Consulta devuelve el resultado real, sin acreditar de nuevo | Sí (A.9) | Sí | ídem | Sí |

**Tests que cuentan:** 8/5 · **(F) que cuentan:** 4 (IT-01, IT-09, IT-10, IT-11) · **Smoke A.8 por ambiente:** dev sí, staging sí, prod sí

## Matriz servicios × ambientes

| Servicio (E2) | Real/Mock | En estado deseado dev | staging | prod | Evidencia kubectl get pods,svc | Contrato conforme (mocks) | Sondas reales |
|---|---|---|---|---|---|---|---|
| acceso-e-identidad | Real | Sí (digest) | Sí | Sí | Sí (docs/evidencias/ambientes.md) | — | Readiness real; liveness constante (C3.11) |
| coleccion-y-sobres | Real | Sí | Sí | Sí | Sí | — | Readiness real; liveness constante (C3.11) |
| intercambios | Mock | Sí | Sí | Sí | Sí | Sí (muestreo) | Presentes (vivo/listo/startup) |
| retos-y-promociones | Mock | Sí | Sí | Sí | Sí | Sí | Presentes |
| panorama-del-coleccionista | Mock | Sí | Sí | Sí | Sí | Sí | Presentes |
| supervision-operativa | Mock | Sí | Sí | Sí | Sí | Sí | Presentes |
| Gateway (Envoy), PostgreSQL, Kafka | Infra | Sí | Sí | Sí | Sí | — | Presentes |
| Valkey (de panorama) | Infra | No | No | No | — | — | Cambio registrado en docs/cicd.md:184 (panorama es mock sin cache) |

## Evidencias verificadas (§5.9)

| Evidencia | Ubicación | Verificación con glab | Resultado |
|---|---|---|---|
| Corrida completa commit → prod | docs/evidencias/corrida-completa.md | #3068115841 success, sha 901c401 (ancestro), 15 jobs en verde, promover-prod #18031211523 manual | Verificada |
| Falla de CI sin imagen | docs/evidencias/ci-fallido.md | #3067087620 failed, sha 808cbb4 (ancestro); #18025147505 failed, promover-\* skipped; artefactos sin imagen-publicada.json | Verificada |
| Integración fallida, prod intacto | docs/evidencias/integracion-fallida.md | #3067723027 failed, sha 1f168d1 (ancestro); #18029090985 failed (IT-05 ×), promover-prod skipped; ningún commit a gitops/environments/prod entre 06:00 y 10:27 UTC | Verificada |
| Rollback vía Git | docs/evidencias/rollback.md | #3067136889 success (llevó 6f6030f a prod); b49fe99 (revert) y 53ea36f en el historial; JSON de ArgoCD con la revisión | Verificada |
| ArgoCD 3 apps + kubectl get pods,svc | docs/evidencias/ambientes.md, datos/ambientes-2026-11-14/argocd.png | Captura abierta: album-dev/staging/prod Healthy + Synced; 6 servicios + gateway + PG + Kafka en cada ns | Verificada |
| Deriva corregida | docs/evidencias/deriva.md | No aplica glab; datos coherentes (mismo Deployment, ns prod, timestamps, operación automática) | Verificada |

## Hallazgos bloqueantes

Ninguno.

## A verificar en la defensa

- **Deploy token de ArgoCD vencido:** argocd-albumreadonly (read_repository) figura expired: true, expires_at: 2026-11-20T03:00Z. Mostrar que ArgoCD sigue leyendo el repo, o rotarlo y volver a sellar infra/argocd/gitlab-repository.sealed.json.
- **Refresh tokens de Google:** vencen a los 7 días (README.md:157). Renovarlos antes de la defensa para que la suite de staging no falle.
- **Aprobación a prod:** explicar quién puede ejecutar promover-prod y por qué no se protegió el environment prod. Mostrar que un Developer no puede dispararlo.
- **Commits manuales al overlay de prod sin MR** (c3d65e2, 8f5cbbd): explicar su contexto y cómo se evita que se repitan.
- **Manifiestos comunes:** mostrar qué pasaría con prod si se mergea un cambio en gitops/workloads/ que rompe staging (llega a prod sin gate).
- Mostrar que el token ci-lector no puede sincronizar ni leer Secrets (p. ej. un POST /api/v1/applications/album-prod/sync rechazado por la ventanilla y por el RBAC).

## Acciones recomendadas al equipo (priorizadas)

1. Proteger el environment prod con aprobadores explícitos (o pasar a MR integrado por otra persona) y dejar de commitear a mano en gitops/environments/prod (C3.6).
2. Rotar el deploy token de ArgoCD (vencido el 2026-11-20) y los refresh tokens de Google antes de la defensa.
3. Hacer que la liveness de acceso-e-identidad y coleccion-y-sobres refleje el estado del proceso (C3.11).
4. Separar el cliente OAuth de prod del de dev/staging, o corregir docs/cicd.md:134 y justificar el cliente compartido (C1.3).
5. Hacer que los cambios en manifiestos comunes también pasen por staging antes de llegar a prod (p. ej. versionar la base por ambiente o fijar targetRevision de prod a un commit/tag promovido) (C3.9, observación).
6. Justificar en el README/docs/cicd.md el lenguaje/framework y el framework de tests (C6.4).

## Registro de verificación

- git rev-parse HEAD → ea67803433b473cbe7fcec02decef4ef7f550974 (repo ya clonado; sin checkout ni modificaciones).
- glab auth status → logueado en gitlab.com. glab api projects/:id → id 85941223, default_branch: main, merge_method: merge, privado.
- glab api projects/85941223/repository/commits/ea67803… → existe (2026-11-14T10:25:09-03:00, Eugenio Migliaro).
- glab api projects/85941223/ci/lint?content_ref=<sha>&include_jobs=true → valid: true, sin errores (guardado en el scratchpad como ci-lint.json / ci-merged.yml).
- glab api projects/85941223/pipelines?sha=<sha> → #3068326255 success (solo verificar-integracion y comprobar-runner: commit de documentación).
- Jobs de #3068115841, #3067087620, #3067723027, #3067136889, #3067870121 listados con include_retried=true. Ninguno tiene allow_failure: true.
- git merge-base --is-ancestor → 901c401, 808cbb4, 1f168d1, 0817660, 2f58a3e, 9fe621e, df4615f son ancestros del commit evaluado.
- Logs descargados: #18031211513 y #18031211514 (lint, cobertura unitaria/funcional), #18031211521 (gate dev), #18031211522 (ciclo staging, 8/8), #18031211523 (promoción prod), #18025147505 (unitario fallido), #18029090985 (IT-05 fallido).
- Artefactos: #18031211513 (JUnit, Cobertura unitaria y funcional, npm-audit, imagen-publicada.json), #18031211522 (integracion.xml, smoke-staging.xml, promocion-staging.json).
- glab api projects/85941223/protected_branches → main: push y merge solo Maintainers, sin force push. protected_environments → []. approval_rules → []. environments → prod (available).
- glab api projects/85941223/variables (solo metadatos) → 14 variables, todas protected. Los secretos (ARGOCD_TOKEN_LECTURA, GOOGLE_OAUTH_CLIENT_SECRET, GOOGLE_REFRESH_TOKEN_\*, INTEGRACION_SESION_VENCIDA) están masked, y todos salvo el token de ArgoCD también hidden.
- glab api projects/85941223/deploy_tokens → argocd-albumreadonly, read_repository, expired: true (vence 2026-11-20).
- glab api projects/85941223/members/all → 4 integrantes con rol 40 (Maintainer) más cátedra (50).
- glab api projects/85941223/merge_requests?state=all → 12 MRs integrados por Pablo-Goros.
- git log -- gitops/environments/prod y glab api …/commits/<sha>/merge_requests sobre c3d65e2, 8f5cbbd, 2e0c391, b49fe99, 53ea36f → 0 MRs.
- git log --all -p | grep -E 'GOCSPX-|AccountKey=|-----BEGIN … PRIVATE KEY-----|glpat-|ghp_|AKIA…' → sin resultados (gitleaks no instalado).
- No se ejecutó código del grupo ni ningún comando de escritura contra GitLab o el cluster.
