# Requerimientos funcionales del Álbum 2026 (v1.1)

## Estado y alcance

La fuente incorporada es la versión 1.1 de la consigna funcional. Declara obligatorios y demostrables todos los requerimientos listados. La versión aclara especialmente que publicar una oferta no aparta copias, que una copia puede respaldar varias ofertas activas y que el apartado comienza al intentar concretar una aceptación. [E01, p. 1]

Esta síntesis sirve para localizar requisitos. Para implementar o verificar la entrega hay que consultar la redacción exacta de cada RF, RNF y escenario en `E01`.

La [consigna general del trabajo práctico](consigna-general-album-2026.md) (`E03`) relata el mismo producto, pero difiere en puntos centrales: aparta la copia al publicar, deja el login real fuera de alcance y separa lo que se implementa de lo que solo se diseña. [E03, p. 4] [E03, p. 8] Las diferencias están comparadas en esa página y registradas en [Dudas y conflictos](../dudas-y-conflictos.md).

La [primera entrega del TP1](primera-entrega-album-ddd.md) usa esta consigna como fuente de verdad, pero tiene un alcance distinto: pide exclusivamente análisis de dominio DDD y excluye implementación, arquitectura e infraestructura. [E02] La [segunda entrega](segunda-entrega-album-arquitectura.md) documenta la arquitectura. Su versión anterior ponía esta consigna en segundo lugar, detrás de la consigna general. [E04] La versión 1.2 la convierte en la fuente de verdad del producto y le exige a la arquitectura un lugar para cada RF-01 a RF-39. [E05]

## Lenguaje e invariantes de colección

- **Figurita:** tipo del catálogo, identificado por número dentro de la edición.
- **Copia:** unidad de una figurita poseída por un usuario.
- **Repetida:** copia extra cuando el usuario posee más de una.
- **Disponible:** copia poseída que no está apartada.
- **Apartada:** copia reservada temporalmente por un intento de intercambio en curso.
- **Ofrecible:** copia disponible y repetida; nunca la última copia de una figurita.
- **Oferta:** intención publicada de entregar copias a cambio de otras; no equivale a una reserva. [E01, p. 1]

Para cada usuario y figurita deben cumplirse simultáneamente:

```text
poseídas   = disponibles + apartadas
repetidas  = max(poseídas - 1, 0)
ofrecibles = max(disponibles - 1, 0)

poseídas >= 0
disponibles >= 0
apartadas >= 0
```

Una figurita cuenta para el álbum si `poseídas >= 1`. Varias ofertas pueden señalar la misma copia ofrecible sin crear copias ni reservas; recién una aceptación mueve temporalmente la copia de disponible a apartada. [E01, p. 2] La consigna general describe lo contrario: aparta la copia al publicar y la libera al cancelar o vencer la oferta. [E03, p. 4] [E03, p. 5]

## Actores y autenticación

La consigna distingue Coleccionista, Operador y Sistema automático. Coleccionista y Operador se autentican con Google; el rol de Operador se asigna explícitamente, es de inspección y no posee colección. El Sistema vence ofertas, otorga recompensas y actualiza progresos por tiempo o eventos. [E01, p. 2]

Los requisitos `RF-A01` a `RF-A05` exigen OAuth 2.0/OpenID Connect real, inicialización automática del álbum, autorización por propietario, control explícito del rol de Operador y rechazo sin efectos ante una sesión inválida. [E01, p. 2] [E01, p. 3] La consigna general, en cambio, deja fuera de alcance el login real y acepta un `userId` en header. [E03, p. 8]

## Mapa de requerimientos funcionales

| Grupo | IDs | Cobertura |
|---|---|---|
| Álbum y colección | `RF-01`–`RF-04` | Cantidades, porcentaje, faltantes y actualización inmediata sin pérdidas ni negativos. [E01, p. 3] |
| Sobres | `RF-05`–`RF-09` | Acreditación inicial, apertura propia, reintento sin duplicar, rechazo de apertura inválida y fuentes adicionales. [E01, p. 3] |
| Intercambios base | `RF-10`–`RF-22` | Ofertas 1:1, elegibilidad, apartado al aceptar, revalidación, liberación, carreras, cancelación/vencimiento y consulta posterior del resultado real. [E01, p. 3] [E01, p. 4] |
| Intercambios ampliados | `RF-23`–`RF-24` | Ofertas N:M y compatibilidad con la colección del usuario. [E01, p. 4] |
| Retos y promociones | `RF-25`–`RF-29` | Progreso por eventos, recompensas, deduplicación y concurrencia en el último uso de códigos. [E01, p. 4] |
| Resumen y rankings | `RF-30`–`RF-32` | Panorama de colección/actividad y rankings por avance e intercambios. [E01, p. 4] |
| Operador | `RF-33`–`RF-36` | Inspección de ofertas, reservas e inconsistencias, sin modificar datos de negocio. [E01, p. 4] |
| Sistema | `RF-37`–`RF-39` | Vencimiento automático, recompensas e invariantes ante fallos, reintentos y duplicados. [E01, p. 4] [E01, p. 5] |

### Semántica crítica del intercambio

1. Publicar valida que la copia exista, pertenezca al usuario y sea ofrecible; no la aparta. [E01, p. 3]
2. Aceptar vuelve a validar la elegibilidad e intenta apartar las copias necesarias. [E01, p. 3]
3. Antes de confirmar se revalidan oferta, copias y pertenencia de las reservas a esa operación. [E01, p. 3]
4. Si cualquier condición falla, no se transfiere nada y se liberan las reservas parciales. [E01, p. 3]
5. Ante aceptaciones, cancelaciones o vencimientos concurrentes debe existir un único resultado terminal, sin duplicación ni estado intermedio. [E01, p. 3] [E01, p. 4]
6. Si el cliente pierde la respuesta, una consulta posterior debe revelar el resultado real sin repetir la operación. [E01, p. 4]

## Consistencia percibida

Deben ser correctos inmediatamente el estado y contenido de los sobres, las cantidades de colección, el resultado de operar ofertas y el resultado del intercambio. Pueden actualizarse de forma eventual el porcentaje del álbum, los retos, rankings, notificaciones y actividad reciente. [E01, p. 5]

**Inferencia de diseño:** esta separación define qué invariantes no pueden depender de una proyección asíncrona y qué vistas sí toleran retraso. No prescribe por sí sola una tecnología ni una arquitectura concreta.

## Escenarios obligatorios

Los 16 escenarios cubren: apertura normal y reintentada; intercambio 1:1; ofertas concurrentes respaldadas por una copia; interrupciones con una o ambas copias apartadas; recuperación del resultado tras timeout; eventos duplicados y fuera de orden; proyecciones demoradas; aceptaciones concurrentes; carrera entre aceptación y cierre; apertura ajena o repetida; oferta inválida; último uso concurrente de un código; y solicitud sin sesión. [E01, p. 6]

Cada escenario debe poder reproducirse contra el ambiente entregado y demostrar ausencia de pérdidas, duplicados, negativos o reservas indefinidas. [E01, p. 6] [E01, p. 7]

La consigna general tiene 15 escenarios con otra numeración para el 4, deja el 15 como solo diseño y no incluye el escenario sin sesión. La versión anterior de la segunda entrega usaba esa numeración. [E03, p. 10] [E03, p. 11] [E04] La versión 1.2 conserva sus títulos, pero toma a esta consigna como fuente. [E05] Ver la [tabla de correspondencia](consigna-general-album-2026.md#escenarios-mínimos).

## Condiciones no funcionales y entrega

Los `RNF-01` a `RNF-06` son un gate: si no se cumplen, la entrega no se evalúa. Exigen un pipeline que compile y pruebe; despliegue a staging; ejecute integración como gate; y promueva a producción únicamente en AKS. El login debe usar credenciales reales de Google y no puede simularse con un `userId`. [E01, p. 6] La versión 1.2 de la segunda entrega atribuye a esta consigna RNF de "despliegue en contenedores" con Docker y docker compose, que no coinciden con el AKS de esta versión; el conflicto está en [Dudas y conflictos](../dudas-y-conflictos.md). [E05]

La entrega completa requiere:

- AKS desplegado por el pipeline, con al menos dos usuarios y datos para reproducir los 16 escenarios;
- README con pasos exactos y autenticación;
- cobertura observable de Coleccionista y Operador;
- evidencia ejecutada de invariantes bajo fallos, reintentos y duplicados;
- user stories y criterios propios derivados de la consigna;
- toda la funcionalidad implementada, no solo diseñada. [E01, p. 7]

La rúbrica pondera autenticación (10 %), requerimientos core (25 %), ampliados (15 %), reglas e invariantes (20 %), escenarios de aceptación (20 %), consistencia inmediata/eventual (5 %) y reproducibilidad (5 %). [E01, p. 7]

La exigencia de implementar todo y el gate no funcional presentan una incompatibilidad aparente con el umbral general del material introductorio; está documentada en [Dudas y conflictos](../dudas-y-conflictos.md). [T01, p. 24] [E01, p. 1] [E01, p. 6] [E01, p. 7] La consigna general agrega otra formulación, con partes solo de diseño y una rúbrica distinta. [E03, p. 8] [E03, p. 13]

> Nota operativa del repositorio: aunque `E01` exige credenciales reales, no se deben guardar client secrets ni otros secretos en este repositorio. Esta es una política del repositorio, no una afirmación del material académico.
