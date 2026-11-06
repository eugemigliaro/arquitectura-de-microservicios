# Devolución de la segunda entrega: qué corregir para la tercera

## Propósito y fuente

`D02` es la devolución de la cátedra a la [segunda entrega](segunda-entrega-album-arquitectura.md) del grupo 10. Se evaluó con la consigna v1.2 (`E05`). El material corregido fue `AGENTS.md`, `docs/arquitectura.md`, los ADRs 0001 a 0020 y, como insumo, `docs/analisis-ddd.md`. [D02]

Importa para la [tercera entrega](tercera-entrega-album-cicd-aks.md) porque `E06` toma como fuente de verdad `docs/arquitectura.md` y `docs/adr/`, "con las correcciones que surjan de la devolución de la cátedra". Esos documentos definen qué servicios existen, cómo se comunican y qué contratos tienen. [E06] Las observaciones de abajo, entonces, son insumo de la entrega que viene y no solo un comentario sobre la anterior.

## Nota por criterio

| Criterio | Peso | Nivel | Puntaje |
|---|---:|---|---:|
| Drivers y trazabilidad | 10 % | Excelente | 9,0 / 10 |
| Descomposición y datos | 20 % | Excelente | 18,0 / 20 |
| Comunicación y coordinación | 20 % | Excelente | 18,5 / 20 |
| Consistencia, idempotencia y escala | 20 % | Excelente | 18,5 / 20 |
| Despliegue y CI/CD | 20 % | Excelente (límite inferior) | 17,5 / 20 |
| Decisiones de arquitectura | 10 % | Excelente | 9,5 / 10 |
| **Total** | | | **91 / 100** |

Fuente: [D02]. Los criterios y los pesos son los de la rúbrica de `E05`. [E05] La devolución dice que corrigió con `criterios-evaluacion.md`, un archivo que no está incorporado. Ver [Dudas y conflictos](../dudas-y-conflictos.md).

La condición previa de forma se cumplió y no hubo descuentos graves. La cátedra comprobó explícitamente que no hubiera servicios sin contexto de origen, bases escribibles compartidas, nombres genéricos ni ADRs sin alternativas. [D02] Esos controles corresponden a motivos de descuento y reglas de calidad de `E05`. [E05]

El criterio con menos margen fue **Despliegue y CI/CD**, justo el que la tercera entrega reemplaza por AKS y GitOps. [D02] [E06]

## Qué valoró la cátedra

Conviene conservar estos rasgos en `docs/cicd.md` y en el README de la tercera entrega:

- **Trazabilidad contexto → servicio → dato → evento.** Hay un servicio por contexto, un único dueño por dato y proyecciones con origen y atraso aceptable declarados. [D02]
- **Escenarios de calidad medibles**, con estímulo, entorno, respuesta y medida. La cátedra también valoró que se explicara la estimación del pico y que se reconociera que el despliegue inicial no lo alcanza, en lugar de disimularlo. [D02]
- **Saga orquestada con máquina de estados explícita.** Define el punto de no retorno, timeouts, compensación solo antes de decidir y un recuperador por lease que no duplica efectos. [D02]
- **Un mecanismo concreto por garantía**: CAS sobre versión, `UNIQUE` por reserva, `UPDATE` condicional del cupo e idempotencia en la API y en los consumidores. [D02]
- **Contratos de eventos completos** sobre CloudEvents, con `correlationid`/`causationid`, clave de partición, evolución de esquemas y outbox. [D02]
- **Artefacto inmutable**: una imagen construida una vez y promovida por digest, con rollback compatible y migraciones *expand/contract*. [D02]
- **Honestidad en la sección 11**: cambios al modelo con su motivo y riesgos reales con su mitigación. [D02]

*Inferencia:* varias de estas fortalezas coinciden con reglas de calidad de `E06`: "una imagen, muchos ambientes", trazabilidad de cada test a un RF o escenario, coherencia con la segunda entrega y nada de boilerplate. [E06]

## Observaciones y su efecto en la tercera entrega

Ninguna observación es grave; explican los nueve puntos que faltan. [D02] La columna de la derecha relaciona cada una con `E06` y con la teoría. Cuando no hay cita de `D02` o `E06`, es *inferencia de estudio*.

| # | Observación de `D02` | Sugerencia de la cátedra | Relación con la tercera entrega |
|---:|---|---|---|
| 1 | La tabla de escenarios de calidad (1.2) salta de QA-08 a QA-10, y el documento referencia los QA por ID. [D02] | Renumerar o explicar la ausencia de QA-09. [D02] | El README tiene que citar el commit o tag de `docs/arquitectura.md`. [E06] *Inferencia:* conviene corregirlo antes de fijar ese commit. |
| 2 | Los QA se centran en consistencia y operación. Faltan escenarios de rendimiento de lectura y de seguridad más allá del rechazo de sesión. [D02] | Agregar un QA de p99 del resumen y de los rankings, con ID propio. [D02] | `E06` excluye los acuerdos de latencia de lo implementado. [E06] *Inferencia:* es una corrección del documento; no obliga a medir ese p99 en AKS. |
| 3 | Unas 1.500 líneas, muy por encima de las 15 a 25 páginas sugeridas. El ADR-0009 lista unas 28 viñetas de alternativas y el ADR-0011 unas 38, y la tabla 8.2 repite celdas. [D02] [E05] | Reducir cada ADR a las dos o tres alternativas reales y una tabla compacta. Resumir un Dockerfile por servicio con sus diferencias. [D02] | `docs/cicd.md` tiene que ocupar entre 3 y 6 páginas, y `E06` prohíbe el boilerplate. [E06] *Inferencia:* es la observación con más riesgo de repetirse. |
| 4 | Un único contenedor PostgreSQL aloja las cinco bases: el fallo y la saturación son comunes y la restauración es conjunta. Los NFR de 1.3 suponen una configuración (unas 5 réplicas y un host propio de PostgreSQL) que ningún ambiente documentado tiene. [D02] | Separar PostgreSQL en dos instancias, núcleo y vistas, desde el despliegue inicial si el presupuesto lo permite, y decirlo en 8.1. [D02] | Los ambientes no comparten datos, y al menos un servicio real tiene base propia en el cluster o gestionada. [E06] Alta disponibilidad y autoscaling quedan fuera de alcance, pero hay que mencionarlo. [E06] *Inferencia:* hay que decidir por escrito cuántas instancias de PostgreSQL corren por namespace y con qué `requests/limits`, y contrastarlo con la cuota de CPU de la suscripción. |
| 5 | La estrategia "recrear con mantenimiento, hasta 5 minutos" condiciona la promoción a ventanas fuera de los partidos, sin contrastarlas con el calendario real del torneo. [D02] | Definir una política de hotfix durante los partidos. [D02] | La estrategia de despliegue se elige y se justifica, y `prod` exige aprobación manual. [E06] Un rolling update con `maxUnavailable: 0` y drenado evita la pausa. [T18, p. 18] [T21, p. 29] *Inferencia:* el hotfix puede describirse como quién aprueba el cambio a `prod`. No puede saltear el gate de integración de `staging`. |
| 6 | Los healthchecks no reinician: un proceso colgado "requiere reinicio manual". "Un healthcheck del que nadie depende para actuar es información, no protección." [D02] | Un reinicio automático acotado (por ejemplo, un autoheal con límite de reintentos) o una alerta al operador. [D02] | El healthcheck de Compose se parece más a readiness y no reinicia. [T07, p. 18] En Kubernetes, si falla la liveness, el kubelet reinicia el contenedor. [T18, p. 15] La liveness detecta un bloqueo local y no depende de PostgreSQL. La readiness representa capacidad real. [T18, p. 16] `E06` exige sondas que verifiquen algo real. [E06] *Inferencia:* separar liveness (bloqueo local) de readiness (base propia) responde la observación sin abandonar el argumento de QA-04. |
| 7 | La aceptación de oferta (4.2) devuelve `201` aunque la operación termine `NO_CONCRETADA`, y mezcla "recurso creado" con un resultado de negocio negativo. [D02] | `200` o `409` con motivo, o `201` con estado explícito. [D02] | Los códigos de estado tienen que ser correctos; un error dentro de un `200` es un antipatrón. [T09, p. 9] [T09, p. 10] Los mocks respetan los códigos de la segunda entrega, y cualquier cambio se registra en `docs/cicd.md`. [E06] *Inferencia:* conviene decidirlo antes de implementar el servicio real y los tests de intercambio (IT-19 a IT-28). [E06] |
| 8 | `AGENTS.md` del TP pide citar `[E0x]`/`[Txx]` con una tabla en el encabezado de `arquitectura.md`, pero el entregable cita en prosa. [D02] | Alinear la regla interna con el entregable. [D02] | *Inferencia:* fijar un único criterio de citas antes de escribir `docs/cicd.md` y el README. |
| 9 | `supervision-operativa` puede devolver `OBSERVACION_NO_CONCLUYENTE`, pero no define cuándo deja de serlo, y RF-35 pide mostrar inconsistencias. [D02] | Fijar cuántas relecturas o qué ventana antes de reportar `INCONSISTENCIA`. [D02] | IT-36 siembra un estado inconsistente en `staging` y espera que la inconsistencia "quede identificada" cuando el operador consulta. En la entrega final todos los servicios son reales. [E06] *Inferencia:* sin ese criterio, IT-36 no tiene una aserción determinista. |
| 10 | Aceptar una oferta suma hasta 4 llamadas gRPC en secuencia, con un deadline de 2 s y hasta 3 reintentos por llamada. El p99 declarado de 1.500 ms no cubre el peor caso. [D02] | Derivar un presupuesto de latencia por paso y justificar los 1.500 ms. [D02] | Con un deadline propagado, cada salto recibe menos tiempo (2 → 1,5 → 1 → 0,5 s). [T21, p. 10] Se reintenta en un solo nivel de la cadena, con un retry budget de alrededor del 10 %. [T21, p. 11] Kubernetes no aporta timeouts ni retries. [T21, p. 25] *Inferencia:* el presupuesto define la configuración del servicio real que coordine el intercambio. |

## Cómo aplicarla en la tercera entrega

*Inferencia de estudio, ordenada por momento:*

1. **Antes de fijar el commit o tag de `docs/arquitectura.md` que cita el README:** observaciones 1, 2, 7, 9 y 10. Cambian el documento o los contratos que implementan los servicios reales y verifican los tests. [E06]
2. **Al escribir `docs/cicd.md`:** registrar los cambios de nombres, endpoints o códigos, como el de la observación 7, en "cambios respecto de la segunda entrega". [E06] Explicar también que la estrategia de despliegue y las sondas de Kubernetes reemplazan las respuestas de Compose de las observaciones 5 y 6, porque las decisiones de `E06` prevalecen sobre las secciones 8 y 9. [E06]
3. **Al dimensionar el cluster:** la observación 4 se cruza con la cuota y el crédito de Azure. Hay que decidir la topología de PostgreSQL por ambiente y declararla con sus límites. [E06]
4. **En el estilo de toda la entrega:** observaciones 3 y 8. Un documento corto, tablas sin celdas repetidas y alternativas reales en los ADRs nuevos. [D02] [E06]

## Puntos abiertos

- **Criterios aplicados.** `D02` remite a `criterios-evaluacion.md`, que no está incorporado. `E06` aclara que ese archivo no se aplica a la tercera entrega. [D02] [E06]
- **Qué código espera la cátedra en la observación 7.** El `docs/arquitectura.md` del TP (4.2) ya devuelve en el cuerpo del `201` el resultado `COMPLETADA` o `NO_CONCRETADA` con su motivo. Por eso no queda claro en qué se diferencia de la alternativa "`201` con estado explícito". [D02] *Inferencia:* la opción menos ambigua es `200` o `409` con motivo; si se mantiene el `201`, conviene preguntar a la cátedra qué esperaba.
- **Reevaluación.** `D02` no dice si las correcciones vuelven a evaluarse. *Inferencia:* `E06` no tiene un criterio propio para ellas, pero la coherencia con la segunda entrega es una regla de calidad y los contratos de los mocks se controlan. [E06]
