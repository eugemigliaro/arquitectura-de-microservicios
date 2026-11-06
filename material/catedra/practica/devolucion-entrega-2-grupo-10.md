# Devolución Entrega 2 — Grupo 10

**Integrantes:** Pablo Gorostiaga, Francisco Costa, Franco Nicolás Branda, Eugenio Migliaro
**Material evaluado:** `AGENTS.md`, `docs/arquitectura.md` (1.494 líneas), `docs/adr/0001` a `0020`, `docs/analisis-ddd.md` (insumo).
**Consigna:** Entrega 2 v1.2. **Criterios:** `criterios-evaluacion.md`.

## Nota global: 91 / 100 (Excelente)

| Criterio | Peso | Nivel | Puntaje |
|---|---|---|---|
| Drivers y trazabilidad | 10% | Excelente | 9,0 / 10 |
| Descomposición y datos | 20% | Excelente | 18,0 / 20 |
| Comunicación y coordinación | 20% | Excelente | 18,5 / 20 |
| Consistencia, idempotencia y escala | 20% | Excelente | 18,5 / 20 |
| Despliegue y CI/CD | 20% | Excelente (límite inferior) | 17,5 / 20 |
| Decisiones de arquitectura | 10% | Excelente | 9,5 / 10 |
| **Total** | 100% | | **91,0** |

**Condición previa de forma:** cumplida. Están `docs/arquitectura.md` y `docs/adr/` (20 ADRs), todo en castellano, con integrantes, versión de la consigna funcional (1.1) y commit del análisis DDD (`70a42b7`) en el encabezado, y con todos los diagramas en Mermaid. Las 11 secciones siguen el orden y la numeración pedidos. No hay descuentos graves. No se detectaron servicios sin contexto de origen, bases compartidas escribibles, nombres genéricos ni ADRs sin alternativas.

## Fortalezas

- **Trazabilidad fuerte contexto → servicio → dato → evento.** Hay seis servicios, uno por contexto de la Entrega 1 (3.1, 3.2, ADR-0001). `supervision-operativa` no tiene base y consulta a los dueños. La tabla 3.3 da un único dueño por dato, con lectores y proyecciones. Las proyecciones de `panorama-del-coleccionista` indican origen y atraso aceptable (5 s normal, 60 s en pico, QA-11).
- **Escenarios de calidad y NFR medibles.** Los QA-01 a QA-11 tienen estímulo, entorno, respuesta y medida concretos. El 1.3 explica cómo se estimó el pico (1 M de usuarios, 10 % activo, 500 aperturas/s). La disponibilidad "best effort, sin HA" se fundamenta en la sección 6 de la consigna funcional. La sección 7 reconoce que el despliegue inicial no alcanza el pico y no lo disimula.
- **Saga del intercambio muy completa (sección 5, ADR-0005).**
  - Es una saga orquestada con máquina de estados explícita: `EN_CURSO`, `CONFIRMANDO`, `LIBERANDO`, `COMPLETADA` y `NO_CONCRETADA`.
  - Define el punto de no retorno, quién dispara cada transición, los timeouts (60 s) y la compensación solo antes de decidir.
  - El recuperador por lease cada 15 s explica por qué no duplica efectos: comandos idempotentes por operación y registro terminal de la operación en `coleccion-y-sobres`.
  - Se cubren todos los diagramas de secuencia pedidos (1-2, 3, 4, 5-6, 7, 12, retos y escenario 15).
  - El N:M se trata como lote de un elemento (ApartarLote todo-o-nada), y las ofertas compatibles tienen una proyección y una validación al mostrar (ADR-0016).
- **Mecanismos concretos para cada garantía (sección 6).**
  - Cada dato tiene su fila de consistencia con mecanismo: CAS sobre la versión de Tenencia, `UNIQUE` por reserva, `UPDATE` condicional del cupo, versión optimista en la Oferta.
  - La tabla de fallos por escenario cubre del 2 al 16 más fallos de infraestructura.
  - El invariante de oro tiene tabla de mecanismos y de verificación.
  - La idempotencia está en API y en consumidores (clave, ubicación, retención).
  - El orden se resuelve en cada consumidor con versión del Álbum y contribución por acción.
  - La resiliencia lista valores y qué ocurre al activarse cada mecanismo.
  - Se justifica por qué no hay colas de absorción.
- **Contratos de eventos completos (4.3).** Se usa CloudEvents como envelope, con `correlationid`/`causationid`, productor, consumidores, canal, clave de partición con el orden que da y payload. Hay estrategia de evolución de esquemas por `type` versionado en paralelo y outbox con relay por lease. Los códigos de error de negocio tienen tabla propia (4.2).
- **Despliegue y CI/CD con rigor.**
  - Hay tres ambientes en hosts aislados, con proyectos Compose, redes y volúmenes propios. Imágenes, redes, límites de CPU/memoria (con la cuenta que cierra en 8 CPU / 11,75 GiB), `depends_on` y reinicios están definidos.
  - Los healthchecks verifican la base propia y no incluyen dependencias de terceros, con argumento explícito (QA-04).
  - Se justifica el estado en contenedores, con respaldo continuo (ADR-0020).
  - El pipeline tiene 8 etapas con gates. Las imágenes se construyen una vez y se promueven por digest con un manifiesto inmutable.
  - Se detalla el rollback compatible, el expand/contract en releases separados, las credenciales sin exponer secretos y la ventana de despliegue fuera de partidos.
- **ADRs sólidos.** Están los 9 temas obligatorios más 11 adicionales. Todos tienen alternativas y consecuencias, con riesgos aceptados y decisiones condicionadas.
- **Honestidad sobre cambios y riesgos.** La sección 11.1 registra 10 cambios al modelo con motivo, incluida la respuesta a la devolución de la Entrega 1. La sección 11.3 trae 18 riesgos reales, no genéricos, con probabilidad y mitigación. Las preguntas abiertas indican qué parte depende de cada una. La matriz 11.2 cubre RF-A01 a A05, RF-01 a RF-39, RNF-01 a RNF-06 y los 16 escenarios.

## Problemas y observaciones

Ningún problema es grave. Los siguientes explican el puntaje que falta.

1. **Numeración de escenarios de calidad con un salto** (`arquitectura.md` 1.2). La tabla pasa de QA-08 a QA-10 y no hay QA-09. Puede ser un escenario eliminado. Hay que renumerar o explicarlo, porque el documento referencia QA por ID.
2. **Escenarios de calidad centrados en consistencia y operación.** Faltan escenarios medibles de rendimiento puro para consultas (p99 de resumen bajo carga) o de seguridad más allá del rechazo de sesión. Están en la tabla 1.3 pero sin ID de QA.
3. **Volumen del documento.** Son ~1.500 líneas, muy por encima de las 15-25 páginas sugeridas. La consigna dice que no se evalúa el volumen, pero hay redundancia: ADR-0009 tiene unas 28 viñetas de alternativas y ADR-0011 unas 38, y varias celdas de la tabla 8.2 repiten "Misma base y esquema multi-stage". La tabla 8.2 se acerca a boilerplate. Un Dockerfile resumido por servicio con sus diferencias (puertos, secretos) sería más preciso.
4. **Cuello de botella único declarado pero no resuelto** (7, 8.1, 11.3).
   - Un único contenedor PostgreSQL aloja las cinco bases. Las bases y usuarios son separados, así que no es una base escribible compartida. Pero el fallo y la saturación son comunes, y la restauración tiene que hacerse conjunta.
   - La sección 7 admite que el pico de ~6.700 req/s sobre `coleccion-y-sobres` necesita unas 5 réplicas y un host de PostgreSQL propio. El diseño por defecto no lo alcanza y esto queda como riesgo "Alta probabilidad". Es honesto, pero los NFR de 1.3 se presentan para una configuración que ningún ambiente documentado tiene. QA-07 lo verifica solo de forma exploratoria.
5. **Pausa por despliegue contra la disponibilidad** (9.1, QA-10). La estrategia "recrear con mantenimiento, hasta 5 minutos" está justificada y se mitiga con ventanas fuera de partidos. Pero la promoción automática queda condicionada a ventanas, lo que es una política de negocio que no se contrasta con el calendario real del torneo. Convendría un criterio para promociones urgentes (hotfix) durante partidos.
6. **Healthchecks que no reinician** (8.2, "Salud del servicio"). El diseño es defendible, pero un proceso colgado "requiere reinicio manual" y se acepta como riesgo bajo. Un healthcheck del que nadie depende para actuar es información, no protección. Sugerencia: un reinicio automático acotado (por ejemplo, un autoheal con límite de reintentos) o alertar al operador.
7. **Código HTTP de la aceptación** (4.2). Devolver `201` cuando la operación termina `NO_CONCRETADA` mezcla "recurso creado" con "resultado de negocio negativo". Un `200`/`409` con motivo o un `201` con estado explícito es más consistente con la regla "nunca un error dentro de `200`" que el propio documento establece.
8. **Referencias a fuentes externas.** `AGENTS.md` dice que las citas `[E0x]`/`[Txx]` deben figurar en una tabla del encabezado de `arquitectura.md`. No hay tabla ni citas: se cita en prosa ("consigna funcional v1.1, sección 6"). Esto es correcto para quien corrige, pero la instrucción interna del repositorio queda inconsistente con el entregable.
9. **`supervision-operativa` y detección de inconsistencias** (3.1, ADR-0017). La detección "dos lecturas con comparación de versiones" puede dar `OBSERVACION_NO_CONCLUYENTE`. Está bien reconocido, pero RF-35 pide mostrar inconsistencias, y el diseño no define cuándo un diagnóstico inconcluyente deja de serlo (cuántas relecturas, qué ventana). Falta ese criterio operativo.
10. **Tiempos acumulados de la aceptación.** `aceptar oferta` tiene deadline de 2 s por llamada, hasta 3 reintentos y 4 llamadas gRPC en secuencia. El p99 declarado de 1.500 ms no es coherente con el peor caso. El texto lo menciona ("el p99 cubre un reintento"), pero no hay un presupuesto de latencia por paso. Conviene derivarlo.

## Sugerencias

- Renumerar los QA o explicar la ausencia de QA-09; agregar un QA de rendimiento de lectura (p99 del resumen y los rankings) con ID propio.
- Reducir las alternativas listadas de ADR-0009 y ADR-0011 a las dos o tres reales y mover el resto a una tabla compacta. Mantener las tablas 8.2 sin celdas repetidas.
- Especificar un presupuesto de latencia por paso para `aceptar oferta` y justificar el 1.500 ms.
- Definir la política de hotfix durante partidos y un criterio de reinicio automático ante `unhealthy` persistente.
- Fijar un criterio operativo para el diagnóstico inconcluyente de Supervisión (cuántas relecturas o qué tiempo antes de reportar `INCONSISTENCIA`).
- Corregir el uso de `201` en una operación no concretada.
- Considerar separar PostgreSQL en dos instancias (núcleo y vistas) desde el despliegue inicial si el presupuesto lo permite, y decirlo en 8.1.

## Conclusión

Entrega excelente, sin descuentos graves. Es de las más completas posibles para esta consigna: arquitectura derivada del dominio, mecanismos concretos para cada garantía, saga con recuperación bien diseñada, CI/CD con promoción de artefacto inmutable y una sección 11 honesta. Lo que resta son observaciones menores, sobre todo de numeración, volumen y algunos criterios operativos sin cerrar. **Nota: 91/100.**
