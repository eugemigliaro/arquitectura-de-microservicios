# DDD moderno: lectura complementaria

## Autoridad y uso

`B01` es bibliografía externa, no material oficial de cátedra. Sus afirmaciones están por debajo de `T01`–`T05`, `P01`, `P02` y `E01` en la prioridad de esta wiki. La fuente incluye una bibliografía propia, pero esos enlaces no fueron incorporados ni verificados aquí; por eso esta página resume qué sostiene la guía y no convierte sus formulaciones fuertes en criterio oficial.

Cuando la guía presenta DDD como requisito obligatorio para microservicios o aproxima bounded context y frontera de despliegue, entra en tensión con la formulación más cauta de la cátedra. [B01, p. 1] [B01, p. 3] [T03, p. 31] El caso queda registrado en [Dudas y conflictos](../dudas-y-conflictos.md).

## Descubrimiento: Event Modeling

La guía distingue EventStorming, orientado a explorar el dominio, de Event Modeling, que organiza un storyboard con interfaz, commands, eventos y read models. Propone dos clases de paso:

- **State Change:** wireframe, command, domain event y referencia a hechos previos;
- **State View:** wireframe y read model que construye lo que ve el usuario. [B01, p. 7]

Según la fuente, este modelo ayuda a derivar pruebas Given–When–Then y puede usarse aunque la persistencia final sea relacional y no Event Sourcing. [B01, p. 7]

## Recomendaciones tácticas de la guía

La lectura complementaria desarrolla entidades y value objects en términos compatibles con la introducción oficial: identidad persistente para Entity e igualdad por atributos e inmutabilidad para Value Object. [B01, p. 7] [B01, p. 8]

Para aggregates, la guía recomienda:

1. mantenerlos pequeños;
2. modificar uno por transacción local;
3. referenciar otros aggregates por identidad;
4. coordinar aggregates distintos mediante eventos y consistencia eventual;
5. usar control optimista de concurrencia cuando corresponda. [B01, p. 8] [B01, p. 9]

También presenta Domain Services para políticas sin dueño natural, repositories limitados a aggregate roots y arquitectura hexagonal para orientar las dependencias hacia el dominio mediante ports y adapters. [B01, p. 9] [B01, p. 10]

Estas recomendaciones amplían el material oficial disponible; no reemplazan su definición de Aggregate como frontera de consistencia ni demuestran que toda implementación deba usar el mismo patrón de persistencia. [T03, p. 37]

## Eventos públicos y datos distribuidos

La guía separa eventos de dominio internos de eventos de integración versionados. Propone traducir explícitamente el evento interno a un contrato estable antes de publicarlo, en lugar de exponer cada cambio del modelo interno. [B01, p. 10]

Para consumidores que necesitan una copia local de datos upstream, describe *Event-Carried State Transfer*: el evento transporta el estado relevante, el consumidor actualiza su read model y evita una consulta síncrona posterior. El costo declarado es aceptar que esa copia puede quedar temporalmente desactualizada. [B01, p. 11]

## Dual write, outbox e idempotencia

El problema de *dual write* aparece cuando un servicio confirma el cambio de negocio en su base y después intenta publicar en un broker: la primera operación puede tener éxito y la segunda fallar. La guía propone Transactional Outbox, que persiste el estado y el evento pendiente en una misma transacción local. [B01, p. 11]

Un relay publica luego el outbox mediante polling o Change Data Capture. El primero simplifica la infraestructura a costa de latencia y consultas; CDC reduce el polling, pero agrega infraestructura y complejidad operativa. [B01, p. 12]

Como la publicación puede entregar duplicados, la guía exige consumidores idempotentes y describe un Inbox que registra `message_id` ya procesados. Su afirmación de que esto produce semántica “exactly once” necesita una precisión posterior en la unidad oficial de mensajería; por ahora se conserva como postura de la fuente externa. [B01, p. 13]

## Extensión analítica: Data Mesh

La última sección traslada límites de dominio al plano analítico y resume cuatro pilares de Data Mesh: propiedad descentralizada por dominio, datos como producto, plataforma autoservicio y gobierno computacional federado. [B01, p. 13] [B01, p. 14]

Esta extensión todavía no aparece desarrollada en el material oficial incorporado. Debe estudiarse como conocimiento complementario, no como contenido confirmado de evaluación.

