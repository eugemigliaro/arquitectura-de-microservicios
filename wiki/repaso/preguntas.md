# Preguntas de repaso

Preguntas recuperables por tema. Mantener las respuestas separadas o plegadas cuando eso ayude a practicar recuperación activa.

## Fundamentos

1. ¿Qué independencias debe ofrecer un microservicio y qué papel cumple su interfaz?
2. ¿Por qué un servicio no es lo mismo que una librería?
3. Compará monolito, SOA y microservicios en integración, datos y gobierno.
4. ¿Qué beneficios y costos inherentes enumera la cátedra?
5. ¿Qué variables de negocio usarías para elegir entre monolito y microservicios?
6. ¿Qué convierte a un conjunto de servicios en un monolito distribuido?

## Diseño y límites

7. Explicá bajo acoplamiento y alta cohesión con un ejemplo de cambio.
8. ¿Por qué conviene diseñar un contrato alrededor de capacidades y no solo de CRUD?
9. ¿Cómo relaciona la ley de Conway la organización con la arquitectura?
10. ¿Qué problema resuelve el contract testing y qué no reemplaza?
11. Nombrá los módulos de la anatomía genérica de un microservicio y su responsabilidad.

## DDD estratégico

12. ¿Por qué “un servicio por tabla” suele producir malos límites?
13. Diferenciá domain, subdomain y bounded context.
14. ¿Por qué el lenguaje ubicuo no debería ser global a toda la empresa?
15. Compará Core, Supporting y Generic Subdomains.
16. ¿Cuándo elegirías Conformist y cuándo una ACL?
17. ¿Qué diferencia hay entre OHS y Published Language?
18. ¿Por qué un bounded context no equivale necesariamente a un microservicio?

## DDD táctico y EventStorming

19. Diferenciá Entity y Value Object mediante su criterio de igualdad.
20. ¿Qué protege la frontera de un Aggregate y cómo se accede desde afuera?
21. Contrastá Command y Domain Event en tiempo verbal, posibilidad de fallo y dirección.
22. ¿Qué elementos aparecen en la notación de EventStorming?
23. Reconstruí el camino feliz del ejemplo de e-commerce solo con eventos.
24. Modelá el fallo de un cobro: evento inicial, command compensatorio, evento resultante y contextos involucrados.

## Contenedores sobre Linux

25. ¿Por qué un contenedor no es una VM pequeña ni un objeto nativo del kernel?
26. Diferenciá namespaces y cgroups por la pregunta que responde cada mecanismo.
27. ¿Por qué un PID namespace debe combinarse con un mount namespace y un `/proc` propio?
28. Explicá el rol de `lowerdir`, `upperdir`, `workdir`, `merged` y los whiteouts en OverlayFS.
29. ¿Qué cambia al compartir el kernel del host frente a usar una VM?
30. Reconstruí el runtime stack desde `docker run` hasta el kernel y asigná responsabilidad a `containerd` y `runc`.
31. ¿Cómo se complementan capabilities, seccomp y AppArmor/SELinux?
32. Relacioná `create → start → stop → delete` de la demo con el ciclo de vida de Docker.
33. Nombrá tres advertencias operativas que impiden ejecutar la demo a ciegas sobre un host importante.

## Práctica de DDD

34. ¿Qué cinco fronteras propone la resolución 2025 de Pedidos y en qué difiere del ejemplo 2026?
35. Reconstruí el camino feliz y dos fallos del EventStorming de Pedidos.
36. ¿Qué conflictos de lenguaje de Huella señalan posibles bounded contexts?
37. ¿Qué producen E1–E7 en el ejercicio Huella?
38. ¿Qué antipatrones descalifica explícitamente la rúbrica de Huella?

## Entrega Álbum 2026

39. Escribí los invariantes de cantidades para una figurita de un usuario.
40. ¿Por qué una oferta activa no equivale a una copia apartada?
41. Describí la secuencia segura desde aceptar una oferta hasta confirmar o liberar.
42. ¿Qué datos deben ser correctos de inmediato y cuáles pueden converger después?
43. ¿Qué requisitos obligan a tolerar reintentos, duplicados o respuestas perdidas?
44. ¿Qué clases de carrera cubren los escenarios obligatorios?
45. ¿Qué condiciones no funcionales actúan como gate de evaluación?

## Lectura DDD complementaria

46. ¿Cómo diferencia `B01` EventStorming de Event Modeling?
47. ¿Qué dual write intenta evitar Transactional Outbox y cómo se publica luego el evento?
48. ¿Por qué un outbox no elimina por sí solo la necesidad de consumidores idempotentes?

## Imágenes de contenedores

49. ¿Por qué un binario puede existir dentro de una imagen `scratch` y aun así no arrancar?
50. Diferenciá `ENTRYPOINT` y `CMD`, y explicá qué reemplaza `docker run IMAGEN argumentos`.
51. ¿Cómo se invalidan `COPY`/`ADD` y `RUN`, y por qué conviene copiar las dependencias antes que el código?
52. ¿Qué es el build context y qué dos clases de problemas evita `.dockerignore`?
53. ¿Qué queda fuera de una imagen final multi-stage y qué tradeoff introducen `scratch` o distroless?
54. ¿Por qué menos CVEs informadas no demuestra por sí solo que una base sea más segura o esté actualizada?
55. Compará un tag, un tag que el registry impide sobrescribir y un digest como referencias de despliegue.

## Docker Compose

56. Diferenciá `ports`, `expose`, `volumes` y `networks` por el efecto que declara cada uno.
57. ¿Cómo evita Compose hardcodear secretos y dónde aparecen dentro del contenedor?
58. ¿Por qué `depends_on` sin healthcheck no resuelve necesariamente el orden de arranque?
59. Reconstruí los estados de un healthcheck y explicá si un fallo reinicia el contenedor.
60. Diferenciá readiness y liveness; ¿a cuál se aproxima `service_healthy`?
61. ¿Cuándo conviene definir el healthcheck en el Dockerfile y cuándo en Compose?
62. En el ejemplo proxy + backend + MariaDB, asigná el rol de puerto, red, secreto, volumen y dependencia saludable.

## Docker CLI, redes y volúmenes

63. ¿Por qué la CLI no crea contenedores directamente y qué riesgo especial tiene el socket del daemon?
64. Reconstruí los estados principales del ciclo de vida y explicá qué combinan o eliminan `docker run`, `pause` y `rm`.
65. Diferenciá `exec` y `attach`, y describí la parada ordenada que inicia `docker stop`.
66. En una red bridge, separá qué coordina el engine de los objetos y funciones que aporta el kernel.
67. ¿Qué transformación implementa `-p 8080:80` y cómo cambia la exposición al escribir `127.0.0.1:8080:80`?
68. Compará el bridge predeterminado con uno creado por el usuario en DNS y aislamiento.
69. Diferenciá volumen, bind mount y `tmpfs` por responsable, persistencia y caso de uso.
70. ¿Por qué un bind mount puede producir problemas de permisos aunque los nombres de usuario parezcan coincidir?

## Primera entrega DDD del Álbum

71. ¿Cuáles son las cinco secciones obligatorias de `docs/analisis-ddd.md`?
72. ¿Por qué identificar bounded contexts no equivale todavía a diseñar microservicios?
73. ¿Qué debe hacerse cuando una regla de negocio no puede garantizarse dentro de un único agregado?
74. ¿Qué trazabilidad exige la consigna entre `E01` y cada elemento del modelo?
75. ¿Qué tres flujos deben aparecer como mínimo en el encadenamiento de eventos?

<details>
<summary>Respuestas orientativas</summary>

1. Desarrollo, ejecución, despliegue y escalado independientes, coordinados mediante una interfaz estable; los cambios internos no deberían afectar consumidores mientras el contrato se conserve. [T02, p. 4] [T02, p. 11] [T02, p. 42]
2. La librería corre dentro del proceso consumidor y comparte recursos; reemplazarla suele exigir reinicio o recompilación. El servicio es un componente ejecutable y reemplazable por separado. [T02, p. 12]
3. SOA se presenta con ESB, gobierno centralizado y datos compartidos; microservicios con comunicación simple, gobierno descentralizado y datos por servicio. Un monolito concentra la funcionalidad en un proceso y escala replicando el conjunto. [T02, p. 8] [T02, p. 9]
4. Beneficios: fronteras modulares, despliegue independiente y diversidad tecnológica. Costos: distribución, consistencia eventual y complejidad operativa. [T02, p. 18]
5. Frecuencia de release, escalado selectivo, distribución del negocio, cantidad y autonomía de equipos, siempre vinculados con valor de negocio. [T02, p. 29] [T02, p. 30]
6. Separar procesos pero conservar acoplamiento de datos o diseño; estar distribuido no basta para estar desacoplado. [T02, p. 32]
7. Bajo acoplamiento permite cambios locales sin despliegues coordinados; alta cohesión mantiene juntas las tareas de una capacidad. Agregar un campo opcional que consumidores existentes ignoran ilustra evolución desacoplada. [T02, p. 20] [T02, p. 21] [T02, p. 22]
8. Porque la capacidad expresa comportamiento y propósito del contexto; un contrato centrado solo en datos expone detalles y favorece acoplamiento. [T02, p. 25]
9. La estructura del sistema tiende a copiar los canales de comunicación: silos funcionales favorecen capas en silos; equipos multifuncionales favorecen límites por capacidad. [T02, p. 13] [T02, p. 14] [T02, p. 24]
10. Automatiza expectativas proveedor/consumidor en sus pipelines sin un entorno end-to-end; no reemplaza la coordinación del contrato ni del lenguaje. [T02, p. 33]
11. Resources adapta el protocolo; Domain contiene reglas; Service Layer coordina; Repositories accede a persistencia; Gateways/HTTP Client integra servicios; Data Mappers/ORM traduce al datastore. [T02, p. 37] [T02, p. 38] [T02, p. 39] [T02, p. 40]
12. Porque optimiza el esquema relacional, impone un modelo global y distribuye acoplamiento y transacciones sin respetar el lenguaje del negocio. [T03, p. 6] [T03, p. 7]
13. Domain es el área completa de conocimiento; subdomain recorta el problema; bounded context recorta la solución donde un modelo y lenguaje son válidos. [T03, p. 19] [T03, p. 21] [T03, p. 22]
14. Una palabra puede significar cosas distintas por área. En el ejemplo, “Pedido” tiene modelos diferentes en Ventas, Operaciones, Facturación y Logística. [T03, p. 16] [T03, p. 17]
15. Core diferencia competitivamente; Supporting es específico y necesario sin diferenciar; Generic suele estar resuelto por el mercado. [T03, p. 20]
16. Conformist reduce el costo aceptando el modelo upstream; ACL agrega traducción para proteger el modelo propio cuando el lenguaje ajeno no conviene. [T03, p. 25] [T03, p. 26]
17. OHS es el servicio o interfaz estable ofrecido a varios consumidores; Published Language es el contrato público que expresa esa integración. [T03, p. 27]
18. El BC es una frontera de modelo y lenguaje; el microservicio es una frontera de runtime y despliegue. Un monolito modular puede alojar varios BC. [T03, p. 31]
19. Entity se compara por identidad persistente; Value Object por el conjunto de atributos y no tiene identidad propia. [T03, p. 35] [T03, p. 36]
20. Protege invariantes y consistencia transaccional; el exterior accede solo mediante la Aggregate Root. [T03, p. 37]
21. Command expresa una intención imperativa, entra y puede fallar; Domain Event expresa un hecho pasado, ya ocurrió y sale hacia interesados. [T03, p. 38]
22. Actor/persona, command, domain event, aggregate, policy, sistema externo, read model/vista y hotspot/pregunta abierta. [T03, p. 42]
23. `PedidoCreado` → `InventarioReservado` → `PagoConfirmado` → `EnvioProgramado` → `PedidoDespachado` → `EntregaConfirmada`. [T03, p. 50]
24. Una respuesta mínima: Facturación publica `PagoRechazado`; una policy en Operaciones envía `LiberarInventario`; el aggregate de reserva produce `InventarioLiberado`. Los contextos directamente involucrados son Facturación y Operaciones. [T03, p. 47] [T03, p. 50]
25. Es un proceso del host con vistas aisladas, límites y rootfs propio; la imagen aporta userspace y el proceso usa el kernel del host. Una VM, en cambio, ejecuta un kernel invitado sobre hardware virtualizado. [T04, p. 12] [T04, p. 13] [T04, p. 24] [T04, p. 25]
26. Namespaces responden “¿qué puede ver el proceso?”; cgroups, “¿cuánto puede consumir?”. Los primeros aíslan PID, red, montajes, UTS, IPC y usuarios; los segundos limitan CPU, memoria, PIDs e I/O. [T04, p. 14] [T04, p. 18]
27. El PID namespace cambia la vista y numeración de procesos, pero `/proc` sigue reflejando el montaje disponible. Aislar montajes y remontar `/proc` hace que herramientas como `ps` observen la vista correcta. [T05, p. 21] [T05, p. 22] [T05, p. 23]
28. `lowerdir` contiene capas inmutables; `upperdir`, cambios; `workdir`, trabajo interno del kernel; `merged`, la vista resultante. Un whiteout en la capa superior oculta un archivo inferior sin borrarlo de `lowerdir`. [T04, p. 19] [T04, p. 21] [T05, p. 50]
29. Compartir kernel reduce boot y overhead, pero una vulnerabilidad del kernel puede ampliar el impacto de un escape. Una VM agrega aislamiento mediante su propio kernel a cambio de más recursos. [T04, p. 24] [T04, p. 25] [T04, p. 37]
30. `docker run` → Engine → `containerd` → `runc` → kernel. `containerd` descarga/desempaqueta capas y prepara bundle/configuración; `runc` materializa namespaces, cgroups, rootfs y controles de seguridad y ejecuta el proceso. [T04, p. 29] [T04, p. 30] [T04, p. 31]
31. Capabilities reducen los privilegios de kernel; seccomp limita syscalls; AppArmor/SELinux agrega control de acceso obligatorio. Son capas de defensa en profundidad. [T04, p. 32] [T04, p. 33] [T05, p. 48]
32. `create` prepara rootfs y cgroup; `start` entra con aislamiento; `stop` mata el PID visible desde el host; `delete` remueve cgroup y rootfs. Se corresponden con `docker create`, `start/run`, `stop` y `rm`. [T05, p. 54]
33. Ejemplos: el setup modifica AppArmor; desmontar `/proc` sin mount namespace afecta al host; el segundo `veth` debe configurarse del lado correcto; `io.max` no admite asumir un dispositivo fijo; y el cleanup borra rutas de `/tmp`. [T05, p. 8] [T05, p. 9] [T05, p. 21] [T05, p. 27] [T05, p. 45]
34. La resolución 2025 separa gestores de Pedido, Inventario, Cobros, Envíos y Créditos. `T03` propone cuatro contextos candidatos y ubica Crédito dentro de Facturación; son modelos situados, no una taxonomía fija. [P01, p. 7] [T03, p. 39]
35. Camino feliz: pedido creado, inventario disponible y cobro realizado, envío creado, despachado y entregado. Fallos posibles: inventario insuficiente; cobro rechazado con liberación; envío fallido con devolución y crédito. [P01, p. 10] [P01, p. 11]
36. Entre otros: “alta”, paciente/cliente/titular, turno/consulta/práctica y vacuna cambian de significado entre Recepción, Clínica, Depósito y Administración. El ejercicio exige acotar y traducir, no unificar. [P02, p. 2] [P02, p. 3] [P02, p. 4] [P02, p. 5]
37. E1 lenguaje conflictivo; E2 subdominios; E3 bounded contexts; E4 context map; E5 táctica del Core; E6 commands y eventos por contexto; E7 EventStorming mínimo. [P02, p. 6] [P02, p. 7] [P02, p. 8]
38. Cortar por entidad, compartir base, usar un `Paciente` global, adoptar el vocabulario de VetSoft/pasarela dentro del Core, emitir eventos CRUD y publicar todos los eventos internos. [P02, p. 10]
39. `poseídas = disponibles + apartadas`; `repetidas = max(poseídas - 1, 0)`; `ofrecibles = max(disponibles - 1, 0)`; y las tres cantidades base no pueden ser negativas. [E01, p. 2]
40. La oferta es una intención y no modifica cantidades. La misma copia ofrecible puede respaldar varias ofertas; solo una aceptación en curso intenta moverla de disponible a apartada. [E01, p. 1] [E01, p. 2] [E01, p. 3]
41. Revalidar elegibilidad, intentar apartar todas las copias, volver a validar oferta/copias/reservas y confirmar una sola vez. Si algo falla, no transferir y liberar toda reserva parcial. [E01, p. 3] [E01, p. 4]
42. Sobres, cantidades de colección y resultados de ofertas/intercambios son inmediatos. Porcentaje, retos, rankings, notificaciones y actividad pueden converger luego. [E01, p. 5]
43. Entre otros: reapertura de sobre devuelve el mismo contenido; una operación puede consultarse tras timeout; el progreso ignora eventos duplicados; y el sistema no reejecuta intercambios completados. [E01, p. 3] [E01, p. 4] [E01, p. 5]
44. Compiten ofertas respaldadas por una copia, dos aceptaciones de una oferta, aceptación contra cancelación/vencimiento y dos canjes por el último uso de un código. También se prueban fallos después de reservas parciales o completas. [E01, p. 6]
45. Pipeline con compilación y tests; despliegue automático a staging; integración como gate; producción en AKS solo tras pasar el gate; entrega únicamente por ese pipeline; OAuth/OIDC real con Google. [E01, p. 6]
46. La fuente externa usa EventStorming para exploración colaborativa del dominio y Event Modeling para un storyboard más estructurado de UI, commands, eventos y read models. [B01, p. 7]
47. Evita que el cambio de dominio se confirme y la publicación falle por ser dos escrituras independientes. Estado y evento pendiente se guardan en una transacción local; un relay publica luego mediante polling o CDC. [B01, p. 11] [B01, p. 12]
48. El relay puede publicar más de una vez ante reintentos o fallos; por eso el consumidor todavía debe tolerar duplicados, por ejemplo registrando `message_id` procesados. [B01, p. 12] [B01, p. 13]
49. Porque un binario enlazado dinámicamente necesita el intérprete ELF y bibliotecas como `libc`; si `scratch` solo contiene el ejecutable, Docker puede informar `no such file or directory` aunque el archivo esté presente. [T06, p. 10] [T06, p. 11]
50. `ENTRYPOINT` fija el ejecutable y `CMD` aporta comando o argumentos predeterminados. Los argumentos escritos después de la imagen reemplazan `CMD` y se aplican sobre `ENTRYPOINT`; con solo `CMD`, pueden reemplazar el comando entero. [T06, p. 14] [T06, p. 15]
51. `COPY`/`ADD` usan checksum del contenido y `RUN` compara exactamente el string del comando. Un miss invalida esa capa y las posteriores; separar manifiestos de dependencias del código permite reutilizar la instalación mientras aquellos no cambien. [T06, p. 20] [T06, p. 21] [T06, p. 22]
52. Es el conjunto enviado por el cliente al engine y la única entrada de archivos del build aislado. `.dockerignore` reduce tamaño e invalidaciones de cache y evita transferir o incorporar secretos y basura del repositorio. [T06, p. 23] [T06, p. 24]
53. Quedan fuera compiladores, dependencias de desarrollo y todo lo no copiado desde la etapa builder. La imagen se achica y pierde superficie de ataque, pero `scratch` o distroless también quitan shell y utilidades de diagnóstico; además el binario debe incluir sus dependencias necesarias. [T06, p. 27] [T06, p. 28] [T06, p. 29]
54. Porque el conteo depende del scanner, su base y la fecha; incluye hallazgos no necesariamente explotables desde el contenedor y hay que distinguir los que tienen fix. Una base distroless desactualizada puede quedar peor que otra mayor pero actualizada. [T06, p. 30] [T06, p. 31]
55. Un tag puede moverse; una política de registry puede impedir que se sobrescriba; el digest identifica criptográficamente el contenido del manifiesto y no depende de que se respete una convención de nombres. [T06, p. 35] [T06, p. 36]
56. `ports` publica host:contenedor; `expose` declara acceso interno sin publicar al host; `volumes` monta almacenamiento persistente; `networks` define conectividad y resolución por nombre entre servicios. [T07, p. 8] [T07, p. 9]
57. Declara un secreto externo o basado en un archivo excluido de Git y lo referencia desde el servicio; Compose lo monta como `/run/secrets/<nombre>`. [T07, p. 11]
58. Porque contenedor iniciado no equivale a servicio listo. Sin prueba de salud el consumidor puede conectarse durante el warm-up; `condition: service_healthy` lo mantiene esperando. [T07, p. 15] [T07, p. 16]
59. Empieza `starting`; código 0 produce `healthy` y otro código, `unhealthy`, con transiciones posteriores según los resultados. El healthcheck informa estado y no reinicia por sí solo. [T07, p. 17]
60. Readiness indica capacidad de aceptar trabajo; liveness, si el proceso sigue vivo y la plataforma debería reiniciarlo. `service_healthy` se aproxima a readiness y solo coordina a los dependientes. [T07, p. 18]
61. En el Dockerfile cuando la prueba es intrínseca y portable con la imagen; en Compose cuando depende del entorno donde se despliega. [T07, p. 19]
62. El proxy publica HTTP al cliente; los servicios se conectan internamente; la contraseña se monta como secreto en backend y base; el volumen conserva datos; y el backend espera que MariaDB esté `service_healthy`. [T07, p. 21] [T07, p. 22] [T07, p. 23]
63. La CLI es un cliente HTTP que solicita operaciones al daemon, que delega en `containerd`, `runc` y el kernel. El socket permite usar toda esa API, incluidos montajes del host; por eso su acceso equivale en la práctica a root y no debe exponerse a contenedores no confiables. [T08, p. 4] [T08, p. 5]
64. El flujo principal es `created → running → exited → rm`, con `paused` como suspensión reversible. `run` combina create y start; `pause` congela mediante el cgroup freezer; `rm` elimina el objeto y su capa escribible. [T08, p. 7]
65. `exec` crea un proceso nuevo dentro de los namespaces del contenedor; `attach` conecta la terminal al PID 1 y puede enviarle señales. `stop` envía primero `SIGTERM` y, si vence el período de gracia, `SIGKILL`; `--init` ayuda a reenviar señales y recolectar zombis. [T08, p. 13] [T08, p. 14]
66. El engine elige subred mediante IPAM, crea y conecta endpoints y escribe configuración. El kernel aporta net namespaces, pares `veth`, bridge L2, forwarding y reglas de NAT/filtro; el egress se enmascara con la IP del host. [T08, p. 20] [T08, p. 21]
67. Instala DNAT desde el puerto 8080 del host al 80 del contenedor. Sin dirección explícita puede quedar publicado en interfaces externas; anteponer `127.0.0.1` lo limita a loopback. [T08, p. 22]
68. El bridge predeterminado deja una red plana y no ofrece resolución automática por nombre; una red propia aísla el grupo y ofrece DNS embebido por nombre y alias mediante el resolver `127.0.0.11`. [T08, p. 24] [T08, p. 25]
69. Docker gestiona el volumen fuera de OverlayFS y éste sobrevive al contenedor; el usuario gestiona el path de un bind, que persiste como parte del host; el kernel mantiene `tmpfs` en RAM y lo destruye con el contenedor. Se usan, respectivamente, para datos importantes, paths del host y contenido que no debe persistir. [T08, p. 35] [T08, p. 43]
70. El kernel compara UID/GID numéricos, no nombres. Un usuario llamado igual dentro y fuera puede tener otro número; un contenedor root puede además dejar archivos de root en el host. [T08, p. 42]
71. Lenguaje ubicuo, mapeo estratégico, diseño táctico, eventos de dominio, y supuestos y preguntas abiertas. [E02]
72. Porque un bounded context delimita un modelo y un lenguaje; la partición en unidades de runtime y despliegue pertenece a una etapa posterior y está fuera del alcance de esta entrega. [E02] [T03, p. 31]
73. Señalarlo como hallazgo y explicar qué coordinación o mecanismo de dominio la sostiene, en lugar de adjudicarla artificialmente a un agregado incapaz de garantizarla. [E02]
74. Cada término, contexto, agregado y evento debe derivarse de la consigna funcional o justificarse; cada requisito de `E01` debe quedar representado para no convertirse en una omisión. [E02]
75. Apertura de sobre, intercambio completado e intercambio no concretado. [E02]

</details>

## Casos para practicar sin respuesta única

- Una fintech entrega semanalmente con seis equipos; un sistema interno cambia dos veces al año con un equipo de cuatro; un catálogo recibe cien veces más tráfico que checkout; una startup de tres personas construye su MVP. Elegí arquitectura y justificá por equipo, releases y escalado. [T02, p. 30]
- Elegí `PagoRechazado` o `EntregaFallida`, escribí entre tres y cinco eventos y uno o dos comandos, asigná publicador y reactor, y marcá al menos un hotspot. [T03, p. 52]
- Dibujá el proceso y las vistas del kernel para un contenedor que usa PID, mount, user y net namespaces, un cgroup de memoria y OverlayFS. Señalá qué sigue compartiendo con el host. [T04, p. 14] [T04, p. 18] [T04, p. 19]
- Para Huella, elegí cuatro términos conflictivos y proponé cómo se traducen en el borde, sin resolver todavía los bounded contexts. [P02, p. 6]
- Construí una tabla de estados para oferta, operación de intercambio y copia que cubra aceptación, timeout, cancelación y vencimiento sin violar los invariantes. [E01, p. 3] [E01, p. 4] [E01, p. 6]
- Diagnosticá este caso: dos contenedores comparten una red propia, el nombre resuelve, pero el cliente recibe `connection refused` y desde el host tampoco funciona el puerto publicado. Ordená las comprobaciones de red, escucha y DNAT. [T08, p. 22] [T08, p. 32]
