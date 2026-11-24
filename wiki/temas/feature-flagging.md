# Feature flagging

## Fuente y alcance

Síntesis de `T22`, presentación de la cátedra "Feature flagging: liberar funcionalidades en una arquitectura de microservicios" (45 láminas, Microservicios · ITBA · 73.40). **Inferencia de catalogación:** el ciclo 2026 se asigna por la configuración de la materia y la fecha de creación del PDF (6 de noviembre de 2026); la portada no lo indica ni nombra docentes. [T22, p. 1] Tiene cinco bloques: conceptos, capas, propagación, observación, y operación y cierre. [T22, p. 3] Las láminas 40 a 43 son un apéndice de "fragmentos técnicos de consulta" que no se presentan en clase salvo que surja la pregunta; las dos últimas son la autoevaluación y las lecturas. [T22, p. 40]

`T01` ubicaba el feature flagging en la clase de escalabilidad y resiliencia. [T01, p. 13] Dos unidades anteriores lo habían anticipado: los feature toggles de la migración y el "desplegar no es lo mismo que liberar" de resiliencia. [T15, p. 28] [T21, p. 29]

**Caso conductor:** un *checkout express*, asignado por tienda (tenant). [T22, p. 3]

## El problema: un deploy correcto, una experiencia rota

Se quiere liberar el checkout express para algunas tiendas. El navegador muestra el botón express; el gateway rutea a checkout classic; Pagos recalcula la variante y obtiene express; el worker procesa el evento mañana como classic. "Cada componente funciona. Cuatro decisiones independientes ⇒ recorrido incoherente." La pregunta que ordena la clase: qué decisión debe mantenerse estable y durante cuánto tiempo. [T22, p. 2]

Objetivos de la clase: [T22, p. 3]

1. Separar despliegue, activación y exposición.
2. Elegir dónde evaluar una flag: browser, red, backend o datos.
3. Mantener coherencia entre servicios, reintentos y eventos.
4. Observar efectos y detener un rollout con evidencia.
5. Retirar la flag sin romper contratos ni perder datos.

## Conceptos

### Vocabulario

| Término | Definición de la clase |
|---|---|
| Feature flag | Decisión configurable que elige comportamiento en ejecución |
| Toggle point | Lugar del código donde cambia el comportamiento |
| Control plane | Donde se definen y publican las reglas; lento y auditado |
| Data plane | Donde se evalúa la regla en cada operación; rápido |
| Cohorte | Grupo de unidades que reciben la misma variante |
| Stickiness | Misma unidad, misma variante entre requests |
| Tenant | La tienda o cliente que usa el sistema; la unidad de asignación del caso |
| BFF | Backend dedicado a un frontend; puede evaluar flags para el browser |

Fuente: [T22, p. 4].

### Qué es una flag

Una decisión configurable que selecciona comportamiento en ejecución: [T22, p. 5]

```js
const mode = flags.checkoutMode(context) // "classic" | "express"
return mode === 'express' ? expressCheckout(order) : classicCheckout(order)
```

Tiene tres partes: el **toggle point** (la línea del `if`), el **evaluador** (aplica reglas a un contexto y devuelve una variante) y la **configuración** (variantes, audiencia, porcentaje y fallback). Una variable de entorno también puede ser una flag, pero normalmente exige reiniciar para cambiarla. [T22, p. 5]

### Desplegar, activar y exponer

Son "tres cosas distintas, con tres curvas distintas": [T22, p. 6]

| Curva | Cómo evoluciona |
|---|---|
| Código en producción | Desplegado en el 100 % de las réplicas desde el día 1 |
| Flag activada (audiencia) | 0 % → solo internos → cohortes crecientes → 100 % → se elimina |
| Exposición real | Menor o igual a la audiencia: no todos los habilitados ejecutan la variante |

Las etapas son desplegar, activar, exponer a cohortes y retirar. "Un rollback de flag selecciona el camino anterior; los datos ya escritos permanecen." [T22, p. 6]

### Flags con propósitos distintos

La lámina ubica cuatro tipos en dos ejes: estática o dinámica (por request o cliente) y vida corta o larga. [T22, p. 7]

| Tipo | Para qué | Decisión | Vida | Retiro |
|---|---|---|---|---|
| Release | Liberar código gradualmente | Estática | Corta | Tras estabilizar el lanzamiento |
| Experimento | Comparar variantes con asignación estable | Dinámica | Corta | Al concluir el análisis |
| Operacional | Apagar trabajo costoso o riesgoso | Estática | Larga | Revisión periódica y simulacros |
| Entitlement | Habilitar una capacidad por plan o cliente | Dinámica | Larga | Dura mientras exista el producto |

En el entitlement, "el servidor debe autorizar igual". "Habilitación comercial y autorización requieren controles del servidor. No todas las flags son booleanas ni deben durar para siempre." [T22, p. 7]

### Arquitectura: control y evaluación

- **Control plane:** cambia políticas; es lento, humano y auditado. Un operador hace un cambio, que queda auditado, sobre reglas versionadas (`checkout-express · r42`).
- **Data plane:** resuelve decisiones por request, en memoria y rápido. En el backend, un SDK con caché local evalúa en memoria; para el browser evalúa un evaluador remoto en el BFF o en el edge.
- Las reglas se **distribuyen** del control plane a los evaluadores. "Evitar convertir cada evaluación en una consulta a la base central de flags." [T22, p. 8]

Los mismos términos se usan en la malla de servicios. Ver [Service mesh](service-mesh.md#de-la-librería-al-proxy).

### Contrato de una flag

```yaml
key: checkout-express
owner: equipo-checkout
purpose: release
variants: [classic, express]
default: classic
assignment_unit: tenant_id
expires_when: rollout estable y camino anterior sin consumidores
```

Además: ambientes aislados, permisos de cambio, audiencia y plan de reversión. "La semántica es parte del contrato: ¿activar significa habilitar o bloquear?" [T22, p. 9]

### Contexto y audiencia

El contexto de evaluación del ejemplo lleva `targetingKey: "tenant-42"`, `region`, `plan` y `appVersion`. Describe al sujeto de la decisión, que no siempre es un usuario: se elige usuario, sesión, dispositivo o tenant según la experiencia. El plan y el tenant se derivan de una identidad verificada en el servidor. Se envían solo los atributos necesarios, sin emails ni datos sensibles. [T22, p. 10]

### 10 % de requests no es 10 % de tiendas

La lámina pregunta qué ve la misma tienda en tres clicks seguidos si se sortea express al azar en cada request. [T22, p. 11] *Inferencia:* variantes distintas en cada click. Por eso la asignación tiene que ser estable.

**Asignación estable.** `bucket = hash(salt + tenant_id) mod 10000`. En el ejemplo, `tenant-42` cae en el bucket 732 y recibe express. Subir de 10 % a 25 % **conserva** la cohorte previa, si no cambian la clave ni el salt. [T22, p. 11]

**La unidad no es el tráfico.** El 10 % de los tenants es 1 de 10, pero ese tenant puede generar el 55 % del tráfico. "Medí por variante y por tenant, no por request." Y si la identidad cambia al iniciar sesión (anónimo → tenant), la asignación puede cambiar. [T22, p. 11]

## Capas: en qué nivel se decide

Una sola flag puede tocar el browser, la red, el backend y los datos; la pregunta es quién decide en cada capa y quién garantiza qué. [T22, p. 12]

| Capa | Controla | No garantiza |
|---|---|---|
| Browser | UI, navegación, componentes | Permisos ni ejecución del servidor |
| Red L7 | La ruta a una versión o destino | La semántica dentro del proceso |
| Backend | Reglas y efectos del negocio | Revertir datos ya escritos |
| Datos | Lectura y escritura compatible | Deshacer una migración destructiva |

"Combinar capas requiere contratos, no cuatro sorteos independientes." [T22, p. 13]

### Browser: dónde se evalúa

| Opción | Cómo funciona | Costo |
|---|---|---|
| SDK con reglas locales | El browser descarga reglas públicas (CDN) y evalúa en el cliente | Reglas y atributos visibles para cualquiera |
| Evaluación remota | El browser envía el contexto; el BFF o el edge decide y devuelve la variante | Red, latencia y disponibilidad |
| Bootstrap del servidor | El servidor evalúa antes y el HTML ya trae el resultado inicial | Renovar al cambiar login, logout o tienda |

En las tres, "el resultado en el browser es una preferencia, no una prueba": solo credenciales para clientes públicos, y el backend valida igual. [T22, p. 14]

El apéndice agrega: resolver el estado inicial antes del primer render evita parpadeos; SSR e hidratación deben partir de la misma decisión; cachear por identidad o cohorte, para no servir HTML express a otra tienda; y al confirmar, el servidor devuelve el modo realmente aceptado. [T22, p. 41]

### Red: routing por cohorte

El gateway autentica y asigna la cohorte: manda classic a Checkout v1 y express a Checkout v2. [T22, p. 15]

- Elimina los headers externos que pretendan elegir la cohorte.
- Inyecta metadata interna después de verificar la identidad.
- Un reparto por pesos distribuye tráfico; estabilizar usuarios requiere afinidad adicional. [T22, p. 15]

El apéndice muestra una regla concreta: un fragmento de `VirtualService` de Istio que manda al subset `v2` los requests con el header `x-checkout-mode: express` y el resto al `v1`. Requiere un `VirtualService` completo, subsets en una `DestinationRule` y workloads etiquetados. El header es un contrato interno del ejemplo: hay que restringir quién puede llegar a ese proxy y emitirlo. [T22, p. 42]

Es el mismo criterio que el gateway aplica a la identidad del usuario. Ver [API Gateway](api-gateway.md#propagación-de-identidad-del-token-al-servicio). [T24, p. 17]

### Canary, release con flag, experimento y shadow

| Técnica | Unidad | Pregunta que responde |
|---|---|---|
| Canary de versión | Tráfico hacia un binario | ¿El nuevo deploy es saludable? |
| Release con flag | Audiencia de una capacidad | ¿A quién habilitamos la función? |
| Experimento | Unidad aleatorizada estable | ¿Qué efecto causa la variante? |
| Shadow traffic | Copia de operaciones | ¿Qué haría el nuevo camino? |

"Se pueden desplegar binarios nuevos al 100 % con la funcionalidad al 0 %." En un shadow de pagos hay que aislar los efectos: nunca duplicar cobros ni notificaciones. [T22, p. 16] El canary como estrategia de despliegue está en [DevOps y CI/CD](devops-y-ci-cd.md), y el shadowing de escrituras, en [Del monolito a microservicios](del-monolito-a-microservicios.md). [T13, p. 42] [T15, p. 26]

### Backend: decidir cerca del comportamiento

```js
// Ejemplo con API de OpenFeature; provider configurado al iniciar
const decision = await client.getStringDetails(
  'checkout-express', 'classic',
  { targetingKey: tenant.id, region: tenant.region },
)
const mode = decision.value === 'express' ? 'express' : 'classic'
const strategy = mode === 'express' ? expressFlow : classicFlow
return strategy.createOrder(command)
```

- OpenFeature desacopla la API de evaluación del proveedor.
- Evaluar **una vez por operación** y pasar una estrategia explícita.
- Registrar el uso del default y los errores: un fallback puede ocultar una falla. [T22, p. 17]

La operación decide una sola vez y elige entre `ClassicFlow` y `ExpressFlow`, que comparten un contrato estable de API y eventos. Se mantiene ejecutable el camino anterior durante la transición, y un consumidor antiguo acepta campos nuevos opcionales. "No repartir `if (flag)` por los servicios; evitar la flag global gigante que acopla cinco equipos." [T22, p. 18]

### Datos: expandir, migrar, contraer

| Fase | Esquema | Escribe | Lee |
|---|---|---|---|
| 1. Expandir | + columna nullable | Formato viejo | Viejo (tolera nuevo) |
| 2. Migrar | Backfill + validar | Viejo + nuevo | Viejo |
| 3. Activar lectura | Ambos formatos | Nuevo | Nuevo, por cohorte |
| 4. Contraer | − formato viejo | Solo nuevo | Solo nuevo |

Las fases 1 a 3 son reversibles, porque el camino viejo sigue siendo válido; contraer no tiene retorno. Antes de contraer hay que esperar consumidores, jobs y eventos atrasados. "La reversión necesita un camino de datos todavía válido. Una flag no recupera una columna eliminada." [T22, p. 20]

En SQL: se agrega `checkout_mode text` a `orders`, donde `NULL` significa classic para las filas históricas; la escritura nueva guarda el modo fijado al crear la orden; y la lectura compatible usa `COALESCE(checkout_mode, 'classic')`. El DDL va por migraciones controladas, revisando locks y duración, y los lectores viejos deben tolerar el esquema expandido. [T22, p. 43]

**Contexto y doble escritura.** [T22, p. 21]

- **Misma base:** escritura compatible en una transacción, cuando sea viable; el modo se pasa como parámetro explícito o contexto de transacción, sin fugas entre usuarios de un pool.
- **Entre stores:** dos writes sueltos pueden divergir; usar outbox o CDC, idempotencia y reconciliación.
- **Antes de cambiar la fuente:** una sola fuente autoritativa; comparar lecturas en sombra, con volumen limitado.

"Desactivar una flag cambia el trabajo futuro. Los efectos previos requieren compensación o reparación." [T22, p. 21] La unidad de migración dice lo mismo con otras palabras: un feature flag no revierte datos. [T15, p. 40] El outbox está en [Comunicación asincrónica y Kafka](comunicacion-asincrona-y-kafka.md). [T14, p. 47]

## Propagación: de la regla a la decisión

La pregunta del bloque: si la regla es la misma para todos, ¿por qué dos servicios pueden decidir distinto para la misma orden? [T22, p. 22]

### Tres cosas que viajan

| Qué viaja | De dónde a dónde | Ejemplo |
|---|---|---|
| Configuración | Control plane → evaluadores | Las reglas de la revisión 42 |
| Contexto de evaluación | Entrada → servicios | Tenant verificado, región |
| Decisión de operación | Orquestador → participantes | Esta orden usa express |

"Distribuir reglas no hace atómica una decisión entre servicios." Hay que elegir qué fijar por request, por sesión o por entidad de negocio. [T22, p. 23]

### Configuración: polling, streaming y caché

Con **polling** cada 30 s, cada réplica consulta en su momento, así que después de publicar `r42` hay una ventana de hasta un intervalo en la que unas réplicas ya tienen `r42` y otras siguen en `r41`. Con **streaming** la adopción es casi inmediata, salvo si se pierde la actualización, por ejemplo una réplica que se reconectó tarde. "Un kill switch necesita un objetivo medido de propagación." Hay que versionar la configuración, descartar actualizaciones atrasadas y medir la edad del snapshot. [T22, p. 24]

### La carrera: el mismo tenant, dos decisiones

El gateway dijo express con las reglas `r42`. Checkout le pide a Pagos cobrar la orden, y Pagos, que todavía tiene `r41`, reevalúa y obtiene classic: el resultado es de otro flujo. "Hash estable + snapshots distintos todavía puede producir variantes distintas." Opciones: aceptar la divergencia si es inocua, o **fijar la decisión de la operación**. [T22, p. 25]

### Fijar una decisión de negocio

```json
{
  "orderId": "ord-731",
  "checkoutMode": "express",
  "decisionRevision": "r42",
  "eventSchemaVersion": 2
}
```

- Checkout decide, valida compatibilidad y **persiste el modo con la orden**.
- Pagos lo recibe por llamada interna o por un evento confiable.
- Los reintentos recuperan la misma orden y la misma decisión (idempotencia).
- El consumidor respeta esa semántica; no vuelve a sortear.
- La revisión ayuda a auditar; no todos los providers permiten evaluar una revisión histórica. [T22, p. 26]

### Propagación segura entre servicios

Internet no es confiable; el interior tiene identidad de servicio y canal autenticado. El gateway autentica la identidad, descarta los headers externos, y asigna y emite la metadata. Un `x-checkout-mode: express` que llega del browser se descarta; adentro viaja `modo=express`, con valores permitidos y emitido solo por el gateway. "Trace context y baggage sirven para correlación, no prueban autorización." Se propagan solo las decisiones necesarias; si un destino no soporta el modo, se rechaza antes de producir efectos o se elige un fallback compatible. [T22, p. 27]

### Eventos, replays y workflows largos

El lunes Checkout emite un evento con `mode = express` y schema v2. El martes se apaga la flag (express → classic). El miércoles el worker consume el evento del lunes. [T22, p. 28]

- **Mal:** consultar la flag actual, que dice classic, y reinterpretar el evento.
- **Bien:** usar el modo que trae el evento, con la misma semántica.
- **Excepción:** un bloqueo operacional puede detener el procesamiento, nunca cambiar el significado. [T22, p. 28]

### Cuando la plataforma de flags falla

| Pregunta | Si la respuesta es no |
|---|---|
| ¿SDK inicializado? | Readiness: no recibir tráfico, o un default seguro explícito |
| ¿Control plane responde? | Última configuración válida, con límite de edad (medida y alertada) |
| ¿Snapshot dentro del límite? | Política por riesgo: Pagos rechaza lo nuevo; Recomendaciones omite |
| ¿Contexto incompleto? | Fallback explícito + señal observable |
| ¿El kill switch no llega? | Ruta de emergencia independiente y ensayada |

Si las tres primeras respuestas son sí, se evalúa. "Seguro no siempre significa `false`. El valor seguro depende del negocio y de la semántica de la flag." [T22, p. 29]

## Observación: ¿lo que liberamos funciona?

La pregunta del bloque: cómo se sabe que el problema, o la mejora, lo causó la flag y no otra cosa. [T22, p. 30]

### Cuatro preguntas

| Nivel | Pregunta | Ejemplo |
|---|---|---|
| 1. Configuración | ¿Quién cambió qué y cuándo? | `r42`: 5 % → 25 %, por ana, 14:02 |
| 2. Evaluación | ¿Qué variante y por qué? | `variant=express`, `reason=targeting` |
| 3. Exposición | ¿Se ejecutó ese camino? | `createOrder(mode=express)` |
| 4. Resultado | ¿Cómo cambió el efecto? | Errores 10 %, confirmados 3,1 % |

"Evaluar no es ejecutar: abrir la pantalla ≠ confirmar el checkout." Se correlaciona con el `trace_id` más la versión de la configuración. [T22, p. 31]

### Telemetría de evaluación

OpenTelemetry define los nombres del evento lógico `feature_flag.evaluation`: `feature_flag.key`, `feature_flag.result.variant`, `feature_flag.result.reason` y `feature_flag.version`. Se emite con contexto de traza, para correlacionarlo con la operación, y se instrumenta desde hooks o desde el SDK, controlando volumen, muestreo y datos sensibles. Esas convenciones estaban en *Release Candidate* al prepararse la clase. [T22, p. 32]

### Métricas que permiten decidir

| Dimensión | Qué medir |
|---|---|
| Salud por variante | Errores, p95/p99, timeouts y saturación |
| Negocio por exposición | Checkouts confirmados / intentos expuestos |
| Evaluador | Defaults, errores, readiness y edad de la configuración |
| Propagación | Latencia de adopción y desacuerdos de modo |

Los labels de las métricas llevan variantes acotadas y un conjunto seleccionado de flags. No se usan tenant, usuario, trace ID ni todas las combinaciones de flags; en logs y trazas se aplican privacidad y retención. [T22, p. 33] Es la regla de cardinalidad de [Observabilidad](observabilidad.md#cardinalidad). [T23, p. 13]

### El promedio global esconde el incidente

El 1 % del tráfico usa express y falla el 10 %; el resto falla el 0,1 %. [T22, p. 34]

| Variante | Requests | Fallas | Tasa de error |
|---|---:|---:|---:|
| Classic | 99.000 | 99 | 0,1 % |
| Express | 1.000 | 100 | 10 % |
| Global | 100.000 | 199 | 0,199 % |

Express falla 100 veces más y el promedio global, por debajo del umbral de pausa de 1 %, ni lo insinúa. "Comparar siempre por variante." El ejemplo es hipotético, y la diferencia observada no prueba causalidad. [T22, p. 34]

## Operación: de 1 % a 100 % y retirar

La pregunta del bloque: cómo aumentar la exposición sin apostar el negocio y cómo cerrar la flag cuando ya no hace falta. [T22, p. 35]

### Rollout con guardrails

El porcentaje de tenants expuestos sube por etapas —internos, 1 %, 5 %, 25 %, 100 %—, con un candado entre una y otra. [T22, p. 36]

**Para avanzar a la siguiente etapa:**

- exposiciones reales y configuración adoptada;
- ventana y volumen mínimos (ciclos de carga);
- errores y latencia frente a classic, sin degradación;
- conversión revisada.

**Pausar o revertir** ante degradación, manteniendo capacidad para el retorno. Regla de ejemplo: errores de express mayores a 1 % con al menos 500 intentos; hay que ajustarla al SLO real. [T22, p. 36]

### Un rollout no es automáticamente un experimento

| | Rollout | A/B test |
|---|---|---|
| Objetivo | Limita riesgo | Mide un efecto |
| Cómo | Puede empezar por clientes internos; sube el porcentaje cuando es seguro | Asignación aleatoria y estable; hipótesis, métrica primaria y duración fijadas antes |
| Pregunta | ¿Es seguro avanzar? | ¿Qué efecto causa la variante? |

Hay que registrar la exposición real y deduplicar conversiones. Si se asigna por tenant, el análisis debe considerar esa agrupación, porque los usuarios de una tienda comparten estado. Aumentar el porcentaje durante el test puede cambiar la interpretación. [T22, p. 37]

### Probar un sistema con flags

Con 10 flags booleanas hay 1.024 combinaciones teóricas (2¹⁰). "No se prueban todas: se prueban las que importan." [T22, p. 38]

- ambas variantes de la capacidad modificada;
- el default, el provider caído y la configuración obsoleta;
- las combinaciones que comparten estado o contratos;
- versiones anteriores de servicios y consumidores;
- reintentos, eventos atrasados y cambio a mitad del flujo;
- la reversión después de escribir datos nuevos.

En los tests se usa un evaluador controlable, sin depender de la consola de producción. [T22, p. 38]

### Operar y retirar sin acumular deuda

| Etapa | Qué incluye |
|---|---|
| Crear | Owner, propósito, default y retiro |
| Rollout | Cohortes y guardrails |
| 100 % estable | Verificar clientes, jobs y mensajes |
| Retirar código | El camino viejo y las evaluaciones |
| Archivar | La configuración, con auditoría |
| Contraer datos | Sin compatibilidad pendiente |

Si nadie la retira, la flag y su camino viejo se quedan: es deuda técnica. Mientras tanto: permisos por ambiente, auditoría, ensayo de reversión y pocos cambios simultáneos. "La flag desaparece; la semántica histórica de las órdenes permanece." [T22, p. 39]

## Autoevaluación de la cátedra

Preguntas de mitad de clase: [T22, p. 19]

1. Un usuario con el modo express pide `/checkout` dos veces. ¿Qué garantiza que vea lo mismo?
2. ¿Por qué el browser no puede ser quien autoriza una capacidad paga?
3. ¿Qué ventaja tiene evaluar una vez y pasar una estrategia, en vez de `if (flag)` en cada servicio?
4. ¿Qué debe aceptar un consumidor antiguo cuando el productor agrega campos nuevos?

Preguntas de cierre, para responder con un ejemplo del checkout: [T22, p. 44]

1. **Despliegue, activación, exposición:** ¿qué cambia entre una y otra, y qué mido en cada una?
2. **Dónde evaluar:** ¿qué decido en browser, red, backend y datos, y qué valido siempre en el servidor?
3. **Coherencia:** ¿qué fijo en la orden para que reintentos y eventos no cambien de modo?
4. **Evidencia:** ¿qué tres señales por variante me hacen pausar un rollout?
5. **Retiro:** ¿qué debo verificar antes de borrar la flag y contraer los datos?

Las respuestas orientativas están en [Preguntas de repaso](../repaso/preguntas.md).

**Lecturas para profundizar:** *Feature Toggles* de Pete Hodgson (categorías y diseño); la especificación de OpenFeature (evaluación, providers y contexto); Unleash (arquitectura y stickiness); *Request Routing* de Istio; *Parallel Change* (evolución compatible); y los eventos de feature flags de OpenTelemetry. [T22, p. 45]

## Relación con otras unidades y con el TP

- **Migración.** El feature toggle separa el deploy del release y elige entre la implementación vieja y la nueva en Branch by Abstraction. [T15, p. 25] [T15, p. 28] `T22` agrega lo que falta cuando la decisión cruza varios servicios: asignación estable, decisión fijada en la entidad y retiro.
- **Resiliencia.** Los feature flags permiten degradar manualmente durante un incidente: es la flag operacional o *kill switch* de esta unidad. [T21, p. 23] [T22, p. 7] [T22, p. 24]
- **TP.** *Inferencia:* la consigna de la tercera entrega no pide feature flags. Sí pide elegir y justificar la estrategia de despliegue (rolling update, blue-green, canary u otra), que es la distinción de la lámina 16 entre canary de versión y release con flag. [E06] [T22, p. 16]
