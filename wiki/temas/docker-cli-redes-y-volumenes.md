# Docker CLI, redes y volúmenes

## Del cliente al proceso

La CLI `docker` es un cliente HTTP: envía pedidos al daemon `dockerd` mediante su API, normalmente a través de `/var/run/docker.sock`. El daemon gestiona imágenes, redes y volúmenes y delega la ejecución en `containerd`, `runc` y, finalmente, el kernel Linux. Por eso una CLI local puede operar contra un daemon remoto si cambia `DOCKER_HOST`. [T08, p. 4]

El socket no es una frontera menor de administración. Quien puede usarlo puede pedir montajes del host y crear contenedores privilegiados; pertenecer al grupo `docker` equivale, en la práctica presentada por la cátedra, a disponer de acceso root sobre la máquina. Montar el socket dentro de un contenedor extiende ese riesgo al proceso contenido. [T08, p. 5]

`docker run` combina `create` y `start`. Un contenedor puede pasar por `created`, `running`, `paused` y `exited`; `docker rm` elimina el objeto y su capa de escritura. `pause` usa el freezer del cgroup, por lo que suspende los procesos sin enviarles señales. [T08, p. 7]

Al ejecutar `docker run`, todo lo que aparece después de la referencia de imagen reemplaza el `CMD`; opciones como `--network`, `-p`, `--mount`, `--memory` o `--cpus` terminan materializándose como namespaces, reglas de red, mounts y controles de cgroup. El flujo completo incluye crear el contenedor por API, resolver y desempaquetar la imagen, preparar red y configuración OCI, y hacer que `runc` aplique aislamiento y ejecute el proceso. [T08, p. 8] [T08, p. 9] [T08, p. 10]

## Operación y diagnóstico

Las herramientas se complementan según la pregunta operativa:

| Pregunta | Herramienta |
|---|---|
| ¿Qué corre o terminó y con qué código? | `docker ps -a`. [T08, p. 11] |
| ¿Qué emitió el proceso principal? | `docker logs`; el daemon captura `stdout` y `stderr`, no cualquier archivo interno. [T08, p. 12] |
| ¿Cuál es la configuración efectiva? | `docker inspect`, incluidos estado, red y mounts. [T08, p. 11] |
| ¿Qué cambió respecto de la imagen? | `docker diff`, que observa cambios de la capa escribible. [T08, p. 11] |
| ¿Responde el proceso dentro de sus namespaces? | `docker exec`, que inicia un proceso adicional dentro del contenedor. [T08, p. 13] |

`docker attach`, en cambio, conecta la terminal al `stdin/stdout` del PID 1; un `Ctrl-C` puede enviarle `SIGINT` y detener el contenedor. En una parada normal, `docker stop` envía `SIGTERM` al PID 1 y, si no termina dentro del período de gracia, recurre a `SIGKILL`; `--init` agrega un init mínimo que reenvía señales y recolecta zombis. [T08, p. 13] [T08, p. 14]

## Redes: del namespace al tráfico exterior

Cada contenedor comienza con un network namespace propio, que separa interfaces, rutas, firewall y puertos. Esto permite que varios contenedores escuchen el mismo puerto interno sin colisionar. [T08, p. 18]

En una red bridge, el engine coordina varios objetos del kernel: asigna una subred mediante IPAM, crea un bridge, conecta cada namespace con un par `veth`, configura IP y ruta predeterminada, habilita forwarding y agrega reglas de filtrado y NAT. El tráfico saliente usa SNAT/MASQUERADE; para el exterior el contenedor no aparece con su dirección privada. [T08, p. 20] [T08, p. 21]

Publicar `-p 8080:80` agrega DNAT desde el puerto 8080 del host al 80 del contenedor. Si no se especifica dirección, el puerto puede quedar expuesto en las interfaces del host; `-p 127.0.0.1:8080:80` lo restringe a loopback. Docker inserta sus propias reglas de firewall, y el material indica `DOCKER-USER` como la cadena prevista para restricciones anteriores a las reglas administradas por el engine. [T08, p. 22] [T08, p. 23]

Conviene distinguir el bridge predeterminado de una red bridge creada por el usuario. La segunda ofrece DNS embebido por nombre de contenedor o alias y aislamiento respecto de otras redes; dentro del namespace, el resolver del daemon aparece como `127.0.0.11`. [T08, p. 24] [T08, p. 25]

| Modo | Propiedad central y uso |
|---|---|
| Bridge propio | Opción normal para un stack local: namespace por contenedor, conectividad interna y NAT. [T08, p. 19] [T08, p. 30] |
| `host` | Comparte la red del host, sin `veth`, bridge ni NAT; pierde aislamiento y `-p` deja de tener sentido. [T08, p. 26] |
| `none` | Conserva un namespace con loopback pero sin salida; sirve para trabajo que no necesita red. [T08, p. 27] |
| `container:<id>` | Comparte IP, interfaces y `localhost` con otro contenedor; representa el modelo de red de un pod con sidecars. [T08, p. 27] |
| `macvlan` / `ipvlan` | Presenta al contenedor con una IP de la LAN, evitando NAT, con restricciones propias de la red física. [T08, p. 28] |
| `overlay` | Extiende una red virtual entre hosts mediante VXLAN; agrega encabezado, reduce MTU efectiva y requiere plano de control distribuido. [T08, p. 29] |

Para diagnosticar conectividad, la secuencia propuesta es verificar pertenencia a la misma red, resolución de nombre, alcance del puerto, dirección donde escucha la aplicación y, por último, reglas DNAT. [T08, p. 32]

## Estado y mounts

La capa escribible del contenedor pertenece a su ciclo de vida: `docker rm` la elimina. Además, modificar allí un archivo proveniente de una capa inferior exige *copy-up*, por lo que una base de datos, uploads u otro estado importante no deberían depender de esa capa. [T08, p. 34]

| Tipo | Quién lo gestiona | Persistencia | Uso principal |
|---|---|---|---|
| Volumen | Docker, fuera de la unión OverlayFS. | Sobrevive a `docker rm`. | Datos importantes, bases y datos compartidos. [T08, p. 35] [T08, p. 37] |
| Bind mount | El usuario, mediante un path existente del host. | Es el mismo dato del host. | Código de desarrollo y configuración; puede montarse read-only. [T08, p. 35] [T08, p. 38] |
| `tmpfs` | Kernel, sobre RAM del host. | Se destruye con el contenedor. | Secretos, temporales y scratch de alta frecuencia. [T08, p. 35] [T08, p. 39] |

La sintaxis `-v` es posicional y puede crear silenciosamente el directorio de origen de un bind inexistente; `--mount` expresa `type`, `src` y `dst` con claridad y falla cuando falta ese origen. [T08, p. 36]

Un volumen vacío recibe inicialmente el contenido y permisos que la imagen tenga en el path montado. Los volúmenes anónimos obtienen un identificador automático y pueden quedar huérfanos; la instrucción `VOLUME` de un Dockerfile crea uno nuevo en cada `docker run`. El material también advierte que `docker volume prune` puede borrar datos cuando ningún contenedor referencia el volumen, incluso si el stack sólo está detenido y removido. [T08, p. 37] [T08, p. 41]

Un bind mount expone el mismo inodo del host dentro del mount namespace y tapa el contenido que la imagen tuviera en el destino. Los permisos se comparan por UID/GID numéricos: los nombres de usuario del host y de la imagen no traducen identidades, y un contenedor que escribe como root puede dejar archivos de root en el host. [T08, p. 38] [T08, p. 42]

`tmpfs` no toca disco y su consumo cuenta contra el límite de memoria del cgroup. Es útil junto a un root filesystem read-only para ofrecer únicamente paths temporales escribibles; `/dev/shm` es un caso relacionado cuyo tamaño puede requerir ajuste explícito. [T08, p. 39]

La regla de selección de la clase es: volumen si importa el dato, bind mount si importa el path del host y `tmpfs` si el contenido no debe persistir. [T08, p. 43]

## Relación con Compose

Esta unidad muestra los objetos imperativos que luego Compose declara: `ports` expresa publicación y DNAT, `networks` crea conectividad y DNS entre servicios, y `volumes` o bind mounts definen dónde vive el estado. La propia presentación enlaza este cierre con la clase siguiente de Compose. [T08, p. 45] [T07, p. 8] [T07, p. 9]
