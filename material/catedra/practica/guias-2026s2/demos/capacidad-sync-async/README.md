# Demo de capacidad: sincrónico vs. cola

Qué pasa cuando llegan más pedidos de los que el sistema puede atender, y qué
cambia según cómo lo enfrentes: aceptar todo, acotar la admisión, reintentar, o
meter una cola en el medio.

No es una simulación. Son procesos reales, con límites de CPU y memoria
impuestos por cgroups, y las métricas se leen del cgroup y del reloj del
cliente. Se usó en vivo en la **clase 6 — Testing / Comunicación Sync**.

---

## Requisitos

| | |
|---|---|
| **Linux** | `scripts/run.sh` lee `/sys/fs/cgroup` directo. En macOS/WSL2 las corridas andan, pero las columnas de CPU y memoria salen en cero |
| **cgroups v2** | verificalo con `stat -fc %T /sys/fs/cgroup` → tiene que decir `cgroup2fs` |
| **Docker + Compose v2** | `docker compose version` (con espacio, no `docker-compose`) |
| `bash`, `curl`, `bc` | `bc` viene aparte en varias distros: `sudo apt install bc` |

No hace falta Go instalado: compila adentro del contenedor.

## Arrancar

```bash
docker compose build          # primera vez: 1-2 min

./scripts/run.sh sync-naive      # 1. el sync que escribiría cualquiera
./scripts/run.sh sync-bulkhead   # 2. + sala de espera acotada, 503 al llenarse
./scripts/run.sh sync-retry      # 3. + reintentos del cliente
./scripts/run.sh queue           # 4. NATS JetStream en el medio
./scripts/run.sh all             # las cuatro en orden

./scripts/run.sh naive-retry     # extra: retry SIN admisión acotada
```

**Las corridas 2 y 3 se suman a la anterior**: la 3 es bulkhead **y** retry, no
retry solo. Para ver qué pasa cuando reintentás sin acotar la admisión está
`naive-retry`, que es la corrida 1 más reintentos.

Cada corrida levanta su perfil desde cero, corre el generador de carga, imprime
el reporte y lee los recursos del cgroup. La 1 además se queda mirando el server
después de que el cliente se rindió: ahí se ve el trabajo huérfano.

Para bajar todo: `docker compose --profile sync --profile queue down`.

## Calibrar antes de presentar

**Esto no es opcional.** `WORK_ITERS` es una cantidad fija de trabajo
(SHA-256), pero cuánto *tarda* depende de la máquina: `crypto/sha256` usa la
instrucción SHA-NI cuando el procesador la tiene, y la diferencia es de varias
veces. Sin calibrar, en una máquina lenta la corrida 1 completa 0 de 100 y la 4
tarda 80 segundos con la clase esperando; en una rápida no hay sobrecarga y no
se ve nada.

El objetivo es **~0,25 CPU-segundos por request**:

```bash
./scripts/run.sh sync-naive
# de la salida, tomá el total de CPU-segundos y dividilo por 100:
#   TOTAL   25.30 CPU-s en total   ->  0.25 por request  -> está calibrado
#   TOTAL   12.00 CPU-s en total   ->  0.12 por request  -> subí WORK_ITERS
```

Y volvés a correr con el valor nuevo:

```bash
WORK_ITERS=4000000 ./scripts/run.sh sync-naive
```

Cuando encontrás el número, dejalo fijo para toda la sesión (exportalo o
cambiá el default en `docker-compose.yml`). Si cambia entre corridas, la
comparación no vale.

`CONCURRENCY` **no** entra en esta cuenta: con la CPU como cuello de botella, μ
la fijan `cpus` y `WORK_ITERS`. `CONCURRENCY` controla cuántos requests están en
vuelo a la vez — o sea memoria y latencia de cada uno, no capacidad.

## Cómo leer la salida

```
=== 1. sync naive (sin bulkhead) ===
  requests enviados         100
  intentos HTTP del cliente 100
  completados (ok)          6
    de esos, en < 3000 ms   6      <-- mismo SLA para las dos corridas
  NO COMPLETADOS            94  (503:0 timeout:94 error:0)
  en DLQ (apartado, no perdido en silencio)  0
  latencia p50/p95/p99      3000 / 3000 / 3000 ms   (n=6, solo completados)
  cliente ocupado           15.02 s   <-- acoplamiento temporal
  corrida terminada en      30.14 s
  intentos por completado   1.00 promedio  (peor caso 1)
  goodput                   0.40 completados/s
```

Las líneas que importan:

- **completados** vs. **de esos, en < 3000 ms** — no es lo mismo. La corrida 4
  completa 99 y entrega 4 dentro del SLA.
- **NO COMPLETADOS** desglosado: un `503` es un rechazo explícito y barato; un
  `timeout` es el cliente que se cansó mientras el server seguía trabajando.
- **cliente ocupado** — cuánto tiempo el que llama estuvo esperando en vez de
  hacer otra cosa. Es el costo que no aparece en las métricas del servidor.
- **intentos por completado** — cuánto hubo que insistir. Es el dato con el que
  se elige un `maxAttempts` de verdad, en vez de poner 3 porque sí.

Al final de cada corrida el `loadgen` verifica un invariante:

```
ok + rechazados + timeouts + errores + dlq == enviados
```

Si no cierra, sale con código 1. Sin eso la comparación no probaría nada:
podrían estar desapareciendo requests sin que nadie los cuente.

## Las perillas

Todas se pasan como variables de entorno delante del script. Los defaults
de abajo son los que fija `docker-compose.yml`, que es lo que ves al correr;
el código tiene otros para cuando se lo usa suelto.

```bash
CLIENTS=40 CLIENT_TIMEOUT_MS=5000 ./scripts/run.sh sync-naive
```

**Carga** (las lee el `loadgen`):

| Variable | Default | Qué hace |
|---|---|---|
| `CLIENTS` | 20 | Clientes concurrentes. Es **la perilla de la sobrecarga** (ver abajo) |
| `REQS_PER_CLIENT` | 5 | Pedidos que hace cada uno, en orden |
| `LOAD_MODE` | `seq` | `seq` = closed-loop (espera la respuesta). `burst` = los 100 de una |
| `CLIENT_TIMEOUT_MS` | 3000 | Paciencia del cliente. También es el SLA del reporte |
| `RETRY_MAX` | 0 | Reintentos del cliente. Lo setea `run.sh` según la corrida |

**Servicio** (las leen `service` y `worker`):

| Variable | Default | Qué hace |
|---|---|---|
| `WORK_ITERS` | 2000000 | Cuánto trabajo hace un request. **Calibrar** |
| `CONCURRENCY` | 3 | Requests procesándose a la vez por worker |
| `ADMIT_QUEUE` | 2 | Sala de espera del bulkhead. `CONCURRENCY + ADMIT_QUEUE` = admitidos |
| `BULKHEAD` | `false` | Si acota la admisión y devuelve 503, o acepta todo |
| `FAILURE_RATE` | 0.2 | Probabilidad de falla, **después** de haber hecho el trabajo |
| `HOLD_KB` | 2048 | Memoria que retiene cada request en vuelo |
| `MAX_DELIVER` | 3 | Reintentos del broker antes de mandar a la DLQ |
| `MAX_QUEUE_MSGS` | 500 | Techo del stream. Al llenarse, el producer vuelve a dar 503 |

**Por qué 20 clientes y no 10:** en closed-loop la tasa de llegada la fija la
propia espera. Con 3 s de paciencia cada cliente pide como mucho una vez cada
3 s, así que 20 demandan ≈ 6,7 req/s contra los ≈ 4 que el sistema puede: λ > μ
de verdad. Con 10 la demanda queda en ≈ 3,3, *por debajo* de μ, no hay
sobrecarga y los números oscilan salvajemente entre corridas.

**Escalado vertical** — sin tocar el código, solo el quota:

```bash
cat > docker-compose.override.yml <<'EOF'
services:
  service-1:
    cpus: 4
  service-2:
    cpus: 4
EOF
./scripts/run.sh sync-naive
rm docker-compose.override.yml   # acordate de borrarlo
```

## Dos cosas del `docker-compose.yml` que no vimos en clase

### `profiles:` — servicios que no arrancan solos

Todos los servicios de este compose declaran un perfil:

```yaml
service-1:
  profiles: ["sync"]
nats:
  profiles: ["queue"]
loadgen:
  profiles: ["tools"]
```

Un servicio con `profiles` **no arranca** con un `docker compose up` pelado:
hay que pedirlo explícitamente.

```bash
docker compose up -d                    # no levanta NADA acá
docker compose --profile sync up -d     # service-1 y service-2
docker compose --profile queue up -d    # nats, producer, worker-1, worker-2
docker compose run --rm loadgen         # `run` activa el perfil del servicio solo
```

Acá no es cosmético: los dos perfiles compiten por la misma CPU y por los
mismos puertos. Si arrancaran juntos, la comparación no valdría. El perfil
`tools` es el otro uso típico — un servicio que nunca corre como daemon, solo
a demanda con `docker compose run`.

Un servicio puede estar en varios perfiles (`profiles: ["sync", "debug"]`), y
las dependencias de `depends_on` se activan solas aunque no las pidas.

### `x-`, `&`, `*` y `<<:` — no repetir tres veces lo mismo

Esto no es de Docker: es **YAML**, y lo resuelve el parser antes de que Compose
vea nada.

```yaml
x-work: &work          # 1. clave `x-`: Compose ignora todo lo que empieza así
  WORK_ITERS: "2000000"
  CONCURRENCY: "3"

services:
  service-1: &svc      # 2. `&svc` define un anchor sobre este mapa
    environment:
      <<: *work        # 3. `<<:` mete acá adentro las claves de *work
      ROLE: sync
    ports: ["8081:8080"]

  service-2:
    <<: *svc           # service-2 es igual a service-1...
    ports: ["8082:8080"]   # ...salvo por lo que redeclara
```

Tres piezas:

- **`x-<algo>`** es un *extension field*. Compose ignora cualquier clave de
  primer nivel que empiece con `x-`, así que es el lugar donde declarar un
  bloque para reusar sin que Docker intente interpretarlo como un servicio.
- **`&nombre`** es un **anchor**: le pone nombre a ese nodo del YAML.
  **`*nombre`** es el **alias** que lo referencia.
- **`<<:`** es la **merge key**: copia las claves del mapa referenciado dentro
  del mapa actual. Lo que declarás localmente **pisa** lo que viene del merge.

**El detalle que sorprende:** el merge es de **un solo nivel**, no recursivo.
Por eso `worker-2` vuelve a escribir su bloque `environment` entero en vez de
agregarle `WORKER_ID` al de `worker-1`: si pusiera `<<: *worker` y después un
`environment` con una sola clave, ese `environment` **reemplaza** al heredado,
no se fusiona con él.

Si te suena a herencia, ojo: no hay herencia acá. Es copiar y pegar, hecho por
el parser. Compose tiene además `extends:`, que sí es un mecanismo propio de
Docker y funciona entre archivos distintos.

## El diseño del experimento

### Los parámetros de la consigna, hechos reales

| Consigna | Implementación |
|---|---|
| clientes | 20 goroutines, una por cliente (`CLIENTS`) |
| N pedidos cada uno | 5 por cliente = 100 (`REQS_PER_CLIENT`) |
| **cómo los manda** | `LOAD_MODE=seq`: closed-loop, no manda el siguiente hasta que vuelve el anterior |
| 2 workers | dos services con nombre propio, para verlos separados |
| que pueden 3 por X tiempo | `CONCURRENCY=3` + `cpus: 0.5` por worker |
| con Y% de posibilidades de falla | `FAILURE_RATE=0.2`, y la falla ocurre **después** del trabajo |

### Por qué closed-loop y no un burst

La primera versión mandaba los 100 requests de una (`LOAD_MODE=burst`). Es un
escenario que no existe: ningún cliente dispara diez pedidos simultáneos y se
queda esperando los diez. Y tenía un costo concreto — con el burst, "10 clientes
× 10 pedidos" es indistinguible de "100 clientes × 1 pedido": la identidad del
cliente no transporta nada, y el retry parece amplificación gratuita.

En closed-loop cada cliente tiene una lista de cosas que hacer y las hace en
orden, esperando. Eso cambia tres cosas:

1. **La tasa de llegada la fija la propia espera**, y el número de clientes pasa
   a ser la perilla que regula la sobrecarga.
2. **El retry pasa a tener sentido**: el pedido sigue pendiente, el cliente
   todavía lo necesita. Duplica los completados en vez de ser ruido.
3. **El trabajo huérfano es peor**, no mejor: el cliente que se rindió ya mandó
   el pedido siguiente, así que el server compite consigo mismo.

`LOAD_MODE=burst` sigue existiendo y sirve para mostrar el pico instantáneo,
pero los números de abajo son de closed-loop.

### Las dos decisiones que hacen que la comparación valga

**Un solo binario, tres roles.** `cmd/service` corre como `sync`, `producer` o
`worker` según `ROLE`. La función de trabajo, el runtime, el GC y el código de
medición son *el mismo código* en las cuatro corridas. Lo único que cambia es el
camino de admisión. Si fueran dos servicios distintos quedaría siempre la
objeción "y si la diferencia la hace el cliente de AMQP, u otro framework".

**El mismo límite de CPU en todas.** El presupuesto del trabajo pesado es
idéntico en los dos perfiles: 2 × 0,5 = **1 core**. Por eso μ es la misma y la
comparación vale. Lo que async agrega (`producer` + `nats`) tiene sus propios
límites declarados a propósito: es el costo de infraestructura de no perder
requests, y en la demo **se muestra, no se esconde**.

## Resultados medidos

Medidos el 2026-09-10 con `WORK_ITERS=2000000`, `CLIENTS=20`,
`REQS_PER_CLIENT=5`, `LOAD_MODE=seq`. Varían entre corridas porque
`FAILURE_RATE` es aleatorio y porque el cgroup mide tiempo y no ciclos; los
órdenes de magnitud son estables. **Los números de CPU no son reproducibles
entre máquinas** — regeneralos en la que vayas a usar.

| | 1. sync naive | 2. + bulkhead | 3. + retry | 4. queue |
|---|---|---|---|---|
| completados | 6 | 21 | 41 | **99** |
| **— dentro del SLA de 3 s** | 6 | 21 | **41** | **4** |
| no completados | 94 (timeout) | 79 (68 × 503) | 59 (56 × 503) | **0** |
| en DLQ (apartado, no perdido) | — | — | — | 1 |
| intentos HTTP del cliente | 100 | 100 | **311** (×3,11) | 100 |
| ejecuciones del trabajo | 100 | ~32 | ~52 | **120** (×1,20) |
| **intentos por completado** | 1,00 | 1,00 | **1,83** (peor caso 4) | — |
| cliente ocupado | 15,0 s | 9,1 s | 20,8 s | **0,05 s** |
| p50 de latencia | 3,0 s | 1,7 s | 1,7 s | 17,5 s |
| trabajo huérfano | **15 s** | — | — | — |
| **pico de memoria** | **93 MiB** | 28 MiB | 28 MiB | **21 MiB** |
| infra extra | — | — | — | nats 12,7 + producer 9,7 MiB |
| profundidad de cola | — | — | — | 100 mensajes en el pico |

CPU y memoria se leen del **cgroup** (`cpu.stat/usage_usec` y `memory.peak`), no
se muestrean: una corrida de 3 s puede caer entre dos muestras de `docker stats`
y reportar un pico que no es el pico.

El modo extra, retry **sin** admisión acotada: **8 completados, 376 intentos,
218 MiB, 65 s de cliente ocupado** — contra 6 completados y 93 MiB del naive
pelado. Gana dos pedidos y duplica la memoria, porque los reintentos se apilan
adentro del server como requests vivos.

### Escalado vertical

Misma carga, solo cambia el quota:

| | 1 core | 4 cores | 8 cores |
|---|---|---|---|
| completados | 5 | 88 | 79 |
| timeouts | 94 | **0** | **0** |
| pico de memoria | 89 MiB | 53 MiB | 53 MiB |

A 4 cores no queda un timeout: lo que falta para 100 son las fallas inyectadas
del 20 %, y por eso 8 puede dar menos que 4 (es el sorteo, no el hardware). La
memoria baja a medias — 89 a 53 MiB — contra los 28 que consigue el bulkhead
sin tocar el hardware.

## Lo que dicen los números

**La fila que importa no es "completados": es "dentro del SLA".** La cola
completa 99 de 100 y no pierde nada — es cierto. Pero bajo el mismo deadline de
3 segundos entrega **4**, contra **41** de sync con bulkhead y retry. La cola no
reduce pérdidas bajo el mismo deadline: **convierte un problema de pérdida en un
problema de latencia**, con un p50 de 17 s.

**Acotar la admisión protege al servicio y además mejora el resultado.** De 6 a
21 completados, y la memoria de 93 a 28 MiB. Al acotar cuántos entran, lo que
entra termina en 1,7 s en vez de morir a los 3. No hace falta un broker para
esto.

**Con clientes reales, el retry es la otra mitad.** De 21 a 41 completados,
todos dentro del SLA, a cambio de ×3,11 en intentos. Y funciona **solo porque el
bulkhead rechaza barato**: reintentar contra un server sin admisión acotada es
echar nafta. El costo lo paga el cliente con su tiempo: de 9 a 21 segundos
ocupado.

**Cuánto hay que insistir: 1,83 intentos por pedido exitoso, peor caso 4.** Más
de la mitad sale al primer intento. Es la pregunta práctica de cualquiera que
implementa un retry, y la respuesta se mide, no se adivina.

**El 202 instantáneo destruye la contrapresión.** En sync, el cliente esperando
es lo que frena la llegada de trabajo nuevo. Con la cola el closed-loop se
degenera en un burst: los 100 pedidos entran en 0,05 s y el backlog aparece
entero en el broker (profundidad máxima: 100). La cola no absorbió el pico: lo
mudó de lugar.

**Async no es gratis.** Dos procesos nuevos, ~22 MiB, y las redeliveries del
broker son trabajo real: 120 ejecuciones para 100 pedidos.

**El criterio que sale de todo esto:** si el que llama necesita la respuesta
para seguir, la cola no es la solución — acotar la admisión y reintentar sí lo
es. La cola gana cuando nadie está esperando: 99 terminados, cero perdidos, cero
trabajo huérfano. Autorizar un pago necesita sync; emitir el comprobante, no.

## Lo que esta demo NO prueba

El stream es `MemoryStorage`: **si el proceso de NATS se muere, el backlog se
pierde entero.** La cola te salvó de perder requests por *sobrecarga*, no de
perderlos por un *crash*.

El broker puede **entregar el mismo mensaje dos veces** (at-least-once). El
`loadgen` deduplica por ID en silencio para poder contar; un consumidor real
necesita ser idempotente.

**Con λ > μ sostenida, ninguna cola alcanza.** El stream está acotado
(`MAX_QUEUE_MSGS=500`, `DiscardNew`) justamente para que eso sea verdad y no una
promesa: cuando el backlog llega al techo, el producer vuelve a devolver 503 —
solo mucho más tarde que un server sin buffer. La cola absorbe picos
**transitorios**; si la sobrecarga es permanente, solo posterga el colapso.

Persistencia, garantías de entrega, particiones, consumer groups y EDA son tema
de la **clase 7**. `WorkQueuePolicy` es una decisión deliberada para que esto
sea una cola de trabajo y nada más.

## Estructura

```
cmd/service/     sync | producer | worker  (un binario, tres roles)
cmd/loadgen/     generador de carga y única herramienta de medición
internal/work/   Hold() = costo en memoria del request en vuelo
                 Burn() = costo en CPU, cantidad fija de trabajo
internal/report/ tipos compartidos, percentiles y contabilidad
scripts/run.sh   orquesta cada corrida y lee los recursos del cgroup
```

## Si algo no anda

| Síntoma | Qué pasa |
|---|---|
| CPU y memoria en `0.00` | El script no encontró el cgroup: no estás en Linux, o es cgroups v1 |
| `bc: command not found` | `sudo apt install bc` |
| `bind: address already in use` | Quedó un perfil levantado. `docker compose --profile sync --profile queue down` |
| La corrida 1 completa 0 de 100 | `WORK_ITERS` muy alto para esta máquina. Ver **Calibrar** |
| La corrida 1 completa 80 de 100 | `WORK_ITERS` muy bajo: no hay sobrecarga que mostrar |
| La corrida 4 tarda muchísimo | Lo mismo, o quedó `WAIT_FOR_S` alto con el broker vacío |
| `la contabilidad no cierra` | Bug real de contabilidad en el `loadgen`. No lo tapes: los números de esa corrida no valen |
