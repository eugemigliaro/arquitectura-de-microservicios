# Corrección del simulacro — Segundo parcial

Corrección de las respuestas guardadas en `simulacro-segundo-parcial.md`. Se tomó la hoja de respuestas como registro principal; tus respuestas originales se conservan intactas.

## Resultado

**32/35 correctas (91,4 %), 1 incorrecta y 2 sin resolver.** Entre las 33 respondidas, acertaste 32. Esto es un resultado de práctica, no una nota oficial.

| Tema | Correctas | Incorrectas | Sin resolver |
|---|---:|---:|---:|
| Testing | 4 | 1 | 0 |
| Comunicación síncrona | 5 | 0 | 0 |
| DevOps y CI/CD | 5 | 0 | 0 |
| Estado distribuido y patrones | 4 | 0 | 1 |
| Sistemas distribuidos | 4 | 0 | 1 |
| Comunicación asincrónica y Kafka | 5 | 0 | 0 |
| Del monolito a microservicios | 5 | 0 | 0 |

No hay justificaciones escritas. Se puede evaluar la opción elegida y la confianza declarada, pero todavía no distinguir todos los aciertos razonados de los obtenidos por descarte. Tampoco cinco preguntas por tema demuestran dominio del documento completo.

## Prioridades de repaso

**Diagnóstico basado en tus respuestas, no en una clasificación oficial de dificultad:**

1. **SAST/SCA/DAST (5):** confusión comprobada entre código propio y dependencias.
2. **Recuperación de Streams (20) y locking optimista (23):** quedaron sin resolver.
3. **Liberación de locks, fencing e idempotencia (17–19):** correctas con confianza media-baja.
4. **Particiones y consumidores de Kafka (27):** correcta con confianza baja.
5. **Deadlines y DataLoader (7–8):** correctas con confianza media-baja.

DevOps muestra cinco aciertos con confianza entre media y alta. Migración también tiene cinco aciertos, pero no declaraste confianza en ese bloque. Las preguntas 21 y 28–30 tampoco tienen confianza registrada; no se les atribuye seguridad alta por haber acertado.

## Las tres preguntas que requieren corrección

### 5. Elegiste A — Correcta: C, SCA

El caso dice que la vulnerabilidad está en **una librería incorporada**, no que se encontró una vulnerabilidad al analizar el código que escribió el equipo.

| Control | Qué revisa | Ejemplo de la cátedra |
|---|---|---|
| Linter | Convenciones y patrones de escritura | ESLint, Checkstyle, Hadolint |
| SAST | Vulnerabilidades en el código propio, mediante análisis estático | SonarQube, Semgrep |
| SCA | Vulnerabilidades en dependencias | Trivy |
| DAST | La aplicación en ejecución, atacada desde afuera | OWASP ZAP |

[T10, p. 20]

**Por qué falla A:** le atribuye a SAST el análisis exclusivo de dependencias, que corresponde a SCA. B falla porque cumplir convenciones no demuestra que las librerías sean seguras. D falla porque ejecutar líneas no descarta vulnerabilidades conocidas ni prueba que se hayan evaluado correctamente. [T10, p. 19] [T10, p. 20]

**Para recordarlo:** SAST → código; SCA → componentes/dependencias; DAST → aplicación corriendo. Que pasen los tests funcionales no garantiza seguridad: evalúan preocupaciones distintas. [T10, p. 20]

### 20. Respondiste «no sé» — Correcta: B, XAUTOCLAIM

En un consumer group de Streams, entregar un mensaje no equivale a confirmar que terminó. Se espera `XACK`. Si el consumidor cae antes de enviarlo, queda trabajo pendiente. [T12, p. 32]

Hay dos situaciones:

- **Recovery:** el consumidor vuelve y lee sus pendientes con `XREADGROUP ... 0`.
- **Claim:** el consumidor no vuelve; otro reclama los pendientes mediante `XAUTOCLAIM` cuando cumplen el tiempo mínimo de inactividad.

El caso pregunta por la segunda. [T12, p. 33]

A falla porque no hay ACK automático por entregar. C falla porque Valkey Pub/Sub no conserva historial para recuperar mensajes. D falla porque borrar el stream no hace que el productor reconstruya automáticamente el trabajo. [T12, p. 25] [T12, p. 32] [T12, p. 33]

**Conexión con idempotencia:** el consumidor original podría haber aplicado el cambio y caído antes del ACK. Por eso una reentrega puede duplicar el efecto si no se protege, por ejemplo con un Inbox transaccional. [T12, p. 21] [T12, p. 27]

### 23. Respondiste «no sé» — Correcta: B, locking optimista

Ambos clientes leyeron versión 7. El primero actualizó a versión 8. El segundo exige que la versión siga siendo 7: esa condición ya no se cumple y la escritura no procede.

**Ejemplo didáctico del mecanismo:**

```sql
UPDATE perfiles
SET nombre = 'Nuevo nombre', version = 8
WHERE id = 42 AND version = 7;
```

Si el registro está en versión 8, no se actualiza ninguna fila. El cliente detecta el conflicto en lugar de sobrescribir silenciosamente el cambio anterior. Debe manejarlo, por ejemplo releyendo y reintentando según la operación. [T11, p. 28]

**Pesimista:** bloquear antes de operar; el costo son las esperas y locks. **Optimista:** detectar conflictos al intentar aplicar el cambio; el costo son los conflictos y reintentos. El material sugiere pesimista con conflictos frecuentes y optimista con conflictos poco frecuentes. [T11, p. 27]

A falla porque no se describió un lock adquirido al leer. C falla porque dos clientes concurrentes no son por eso participantes de 2PC: no hay preparación ni decisión global. D falla porque la condición de versión justamente impide sobrescribir siempre. [T11, p. 28] [T11, p. 29]

## Revisión de todas las preguntas

Las explicaciones siguientes indican el concepto correcto y el error central de los distractores.

### Testing

| Nº | Tu respuesta | Correcta | Devolución |
|---|---|---|---|
| 1 | C | C | Se prueba el límite con persistencia: mapeo, conexión y transaccionalidad. No es unitario por tener base efímera, no recorre todo el producto y no verifica un contrato HTTP. [T10, p. 7] [T10, p. 8] |
| 2 | A | A | Un fake implementa comportamiento simplificado. Un stub devuelve respuestas prefijadas; un mock verifica interacciones. Tener estado no lo vuelve end-to-end. [T10, p. 6] |
| 3 | D | D | Cobertura mide ejecución de código, no corrección de las comprobaciones ni todas las combinaciones de entradas. Sirve para encontrar huecos, aunque no sea suficiente. [T10, p. 18] [T10, p. 19] |
| 4 | B | B | Contract testing valida expectativas del consumidor contra el proveedor por separado. Mocks privados no garantizan compatibilidad real; no reemplaza todas las pruebas ni congela la API para siempre. [T10, p. 10] |
| 5 | A | C | SCA revisa dependencias; SAST revisa código propio. Linting y cobertura no garantizan seguridad de una librería. [T10, p. 20] |

### Comunicación síncrona

| Nº | Tu respuesta | Correcta | Devolución |
|---|---|---|---|
| 6 | A | A | REST encaja en APIs públicas; gRPC en comunicación interna compacta; GraphQL en consultas complejas del frontend. Elegir uno para todo no elimina red, contratos ni fallas. [T09, p. 17] |
| 7 | D | D | El deadline limita la espera de la llamada. Streaming y quitar el `.proto` no resuelven esa espera; retries inmediatos ilimitados pueden agravar sobrecarga. [T09, p. 12] [T09, p. 27] |
| 8 | B | B | DataLoader agrupa cargas para abordar N+1. Más profundidad no reduce consultas; mover reglas a resolvers ni pedir menos campos garantiza eliminar N+1. [T09, p. 15] |
| 9 | C | C | Si llegan más trabajos de los que se completan, crece el backlog. Una cola finita termina saturándose: no crea capacidad ni garantiza latencia constante. [T09, p. 22] [T09, p. 30] [T09, p. 33] |
| 10 | A | A | Bulkhead acota concurrencia y espera, rechazando exceso para proteger el servicio. No persiste automáticamente trabajos, no agrega CPU y no elimina duplicados. [T09, p. 26] [T09, p. 30] |

### DevOps y CI/CD

| Nº | Tu respuesta | Correcta | Devolución |
|---|---|---|---|
| 11 | D | D | Continuous Delivery deja releases listos para desplegar; Continuous Deployment automatiza su puesta en producción. Un paso manual final no invalida Delivery. [T13, p. 17] [T13, p. 18] |
| 12 | B | B | Se promueve el artefacto validado sin modificarlo. Reconstruir puede cambiarlo; el nombre del tag no demuestra identidad de contenido y omitir pruebas no aporta reproducibilidad. [T13, p. 35] |
| 13 | C | C | Rolling requiere compatibilidad mientras conviven versiones. No detiene necesariamente todo el servicio ni garantiza por sí mismo compatibilidad del esquema. [T13, p. 40] |
| 14 | A | A | Canary expone una fracción de tráfico, evalúa el resultado y amplía progresivamente. Basic detiene todo; multi-service despliega varios servicios; rolling describe reemplazo progresivo de instancias. [T13, p. 38] [T13, p. 39] [T13, p. 40] [T13, p. 42] |
| 15 | D | D | Shift-left devuelve temprano el error al desarrollador. No exige publicar cambios que fallaron ni reemplazar controles por exploración manual; Leftover Principle describe las tareas que quedan al operador tras automatizar. [T13, p. 13] [T13, p. 45] |

### Estado distribuido y patrones

| Nº | Tu respuesta | Correcta | Devolución |
|---|---|---|---|
| 16 | B | B | La condición y la modificación necesitan una frontera atómica compartida. Mutex locales, TTL y balanceo no coordinan esa secuencia entre instancias. [T12, p. 8] [T12, p. 9] |
| 17 | A | A | Comparar propietario y borrar debe ser atómico. Un ID fijo no distingue adquisiciones; separar GET y DEL deja una carrera; quitar expiración dificulta recuperarse de un dueño caído. [T12, p. 9] [T12, p. 10] |
| 18 | C | C | El recurso rechaza tokens anteriores al mayor observado. Owner-id protege liberación, no por sí solo escrituras; un TTL mayor no garantiza que no haya pausas más largas; mutex local no coordina a B. [T12, p. 11] |
| 19 | D | D | La clave corresponde a una operación con parámetros determinados. Cambiarlos debe rechazarse; ni devolver silenciosamente otra respuesta ni generar otra clave y cobrar de nuevo preserva ese acuerdo. [T12, p. 19] |
| 20 | No sé | B | Otro consumidor reclama pendientes con XAUTOCLAIM tras la inactividad mínima. Entregar no confirma; Pub/Sub no guarda historial; borrar el stream no recupera trabajo. [T12, p. 25] [T12, p. 32] [T12, p. 33] |

### Sistemas distribuidos

| Nº | Tu respuesta | Correcta | Devolución |
|---|---|---|---|
| 21 | A | A | Timeout deja incertidumbre sobre el resultado. No demuestra aborto ni caída permanente, y RPC no elimina el riesgo de duplicar al reintentar. [T11, p. 18] |
| 22 | C | C | Serializabilidad exige equivalencia con algún orden secuencial válido. No consiste en ordenar requests por llegada, compartir un reloj o terminar antes de un timeout. [T11, p. 22] |
| 23 | No sé | B | La condición de versión detecta una modificación concurrente: locking optimista. No hubo bloqueo previo ni protocolo 2PC; la condición impide la sobrescritura indiscriminada. [T11, p. 27] [T11, p. 28] |
| 24 | D | D | Tras votar SÍ, el participante debe esperar la decisión y puede quedar bloqueado. Su voto no demuestra la decisión global; un timeout no habilita decidir solo ni cambiar espontáneamente a Saga. [T11, p. 29] [T11, p. 34] |
| 25 | A | A | El dueño explícito del flujo define orquestación. Puede usar mensajes asíncronos; un broker no implica coreografía ni garantiza un commit atómico. [T11, p. 20] |

### Comunicación asincrónica y Kafka

| Nº | Tu respuesta | Correcta | Devolución |
|---|---|---|---|
| 26 | C | C | Dentro de Facturación se comparte grupo para repartir trabajo; Analytics usa otro para recibir el historial independientemente. Un grupo común repartiría eventos entre ambos servicios, y un grupo por worker replicaría el consumo. [T14, p. 42] |
| 27 | B | B | Cada partición se asigna a un solo consumidor del grupo a la vez. Cuatro particiones limitan a cuatro consumidores con asignación; rebalancear no borra backlog ni vuelve independientes a miembros del mismo grupo. [T14, p. 40] |
| 28 | D | D | Una key estable por entidad mantiene sus eventos en la misma partición. Tipo de evento o UUID no garantiza reunir los de una orden; quitar keys y sumar consumidores no da orden global. El handler también debe preservar el procesamiento ordenado. [T14, p. 35] [T12, p. 28] |
| 29 | A | A | Outbox une estado y registro del evento en una transacción local; el relay puede publicar dos veces. Marcar antes arriesga pérdida. Inbox necesita registrar ID y efecto en la misma transacción, no en dos independientes. [T14, p. 47] [T14, p. 48] [T14, p. 50] |
| 30 | C | C | Poison messages requieren intentos limitados, apartamiento y alerta. Retry infinito puede atascar la partición; reiniciar no corrige un esquema y cambiar keys no resuelve el payload ni garantiza orden. [T14, p. 52] [T14, p. 53] |

### Del monolito a microservicios

| Nº | Tu respuesta | Correcta | Devolución |
|---|---|---|---|
| 31 | B | B | La decisión necesita valor de negocio; modernidad no basta y un monolito puede escalar horizontalmente. Broker, tablas y líneas de código no justifican por sí solos la migración. [T15, p. 8] [T15, p. 9] |
| 32 | D | D | Strangler intercepta rutas externas; Branch by Abstraction cubre accesos internos, incluido el cron. El proxy no reemplaza llamadas locales; branch largo y shadowing sin control no cubren esa extracción. [T15, p. 23] [T15, p. 25] |
| 33 | A | A | Ejecutar dos motores con efectos puede cobrar dos veces. Devolver una sola respuesta no limita los efectos; logs, base y timeout no deduplican automáticamente ambas ejecuciones. [T15, p. 26] |
| 34 | C | C | El flag cambia la implementación activa, no revierte ni reconcilia datos. Una réplica atrasada necesita evaluación antes del rollback; binarios y número de topics no resuelven divergencia. [T15, p. 40] |
| 35 | B | B | Hay que medir entrega, fallas, recuperación y coordinación frente a objetivos y línea de base. Más servicios o lenguajes no demuestra autonomía; eliminar métricas impide evaluar la mejora. [T15, p. 46] |

## Refuerzo de los aciertos con dudas

### Deadlines y N+1 — Preguntas 7 y 8

**Deadline:** establece el límite para esperar la llamada. No hace más rápido al proveedor ni convierte el timeout en prueba de que no hubo efecto. [T09, p. 12] [T11, p. 18]

**N+1:** una consulta inicial y luego una por cada elemento. DataLoader permite agrupar cargas relacionadas. Limitar profundidad aborda otra preocupación: consultas excesivamente profundas. [T09, p. 15]

### Owner-id, fencing e idempotencia — Preguntas 17 a 19

Estos mecanismos protegen cosas diferentes:

| Mecanismo | Pregunta que responde |
|---|---|
| Owner-id, validado al liberar | ¿Sigo siendo dueño de esta adquisición del lock? |
| Fencing, validado por el recurso | ¿Esta escritura viene de un dueño obsoleto? |
| Clave de idempotencia | ¿Esta operación lógica ya se completó o está en curso? |

[T12, p. 10] [T12, p. 11] [T12, p. 19]

No son sustitutos entre sí. Una clave completada repite la respuesta para los mismos parámetros; no autoriza a cambiar la operación original. [T12, p. 19]

### Particiones y consumidores — Pregunta 27

**Ejemplo didáctico con cuatro particiones:** dos consumidores pueden atender dos cada uno; cuatro pueden atender una cada uno; ocho no pueden atender las cuatro simultáneamente dentro del mismo grupo. Algunos quedarán sin asignación. [T14, p. 40]

Si crece el lag, las opciones incluyen optimizar el handler, aumentar consumidores hasta el límite de particiones, revisar particionamiento o aceptar latencia durante el pico. Más consumidores por encima del límite no aumentan el paralelismo de asignación. [T14, p. 43]

## Cómo seguir

Primero responder preguntas nuevas de seguridad, recuperación de Streams y locking optimista. Después explicar con palabras propias locks/fencing y particiones/consumer groups. En la siguiente tanda conviene incorporar temas no evaluados aquí, como modelos de falla, compensaciones de Sagas, CQRS y otros controles de testing, para ampliar la evidencia sobre los siete documentos.
