# Primera entrega TP1: análisis DDD del Álbum 2026

## Propósito y fuente

La primera entrega del trabajo final toma como fuente de verdad la consigna funcional v1.1 (`E01`) y pide producir un modelo de dominio trazable antes de diseñar o implementar el sistema. El objetivo es acordar vocabulario, fronteras, modelo táctico, invariantes y hechos de negocio. [E02]

La entrega es grupal y consiste en un único archivo `docs/analisis-ddd.md`, escrito en castellano, con integrantes y versión de la consigna funcional en el encabezado. La extensión sugerida es de 8 a 15 páginas equivalentes, aunque se evalúa precisión y no volumen. [E02]

## Alcance estricto

La entrega incluye:

1. lenguaje ubicuo;
2. mapeo estratégico;
3. diseño táctico;
4. eventos de dominio;
5. supuestos y preguntas abiertas. [E02]

Quedan fuera de alcance el código, los esquemas de base de datos, endpoints y contratos de API, tecnologías, brokers, frameworks, infraestructura, CI/CD y despliegue. Un bounded context es una frontera de modelo y lenguaje, no una decisión anticipada sobre microservicios; mencionar tecnologías concretas resta puntos. [E02] [T03, p. 31]

## 1. Lenguaje ubicuo

El glosario mínimo de `E01` es solo el punto de partida. Deben incorporarse los conceptos usados pero no definidos formalmente alrededor de aceptaciones, retos, recompensas, canjes, rankings e inspección del operador. Los términos se organizan en entidades, acciones o procesos, estados y conceptos de soporte, usando definiciones de negocio sin jerga técnica. [E02]

Cuando una palabra tenga significados diferentes en distintos contextos, el documento debe conservar y ubicar cada acepción en vez de forzar una definición global. También debe evitar sinónimos para un mismo concepto. [E02]

## 2. Mapeo estratégico

Para cada bounded context se exige:

- nombre y lenguaje propios;
- clasificación como núcleo, soporte o genérico, con justificación;
- responsabilidades explícitas;
- aquello que queda fuera de su frontera. [E02]

El mapa debe indicar las relaciones entre contextos con patrones de DDD, la dirección upstream/downstream y la información que cruza cada frontera. También debe incluir un diagrama textual o ASCII. La separación de `E01` entre información correcta inmediatamente y proyecciones que pueden demorarse es una señal que debe analizarse al justificar las fronteras. [E02] [E01, p. 5]

Nombrar contextos acotados no define la futura partición en servicios. Esa decisión pertenece a una etapa posterior y no se evalúa en esta entrega. [E02]

## 3. Diseño táctico

Los contextos clasificados como núcleo deben desarrollarse en profundidad; los de soporte y genéricos deben aparecer, pero admiten un tratamiento breve. El modelo debe distinguir entidades por identidad persistente y objetos de valor por sus valores e inmutabilidad. Para cada elemento se piden atributos clave, comportamiento y reglas. [E02]

Cada agregado debe declarar raíz, miembros e invariantes. Las fórmulas de cantidades y todas las reglas de negocio de `E01` deben quedar asignadas explícitamente a un agregado. Si una regla no puede ser garantizada por un único agregado, el documento debe registrarlo y explicar qué coordinación la sostiene. [E02] [E01, p. 2] [E01, p. 5]

La frontera crítica que debe resolver el modelo es la diferencia entre poseer, ofrecer y apartar una copia, especialmente durante la publicación y aceptación de ofertas. Esa decisión condiciona el tamaño y las responsabilidades de los agregados. [E02] [E01, p. 3] [E01, p. 4]

Los servicios de dominio solo corresponden a operaciones de negocio que no pertenecen naturalmente a una entidad ni a un objeto de valor. Para cada servicio se debe justificar esa ubicación e indicar propósito, entradas y salida. [E02]

## 4. Eventos de dominio

Los eventos deben representar hechos de negocio ocurridos, nombrarse en pasado e indicar disparador, datos relevantes, consumidores y si cruzan una frontera. Se agrupan por el contexto que los publica. [E02]

El documento debe cerrar esta sección con el encadenamiento entre contextos de, como mínimo:

- apertura de un sobre;
- intercambio completado;
- intercambio no concretado. [E02]

No se piden comandos técnicos ni mensajes de infraestructura, y no todo cambio interno tiene que publicarse fuera del contexto. [E02]

## 5. Supuestos y preguntas abiertas

Toda ambigüedad de `E01` debe quedar visible. Si el equipo toma una decisión, la registra como supuesto con su justificación; si todavía no puede decidir, formula una pregunta abierta e indica qué parte del modelo depende de la respuesta. Detectar una ambigüedad explícitamente forma parte de la evaluación. [E02]

## Trazabilidad y control de cobertura

Todo término, contexto, agregado y evento debe poder rastrearse hasta un requisito de `E01` o contar con una justificación explícita. La consigna establece dos controles complementarios:

- un concepto sin origen requiere justificación;
- un requisito funcional sin representación en el modelo constituye una omisión. [E02]

Una matriz `RF/regla/escenario -> elemento del modelo` no se exige como formato, pero es una forma razonable de verificar cobertura antes de entregar. **Inferencia de trabajo:** ayuda a satisfacer el criterio de trazabilidad sin introducir decisiones técnicas.

## Rúbrica

| Criterio | Peso |
|---|---:|
| Diseño táctico | 30 % |
| Mapeo estratégico | 25 % |
| Lenguaje ubicuo | 20 % |
| Eventos de dominio | 15 % |
| Trazabilidad y rigor | 10 % |

La mayor concentración de puntaje está en justificar fronteras y asignar invariantes a agregados pequeños y consistentes. Restan puntos el vocabulario genérico, la tecnología prematura, un glosario que solo copie `E01`, reglas sin responsable y contextos sin frontera argumentada. [E02]

## Aspectos administrativos pendientes

La fuente no fija fecha límite ni canal: remite al Campus. Tampoco contiene los nombres de los integrantes, que no deben inventarse. Ambos datos deben completarse desde información del equipo o de la cátedra antes de entregar. [E02]
