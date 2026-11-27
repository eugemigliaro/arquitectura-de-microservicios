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

## Comunicación síncrona

76. ¿Qué acoplamiento introduce una llamada síncrona y qué fallos propios de la red deben asumirse?
77. Compará REST, gRPC y GraphQL por contrato, fortaleza y borde recomendado.
78. ¿Por qué un deadline es obligatorio en gRPC y qué riesgo evita?
79. ¿Qué problema resuelve ConnectRPC sin abandonar el contrato `.proto`?
80. Diferenciá `λ` y `μ`; ¿qué ocurre si `λ > μ` de manera sostenida?
81. ¿Qué protege un bulkhead y por qué un retry sin esa protección puede empeorar la sobrecarga?
82. ¿Por qué la cola no agrega capacidad y qué intercambia por evitar pérdida?
83. ¿Qué limitaciones impiden generalizar la demo de cola a garantías reales de entrega?

## Testing

84. ¿Qué dimensiones cambian al subir en la pirámide de testing y qué áreas quedan fuera de ella?
85. ¿Por qué `component` y `functional` no representan necesariamente dos escalones?
86. ¿Cómo conviven la nota oral sobre `integration test`/`functional testing` y la clasificación escrita de `T10`?
87. Compará unit, integration, component, contract y end-to-end por alcance.
88. Diferenciá stub, mock y fake.
89. ¿Cómo valida un contract test al consumidor y al proveedor sin levantar el sistema completo?
90. Reconstruí Arrange–Act–Assert–Clean-up y nombrá una trampa de cada fase.
91. ¿Cómo se relaciona Given–When–Then con las cuatro fases y por qué es business-facing?
92. ¿Qué puede demostrar la cobertura y qué no garantiza ni siquiera al 100 %?
93. Diferenciá linter, SAST, SCA y DAST por objeto y momento de análisis.
94. ¿Qué riesgo aparece si un modelo genera a la vez la implementación y el test?

## Sistemas distribuidos

95. ¿Qué ventajas de un sistema centralizado se pierden al distribuir y qué razones justifican hacerlo igual?
96. Compará los modelos de falla de red, de nodos y de tiempo. ¿Qué modelo de enlace mitiga TLS?
97. ¿Por qué no se puede distinguir un nodo caído de uno lento y qué consecuencia tiene para un timeout en RPC?
98. En el ejemplo de transferencia y auditoría, ¿qué ejecución viola la invariante `X + Y = 20` y por qué?
99. Compará control de concurrencia pesimista y optimista por mecanismo, costo y caso de uso.
100. ¿Qué pasa en 2PC si un participante cae después de votar SÍ, o si el coordinador cae después de enviar algunos commits?
101. ¿Cuándo recomienda la cátedra usar 2PC?
102. Contrastá orquestación y coreografía, y describí la compensación del ejemplo de venta cuando falla el pago.

## Estado distribuido

103. ¿Por qué un mutex local no alcanza cuando un servicio corre en varias instancias?
104. Clasificá cupos, pedidos, un pool de conexiones y un ranking según el tipo de estado y su propiedad esperada.
105. ¿Qué protege el `owner-id` de un lock y qué protege un fencing token?
106. ¿Qué ventana de falla aparece si el registro de idempotencia se hace antes del efecto y cuál si se hace después?
107. ¿Cómo evita el inbox transaccional aplicar dos veces un mensaje cuya confirmación se perdió?
108. ¿Qué garantiza y qué no garantiza `SADD claims job:42`?
109. Diferenciá at-most-once, at-least-once, efecto idempotente y exactly-once. ¿Qué semántica tiene Valkey Pub/Sub?
110. ¿Qué garantizan los consumer groups de un stream y cómo se recupera un mensaje pendiente de un consumidor caído?
111. Si un stream conserva el orden de sus IDs, ¿por qué puede alterarse el orden en que se aplica el estado?

## DevOps y CI/CD

112. Nombrá las etapas del ciclo DevOps y ubicá release y delivery.
113. Diferenciá Continuous Integration, Continuous Delivery y Continuous Deployment.
114. ¿Cómo se integra un pull request con el pipeline de CI/CD?
115. Definí stage, job y runner, y explicá cómo intercambian archivos los pasos en GitLab.
116. Enumerá las etapas típicas de un pipeline y qué se preserva de la etapa de build.
117. Compará basic, rolling, blue/green y canary por riesgo y requisitos.
118. ¿Qué es shift-left y qué costo reduce?

## Comunicación asincrónica y Kafka

119. Compará síncrono y asíncrono en acoplamiento temporal, latencia percibida, falla en cascada, complejidad operativa y trazabilidad. ¿Cuándo no conviene async?
120. Diferenciá cola y pub/sub, y explicá cómo se obtiene cada modelo con consumer groups de Kafka.
121. Compará event notification, event-carried state transfer y comando. ¿Cuándo aplicarías claim check?
122. ¿Qué es CQRS light y en qué se diferencia de Event Sourcing?
123. Describí la compensación coreografiada cuando falla el pago y explicá qué agrega un motor de orquestación durable.
124. ¿Por qué una Saga necesita timeouts explícitos?
125. ¿Qué orden garantiza Kafka y qué rol cumple la key? Nombrá tres formas de elegirla mal.
126. ¿Por qué en Kafka leer no es sacar? Explicá offset, replay y consumer lag.
127. Un topic tiene 4 particiones y su consumer group, 5 consumers. ¿Qué ocurre? ¿Y si en un grupo de 2 consumers cae uno?
128. ¿Qué es el dual write y cómo lo resuelve el Transactional Outbox? ¿Qué falla produce publicaciones duplicadas?
129. Compará poller y CDC para vaciar el outbox.
130. ¿Qué evita el Inbox y por qué el exactly-once de Kafka no reemplaza la idempotencia del handler?
131. ¿Por qué reintentar para siempre es especialmente dañino en Kafka y qué regla propone la cátedra?
132. ¿Para qué sirven `correlationId` y `causationId`, y cómo se evoluciona el esquema de un evento sin romper consumidores?

## Consigna general del Álbum

133. Según la consigna general, ¿qué debe implementarse y demostrarse, y qué basta diseñar?
134. Enunciá el invariante de oro. ¿Qué agrega a las fórmulas de cantidades por usuario y figurita?
135. ¿En qué momento se aparta la copia ofrecida según la consigna general y según la consigna funcional v1.1? ¿Qué cambia al cancelar o vencer una oferta?
136. ¿Qué evidencia de observabilidad exige la demostración y qué alcanza para producirla?
137. ¿Qué condiciones debe cumplir el pipeline para que la entrega sea evaluada?
138. Elegí cinco preguntas del diseño para un millón de usuarios y, para cada una, nombrá la unidad de la materia que aporta herramientas.
139. ¿Qué escenarios difieren entre la consigna general y la consigna funcional v1.1?

## Segunda entrega: arquitectura del Álbum

140. ¿Qué decisiones vienen impuestas y por qué "porque corre en contenedores" no justifica ninguna otra?
141. ¿Qué cuatro reglas de trazabilidad ligan la arquitectura con el análisis DDD de la primera entrega?
142. Diferenciá un atributo de calidad expresado como adjetivo de uno expresado como escenario medible. Escribí uno para la apertura de sobres.
143. ¿Qué debe especificar cada contrato de evento y qué campos lleva el envelope común?
144. ¿Qué diagramas de secuencia pide la sección 5 y qué debe incluir la máquina de estados del intercambio?
145. ¿Por qué un healthcheck que siempre responde OK es un error de diseño?
146. ¿Qué garantiza que lo probado en staging sea exactamente lo que llega a producción?
147. ¿Qué ADRs son obligatorios y qué convierte a un ADR en una decisión y no en una afirmación?

## Del monolito a microservicios

148. ¿Qué enseñan juntas las historias de Amazon, Segment y Prime Video sobre la decisión de migrar?
149. Distinguí razones que alcanzan y razones que no alcanzan para migrar. ¿Por qué "para escalar" no basta por sí sola?
150. ¿Qué es un monolito modular y qué beneficios da sin pagar los costos de la distribución?
151. ¿Qué prerrequisitos deben existir antes de extraer el primer servicio?
152. Nombrá los cuatro tipos de seam y qué revela cada uno que los demás no muestran.
153. ¿Qué criterios hacen de Notificaciones un buen primer servicio y de Pagos uno malo?
154. ¿Qué síntomas delatan nano-servicios y por qué conviene empezar más grande?
155. ¿Cuáles son las tres puertas antes de extraer y qué riesgo cubre el inventario de efectos laterales?
156. Elegí el patrón para cada situación: ruta HTTP externa, uso interno del monolito, error caro, comportamiento nuevo sin tocar el monolito, pasar tráfico de a poco y convivencia con el modelo viejo.
157. ¿Cuándo un Strangler Fig se convierte en un segundo monolito y cuándo un ACL en un monolito distribuido?
158. ¿Por qué el shadowing de una escritura es peligroso y cómo se evita?
159. ¿Por qué la base compartida es "el acoplamiento más peligroso"? ¿Cuándo conviene separar datos primero?
160. Describí Split Table y Move FK to Code. ¿Qué se pierde y qué hay que decidir en cada caso?
161. Reconstruí las tres fases de sincronización con CDC, quién escribe y cómo se vuelve atrás en cada una.
162. ¿Qué significa que una extracción está terminada y por qué un feature flag no alcanza para revertirla?
163. ¿Qué tres preguntas hay que hacerle al negocio antes de partir una transacción, y qué indica que una invariante debe ser inmediata?
164. ¿Qué opciones hay para el reporting que antes hacía joins sobre una sola base, y cuál está prohibida?
165. ¿Qué es el Inverse Conway Maneuver y qué roles define Team Topologies?
166. ¿Qué métricas dicen si una migración funciona?
167. Enumerá los anti-patterns de migración y explicá por qué "extraer sin sacar" duplica el costo.

## Kubernetes — Parte 1

168. ¿Qué hacen API server, scheduler, controller manager, etcd y kubelet? ¿Dónde intervienen CRI y CNI?
169. ¿Qué comparten los contenedores de un Pod y qué conservan separado? ¿Un Pod migra entre nodos?
170. Diferenciá requests y limits. ¿Qué ocurre al limitar CPU o memoria?
171. Compará init común, sidecar nativo y contenedor efímero. ¿Resolver DNS prueba readiness?
172. ¿Por qué Running no implica Ready? Diferenciá fases del Pod, estados de contenedor y probes.
173. ¿Cuándo elegir Deployment, StatefulSet, DaemonSet, Job o CronJob?
174. ¿Qué cambia al modificar replicas y qué al modificar el template de un Deployment? ¿Y un ConfigMap?
175. Explicá la cadena Deployment → ReplicaSet → Pod y cómo verificarla con ownerReferences.
176. ¿Por qué la aparición posterior del ServiceAccount default no prueba inconsistencia de etcd?
177. Separá autenticación, autorización y admisión. ¿Qué sucede con get, list y watch?
178. ¿Qué representa apiVersion y qué aislamiento aporta por sí solo un Namespace? Compará el alcance de PVC y PV.

## Kubernetes — Parte 2

179. Distinguí los tres rangos de IP del clúster y las cuatro necesidades de comunicación. ¿Quién resuelve cada una?
180. ¿Qué hace el plugin CNI y qué no hace kube-proxy? Compará ruteo nativo y overlay en mantenibilidad, extensibilidad, seguridad y performance.
181. ¿Cómo sabe un Service a qué Pods reenviar? ¿Qué es un EndpointSlice y por qué se dejó de usar la API Endpoints?
182. ¿Por qué un Pod en `default` puede llegar a `my-nginx-svc.default` sin escribir el nombre completo?
183. Compará ClusterIP y headless. ¿Por qué un Service no garantiza round-robin por solicitud HTTP?
184. Diferenciá ClusterIP, NodePort, LoadBalancer y ExternalName. ¿LoadBalancer exige nube?
185. ¿Qué falta para que un Ingress o un Gateway hagan llegar tráfico? ¿Qué separa Gateway API entre GatewayClass, Gateway y HTTPRoute, y por qué se la recomienda?
186. ¿Cuándo un Pod está aislado por NetworkPolicy? ¿Qué hace falta si ambos extremos están aislados? ¿Qué no hace una NetworkPolicy?
187. ¿Qué sobrevive en la capa escribible, en un `emptyDir` y en un PV? ¿Persistente significa disponible desde cualquier nodo o después de destruir el clúster?
188. Compará aprovisionamiento estático y dinámico. ¿Qué rol cumplen StorageClass, provisioner, CSI y `WaitForFirstConsumer`?
189. ¿Qué garantiza RWO y qué RWOP? ¿Qué pasa con los datos al borrar un PVC con `Retain` y con `Delete`?
190. ¿Qué agrega un Operator a la reconciliación nativa? Explicá el ejemplo de PostgreSQL al pasar de 3 a 5 instancias.
191. ¿Cómo se entera una aplicación de un cambio en su ConfigMap según cómo lo consume?
192. ¿Por qué un Secret no es confidencial por sí solo? Enumerá los riesgos de acceso, almacenamiento y creación, y las buenas prácticas.

## Laboratorios de la cátedra

193. En la guía de contenedores a mano, ¿por qué un PID namespace necesita también un mount namespace? ¿Por qué `curl localhost` no llega a un puerto publicado solo con DNAT en `PREROUTING`?
194. Recorré las mejoras del Dockerfile del laboratorio de Compose y explicá qué problema resuelve cada una: `WORKDIR`, `.dockerignore`, versiones pinneadas, orden de `COPY`, usuario no root y `HEALTHCHECK`.
195. ¿Por qué el laboratorio de Compose pone la configuración en `env_file` y no en `ENV`? ¿Cómo logra que cada Redis solo sea visible para su servicio?
196. ¿Qué etapas del pipeline del laboratorio de CI/CD son simuladas y qué le falta para cumplir lo que pide la segunda entrega?
197. En el monorepo del laboratorio de Valkey, ¿para qué sirven el job `compliance` y el child pipeline?
198. En la demo de capacidad, ¿por qué se usa carga closed-loop y por qué `CONCURRENCY` no cambia la capacidad? ¿Qué muestran los 1,83 intentos por completado?
199. ¿Por qué la cola "destruye la contrapresión" y cuál es el criterio para elegir entre sync con bulkhead y retry o una cola?
200. Explicá el Redlock del laboratorio: quórum, token, validez y liberación. ¿Qué protección de la teoría no implementa?
201. ¿Qué hace el laboratorio de cache-aside ante un producto inexistente y ante Valkey caído? ¿Por qué con concurrencia `db_reads` puede ser mayor que 1?
202. Compará los laboratorios de coreografía y orquestación: qué protocolo implementa cada uno, qué topics usan y qué pasa cuando falla el pago o un participante vota NO.
203. En Minikube, ¿por qué el cluster no ve las imágenes de tu Docker local? ¿Por qué el Pod de depuración no vuelve al borrarlo?
204. En AKS, ¿qué componentes del control plane ya no ves en `kube-system` y qué agentes aparecen? ¿Qué crea Azure ante un Service LoadBalancer, un PVC y un Gateway?
205. ¿Por qué el disco de Postgres necesita `PGDATA` en una subcarpeta? ¿Por qué Postgres tarda más que `web` en volver tras un `drain`?
206. ¿Qué archivos y credenciales del cluster del TP no pueden ir al repositorio y cómo se comparte el cluster con el grupo? ¿Por qué hace falta un seed de datos?

## Programar microservicios para Kubernetes

207. ¿Qué responsabilidades quedan en la aplicación y cuáles en Kubernetes? ¿Qué significa "no reimplementar el control plane dentro del servicio"?
208. Recorré las etapas de la terminación gradual. ¿Qué pasa si el drenado dura más que `terminationGracePeriodSeconds`? ¿Qué hace distinto un worker?
209. ¿Por qué la liveness no debe depender de PostgreSQL? ¿Qué debería representar la readiness?
210. En un rolling update con `maxUnavailable: 0` y `maxSurge: 1`, ¿por qué el Pod viejo tiene que seguir atendiendo después de recibir `SIGTERM`?
211. Un worker cobra un pago, la base confirma y el Pod muere antes del ACK. ¿Qué pasa después y dónde se resuelve?
212. ¿Qué recurso de Kubernetes corresponde a una API HTTP, a un consumidor continuo, a una tarea finita y a una programada? ¿Qué contrato le toca al código en cada caso?
213. Encontrá los cuatro problemas del ejercicio de la API de pagos y proponé una corrección para cada uno.

## Helm

214. ¿Qué diferencia hay entre chart, release y repositorio? ¿Qué distinguen `CHART VERSION` y `APP VERSION`?
215. ¿Qué recursos crea el `helm install` del chart de PostgreSQL de Bitnami y qué patrón de Kubernetes reconocés?
216. ¿Por qué en la tercera entrega no se puede usar `helm upgrade` ni un rollback con `helm` contra el cluster? ¿Qué rol puede cumplir Helm?

## GitOps y entrega continua

217. ¿Cuáles son los cuatro principios de OpenGitOps y qué cambia al pasar de "ejecutar pasos" a "declarar un estado"?
218. ¿Por qué un commit en Git no garantiza reproducibilidad si se usan tags mutables? ¿Qué referencia conviene en producción?
219. Compará push y pull por seguridad, riesgos y contextos en los que tiene sentido cada uno.
220. ¿Qué es drift? Clasificá y decidí qué hacer con un `kubectl edit` en producción, el HPA cambiando réplicas, un sidecar inyectado por el mesh y un Secret rotado por External Secrets.
221. ¿Qué no revierte un rollback por revert del estado deseado y cómo se mitiga?
222. Describí los componentes de ArgoCD y el riesgo de cada uno. ¿Qué se pierde si ArgoCD cae?
223. ¿Qué es "CI con poder total y GitOps decorativo" y por qué es un antipatrón?
224. Compará ArgoCD y Flux por experiencia, arquitectura, multi-clúster y riesgo.

## Resiliencia y escalabilidad

225. Distinguí resiliencia, escalabilidad y disponibilidad. ¿Por qué un 200 tardío puede ser peor que un 503 rápido?
226. ¿Por qué reintentar en cada capa genera una retry storm? ¿Cuántas llamadas llegan a Pagos si tres capas reintentan tres veces? ¿Qué la mitiga?
227. Con un SLO de 99,9 % en 30 días, ¿cuál es el error budget? ¿Qué disponibilidad dan cinco dependencias en serie al 99,9 %? ¿Y dos réplicas independientes?
228. ¿Qué diferencia hay entre timeouts locales y un deadline propagado? ¿Qué consecuencia de negocio muestra la demo?
229. Describí los estados del circuit breaker. ¿Por qué su apertura es un síntoma y no el problema?
230. ¿Cuándo se responde 429 y cuándo 503 con `Retry-After`? ¿Por qué el autoscaling no defiende de un ataque L7?
231. ¿Qué da Kubernetes "gratis" y qué no? ¿Para qué sirven el PDB, `topologySpreadConstraints` y `podAntiAffinity`?
232. En un Job, ¿qué diferencia hay entre `backoffLimit` y `activeDeadlineSeconds`, y entre `restartPolicy: OnFailure` y `Never`? ¿Para qué sirve `podFailurePolicy`?
233. Un HPA tiene 4 réplicas al 90 % de CPU con objetivo de 60 %. ¿Cuántas réplicas pide? ¿Qué requisito tiene y por qué la CPU puede ser una mala métrica?
234. ¿Qué agrega KEDA al HPA? ¿Por qué no combinar HPA y VPA sobre la misma métrica? ¿Por qué más consumidores que particiones no ayuda?
235. ¿Cómo puede escalar un servicio romper la base de datos y qué medidas lo previenen?
236. Ordená backup y restore, pilot light, warm standby y active-active por RTO, RPO y costo. ¿Por qué no se estira un clúster entre regiones?

## Tercera entrega: CI/CD y AKS

237. ¿Qué decisiones vienen impuestas en la tercera entrega y sobre qué partes de la segunda prevalecen?
238. ¿Qué credenciales tiene el pipeline y cuáles no? ¿Cómo llegan los Secrets al cluster?
239. Recorré las cinco etapas de promoción con su disparador y su gate. ¿Qué tiene que pasar con `prod` si fallan los tests de integración en `staging`?
240. ¿Por qué la imagen de release no puede contener código de testing? ¿Cómo se miden entonces las dos coberturas?
241. ¿Qué condiciones tiene que cumplir un test de integración para contar en el mínimo de cinco? ¿Por qué no alcanza uno que ejercite un solo servicio real?
242. ¿Cómo se prueba IT-24 (falla durante un intercambio) sin dar al pipeline credenciales del cluster ni agregar endpoints de test a la imagen?
243. ¿Cómo se hace y se demuestra un rollback en esta entrega? ¿Qué no cuenta?
244. ¿Qué hace que la tercera entrega no se evalúe y qué defectos solo bajan la nota?

## Devolución de la segunda entrega

245. ¿Por qué la devolución de la segunda entrega es insumo de la tercera? ¿Dónde se registran los cambios de contrato que surjan de ella?
246. "Un healthcheck del que nadie depende para actuar es información, no protección." ¿Por qué el healthcheck de Compose del TP no protegía ante un proceso colgado? ¿Cómo lo resuelven la liveness y la readiness en Kubernetes sin que la liveness dependa de PostgreSQL?
247. La aceptación de una oferta hace hasta 4 llamadas gRPC en secuencia, con un deadline de 2 s y reintentos en cada una. ¿Por qué un p99 de 1.500 ms no es coherente? ¿Cómo se arma un presupuesto de latencia con deadline propagado y retry budget?
248. ¿Qué problema tiene devolver `201` en una aceptación que terminó `NO_CONCRETADA`? ¿Qué alternativas propone la cátedra y qué impacto tiene cambiarlo en la tercera entrega?
249. ¿Qué riesgos tiene un único contenedor PostgreSQL con cinco bases, aunque cada servicio tenga base y usuario propios? ¿Qué propuso la cátedra y qué exige la tercera entrega sobre los datos por ambiente?
250. ¿Cómo puede reformularse la política de hotfix durante los partidos en la tercera entrega sin saltear el gate de `staging`?

## Devolución de la tercera entrega

294. La aprobación a `prod` del TP era un job `when: manual` sobre `main` protegida. ¿Por qué la cátedra lo consideró insuficiente si un Developer no podía dispararlo? ¿Qué dos alternativas propuso?
295. ¿Por qué una liveness que devuelve siempre OK es un error? ¿Qué podría verificar un servicio Node.js con consumidores de Kafka y un relay de outbox sin depender de PostgreSQL ni del broker?
296. Los tres ambientes compartían el cliente OAuth de Google. ¿Por qué compartir el client ID sería solo una observación, pero compartir el client secret no?
297. Con ArgoCD en sincronización automática en `prod`, ¿por qué los cambios en los manifiestos comunes eluden el gate de `staging`? ¿Qué dos formas de cerrarlo sugirió la cátedra?
298. ¿Qué credenciales con vencimiento tenía el despliegue del TP y qué se rompe si vencen antes de la defensa?
299. Para la entrega final, ¿qué significa que "un IT de escenario que no está implementado o no pasa equivale a un escenario sin cubrir"? ¿Qué tests necesitan un mecanismo especial para controlar el escenario sin credenciales del cluster?

## Feature flagging

251. ¿Qué diferencia hay entre desplegar, activar y exponer? ¿Qué revierte y qué no revierte un rollback de flag?
252. Clasificá las flags de release, experimento, operacional y entitlement por dinamismo, vida y retiro. ¿Por qué un entitlement no alcanza como autorización?
253. ¿Qué hacen el control plane y el data plane de una plataforma de flags? ¿Por qué no conviene consultar la base central en cada evaluación?
254. ¿Cómo se logra una asignación estable y por qué "10 % de requests no es 10 % de tiendas"? ¿Qué pasa con la cohorte al subir de 10 % a 25 %?
255. ¿Qué controla y qué no garantiza cada capa (browser, red, backend, datos)? ¿Por qué el resultado del browser es "una preferencia, no una prueba"?
256. Distinguí canary de versión, release con flag, experimento y shadow traffic por su unidad y por la pregunta que responden.
257. Describí las fases expandir, migrar, activar lectura y contraer. ¿Hasta dónde es reversible y qué hay que esperar antes de contraer?
258. El gateway decidió express con las reglas `r42` y Pagos todavía tiene `r41`. ¿Qué pasa y cómo se evita? ¿Con qué modo interpreta un worker un evento emitido antes de que se apagara la flag?
259. ¿Qué hace un servicio cuando la plataforma de flags falla (SDK sin inicializar, control plane caído, snapshot viejo)? ¿Por qué "seguro no siempre significa `false`"?
260. El 1 % del tráfico usa express y falla el 10 %; el resto falla el 0,1 %. ¿Cuánto vale la tasa de error global y qué enseña? ¿Qué se mide para decidir un rollout y qué no va como label?
261. ¿Qué condiciones habilitan avanzar de etapa en un rollout con guardrails? ¿En qué se diferencia un rollout de un A/B test?
262. ¿Qué se prueba en un sistema con flags? ¿Cuál es el ciclo de vida completo de una flag y qué pasa si nadie la retira?

## Observabilidad

263. ¿Qué cuatro problemas tiene el monitoreo en microservicios y cómo define la clase la observabilidad?
264. ¿Qué responde cada pilar (logs, métricas, trazas) y qué papel cumple cada uno en un diagnóstico?
265. ¿Qué campos mínimos tiene un log estructurado? ¿Qué es un wide event y qué ventaja tiene frente a los narrow logs?
266. Distinguí counter, gauge e histogram. ¿Qué es la cardinalidad y por qué `userId` no va como label?
267. ¿Por qué "miente" el promedio? Explicá P50, P95 y P99. ¿Cuántas personas representa el P99 con un millón de usuarios diarios?
268. ¿Qué miden RED y USE, a qué se aplica cada uno y cómo se combinan?
269. ¿Cómo obtiene Prometheus las métricas y qué papel cumplen Grafana y Alertmanager? ¿Qué compromiso hay al elegir el `scrape_interval`?
270. Definí trace y span. ¿Cómo se lee un waterfall? ¿Qué lleva `traceparent` y qué pasa si un servicio no está instrumentado?
271. ¿Qué aporta OpenTelemetry? Compará la instrumentación automática con la manual. ¿Qué hace el Collector?
272. Describí el flujo de investigación desde la alerta hasta la causa raíz. ¿Qué es un exemplar?
273. En la traza del intercambio del TP, ¿por qué hay que propagar a mano en NATS y qué pasa si no se hace? Compará `traceId` y `correlationId`. ¿Cómo se une una compensación disparada por un vencimiento?

## API Gateway

274. ¿Qué problema tiene que el cliente acceda directo a cada servicio? ¿Cuáles son las cuatro decisiones que el gateway toma sobre cada request?
275. En Kubernetes, ¿qué decisión cubren `Gateway` y `HTTPRoute` y cuáles agrega cada implementación? Si `GET /envios/estado/42` coincide con las reglas `/envios` y `/envios/estado`, ¿cuál se aplica?
276. Explicá cómo el gateway implementa un Strangler Fig, la traducción de protocolo y la agregación. ¿Qué riesgo tiene la agregación?
277. ¿Qué es un BFF y de quién es cada uno?
278. Distinguí autenticación de autorización y los códigos 401 y 403. ¿De dónde sale el token y qué ve el gateway de ese proceso?
279. ¿Qué partes y qué claims tiene un JWT y cómo se verifica? ¿Qué pasa si alguien cambia el scope en el camino?
280. Compará la validación local con la introspección ante un token revocado al que le quedaban 10 minutos.
281. Beto envía `X-User-Id: ana` desde Internet. ¿Qué hace el gateway con ese header? Si Ana pide el pedido de Beto con un scope válido, ¿quién la rechaza y con qué dato?
282. Con tres réplicas y un contador local de 100 por minuto en cada una, ¿cuánto puede pedir un cliente? ¿Cómo se corrige y qué dilema aparece si el contador compartido no responde?
283. ¿Por qué el gateway se puede replicar? De estas reglas, ¿cuál va en el gateway: rechazar bodies de más de 1 MB, no enviar a direcciones bloqueadas por fraude, envío gratis desde $ 50.000? ¿Qué tráfico no ve el gateway?

## Service mesh

284. ¿Cuáles son las tres preguntas del tráfico este-oeste y qué problema tiene resolverlas con una librería en cada servicio?
285. ¿Qué es un proxy sidecar y cómo intercepta las llamadas? ¿Qué hay que cambiar en el código de la app?
286. ¿Qué hacen el control plane y el data plane de la malla? ¿Qué pasa con las llamadas si el control plane se cae?
287. ¿Qué garantiza mTLS y quién lo implementa? ¿Qué es un SPIFFE ID y en qué se parece al JWT de un usuario?
288. Envíos tiene un certificado válido y envía `X-User-Id: ana` a Pedidos, con `PeerAuthentication` en `STRICT`. ¿Qué pasa? ¿Qué agrega la `AuthorizationPolicy`?
289. ¿Qué observabilidad da la malla sin tocar el código y qué sigue dependiendo de la app?
290. El proxy reintenta un `POST /facturas` después de un timeout. ¿Qué tiene que garantizar el servicio? ¿Qué es un retry budget? Hacé la cuenta para 100 requests con 3 intentos en tres saltos, sin tope y con un budget de 20 %.
291. ¿Qué es outlier detection, en qué se diferencia de la readiness y por qué tiene un máximo de expulsados?
292. Compará sidecar, ambient y eBPF por dónde corren la capa 4 y la capa 7. ¿Qué políticas necesitan un waypoint? ¿Qué hay que reiniciar para actualizar un sidecar?
293. ¿Qué se paga y qué se obtiene al adoptar una malla?

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
76. El cliente espera una respuesta y queda temporalmente acoplado a la disponibilidad del servidor; la red agrega latencia, fallos y particiones. [T09, p. 4] [T09, p. 7]
77. REST usa JSON/HTTP y OpenAPI y se orienta a APIs públicas; gRPC usa Protobuf/HTTP2 y `.proto` y se orienta al tráfico interno; GraphQL usa schema SDL y queries flexibles y se orienta a BFF/frontends complejos. [T09, p. 17]
78. Impide que una llamada lenta quede bloqueada indefinidamente y propague un hang por toda la cadena de servicios. [T09, p. 12] [T09, p. 13]
79. Permite servir desde el mismo `.proto` Connect con HTTP/1.1 y JSON, gRPC y gRPC-Web, habilitando browsers y depuración con `curl`. [T09, p. 20]
80. `λ` es la tasa de llegada y `μ` la tasa que el sistema termina. Si la llegada supera sostenidamente la capacidad, el backlog crece hasta que el trabajo espera, se rechaza o se pierde; ninguna cola finita evita ese límite. [T09, p. 22] [T09, p. 33]
81. Acota ejecución y espera, rechazando barato antes de consumir recursos. Un retry sin ese límite multiplica intentos sobre un servidor ya saturado; en la demo rindió peor que no reintentar. [T09, p. 26] [T09, p. 27]
82. Conserva la misma tasa de procesamiento: guarda el exceso para terminarlo más tarde. En la demo convierte pérdida en latencia y agrega el costo del broker. [T09, p. 29] [T09, p. 30]
83. La cola era en memoria, perdía el backlog ante un crash, podía entregar duplicados y no resolvía una sobrecarga sostenida. [T09, p. 33]
84. Aumentan alcance, fragilidad, duración, costo y trabajo manual, y disminuye la cantidad. La pirámide clásica cubre pruebas automatizadas de desarrollo; quedan fuera testing exploratorio, usabilidad, performance y seguridad. [T10, p. 3]
85. `Component` define alcance —el servicio entero aislado— y `functional` define intención —cumplir el requerimiento—; una misma prueba puede describirse en ambos ejes. [T10, p. 9]
86. La nota conserva el uso terminológico oral que equipara los nombres, pero el material escrito define integration por alcance y functional por intención. Hasta una aclaración, conviene explicitar qué eje se está usando. [N-2026-09-11-integration-test-functional-testing] [T10, p. 7] [T10, p. 9]
87. Unit cubre una función; integration, la interacción mínima con otro límite; component, un servicio entero aislado; contract, la interfaz consumidor/proveedor; end-to-end, gran parte de la infraestructura y el recorrido de usuario. [T10, p. 5] [T10, p. 7] [T10, p. 9] [T10, p. 10] [T10, p. 12]
88. Stub devuelve una respuesta fija; mock también verifica la interacción; fake es una implementación real simplificada. [T10, p. 6]
89. El consumidor prueba contra un provider mock y publica expectativas; el verificador ejecuta ese contrato contra el proveedor real. [T10, p. 10] [T10, p. 11]
90. Arrange prepara datos; Act ejecuta una acción; Assert verifica un concepto; Clean-up restaura el ambiente. Trampas: setup desproporcionado, varios pasos en Act, ausencia de assert y contaminación entre pruebas. [T10, p. 16]
91. Given expresa el estado preparado, When la acción y Then el resultado esperado. Lo cuenta en lenguaje de negocio para que producto pueda leer el comportamiento. [T10, p. 17]
92. Demuestra qué código se ejecutó y detecta huecos. Incluso al 100 % no garantiza que los asserts ni los casos elegidos comprueben el comportamiento correcto. [T10, p. 18] [T10, p. 19]
93. El linter mira convenciones al escribir; SAST busca vulnerabilidades en código en el commit; SCA analiza dependencias en el build; DAST ataca la app en ejecución en un ambiente de prueba. [T10, p. 20]
94. Ambos artefactos pueden acomodarse al mismo supuesto equivocado y hacer que el test pase sin validar independientemente el requerimiento. [T10, p. 21]
95. Se pierden artefacto único, debugging directo, ACID real, latencia mínima y consistencia inmediata. Se distribuye por aplicaciones inherentemente distribuidas, confiabilidad, performance, datos que no caben en una máquina y tolerancia a fallos. [T11, p. 2] [T11, p. 5]
96. La red puede ser confiable, fair-loss, arbitraria, con demoras arbitrarias o particionada; los nodos, crash-stop, crash-recovery o bizantinos; el tiempo, sincrónico o asincrónico. TLS mitiga los enlaces arbitrarios. [T11, p. 8] [T11, p. 9] [T11, p. 10]
97. No hay reloj global ni memoria compartida y la red puede demorar sin límite, por lo que un timeout no demuestra que la operación falló; reintentar puede duplicarla. [T11, p. 11] [T11, p. 18]
98. La ejecución entrelazada: la auditoría lee `X = 11` después de la primera escritura y `Y = 10` antes de la segunda, e imprime una suma de 21. No equivale a ningún orden secuencial, por lo que no es serializable. [T11, p. 22] [T11, p. 26]
99. El pesimista bloquea antes de operar, paga con esperas y sirve con conflictos frecuentes, como reservar asientos; el optimista detecta conflictos al final mediante versión o CAS, paga con reintentos y sirve con conflictos raros, como editar un perfil. [T11, p. 27] [T11, p. 28]
100. El participante ya se comprometió y los demás harán su parte, por lo que su voto debe ser durable. Si el coordinador cae tras algunos commits, debe recordar la transacción de forma durable; si cae antes de enviarlos, se aborta. [T11, p. 32] [T11, p. 33]
101. Cuando se necesita consistencia inmediata sin estados intermedios ni compensaciones: finanzas core, transacciones entre bases, operaciones regulatorias y sistemas legacy. [T11, p. 35]
102. El orquestador conoce y controla toda la secuencia; en la coreografía cada servicio conoce solo su parte y reacciona a eventos. Al fallar el pago se publica `PAYMENT_FAILED`, inventario revierte la reserva y publica `ROLLBACK_INVENTORY`, y la venta queda `CANCELED`. [T11, p. 20] [T11, p. 37] [T11, p. 39]
103. Serializa solo los threads de su proceso; otra instancia, con su propio mutex, puede acceder al mismo recurso a la vez. [T12, p. 9]
104. Cupos son estado compartido de coordinación, que requiere atomicidad y expiración; pedidos, estado durable de negocio; el pool, estado local y efímero; el ranking, derivado y reconstruible con obsolescencia controlada. [T12, p. 5]
105. El `owner-id` impide liberar un lock ajeno; el fencing token hace que el recurso rechace escrituras atrasadas de quien perdió el lease por una pausa. [T12, p. 10] [T12, p. 11]
106. Antes del efecto, una caída deja el mensaje marcado sin aplicarse y se descartan las reentregas; después, una caída provoca que la reentrega vuelva a aplicarlo. [T12, p. 20]
107. Inserta el identificador con restricción `UNIQUE` en la misma transacción que el cambio de dominio y confirma el mensaje recién después del commit; la reentrega encuentra el identificador y omite el cambio. [T12, p. 21]
108. Garantiza que un solo worker adquiera el procesamiento de forma atómica; no prueba que haya terminado, por lo que requiere expiración, finalización y recuperación. [T12, p. 22] [T12, p. 23]
109. At-most-once admite pérdida sin reentrega; at-least-once reintenta y admite repeticiones; el efecto idempotente aplica el cambio una sola vez pese a entregas múltiples; exactly-once exige declarar su alcance. Pub/Sub es at-most-once. [T12, p. 25] [T12, p. 26]
110. Cada elemento va a un solo consumidor del grupo, siempre se entrega el primer ID no consumido y se espera un ACK. El consumidor puede releer sus pendientes con `XREADGROUP ... 0`, u otro reclamarlos con `XAUTOCLAIM`. [T12, p. 32] [T12, p. 33]
111. Porque los consumidores paralelos pueden terminar en otro orden y los productores independientes no fijan un orden causal; una entidad que requiere orden debe tener una ruta serializada. [T12, p. 28]
112. Plan, code, build, test, release, deploy, operate y monitor. El release es la versión habilitada para producción; su delivery equivale a publicarla en la registry. [T13, p. 5] [T13, p. 9]
113. CI construye y prueba automáticamente cada cambio; Continuous Delivery agrega integración, performance y UAT y deja listo para desplegar sin hacerlo; Continuous Deployment despliega a producción automáticamente todo lo que pasa. [T13, p. 16] [T13, p. 17] [T13, p. 18]
114. El SCM dispara un webhook en cada estado del PR, el CI ejecuta build y tests y publica el resultado en el mismo PR, que luego pasa por revisión de pares antes del merge. [T13, p. 24] [T13, p. 26]
115. Stage es una etapa con función propia; job, un paso dentro de ella; runner, el ambiente de ejecución. En GitLab cada paso corre en un contenedor aislado y comparte archivos por artefactos o cache. [T13, p. 29] [T13, p. 30]
116. Preparation, build, unit test, deliver, integration test y deploy. Cada artefacto de build es un posible release candidate que debe preservarse hasta ser rechazado. [T13, p. 31] [T13, p. 33]
117. Basic es el más simple pero corta el servicio; rolling es progresivo y requiere retrocompatibilidad; blue/green conmuta tráfico entre dos ambientes; canary expone primero a un grupo y limita el impacto. La elección depende del riesgo e impacto de negocio. [T13, p. 38] [T13, p. 40] [T13, p. 41] [T13, p. 42] [T13, p. 43]
118. Avisar al desarrollador de inmediato de cualquier falla funcional, de performance o de seguridad del pipeline; reduce los costos de operación al resolver antes de producción. [T13, p. 45]
119. Async baja el acoplamiento temporal, reduce la latencia percibida al primer hop y amortigua fallas en cascada, pero suma broker, DLQ, idempotencia y observabilidad, y la trazabilidad se vuelve indirecta. No conviene si se necesita la respuesta para seguir, el flujo es simple, no se puede operar un broker o importa la consistencia inmediata. [T14, p. 6] [T14, p. 8]
120. Una cola entrega cada mensaje a un grupo consumidor para repartir trabajo; pub/sub lo entrega a N suscriptores. En Kafka, consumers del mismo grupo compiten como en una cola, y grupos distintos reciben todos los mensajes como en pub/sub. [T14, p. 10] [T14, p. 42]
121. Notification lleva un payload mínimo y genera consultas de vuelta; state transfer lleva el estado completo y duplica datos; un comando no es un evento, sino una instrucción que espera resultado. Claim check aplica cuando el payload es pesado o tiene PII: el evento lleva un puntero al store. [T14, p. 17] [T14, p. 18]
122. Separa el modelo de escritura de read models alimentados por eventos; no convierte al historial en la única fuente de verdad como Event Sourcing. [T14, p. 19]
123. Payment publica `PaymentFailed`, Inventory libera y publica `InventoryReleased`, y Orders cancela y publica `OrderCancelled`, sin dueño único. Un motor como Temporal vuelve explícito el flujo y la compensación, y persiste cada paso para retomar tras una caída sin cobrar dos veces. [T14, p. 23] [T14, p. 25] [T14, p. 26]
124. Porque un paso que no responde deja la Saga colgada en un estado intermedio; al vencer el timeout se toma la rama de compensación, y en Temporal o Cadence los timers sobreviven al reinicio. [T14, p. 27]
125. Orden dentro de una partición, no entre particiones. La key elige la partición por hash, así que misma key preserva el orden de esa entidad. Errores: omitir la key (round-robin), usar una key inestable como `uuid()` o una key demasiado gruesa como `tenantId`, que crea una partición caliente. [T14, p. 32] [T14, p. 33] [T14, p. 35]
126. El log no cambia al leer: cada consumer avanza su propio offset, guardado aparte, y puede volverlo a 0 para replay. El lag es la distancia entre el final del log y ese offset. [T14, p. 31] [T14, p. 38] [T14, p. 43]
127. Cada partición se asigna a un solo consumer del grupo, por lo que el quinto queda ocioso. Si cae uno de dos, el grupo rebalancea y el que queda recibe sus particiones. [T14, p. 40] [T14, p. 41]
128. Es escribir en la base y publicar al broker sin transacción compartida. El outbox guarda el evento en la misma transacción que el cambio y un relay lo publica después. Si el relay cae entre publicar y marcar la fila, se publica dos veces: la garantía es at-least-once. [T14, p. 46] [T14, p. 47] [T14, p. 48]
129. El poller es simple, pero su latencia es el intervalo de poll y requiere cuidado con locks y con marcar lo publicado. CDC lee el write-ahead log con herramientas como Debezium, con menos latencia y una pieza operativa más. [T14, p. 49]
130. Evita aplicar dos veces un mensaje reentregado: inserta `message_id` y el efecto en la misma transacción, y un duplicado es no-op. El broker entrega at-least-once en la práctica, así que el handler debe ser idempotente aunque se active el EOS de Kafka. [T14, p. 50] [T14, p. 51]
131. Un consumer que no avanza su offset atrasa toda su partición y deja a esa entidad sin procesar. Regla: tope de intentos, luego DLQ u otro topic y alerta humana. [T14, p. 53]
132. `correlationId` agrupa todos los eventos de un flujo de negocio y `causationId` apunta al evento o request que causó el actual. Para evolucionar el esquema: versionar el `type` o mantener compatibilidad hacia atrás, preferir cambios aditivos y documentar con AsyncAPI y CloudEvents. [T14, p. 54] [T14, p. 55]

133. Implementar: colección, apertura idempotente con contenido congelado, intercambio 1:1 (apartar, confirmar, rechazar, cancelar, vencer y compensar), progreso eventual, escenarios mínimos y observabilidad de extremo a extremo. Solo diseñar: retos, recompensas, códigos promocionales, rankings, matching, ofertas N:M, vista de operador y campañas. [E03, p. 8]
134. Tras cualquier secuencia con fallos, reintentos, duplicados o desorden: cantidades no negativas, ninguna copia perdida ni acreditada dos veces, contenido de sobre estable, ningún intercambio completado reejecutado y ninguna copia apartada por una oferta u operación terminada. Agrega estabilidad frente a reintentos y ausencia de reservas huérfanas. [E03, p. 2] [E03, p. 5]
135. En `E03`, al publicar; cancelar o vencer libera la copia. En `E01`, recién cuando una aceptación inicia el intento. Publicar, cancelar o vencer no cambian cantidades y una copia puede respaldar varias ofertas. El conflicto sigue abierto. [E03, p. 4] [E03, p. 5] [E01, p. 2] [E01, p. 3]
136. Un mismo `correlationId` o `traceId` recorriendo los servicios en una apertura (escenario 1 o 2), un intercambio exitoso (3) y un intercambio con fallo y compensación (4, 5 o 6). Alcanzan logs estructurados o un backend de tracing, sin mesh ni cluster. [E03, p. 11]
137. Build y pruebas unitarias y funcionales por servicio en cada cambio; despliegue automático a staging; tests de integración allí; gate que deja producción intacta si fallan; producción en AKS solo si pasa. Además, documentar el pipeline y evidenciar una corrida completa y un bloqueo. [E03, p. 11] [E03, p. 12]
138. *Inferencia:* hot spots → locks con fencing y operaciones atómicas [T12, p. 11]; ráfagas → bulkhead y rechazo temprano [T09, p. 26]; clave de partición → orden solo dentro de la partición [T14, p. 33]; retención de claves de idempotencia → ciclo de vida de la clave [T12, p. 19]; cambios sin corte → rolling, blue/green o canary [T13, p. 40] [T13, p. 41] [T13, p. 42]. Preguntas: [E03, p. 12]
139. El 4: en `E03` es un intercambio rechazado porque la copia de quien acepta dejó de estar disponible; en `E01`, dos ofertas respaldadas por la misma copia. El 14 difiere en detalle, el 15 es solo diseño en `E03` y el 16 de `E01` (sin sesión) no existe en `E03`. [E03, p. 10] [E03, p. 11] [E01, p. 6]
140. El sistema corre en contenedores, con un Dockerfile por servicio y ambientes levantados con Docker y docker compose, y se construye con CI/CD con gate de integración en staging. El contenedor es el entorno de ejecución, no la arquitectura: el recorte, la persistencia o la comunicación deben responder a drivers propios. [E05] La versión anterior imponía Kubernetes con producción en AKS. [E04]
141. Cada servicio se rastrea a uno o más bounded contexts, cada agregado tiene un servicio dueño, cada evento que cruza fronteras tiene contrato y cada invariante tiene un mecanismo. Los cambios del modelo se registran con su motivo en la sección 11.1. [E05]
142. "Escalable" es un adjetivo. Un escenario fija estímulo, entorno, respuesta, medida y prioridad; por ejemplo, 20.000 aperturas por minuto durante 5 minutos con p99 menor a 500 ms y ninguna apertura duplicada. [E05]
143. Productor, consumidores, tópico o canal, clave de partición con el orden que garantiza y payload. El envelope lleva identificador único del evento, tipo, versión del esquema, momento de ocurrencia y `correlationId`. Además se piden la estrategia de evolución y el mecanismo contra la divergencia entre estado y publicación. [E05]
144. Apertura con reintento (1–2), intercambio exitoso (3), rechazado (4), interrupción y compensación (5–6), resultado desconocido (7) y cierre contra aceptación (12); más retos y el escenario 15 como solo diseño. La máquina de estados incluye transiciones, quién dispara cada una, timeouts y compensaciones. [E05]
145. El healthcheck debe indicar si el contenedor puede aceptar trabajo; si siempre responde OK, no informa nada y la consigna resta puntos por healthchecks triviales. [E05] En Compose solo informa el estado y no reinicia el contenedor, así que se parece más a readiness que a liveness. [T07, p. 17] [T07, p. 18] *Inferencia:* un dependiente que espera con `service_healthy` arrancaría igual contra una instancia que perdió, por ejemplo, su base de datos.
146. Promover un artefacto inmutable: versionar y etiquetar las imágenes, y desplegar en producción exactamente la imagen probada. Un tag puede moverse, mientras que un digest identifica el contenido de forma inmutable. [E05] [T06, p. 35] [T06, p. 36]
147. Recorte de servicios, estilo de comunicación, broker y clave de partición, coordinación del intercambio, idempotencia, publicación confiable, persistencia de las cantidades de la colección, organización de ambientes y empaquetado en contenedores, y despliegue y promoción. Cada uno necesita al menos dos alternativas reales con el motivo de su descarte. [E05] La versión anterior pedía además observabilidad y ambientes en Kubernetes. [E04]
148. Que migrar no es progreso: Amazon migró porque el monolito frenaba a cientos de desarrolladores; Segment volvió a un monolito porque mantener ~140 servicios consumía al equipo; Prime Video pasó a un solo proceso y bajó ~90 % el costo. Es una decisión con costos y a veces la correcta es la inversa. [T15, p. 7]
149. Alcanzan el despliegue independiente, la autonomía de equipos, el escalado selectivo, el aislamiento de fallas y un stack distinto donde hace falta. No alcanzan el hype, el código feo, la performance, "para escalar" y el CV-driven development. Un monolito escala horizontalmente detrás de un load balancer: hay que mostrar que una parte escala distinto que el resto. [T15, p. 8] [T15, p. 9]
150. Un solo deployable con módulos que son bounded contexts, con API pública, prohibición de llamar internals verificada en CI e idealmente schema propio. Da los beneficios de diseño sin red, deploy distribuido ni consistencia eventual, y puede ser el destino final. [T15, p. 10]
151. CI/CD que haga trivial desplegar, observabilidad básica con `correlationId`, tests de caracterización y contract tests, y provisioning de un servicio en horas. [T15, p. 11]
152. Dependencias del código (qué depende de qué, ciclos, god modules); change coupling en git (qué cambia junto aunque no se importe); datos de runtime (carga, disponibilidad, tablas calientes); y organización (dueños, frecuencia y motivo del cambio). [T15, p. 15] [T15, p. 16] [T15, p. 17]
153. Se busca bajo acoplamiento, valor visible y riesgo bajo. Notificaciones solo depende de `common`; pagos y órdenes son centrales, muy acoplados y caros si fallan. El primer servicio sirve para aprender el camino. [T15, p. 15] [T15, p. 18]
154. Seis llamadas por red para un caso de uso, servicios que siempre se despliegan juntos y un servicio por entidad. Juntar obliga a migrar datos y reescribir contratos; partir un módulo bien encapsulado no. [T15, p. 19]
155. Tests de caracterización sobre el borde público, reorganización detrás de una interfaz local e inventario de crons, listeners, mailers, archivos y batch. Sin el inventario, el monolito sigue escribiendo los datos del servicio por atrás. [T15, p. 20]
156. Strangler Fig; Branch by Abstraction; Parallel Run o dark launching; Decorating Collaborator o CDC; feature toggles y canary; Anti-Corruption Layer. Se combinan. [T15, p. 30]
157. La fachada se vuelve segundo monolito si acumula reglas de negocio en lugar de solo reenviar, registrar y permitir volver. El ACL encubre un monolito distribuido si el servicio llama sincrónicamente al monolito en cada request. [T15, p. 24] [T15, p. 29]
158. Porque repite el efecto: manda, cobra o reserva dos veces. Se apaga el efecto en la sombra con stubs o se usa una clave de idempotencia. [T15, p. 26]
159. Porque la base se vuelve la API real sin haber sido diseñada como tal: nadie cambia el schema sin coordinar, no hay dueño del dato y las reglas se duplican o se saltean. Datos primero conviene cuando hay dudas sobre la frontera por joins o transacciones. [T15, p. 33] [T15, p. 34]
160. Split Table reparte columnas entre contextos: copiar, mover lecturas, mover escrituras, borrar columnas; si dos contextos escriben la misma columna, primero se decide el dueño. Move FK to Code reemplaza la FK por un ID validado en código o con eventos; se pierde la integridad referencial y hay que decidir qué pasa al borrar la entidad referida. [T15, p. 36] [T15, p. 37]
161. Fase 1: escribe el monolito y el servicio lee una réplica vía CDC; rollback, dar vuelta la ruta. Fase 2: escribe el servicio y el monolito recibe copia; rollback difícil, requiere reconciliar. Fase 3: escribe el servicio sin sincronización; no hay vuelta barata. [T15, p. 39] [T15, p. 40]
162. Ruta y módulo viejos borrados, un solo escritor por tabla, grant de escritura revocado al monolito y rollback ensayado. El flag cambia el código activo, pero no revierte datos divergentes. [T15, p. 40]
163. Cuánto tiempo puede estar inconsistente el dato, qué pasa si se reservó stock y falló el pago, y si la invariante tiene que ser inmediata. Si debe serlo, quizás esos datos van en el mismo servicio. [T15, p. 41]
164. Composición por API, read models con CQRS y CDC hacia un data warehouse o lake. Está prohibido abrir una conexión SQL de solo lectura a la base de otro servicio. [T15, p. 42]
165. Diseñar los equipos con la forma que se quiere para la arquitectura. Team Topologies define equipos stream-aligned, platform y enabling. [T15, p. 44]
166. DORA (deploy frequency, lead time, change failure rate, MTTR) y métricas propias: deploys coordinados, porcentaje de tráfico fuera del monolito, tablas del monolito leídas por otros servicios y, por tajada, error rate, lag del CDC y rollback ensayado. [T15, p. 46]
167. Monolito distribuido, base compartida "por ahora", librería compartida con lógica de dominio, llamadas sincrónicas encadenadas, big-bang disfrazado y extraer sin sacar. Si el código viejo sigue vivo "por las dudas", hay dos implementaciones que mantener. [T15, p. 47]

168. La API coordina clientes y componentes; scheduler asigna nodo; controller manager reconcilia; etcd persiste; kubelet mantiene los contenedores asignados. El runtime CRI los ejecuta y CNI configura conectividad. [T16, p. 7] [T16, p. 8] [T16, p. 9]
169. Comparten red/IP/puertos, normalmente IPC y hostname; PID está separado por defecto y cada raíz es propia, con volúmenes compartibles. No migra: se reemplaza por otro UID. [T16, p. 10] [T16, p. 15]
170. Requests orientan al scheduler; limits restringen ejecución. CPU puede sufrir throttling y memoria, OOM; almacenamiento efímero mide bytes, no throughput. [T16, p. 16]
171. Los init comunes preparan en secuencia y terminan con éxito; el sidecar nativo queda activo con restartPolicy Always dentro de initContainers; el efímero depura un Pod existente y no se reinicia automáticamente. DNS solo prueba que el nombre existe. [T16, p. 17] [T16, p. 23]
172. Running describe ejecución, no preparación para tráfico. Las fases son Pending, Running, Succeeded, Failed y Unknown; los estados del contenedor, Waiting, Running y Terminated. Startup demora las otras probes, readiness decide preparación y liveness detecta necesidad de reinicio. [T16, p. 18] [T16, p. 19] [T16, p. 20]
173. Deployment para réplicas y actualización declarativa; StatefulSet para identidad estable; DaemonSet para un Pod por nodo elegible; Job para finalizaciones exitosas; CronJob para programación, con tareas idempotentes por posibles duplicados. [T16, p. 26] [T16, p. 31] [T16, p. 33] [T16, p. 37] [T16, p. 38]
174. Replicas escala; el template inicia rollout. Cambiar un ConfigMap por sí solo no cambia el template ni inicia rollout. [T16, p. 26] [T16, p. 42]
175. El Deployment gestiona ReplicaSets y el ReplicaSet crea/elimina Pods; el kubelet ejecuta contenedores. ownerReferences con controller true permite seguir Pod → ReplicaSet → Deployment. [T16, p. 30] [T16, p. 43] [T16, p. 46] [T16, p. 47]
176. Son operaciones asíncronas distintas: persistir Namespace, observarlo, solicitar crear la cuenta y persistirla. Convergencia del clúster no equivale a inconsistencia de etcd. [T16, p. 48] [T16, p. 49]
177. Autenticación identifica, autorización permite la acción y admisión puede mutar o rechazar el objeto. get/list/watch no pasan por admisión ni persisten objetos. ServiceAccount es identidad; RBAC concede permisos aparte. [T16, p. 52] [T16, p. 53] [T16, p. 54]
178. apiVersion es la versión de la API del recurso, no del clúster. Namespace delimita nombres y organiza, sin aislamiento de seguridad o red por sí solo; PVC es namespaced y PV tiene alcance de clúster. [T16, p. 55] [T16, p. 56]
179. Nodos en el underlay, Pods en el rango del plugin de red y Services en un rango virtual. Contenedores del mismo Pod usan localhost; Pod–Pod, la red del CNI sin NAT; Pod–Service, la ClusterIP programada por kube-proxy o una alternativa; desde afuera, NodePort, LoadBalancer, Ingress, Gateway o port-forward. [T17, p. 2] [T17, p. 3] [T17, p. 4] [T17, p. 5] [T17, p. 7] [T17, p. 8]
180. CNI conecta al Pod y le asigna IPs; kube-proxy solo programa el reenvío de Services. El ruteo nativo conserva IPs de Pods y suele ser más eficiente; la overlay encapsula, no requiere rutas de Pods en el underlay pero obliga a operar túneles y MTU, y no cifra ni reemplaza NetworkPolicy. [T17, p. 5] [T17, p. 6]
181. Por el selector; Kubernetes mantiene direcciones y puertos en EndpointSlices que el controlador actualiza según Pods y readiness. Endpoints está deprecada desde v1.33 porque no soporta dual-stack y trunca más de 1000 endpoints. [T17, p. 9] [T17, p. 10] [T17, p. 12]
182. CoreDNS resuelve el Service y el `resolv.conf` del Pod incluye la búsqueda `default.svc.cluster.local svc.cluster.local cluster.local`. [T17, p. 13] [T17, p. 16]
183. ClusterIP da IP virtual estable con reenvío; headless (`clusterIP: None`) devuelve IPs de Pods y el cliente elige. iptables/nftables eligen backend por conexión. [T17, p. 17]
184. ClusterIP es interna; NodePort publica un puerto en los nodos (30000–32767 por defecto); LoadBalancer requiere integración con proveedor o controlador, no necesariamente nube, y suele usar NodePorts; ExternalName es un CNAME sin proxy. [T17, p. 19]
185. Un controlador instalado; Gateway API además requiere sus CRDs. GatewayClass define tipo y controlador, Gateway la infraestructura y listeners, HTTPRoute las reglas: el operador administra la infraestructura y las aplicaciones las rutas. Se recomienda para nuevos despliegues; la API Ingress está congelada y el ingress-nginx comunitario fue retirado. [T17, p. 8] [T17, p. 21] [T17, p. 23]
186. Cuando alguna política lo selecciona en esa dirección. Si ambos están aislados, debe permitirlo el egress del origen y el ingress del destino; las reglas se suman sin orden. No autentica, no cifra, no filtra URLs y sin enforcement de la red no tiene efecto. [T17, p. 25] [T17, p. 26]
187. Capa escribible: nada tras reinicio o reemplazo; `emptyDir`: reinicios del contenedor, no la eliminación del Pod; PV: reemplazo del Pod según backend y retención. Un volumen puede estar ligado a una zona o un nodo, y sobrevivir al clúster no es garantía de Kubernetes. [T17, p. 28] [T17, p. 29]
188. En estático el administrador crea el PV y el PVC lo elige; en dinámico el PVC pide una clase y su provisioner, por ejemplo un driver CSI instalado aparte, crea el PV. `WaitForFirstConsumer` demora la vinculación hasta conocer el Pod y su topología. [T17, p. 33] [T17, p. 34] [T17, p. 35] [T17, p. 36] [T17, p. 37] [T17, p. 38]
189. RWO admite lectura/escritura desde un nodo, aunque haya varios Pods en él; RWOP restringe a un único Pod y requiere CSI. `Retain` conserva volumen y datos para recuperación manual; `Delete` borra PV y almacenamiento si el backend lo soporta. [T17, p. 31]
190. Un tipo propio registrado con un CRD y un controlador con la lógica de la aplicación. El controlador crea las instancias, ajusta replicación y configuración, reporta en `status` y repite hasta converger. [T17, p. 39] [T17, p. 40]
191. Variables y argumentos no cambian hasta recrear el Pod; un volumen proyectado se actualiza eventualmente salvo con `subPath`; la aplicación debe releer el archivo; un cliente de la API puede observar cambios. Nada de esto dispara un rollout. [T17, p. 48] [T16, p. 47]
192. Quien puede hacer get/list/watch, acceder a etcd o crear Pods del namespace puede leerlo; en etcd está sin cifrar salvo cifrado at rest; Base64 no cifra. Prácticas: cifrado at rest, RBAC mínimo, acceso limitado, rotación y no versionar credenciales reales. [T17, p. 50] [T17, p. 51] [T17, p. 52] [T17, p. 53]
193. Sin un mount namespace propio, `/proc` sigue mostrando la vista del host, y montarlo de nuevo puede afectarlo. `PREROUTING` solo procesa paquetes que llegan de afuera; el tráfico originado localmente necesita una regla DNAT en `OUTPUT`. [G01-S05] [G01-S07] [T05, p. 21]
194. `WORKDIR` evita trabajar en `/`; `.dockerignore` achica el contexto y evita meter `.env` en una capa; pinnear evita builds distintos entre integrantes; copiar `requirements.txt` primero preserva el cache de dependencias; el usuario 999 evita ejecutar como root; el healthcheck con Python reporta salud sin instalar herramientas. [G02]
195. Para que la imagen no quede atada a un ambiente: la configuración se fija al crear el contenedor y los valores quedan fuera de git. Con redes `internal` separadas: una compartida entre gateway y backends y una por backend con su Redis. [G02]
196. `release` y `deploy` solo imprimen los comandos. Le falta un ambiente de staging con tests de integración como gate antes de producción, que exige `E05`. [G03] [E05]
197. `compliance` valida rápido la estructura del monorepo antes de construir; el child pipeline se genera con los servicios afectados por el cambio, así que solo esos se construyen y prueban. [G05-MONOREPO]
198. Porque la espera del cliente fija la tasa de llegada y hace que el retry tenga sentido; con la CPU como cuello de botella, `μ` depende de los cores y del trabajo por request, no de cuántos pedidos están en vuelo. El 1,83 dice cuánto hay que insistir en la práctica: es el dato para fijar `maxAttempts`. [G10]
199. Porque responde 202 de inmediato: el cliente ya no espera, los 100 pedidos entran juntos y el backlog se muda al broker. Si quien llama necesita la respuesta, conviene acotar admisión y reintentar; la cola gana cuando nadie espera. [G10]
200. `SET NX PX` con un UUID v7 en tres masters, lock con al menos dos; validez = TTL − transcurrido − drift, verificada antes de la sección crítica; liberación por Lua solo si coincide el token. No usa fencing tokens verificados por el recurso. [G05-LOCK] [T12, p. 11]
201. Devuelve `null` y cachea `{"_absent":true}` con TTL; si Valkey falla, responde desde PostgreSQL y cuenta la falla. Con el cache frío, varias solicitudes concurrentes fallan antes de que el primer resultado se guarde. [G05-CACHE] [G05-EVAL]
202. Coreografía: saga por eventos en un topic único; si falla el pago, inventario repone stock y la venta queda `CANCELED`. Orquestación: 2PC con `checkout-events` para prepare y decisión, y `checkout-notifications` para votos y ACKs; con un NO, el coordinador aborta. [G06-CONTEXTO] [G07] [T11, p. 29]
203. Minikube tiene su propio almacén de containerd, así que hay que construir con `minikube image build` o cargar la imagen. El Pod de depuración no tiene un controlador que declare que debe existir. [G08]
204. No están etcd, apiserver, scheduler, controller-manager ni kube-proxy; aparecen `konnectivity-agent`, `cloud-node-manager`, drivers CSI y Cilium. LoadBalancer → IP pública y regla en el balanceador; PVC → Azure Disk; Gateway → IP pública del proxy de Envoy. [G09]
205. Un disco nuevo trae `lost+found` y Postgres no inicializa en un directorio no vacío. *Inferencia:* el Azure Disk es `ReadWriteOnce` y debe desengancharse del nodo viejo antes de engancharse en el nuevo. [G09] [T17, p. 31]
206. `terraform.tfstate`, el kubeconfig, los deploy tokens y las contraseñas. El kubeconfig se comparte por un canal privado, o cada integrante levanta su cluster. El disco se borra al destruir el cluster, así que los datos de prueba deben poder recrearse. [G09]
207. La aplicación se ocupa de reglas de negocio y contratos, idempotencia y transacciones, ACK/NACK, timeouts y compensaciones, y de saber cuándo está lista. Kubernetes crea, reemplaza y distribuye Pods, descubre servicios, enruta tráfico, programa Jobs, aplica límites y escala. No reimplementar el control plane es no escribir supervisores, descubrimiento de IPs, esperas eternas ni reintentos infinitos que la plataforma ya resuelve: la plataforma administra procesos y el código protege la semántica del negocio. [T18, p. 2] [T18, p. 3] [T18, p. 32]
208. `SIGTERM`, drenar (no aceptar trabajo nuevo), finalizar o cancelar lo en curso, cerrar conexiones, ACK y flush, y salir antes del plazo. Si se agota, llega `SIGKILL`; por eso el timeout del `Shutdown` (25 s en el ejemplo) tiene que ser coherente con el período de gracia. El worker deja de consumir, espera lo que está en vuelo, hace ACK solo de lo completado y devuelve lo demás a la cola. [T18, p. 6] [T18, p. 7] [T18, p. 21]
209. Si cae la base, el kubelet reiniciaría todos los Pods sin arreglar nada: reinicios en cadena. La liveness detecta un bloqueo local; la readiness representa la capacidad real y evalúa las dependencias según la degradación posible. [T18, p. 16] [T18, p. 17] [T21, p. 26]
210. Porque el cambio de endpoints es asíncrono y no garantiza que el Pod saliente deje de recibir solicitudes nuevas; si corta de golpe, esas requests fallan. [T18, p. 18]
211. El broker reentrega el mensaje y otro worker intenta cobrar de nuevo. La unicidad se diseña en el dominio y la persistencia: clave idempotente, restricción única y consulta del resultado previo. [T18, p. 22]
212. API HTTP: Deployment + Service, con readiness, shutdown y sin estado. Consumidor continuo: Deployment, con KEDA opcional, con ACK, backpressure y drenaje. Tarea finita: Job, con exit codes e idempotencia. Tarea programada: CronJob que crea Jobs, con idempotencia y control de concurrencia. [T18, p. 20]
213. Sesión en memoria → estado fuera del Pod. ACK antes del commit → ACK solo después del commit, con idempotencia. Worker sin `SIGTERM` → drenaje que confirma solo lo completado. Retry infinito y logs a archivo → reintentos acotados a operaciones seguras e idempotentes, y logs a `stdout`. [T18, p. 30] [T18, p. 10] [T18, p. 21] [T18, p. 22] [T18, p. 24] [T18, p. 8]
214. El chart es el paquete con los recursos configurables; la release, una instalación concreta y versionada, que permite rollback; el repositorio, la fuente de charts. `CHART VERSION` es la versión del empaquetado y `APP VERSION` la de la aplicación: `bitnami/argo-cd` 7.0.14 empaqueta Argo CD 2.12.4. [T19, p. 5] [T19, p. 6]
215. Un StatefulSet con el Pod `my-postgresql-0`, un Service ClusterIP y uno headless (`-hl`) en el 5432, y un PVC `data-my-postgresql-0` de 8 Gi `RWO`. Es el patrón de StatefulSet con Service headless y `volumeClaimTemplates`. [T19, p. 7] [T16, p. 33]
216. Porque el despliegue es pull con ArgoCD: el pipeline no toca el cluster, y un rollback con `kubectl` o `helm` directo no cuenta; se hace por Git. Helm sirve como base común con valores por ambiente, que el `repo-server` de ArgoCD renderiza. [E06] [T20, p. 18]
217. Declarativo, versionado e inmutable, obtenido automáticamente y reconciliado continuamente. Cambian la seguridad, la auditoría, el rollback, la propiedad del cambio y la respuesta a incidentes. [T20, p. 3] [T20, p. 4]
218. Git solo versiona lo declarado: si el registry permite sobrescribir el tag, el artefacto cambia sin que Git lo registre. En producción, digest, o SemVer + digest como compromiso. [T20, p. 8] [T20, p. 9]
219. Push es simple, familiar y de baja latencia, pero deja kubeconfigs en los runners, expone el API server y agranda el *blast radius* si se compromete el CI; sirve en laboratorios, ambientes efímeros y despliegues no críticos. Pull deja las credenciales dentro del clúster y reconcilia continuamente, a costa de más componentes, consistencia eventual y dependencia del agente; sirve en producción, multi-clúster, sectores regulados y redes cerradas. [T20, p. 11]
220. Es la divergencia entre lo declarado y lo observado. El `kubectl edit` se audita y vuelve a Git o se revierte; el HPA no debe pelearse con GitOps (ignorar el campo o delegar su propiedad); el sidecar es esperado si lo gobierna la plataforma; el Secret externo es correcto: Git declara la intención de consumirlo, no el valor. [T20, p. 12] [T20, p. 13]
221. Datos, migraciones y efectos externos. Se mitiga separando el rollback de la aplicación, el de los datos y el del tráfico. [T20, p. 7]
222. `argocd-server` (API, UI, SSO, RBAC; superficie de administración), `repo-server` (clona y renderiza; puede saturarse), `application-controller` (compara y reconcilia; una mala configuración produce syncs peligrosos), Redis (caché; afecta la performance) y ApplicationSet Controller (genera Applications; multiplica errores). Si cae, las aplicaciones siguen corriendo, pero se pierden la sincronización, la visibilidad, la detección de drift y la operación declarativa. [T20, p. 18] [T20, p. 19] [T20, p. 20]
223. Kubeconfigs en el CI y ArgoCD instalado solo para mirar dashboards: se conservan los riesgos del push sin ganar la seguridad del pull. [T20, p. 14] [T20, p. 11]
224. ArgoCD: UI fuerte, modelo `Application` claro, control plane centralizado, multi-clúster práctico con ApplicationSets; riesgo de centralización y credenciales amplias. Flux: CLI, eventos y CRDs, controladores componibles, multi-clúster sólido pero menos visual; riesgo de complejidad distribuida al depurar. [T20, p. 25]
225. Resiliencia: qué pasa cuando algo falla; escalabilidad: qué pasa cuando crece la carga; disponibilidad: qué porcentaje del tiempo funciona. Un 200 tardío retiene un slot en cada salto de la cadena hasta agotarlos, mientras un 503 rápido los libera. [T21, p. 3] [T21, p. 5]
226. Cada capa multiplica los intentos de la anterior: 3 × 3 × 3 = 27. Se mitiga reintentando en un solo nivel, solo errores transitorios, con backoff y jitter, y con un budget de alrededor del 10 %. [T21, p. 6] [T21, p. 11]
227. Unos 43 minutos por mes. Cinco en serie dan ≈ 99,5 %, unas 3,6 horas caído por mes; dos réplicas independientes, 99,9999 %. [T21, p. 7] [T21, p. 8]
228. Con timeouts locales cada capa espera por su cuenta y la cadena sigue trabajando aunque el cliente se haya ido; con el deadline propagado, cada salto recibe lo que queda del presupuesto y la cadena se cancela. En la demo, sin propagación el cobro queda hecho después del timeout del cliente. [T21, p. 10]
229. Cerrado (las llamadas pasan), abierto tras el umbral de fallas (503 local sin llamar a la dependencia) y semiabierto tras esperar (una prueba: un 200 cierra, un 500 reabre). Su apertura indica que una dependencia está fallando: hay que alertar e investigar esa causa. [T21, p. 15] [T21, p. 16]
230. 429 cuando un cliente supera su límite; 503 con `Retry-After` cuando el servicio está sobrecargado. Un ataque L7 parece tráfico legítimo y el autoscaling escala el costo junto con el ataque (*denial of wallet*); el volumen se absorbe en el borde. [T21, p. 18] [T21, p. 19]
231. Da reinicio de contenedores, reprogramación ante la caída de un nodo, cantidad de réplicas y salida del Service de los Pods no `Ready`. No da timeouts, retries, circuit breakers, idempotencia ni capacidad ante la caída de una zona o región. El PDB evita que un drenado baje todas las réplicas; `topologySpreadConstraints` reparte por zona o nodo; `podAntiAffinity` evita réplicas en el mismo nodo. [T21, p. 25] [T21, p. 26] [T21, p. 27]
232. `backoffLimit` cuenta fallas (6 por defecto); `activeDeadlineSeconds` corta el Job por tiempo total aunque queden reintentos. `OnFailure` reinicia el contenedor en el mismo Pod; `Never` crea un Pod por intento y conserva los logs. `podFailurePolicy` distingue errores: `FailJob` para uno que no vale la pena reintentar, `Ignore` para una disrupción que no debe gastar el presupuesto. [T21, p. 12] [T21, p. 13]
233. `ceil(4 × 90 / 60)` = 6. Requiere requests de CPU definidos. La CPU rara vez refleja la carga real; conviene una métrica que se mueva al agregar réplicas, como RPS por réplica, p95 o profundidad de cola. [T21, p. 32] [T21, p. 33]
234. KEDA escala por eventos, como el lag de Kafka, incluso hasta cero, creando un HPA por debajo. HPA y VPA sobre la misma métrica se pelean. Dentro de un grupo, cada partición tiene como máximo un consumidor, así que los consumidores de más quedan ociosos. [T21, p. 35] [T21, p. 34] [T21, p. 37]
235. 50 réplicas × 20 conexiones son 1000 conexiones a la base. Se previene con connection pooling, un máximo de réplicas según la capacidad de la base, y rate limit y colas delante de los recursos no elásticos. [T21, p. 38]
236. Backup y restore: horas/horas, bajo costo; pilot light: decenas de minutos, bajo-medio; warm standby: minutos, medio; active-active: segundos/≈ 0, alto. No se estira porque etcd y el control plane esperan latencias de pocos milisegundos: un clúster por región. [T21, p. 48] [T21, p. 42]
237. Un único AKS con `dev`, `staging` y `prod` como namespaces; GitOps pull con ArgoCD; registro y credenciales sin secretos en el repo; tests unitarios y funcionales en Docker. Prevalecen sobre las secciones 8 y 9 de la segunda entrega, y los cambios se registran en `docs/cicd.md`. [E06]
238. Push al ACR, escritura en el repositorio de estado deseado y, si hace falta consultar ArgoCD, un token de solo lectura; nunca credenciales de administración del cluster. Los Secrets llegan por Sealed Secrets, SOPS o External Secrets, nunca con `kubectl create secret`. [E06]
239. CI en push o PR (checks en verde); `dev` en el merge (*Synced*, *Healthy* y smoke test); `staging` con las mismas imágenes (*Synced* y *Healthy*); integración en `staging` (todos pasan); `prod` con aprobación manual (*Synced*, *Healthy* y smoke test). Si falla la integración, no se genera ningún cambio en el estado deseado de `prod`. [E06]
240. Porque `prod` ejecuta la misma imagen que se probó en `staging`, y no puede haber testing en producción ni, por lo tanto, en `staging`. Los unitarios corren en una etapa o imagen de test; la cobertura funcional se mide con el servicio instrumentado en una imagen de test aparte. Cada una tiene que superar el 80 % por separado, con gate. [E06]
241. Trazable a un RF o escenario, con datos conocidos y repetible, que verifique un resultado de negocio y no solo un 200, y cuya aserción la produzcan los dos servicios reales. Un test de un solo servicio no prueba la integración entre ellos. [E06]
242. Con un mecanismo versionado y acotado a `staging`: un mock cuya confirmación el test puede retener, un proxy de fallas desplegado por ArgoCD o un Job de ArgoCD con permiso solo para borrar Pods de ese namespace, documentado con su limpieza en `docs/cicd.md`. [E06]
243. Con `git revert` del cambio de versión en el estado deseado, o fijando la versión anterior, para que ArgoCD converja; se muestran el commit, la sincronización y el resultado. Un rollback con `kubectl` o `helm` contra el cluster no cuenta. [E06] [T20, p. 7]
244. No se evalúa si no puede reproducirse un despliegue en AKS vía pipeline con promoción `staging` → `prod` (RNF-04 y RNF-05). La falta del gate de integración, el incumplimiento de RNF-06 y otros defectos bajan la nota en esta entrega, pero en la final rigen RNF-01 a RNF-06 completos. [E06] [E01, p. 6]
245. Porque `E06` toma `docs/arquitectura.md` y los ADRs, "con las correcciones que surjan de la devolución", como fuente de verdad de los servicios, su comunicación y sus contratos. Los mocks tienen que respetar esos contratos. Los cambios de nombres, endpoints o códigos se registran en `docs/cicd.md`. [E06] [D02]
246. El healthcheck de Compose se parece a una readiness y no reinicia el contenedor. [T07, p. 18] Por eso un proceso colgado pero vivo quedaba `unhealthy` hasta que alguien lo reiniciara a mano. [D02] En Kubernetes, si falla la liveness, el kubelet reinicia el contenedor; si falla la readiness, el Pod sale de los endpoints. [T18, p. 15] La liveness detecta un bloqueo local y no depende de PostgreSQL; la readiness representa capacidad real. [T18, p. 16]
247. Un solo paso que agote su deadline de 2 s ya supera los 1.500 ms, y la devolución señala que no hay presupuesto por paso. [D02] El presupuesto reparte el total entre las llamadas y lo hace cumplir con un deadline propagado que baja en cada salto (2 → 1,5 → 1 → 0,5 s). [T21, p. 10] Se reintentan solo errores transitorios, en un único nivel de la cadena y con un retry budget de alrededor del 10 %. [T21, p. 11]
248. Mezcla "recurso creado" con un resultado de negocio negativo y contradice la regla del propio documento de no esconder errores en un éxito. [D02] La teoría pide códigos correctos y señala como antipatrón el `200` con error en el cuerpo. [T09, p. 9] [T09, p. 10] Alternativas: `200` o `409` con motivo, o `201` con estado explícito. [D02] Cambiarlo modifica el contrato que implementan el servicio real y los mocks y que verifican los tests; se registra en `docs/cicd.md`. [E06]
249. Las bases no son escribibles en común, pero el fallo y la saturación son comunes y la restauración es conjunta. Además, los NFR suponían un host propio de PostgreSQL y unas 5 réplicas que ningún ambiente tenía. [D02] La cátedra propuso dos instancias, núcleo y vistas, si el presupuesto lo permite. [D02] La tercera entrega exige que los ambientes no compartan datos y que al menos un servicio real tenga base propia. [E06]
250. *Inferencia:* un rolling update con `maxUnavailable: 0` y drenado no necesita ventana de mantenimiento. [T18, p. 18] El hotfix recorre el mismo flujo `dev → staging → prod`, gate de integración incluido; la política define quién aprueba el cambio a `prod` y con qué urgencia. [E06]
251. Desplegar es tener el código en producción, en el 100 % de las réplicas desde el día 1; activar es habilitar la flag para una audiencia; exponer es que la variante se ejecute de verdad, y siempre es menor o igual a la audiencia. El rollback de flag selecciona el camino anterior, pero los datos ya escritos permanecen. [T22, p. 6]
252. Release: estática, vida corta, se retira tras estabilizar el lanzamiento. Experimento: dinámica, vida corta, se retira al concluir el análisis. Operacional: estática, vida larga, con revisión periódica y simulacros. Entitlement: dinámica, vida larga, dura mientras exista el producto. La habilitación comercial y la autorización requieren controles del servidor: el servidor debe autorizar igual. [T22, p. 7]
253. El control plane define y publica reglas versionadas; es lento, humano y auditado. El data plane evalúa por request y en memoria: un SDK con caché local en el backend y un evaluador remoto en el BFF o el edge para el browser. La clase pide evitar que cada evaluación sea una consulta a la base central. [T22, p. 4] [T22, p. 8] *Inferencia:* esa consulta sumaría latencia y un punto de falla a cada request.
254. Con `bucket = hash(salt + tenant_id) mod 10000` la misma unidad cae siempre en el mismo bucket. La unidad de asignación es el tenant, no el request: un tenant de diez puede generar el 55 % del tráfico, así que se mide por variante y por tenant. Subir de 10 % a 25 % conserva la cohorte previa si no cambian la clave ni el salt. [T22, p. 11]
255. Browser: UI y navegación, pero no permisos ni ejecución del servidor. Red L7: la ruta a una versión, pero no la semántica dentro del proceso. Backend: reglas y efectos del negocio, pero no revertir datos ya escritos. Datos: lectura y escritura compatible, pero no deshacer una migración destructiva. [T22, p. 13] Lo que decide el browser es manipulable, así que el backend valida igual. [T22, p. 14] [T22, p. 41]
256. Canary de versión: tráfico hacia un binario; ¿el nuevo deploy es saludable? Release con flag: audiencia de una capacidad; ¿a quién habilitamos la función? Experimento: unidad aleatorizada estable; ¿qué efecto causa la variante? Shadow traffic: copia de operaciones; ¿qué haría el nuevo camino?, aislando los efectos. [T22, p. 16]
257. Expandir: columna nullable, se sigue escribiendo el formato viejo. Migrar: backfill y validación, se escribe viejo + nuevo y se lee viejo. Activar lectura: se escribe nuevo y se lee nuevo por cohorte. Contraer: se elimina el formato viejo. Es reversible hasta la fase 3; contraer no tiene retorno. Antes hay que esperar consumidores, jobs y eventos atrasados. [T22, p. 20]
258. Pagos reevalúa con `r41` y obtiene classic: el mismo tenant recibe dos decisiones. Se evita fijando la decisión de la operación: Checkout persiste el modo con la orden y Pagos lo recibe sin volver a sortear; los reintentos recuperan la misma decisión. [T22, p. 25] [T22, p. 26] El worker usa el modo que trae el evento, no la flag actual; un bloqueo operacional puede detener el procesamiento, nunca cambiar el significado. [T22, p. 28]
259. SDK sin inicializar: no recibir tráfico (readiness) o un default seguro explícito. Control plane caído: última configuración válida, con un límite de edad medido y alertado. Snapshot fuera del límite: política por riesgo (Pagos rechaza lo nuevo, Recomendaciones omite). El valor seguro depende del negocio y de la semántica de la flag. [T22, p. 29]
260. 199 fallas en 100.000 requests: 0,199 %, por debajo del umbral de pausa de 1 %, aunque express falla 100 veces más. Hay que comparar siempre por variante. [T22, p. 34] Se miden la salud por variante, el negocio por exposición, el evaluador y la propagación; no van como label el tenant, el usuario, el trace ID ni todas las combinaciones de flags. [T22, p. 33]
261. Exposiciones reales y configuración adoptada; ventana y volumen mínimos; errores y latencia frente a classic sin degradación; y conversión revisada. [T22, p. 36] El rollout limita riesgo y pregunta si es seguro avanzar; el A/B test mide un efecto, con asignación aleatoria y estable, y con hipótesis, métrica primaria y duración fijadas antes. [T22, p. 37]
262. Ambas variantes; el default, el provider caído y la configuración obsoleta; las combinaciones que comparten estado o contratos; versiones anteriores de servicios y consumidores; reintentos, eventos atrasados y cambio a mitad del flujo; y la reversión después de escribir datos nuevos, con un evaluador controlable. [T22, p. 38] Ciclo: crear, rollout, 100 % estable, retirar código, archivar y contraer datos. Si nadie la retira, la flag y su camino viejo quedan como deuda técnica. [T22, p. 39]
263. Silos de datos, falta de contexto, requests distribuidas difíciles de rastrear e infraestructura efímera. [T23, p. 4] Observabilidad es qué tan bien pueden inferirse los estados internos de un sistema a partir de sus salidas externas, que en microservicios son logs, métricas y trazas. [T23, p. 5]
264. Logs: qué pasó. Métricas: cuándo y cuánto. Tracing: dónde. [T23, p. 6] En un diagnóstico, las métricas detectan el síntoma, las trazas muestran la cadena causal y los logs dan el detalle; sin los tres, el diagnóstico es parcial. [T23, p. 28]
265. `timestamp` en UTC, `level`, `service`, `traceId`, un `message` de texto fijo y `data` con las variables de negocio. [T23, p. 9] Un wide event es un solo evento por operación con todo el contexto: evita los joins entre servicios y se filtra por cualquier dimensión. [T23, p. 10] [T23, p. 11]
266. Counter: solo sube y se mira con `rate()`. Gauge: sube y baja. Histogram: cuenta valores por rangos y da los percentiles. [T23, p. 12] Cada combinación de valores de labels es una serie: `userId` genera una por usuario y Prometheus se queda sin memoria; ese detalle va en el wide event o en la traza. [T23, p. 13]
267. Una CPU que oscila entre 0 % y 100 % promedia 50 %: el dashboard queda en verde con el servicio saturado la mitad del tiempo. [T23, p. 15] P50 es la mediana, P95 el valor que cubre al 95 % de los requests y P99 el peor 1 %, que con un millón de usuarios diarios son 10.000 personas. [T23, p. 16]
268. RED, por servicio y como lo ve el usuario: rate, errors y duration. USE, por recurso: utilization, saturation y errors. La alerta salta por RED y se investiga con USE. [T23, p. 17]
269. Por pull: va a buscar `/metrics` a cada servicio cada `scrape_interval`; el servicio no sabe que existe. Grafana visualiza con PromQL y Alertmanager notifica a quien está de guardia. [T23, p. 18] [T23, p. 19] Más frecuencia captura más picos y cuesta más; entre 15 y 30 s alcanza para tendencias y alertas, y los picos de milisegundos se ven en las trazas. [T23, p. 14]
270. Trace: la vista de punta a punta de una solicitud. Span: una unidad de trabajo dentro de un servicio, con su parent. [T23, p. 20] En el waterfall el ancho es la duración, la profundidad son las dependencias y el span más ancho marca el cuello de botella. [T23, p. 21] `traceparent` lleva versión, trace-id, parent-id y flags; un servicio sin instrumentar corta la traza. [T23, p. 22]
271. Instrumentar una vez y cambiar de backend por configuración, propagación automática del contexto, SDK más auto-agente y el protocolo OTLP. [T23, p. 24] La automática da la estructura sin tocar el código; la manual agrega el contexto de negocio. [T23, p. 25] El Collector recibe, procesa (sampling, enriquecimiento, filtrado, batching) y exporta a varios backends, y desacopla a los servicios del backend. [T23, p. 26]
272. Alerta → métrica (¿qué servicio?) → trace (¿qué request?) → span (¿qué operación?) → log (¿qué error?) → causa raíz. [T23, p. 30] El exemplar es el trace ID que Prometheus guarda junto a una muestra de la métrica, y permite saltar de la alerta a la traza. [T23, p. 29]
273. NATS en Node no tiene auto-instrumentación: sin inyectar y extraer el contexto en los headers del mensaje quedan tres trazas sueltas. [T23, p. 33] El `traceId` identifica una ejecución técnica, nace con el request, viaja en `traceparent` y lo genera OpenTelemetry; el `correlationid` identifica una operación de negocio, vive días, viaja en el CloudEvent y lo genera el dominio. Un job no tiene request de origen y arranca otra traza, que se une por el `correlationid` como atributo del span o con un span link. [T23, p. 34]
274. El cliente conoce cada servicio, así que un cambio interno (separar Seguimiento) rompe una app que ya está instalada. [T24, p. 3] El gateway decide a dónde va el request, quién es, cuánto puede pedir y qué queda registrado. [T24, p. 4]
275. `Gateway` y `HTTPRoute` cubren el ruteo; identidad, límites y registro son políticas que agrega cada implementación. [T24, p. 5] La lámina deja la segunda pregunta abierta. [T24, p. 7] *Conocimiento general (Gateway API):* gana el prefijo más largo, `/envios/estado`.
276. Strangler Fig: lo ya extraído va al servicio nuevo y el resto, al monolito. [T24, p. 8] Traducción: HTTP + JSON afuera y gRPC adentro, sin que ninguno cambie de protocolo. [T24, p. 9] Agregación: una respuesta a partir de varias llamadas; si un servicio no responde, la pantalla entera espera. [T24, p. 10] *Inferencia:* se mitiga con un timeout por llamada y una respuesta parcial.
277. Un gateway por tipo de cliente (móvil, web, socios); cada uno es del equipo que hace ese cliente. [T24, p. 11]
278. Autenticación: quién es. Autorización: si puede hacer esto. Sin token, 401; con token pero sin permiso, 403. [T24, p. 13] El token lo emite el proveedor de identidad cuando el usuario inicia sesión con OAuth 2.0 + OIDC; el gateway solo ve un request con un token. [T24, p. 14]
279. Encabezado, contenido y firma; los claims del ejemplo son `iss`, `sub`, `scope` y `exp`. El gateway verifica la firma con la clave pública del proveedor (JWKS); si cambia un claim, la firma deja de coincidir y responde 401. [T24, p. 15]
280. La lámina deja la pregunta abierta. [T24, p. 16] *Conocimiento general:* la validación local lo acepta hasta que vence, porque no consulta al proveedor; la introspección lo rechaza desde la revocación, a cambio de una llamada al proveedor por request.
281. Descarta el header externo y escribe el propio a partir del token verificado. [T24, p. 17] [T24, p. 27] [T22, p. 27] *Inferencia:* a Ana la rechaza Pedidos, con el dato de quién es el dueño del pedido; el gateway solo autoriza por scope. [T24, p. 18]
282. *Inferencia:* hasta 300 por minuto, porque cada réplica lleva su propia cuenta. [T24, p. 20] Se corrige con un contador compartido que vive fuera del gateway; si deja de responder hay que elegir entre dejar pasar todo o rechazar todo, y la lámina no lo resuelve. [T24, p. 24]
283. Porque no guarda estado propio: el contador vive afuera. [T24, p. 24] *Inferencia:* solo el límite de 1 MB; las otras dos necesitan datos del negocio y van en un servicio. [T24, p. 25] No ve el tráfico este-oeste, entre servicios. [T24, p. 26]
284. ¿Quién llama?, ¿va cifrado? y ¿cuántos reintentos? [T25, p. 2] Con librerías, la misma política se implementa una vez por lenguaje y queda con valores distintos en cada servicio. [T25, p. 3]
285. Un segundo contenedor del Pod que se queda con la identidad, el TLS y los reintentos. Las reglas de red del Pod desvían la llamada y cada llamada pasa por dos proxies. [T25, p. 4] [T25, p. 5] *Inferencia:* el código no cambia.
286. El control plane distribuye la política a los proxies y firma sus certificados; el data plane son los proxies. [T25, p. 5] [T25, p. 6] [T25, p. 8] *Inferencia:* la llamada sigue funcionando con la última política recibida; lo que no se puede es cambiar políticas ni, si la caída dura, renovar certificados.
287. Que cada extremo sepa con quién habla, porque los dos presentan un certificado; lo hacen los proxies y la app sigue hablando HTTP. [T25, p. 7] El SPIFFE ID nombra al workload por dominio de confianza, namespace y ServiceAccount; como en el JWT, un emisor firma y el receptor verifica, pero el sujeto es un servicio. [T25, p. 8]
288. *Inferencia:* con mTLS solo, Pedidos le cree: el certificado prueba quién llama, no qué puede hacer. [T25, p. 9] La `AuthorizationPolicy` deja llamar solo al gateway, y Envíos recibe 403. [T25, p. 10]
289. Métricas RED por llamada y spans emitidos por el proxy. [T25, p. 11] [T25, p. 12] La app tiene que copiar `traceparent` para que la traza no se corte. [T25, p. 21]
290. *Inferencia:* idempotencia, porque el timeout no prueba que el primer intento no se haya procesado. [T25, p. 13] [T11, p. 18] El retry budget limita los reintentos a una proporción del tráfico; pasado el tope, el proxy devuelve el error. Sin tope: 100 × 3 × 3 × 3 = 2700. Con 20 %: 120, 144 y 173. [T25, p. 14]
291. El proxy que llama deja de enviarle por un tiempo a un Pod que acumula errores y después vuelve a probar. [T25, p. 15] *Inferencia:* lo decide el llamador por lo que observa, aunque el Pod figure `Ready`; el máximo evita quedarse sin destinos si fallan todos.
292. Sidecar: L4 y L7 en el proxy de cada Pod. Ambient: L4 en un `ztunnel` por nodo y L7 en un waypoint aparte y opcional. Cilium: L4 con eBPF en el kernel y L7 con un Envoy por nodo. [T25, p. 19] *Inferencia:* necesitan waypoint las políticas que interpretan HTTP (timeout y reintentos por request, métricas RED, spans); para actualizar un sidecar hay que reiniciar todos los Pods. [T25, p. 16] [T25, p. 18]
293. Se paga un proxy en cada llamada (latencia y memoria), un control plane que operar y actualizar, y una capa más donde buscar cuando algo falla. Se obtiene mTLS sin tocar el código, identidad y política por servicio, y la misma política de reintentos en todos. [T25, p. 20]

</details>

294. Porque los cuatro integrantes eran Maintainers: cualquiera, incluido el autor del cambio, podía ejecutar el job, y no había environment protegido ni reglas de aprobación. [D03] Alternativas: proteger el environment `prod` con aprobadores explícitos, o que `promover-prod` abra un MR al estado deseado de prod que integre otra persona. [D03] `E06` acepta la aprobación del pull request, la protección del environment o la sincronización manual en ArgoCD. [E06]
295. Porque el kubelet decide reiniciar según la liveness: si siempre responde OK, un proceso colgado nunca se reinicia. [E06] [T18, p. 15] La cátedra sugirió medir el lag del event loop contra un umbral, el heartbeat reciente del consumidor y del relay del outbox, o que el servidor gRPC siga sirviendo. [D03] La liveness no tiene que fallar porque PostgreSQL o el broker estén caídos: eso es tarea de la readiness. [T18, p. 17]
296. El client ID es un identificador público, que viaja en la URL de autorización. *Inferencia, conocimiento general.* El secret, en cambio, autentica a la aplicación frente a Google: si se compromete el de `dev`, queda comprometido el de `prod`. [D03] Además, `docs/cicd.md` afirmaba que cada ambiente tenía su cliente, y eso también se marcó. [D03]
297. Porque el job de aprobación solo protege el overlay de `prod` (los digests). `gitops/workloads`, `base` y `entrada` son comunes y llegan a los tres ambientes con el merge a `main`. [D03] Se puede versionar la base por ambiente o fijar el `targetRevision` de prod a un commit o tag ya promovido. [D03]
298. El deploy token de solo lectura que usa ArgoCD para leer el repositorio, que venció el 2026-11-20: sin él, ArgoCD no puede sincronizar. Y los refresh tokens de Google de las cuentas de prueba, que duran 7 días: sin ellos, la suite de integración de `staging` falla y bloquea la promoción. [D03]
299. Que cada escenario de aceptación de `E01` se considera cubierto solo si su IT existe y pasa en el pipeline; un test pendiente equivale a un escenario no demostrado. [E06] IT-20, IT-24, IT-25, IT-28, IT-35 e IT-36 necesitan inyectar fallas, pausar operaciones o sembrar estados inconsistentes sin dar al CI credenciales de administración del cluster. [E06]

## Casos para practicar sin respuesta única

- Tomá los manifiestos de la guía de AKS como punto de partida para el Álbum: listá qué `HTTPRoute` agregarías al Gateway, qué servicios necesitan base y cómo cambia la NetworkPolicy de Postgres, qué Secrets crea el script del grupo y qué seed necesitás para que la demo sobreviva a un `down.sh`. [G09] [E01, p. 6]
- Para el despliegue del Álbum en un clúster, decidí qué Services son ClusterIP o headless, cuál es el único punto de entrada (Ingress o Gateway API), qué NetworkPolicies de *default deny + allow* hacen falta, qué datos van en PVC con qué modo de acceso y reclaim policy, y cómo llegan configuración y credenciales sin versionarlas. [E01, p. 6] [T17, p. 17] [T17, p. 23] [T17, p. 26] [T17, p. 31] [T17, p. 52]
- Una fintech entrega semanalmente con seis equipos; un sistema interno cambia dos veces al año con un equipo de cuatro; un catálogo recibe cien veces más tráfico que checkout; una startup de tres personas construye su MVP. Elegí arquitectura y justificá por equipo, releases y escalado. [T02, p. 30]
- Elegí `PagoRechazado` o `EntregaFallida`, escribí entre tres y cinco eventos y uno o dos comandos, asigná publicador y reactor, y marcá al menos un hotspot. [T03, p. 52]
- Dibujá el proceso y las vistas del kernel para un contenedor que usa PID, mount, user y net namespaces, un cgroup de memoria y OverlayFS. Señalá qué sigue compartiendo con el host. [T04, p. 14] [T04, p. 18] [T04, p. 19]
- Para Huella, elegí cuatro términos conflictivos y proponé cómo se traducen en el borde, sin resolver todavía los bounded contexts. [P02, p. 6]
- Construí una tabla de estados para oferta, operación de intercambio y copia que cubra aceptación, timeout, cancelación y vencimiento sin violar los invariantes. [E01, p. 3] [E01, p. 4] [E01, p. 6]
- Diagnosticá este caso: dos contenedores comparten una red propia, el nombre resuelve, pero el cliente recibe `connection refused` y desde el host tampoco funciona el puerto publicado. Ordená las comprobaciones de red, escucha y DNAT. [T08, p. 22] [T08, p. 32]
- Un endpoint de reserva de entradas corre en tres instancias y recibe reintentos del gateway. Diseñá la protección contra sobreventa y duplicados: qué estado va en Valkey, qué va en la base, dónde queda la frontera atómica y qué pasa si una instancia pausa más que el TTL. [T12, p. 3] [T12, p. 11] [T12, p. 20]
- Elegí estrategia de despliegue para el Álbum 2026 y justificá según riesgo, retrocompatibilidad entre versiones y costo de rollback. [T13, p. 43]
- Para el Álbum 2026, diseñá el flujo desde la apertura de un sobre hasta la actualización de progreso y retos: qué eventos se publican, con qué key, cómo se evitan efectos duplicados, qué pasa con eventos de progreso fuera de orden y cómo detectarías que la actualización se está demorando. [E01, p. 6] [T14, p. 35] [T14, p. 43] [T14, p. 50]
- Redactá el ADR del estilo de coordinación del intercambio con dos alternativas reales, saga orquestada y coreografiada, y mostrá cómo cambia la máquina de estados si la copia se aparta al publicar o al aceptar. [E05] [E03, p. 4] [E01, p. 2] [T11, p. 20] [T11, p. 37] [T14, p. 27]
- Para MegaShop, armá el plan de migración en una hoja: decidí si migrar y qué responderle al directorio, el primer servicio y su patrón, qué hacer con el cron nocturno, el dueño de `orders`, `inventory` y `users`, los estados intermedios del checkout sin transacción, el destino del SQL de BI y un roadmap de tres fases con riesgo, rollback y métrica. [T15, p. 50] [T15, p. 51] [T15, p. 52] [T15, p. 53]
- ¿Migrarían? Una startup de 6 devs buscando product–market fit; un e-commerce de 80 devs en 9 equipos con un release mensual y un Black Friday que tira el sitio; una liquidación de sueldos con 3 devs y cambios regulatorios dos veces por año. Justificá con las razones que alcanzan y los prerrequisitos. [T15, p. 12]
- Elegí los dos servicios reales del Álbum a partir de tu arquitectura de la segunda entrega, justificalos con los criterios de la consigna y armá la lista de cinco tests de integración que cuentan, con al menos uno de fallo o concurrencia y el mecanismo de control que necesita cada uno sin credenciales de cluster. [E06]
- Diseñá el repositorio de estado deseado del Álbum: base común y valores por ambiente, una `Application` por ambiente con su `syncPolicy`, el commit que promueve un digest de `staging` a `prod` y dónde entra la aprobación manual. Marcá qué campos debería ignorar ArgoCD si en el futuro se agrega un HPA. [E06] [T20, p. 13] [T20, p. 17]
- Pasá un servicio del Álbum por las seis preguntas de revisión de Kubernetes y por el checklist de production readiness, y escribí qué cambiarías en su código y en sus manifiestos. [T18, p. 31] [T21, p. 54]
- Resolvé el ejercicio de Black Friday: SLOs, qué es prescindible, patrones entre servicios, métrica y límites del autoscaling, topología multi-región, estrategia de datos y validación previa. [T21, p. 53]
- La tienda de pedidos tiene 4 servicios y 12 Pods. Decidí si adoptarías una malla: cuál de las tres preguntas (¿va cifrado?, ¿quién llama?, ¿cuántos reintentos?) lo justificaría, qué variante de data plane elegirías y qué costo operativo aceptás. [T25, p. 19] [T25, p. 20]
- El contador compartido del rate limit deja de responder. Elegí entre dejar pasar todo y rechazar todo para tres endpoints del Álbum (abrir un sobre, aceptar una oferta, consultar la colección) y justificá con el riesgo de cada uno. [T24, p. 24] [T22, p. 29]
- Repartí entre el gateway y los servicios estas reglas del Álbum: validar el token de Google, limitar requests por usuario, verificar que la copia ofrecida es del usuario, rechazar un body mal formado y registrar el `traceparent`. Aplicá el criterio "si necesita datos del negocio, va en un servicio". [T24, p. 25] [T24, p. 28] [E06]
- Liberá con una flag un cambio en la aceptación de ofertas del Álbum y respondé las cinco preguntas de cierre de la clase: qué medís en despliegue, activación y exposición; dónde evaluás; qué fijás en la operación; qué tres señales te hacen pausar; y qué verificás antes de retirar. [T22, p. 44]
- Armá la evidencia de observabilidad de un intercambio con compensación: qué spans esperás, qué atributos de negocio agregás, cómo cruza la traza la cola y cómo unís la compensación si la dispara un vencimiento. [T23, p. 32] [T23, p. 33] [T23, p. 34] [E03, p. 11]
