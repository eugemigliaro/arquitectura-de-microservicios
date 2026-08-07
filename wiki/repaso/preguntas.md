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

</details>

## Casos para practicar sin respuesta única

- Una fintech entrega semanalmente con seis equipos; un sistema interno cambia dos veces al año con un equipo de cuatro; un catálogo recibe cien veces más tráfico que checkout; una startup de tres personas construye su MVP. Elegí arquitectura y justificá por equipo, releases y escalado. [T02, p. 30]
- Elegí `PagoRechazado` o `EntregaFallida`, escribí entre tres y cinco eventos y uno o dos comandos, asigná publicador y reactor, y marcá al menos un hotspot. [T03, p. 52]
