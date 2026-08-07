# Diseño de servicios

## Bajo acoplamiento

El acoplamiento expresa cuánto depende un servicio de otro. Un diseño tiene menor acoplamiento cuando un cambio local puede publicarse sin obligar a modificar simultáneamente a todos los consumidores. [T02, p. 20]

Ejemplo: si `Pedidos` agrega `couponCode` a `OrderConfirmed`, un contrato rígido que exige recompilar y redesplegar a todos los consumidores genera alto acoplamiento. Si el campo es compatible y los consumidores pueden ignorar lo que no usan, el productor evoluciona de forma independiente. [T02, p. 21]

**Inferencia de estudio:** el bajo acoplamiento no significa ausencia de contrato; significa diseñar el contrato y su evolución para minimizar cambios coordinados.

## Alta cohesión

La cohesión mide cuánto se relacionan las responsabilidades de un servicio con un propósito único. Cuando el arreglo de una capacidad queda localizado en un módulo, el mantenimiento y el despliegue independiente resultan más simples. [T02, p. 22]

Una manera práctica de combinar ambos criterios es preguntar:

1. ¿Qué capacidad de negocio ofrece este contexto, más allá de exponer datos CRUD? [T02, p. 25]
2. ¿Los cambios de esa capacidad quedan mayormente dentro del servicio? [T02, p. 22]
3. ¿Su contrato permite que productor y consumidores evolucionen sin despliegues coordinados? [T02, p. 20] [T02, p. 21]

## Organización, límites y descomposición

La ley de Conway relaciona la estructura del sistema con la estructura de comunicación de la organización. Por eso, equipos funcionales en silos tienden a producir capas en silos, mientras que equipos multifuncionales alrededor de capacidades favorecen límites verticales. [T02, p. 13] [T02, p. 14] [T02, p. 24]

La cátedra presenta SOLID como apoyo conceptual: responsabilidad única, extensión sin rehacer, sustitución, segregación de interfaces e inversión de dependencias. [T02, p. 23] **Inferencia de estudio:** en este contexto, sus consecuencias prácticas más directas son una responsabilidad clara, contratos específicos y dependencia de abstracciones.

Los contextos pueden seguir subdividiéndose, pero un servicio debería concentrar pocos requerimientos funcionales relacionados. Además de los contextos de dominio, se mencionan descomposición por funcionalidad, madurez y patrones de acceso a datos; funcionalidad y madurez son especialmente comunes al refactorizar monolitos. [T02, p. 26] [T02, p. 27]

## Contratos entre equipos

El contrato define qué acuerdan proveedor y consumidor y cómo verifican ambos lados sus expectativas. La propuesta del material es usar contract testing —por ejemplo, Pact— para ejecutar las expectativas de consumidores en los pipelines sin depender de un entorno integrado end-to-end. [T02, p. 33]

Esto complementa, pero no reemplaza, la negociación de lenguaje y dependencias que se desarrolla en [DDD estratégico](ddd-estrategico.md).

## Anatomía genérica de un microservicio

El diagrama de la cátedra coloca dentro del límite de red estos módulos: Resources; Service Layer, Domain y Repositories; Gateways y HTTP Client; y Data Mappers/ORM. Los servicios y datastores externos quedan fuera del límite lógico o de red y se alcanzan mediante adaptadores. [T02, p. 36] [T02, p. 41]

| Módulo | Responsabilidad |
|---|---|
| Resources | Traducir solicitudes del protocolo externo a mensajes internos, validar parámetros y dar formato a respuestas. [T02, p. 37] |
| Domain Model | Implementar la lógica de negocio. [T02, p. 38] |
| Service Layer | Coordinar actividades complejas que requieren estado. [T02, p. 38] |
| Repositories | Gestionar el acceso del dominio a la persistencia. [T02, p. 38] |
| Gateways / HTTP Client | Encapsular comunicaciones salientes, sincronismo, autenticación y manejo de errores. [T02, p. 39] |
| Data Mappers / ORM | Traducir entre objetos del dominio y el datastore. [T02, p. 40] |

La anatomía es genérica: no obliga a usar un framework particular. [T02, p. 36] [T02, p. 41] **Inferencia de estudio:** el diagrama permite razonar por separado sobre protocolo, dominio, integraciones externas y persistencia.
