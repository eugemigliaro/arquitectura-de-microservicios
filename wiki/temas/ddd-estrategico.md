# DDD estratégico

## Por qué el esquema de datos no define servicios

Crear un servicio por tabla —`OrderService`, `CustomerService`, `ProductService`, etc.— parece separar componentes, pero conserva un modelo global y acopla a los equipos por datos. Un cambio de columna puede afectar varios servicios, las transacciones atraviesan la red y la arquitectura deja de reflejar el lenguaje del negocio. [T03, p. 6] [T03, p. 7]

Domain-Driven Design (DDD) prioriza el dominio y su lógica sobre la tecnología. Se usa cuando la complejidad relevante está en las reglas de negocio y busca alinear la implementación con el modelo mental compartido por expertos y desarrolladores. No es un framework ni un stack. [T03, p. 10]

DDD tiene dos escalas complementarias:

- **estratégica:** lenguaje ubicuo, dominio y subdominios, bounded contexts y context mapping;
- **táctica:** entidades, value objects, aggregates, comandos, eventos y repositorios. [T03, p. 11]

## Lenguaje ubicuo

El *Ubiquitous Language* es un vocabulario preciso construido en colaboración por negocio y desarrollo, usado en conversaciones, documentación y código. Debe representar el problema, no los nombres accidentales de una base o framework. [T03, p. 14]

El lenguaje es local a un bounded context. En el ejemplo de e-commerce, “Pedido” significa carrito confirmado en Ventas, reserva de stock en Operaciones, documento a cobrar en Facturación y unidad a despachar en Logística. Forzar una definición global corrompe las reglas particulares. [T03, p. 16] [T03, p. 17]

## Dominio, subdominio y bounded context

- **Domain:** área de conocimiento y reglas del negocio; no es código. [T03, p. 19]
- **Subdomain:** partición del problema, como ventas, inventario, cobros o envíos. [T03, p. 19] [T03, p. 22]
- **Bounded Context (BC):** partición de la solución dentro de la cual un modelo y su lenguaje son válidos y consistentes. Su integración con otros contextos es explícita. [T03, p. 21] [T03, p. 22]

Subdominio y BC no son sinónimos: uno recorta el problema y el otro organiza la solución. Pueden alinearse uno a uno, pero un subdominio también puede requerir varios BC o un BC abarcar más de una clase de subdominio. [T03, p. 22]

### Clasificación de subdominios

| Tipo | Rol | Estrategia orientativa |
|---|---|---|
| Core | Diferenciador competitivo | Máxima atención al modelo y diseño |
| Supporting | Necesario y específico, pero no diferenciador | Implementación propia cuando sostiene una capacidad importante |
| Generic | Problema ya resuelto por el mercado | Preferir SaaS o módulos simples antes que reinventarlo |

La clasificación y los ejemplos de pricing, inventario, logística, pagos, email y autenticación provienen de la propuesta de la cátedra. [T03, p. 20]

## Context mapping

El mapa de contextos muestra límites, dependencias y dirección de influencia entre BC; hace visibles los puntos que necesitan traducción. [T03, p. 24]

| Relación | Significado |
|---|---|
| Customer / Supplier | Un proveedor *upstream* ofrece capacidades y el cliente *downstream* depende de ellas; negocian prioridades e interfaces. [T03, p. 25] |
| Conformist | El downstream adopta el modelo del upstream sin traducirlo; reduce costo pero acepta acoplamiento semántico. [T03, p. 25] |
| Anticorruption Layer (ACL) | Una capa traduce entre lenguajes y protege el modelo propio frente a terceros, legacy u otros contextos. [T03, p. 26] |
| Open Host Service (OHS) | Un contexto expone capacidades estables a varios consumidores mediante un protocolo e interfaz definidos. [T03, p. 27] |
| Published Language | Contrato público de integración —por ejemplo OpenAPI, AsyncAPI, JSON Schema o eventos— distinto del modelo interno. [T03, p. 27] |
| Partnership | Dos equipos coordinan estrechamente por objetivos compartidos. [T03, p. 28] |
| Shared Kernel | Dos contextos comparten una parte del modelo y deben coordinar su evolución. [T03, p. 28] |
| Separate Ways | Los contextos aceptan duplicación y no se integran para conservar autonomía. [T03, p. 28] |

El mapa define **qué** se integra. HTTP/gRPC, eventos de dominio, mensajería o gateways resuelven **cómo** hacerlo. [T03, p. 29]

## Del contexto al despliegue

Un bounded context es una frontera de modelo y lenguaje; un microservicio es una unidad de proceso, despliegue, escala y fallo. El BC es un buen candidato a frontera de servicio, pero no existe una obligación de desplegar cada BC por separado: un monolito modular puede contener varios. [T03, p. 31]

Señales de un buen límite:

- el equipo puede cambiar el modelo interno sin negociar con todos;
- el lenguaje es estable dentro del contexto y se traduce en el borde;
- la consistencia transaccional fuerte cabe dentro de un aggregate y su contexto;
- la integración exterior admite latencia y fallos parciales. [T03, p. 32]

Antipatrones relacionados son los servicios por tabla, la base compartida, un shared kernel excesivo, un “PedidoService” global y adoptar sin ACL un modelo externo que contamina el core. [T03, p. 33]

La estructura interna y el descubrimiento colaborativo de estos límites continúan en [DDD táctico y EventStorming](ddd-tactico-y-eventstorming.md).
