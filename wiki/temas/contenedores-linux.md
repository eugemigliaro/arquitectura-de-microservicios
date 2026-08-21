# Contenedores sobre Linux

## Modelo mental

Linux no implementa un objeto de kernel llamado *container*. Un contenedor en ejecución es un proceso del host al que se le restringen las vistas del sistema, se le limitan recursos y se le presenta un filesystem propio. La imagen aporta binarios, bibliotecas y demás *userspace*, pero usa el kernel del host. [T04, p. 12] [T04, p. 13]

La síntesis de la cátedra combina tres primitivas:

- **namespaces:** controlan qué ve el proceso;
- **cgroups:** controlan cuánto puede consumir;
- **OverlayFS:** construye la vista de filesystem a partir de capas y una capa escribible. [T04, p. 18] [T04, p. 22]

Esto explica a la vez la eficiencia y el principal costo de seguridad: a diferencia de una VM, el contenedor no arranca un kernel invitado, por lo que todos los contenedores dependen del kernel compartido. [T04, p. 24] [T04, p. 25] [T04, p. 37]

## Namespaces: qué se aísla

| Namespace | Vista aislada |
|---|---|
| `pid` | Árbol y numeración de procesos; el proceso inicial se ve como PID 1. |
| `net` | Interfaces, rutas, reglas y sockets de red. |
| `mnt` | Tabla de montajes y puntos de montaje. |
| `uts` | `hostname` y nombre de dominio NIS. |
| `ipc` | Memoria compartida, semáforos y colas de mensajes. |
| `user` | Mapeo de UID/GID; permite que UID 0 dentro no sea root en el host. |

La enumeración y el alcance provienen de la tabla visual de la clase. [T04, p. 14]

Un namespace de PID no alcanza por sí solo para que `ps` muestre únicamente los procesos internos: también hay que aislar los montajes y remontar `/proc`. El PID 1 interno cumple además funciones de `init`, como recolectar procesos zombis. [T05, p. 21] [T05, p. 22] [T05, p. 23]

Para conectar un namespace de red, un par `veth` actúa como un cable virtual con un extremo en el host y otro dentro del namespace. Un bridge agrega conmutación L2 entre varios pares; su IP puede actuar como gateway y el NAT permite salida hacia Internet. [T05, p. 25] [T05, p. 26] [T05, p. 28] [T05, p. 29]

## cgroups v2: cuánto se consume

Los cgroups no ocultan recursos: imponen límites y contabilizan consumo de CPU, memoria, cantidad de procesos e I/O. En cgroup v2 hay una jerarquía unificada bajo `/sys/fs/cgroup`; un runtime ubica los PIDs en un cgroup hoja y configura controladores como `memory.max`, `cpu.max`, `pids.max` o `io.max`. [T04, p. 16] [T04, p. 18] [T05, p. 40] [T05, p. 41]

| Control | Comportamiento mostrado en la demo |
|---|---|
| Memoria | `memory.max` es un límite duro; excederlo puede producir OOM y `oom_kill`. `memory.high` aplica presión antes del límite. [T05, p. 42] |
| CPU | `cpu.max` expresa cuota y período; al agotar la cuota el kernel pausa el proceso, no lo mata. [T05, p. 43] |
| I/O | `io.max` limita por dispositivo de bloque (`MAJ:MIN`), no por path; el page cache puede ocultar el efecto en una prueba. [T05, p. 45] |

## Filesystem y capas

`chroot` cambia la raíz aparente para resolver paths, pero no aísla la tabla de montajes. Un mount namespace separa esa tabla; `pivot_root` intercambia la raíz nueva por la anterior y permite desmontar la raíz vieja, mecanismo más cercano al usado por los runtimes. [T05, p. 11] [T05, p. 14] [T05, p. 17] [T05, p. 19]

OverlayFS expone un *union mount* sin copiar las capas de solo lectura:

| Directorio | Rol |
|---|---|
| `lowerdir` | Capas de imagen de solo lectura, compartibles. |
| `upperdir` | Cambios escribibles particulares del contenedor. |
| `workdir` | Área interna que el kernel usa para operar el overlay. |
| `merged` | Punto de montaje con la vista unificada que consume el proceso. |

Las escrituras usan *copy-on-write* hacia `upperdir`; un borrado se representa mediante un *whiteout* sin modificar `lowerdir`. [T04, p. 19] [T04, p. 21] [T05, p. 50]

## Del comando al kernel

La OCI separa dos contratos: la *Image Spec* define cómo empaquetar una imagen y la *Runtime Spec* cómo ejecutar un contenedor. Ese contrato permite cambiar el runtime manteniendo el formato de bundle. [T04, p. 29]

El recorrido simplificado es:

`docker run` → Docker Engine → `containerd` → `runc` → kernel Linux.

`containerd` descarga y desempaqueta capas, prepara el root filesystem y genera la configuración del bundle. `runc` lee `config.json`, crea namespaces, aplica cgroups, monta el rootfs, ejecuta `pivot_root`, configura controles de seguridad y finalmente reemplaza el proceso con la aplicación. [T04, p. 29] [T04, p. 30] [T04, p. 31]

Un bundle OCI mínimo contiene `rootfs/` y `config.json`. La demo muestra que `runc` aplica namespaces, cgroups y capabilities a partir de esa descripción. [T05, p. 51]

## Defensa en profundidad

Las capabilities dividen los privilegios históricos de root en permisos de kernel independientes. Docker conserva un conjunto acotado por defecto y permite agregar o quitar capabilities; `CAP_NET_BIND_SERVICE`, por ejemplo, habilita bindear puertos privilegiados sin otorgar todos los permisos. [T04, p. 33] [T04, p. 34]

La protección no depende de una sola frontera: seccomp filtra syscalls, AppArmor o SELinux aplican control de acceso obligatorio y el descarte de capabilities sigue el principio de menor privilegio. [T04, p. 32] [T05, p. 47] [T05, p. 48]

Como los contenedores comparten el kernel, un escape que explote una vulnerabilidad del kernel puede alcanzar el host. La clase presenta gVisor —interposición de kernel en userspace— y Kata Containers —una micro-VM por contenedor— como alternativas de mayor aislamiento que conservan compatibilidad OCI. [T04, p. 37]

## Guía del laboratorio manual

La demo recorre nueve fases: filesystem; procesos; red; identidad; usuarios; recursos; seguridad; capas/runtime; e integración con ciclo de vida. [T05, p. 5]

| Fase | Resultado verificable |
|---|---|
| Filesystem | `/` apunta al rootfs aislado y la raíz anterior deja de estar visible. [T05, p. 12] [T05, p. 18] |
| Procesos | El shell se ve como PID 1 y `/proc` muestra solo la vista del namespace. [T05, p. 22] |
| Red | Dos namespaces se comunican mediante `veth` y bridge. [T05, p. 26] [T05, p. 29] |
| Identidad e IPC | Cambiar el hostname o crear IPC dentro no afecta la vista del host. [T05, p. 32] [T05, p. 34] |
| Usuario | UID 0 interno se mapea a un UID sin privilegios externos. [T05, p. 36] [T05, p. 37] |
| Recursos | Los procesos exhiben OOM, throttling de CPU y límites de I/O observables. [T05, p. 42] [T05, p. 43] [T05, p. 45] |
| Runtime | OverlayFS y un bundle OCI reproducen las abstracciones de una imagen y `runc`. [T05, p. 50] [T05, p. 51] |
| Integración | Un script combina namespaces, `pivot_root`, `/proc` y cgroup v2. [T05, p. 53] |

El ciclo de vida manual `create → start → stop → delete` se corresponde conceptualmente con `docker create`, `docker start/run`, `docker stop` y `docker rm`. Para detener se usa el PID del host registrado en `cgroup.procs`, no el PID 1 visible dentro. [T05, p. 54]

### Advertencias operativas del material

- La demo presupone una VM Linux Lima y varias fases requieren privilegios; el setup modifica una restricción de AppArmor y el cleanup elimina rutas de trabajo bajo `/tmp`. [T05, p. 7] [T05, p. 8] [T05, p. 9]
- Desmontar `/proc` sin un mount namespace puede afectar al host. [T05, p. 21]
- Para el segundo namespace de red, la propia fuente corrige que `10.0.1.2/24` debe configurarse sobre `veth3`, el extremo movido al namespace, y no sobre `veth2` en el host. [T05, p. 27]
- El dispositivo usado por `io.max` debe detectarse; no corresponde asumir un `MAJ:MIN` fijo. [T05, p. 45]

