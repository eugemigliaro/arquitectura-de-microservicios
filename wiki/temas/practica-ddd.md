# Práctica de DDD

Esta página ubica dos materiales de práctica distintos: una resolución de Pedidos del ciclo 2025 y un ejercicio abierto de Huella del ciclo 2026. La primera es un ejemplo resuelto; la segunda contiene la consigna y la rúbrica, pero no una solución oficial.

## Pedidos: resolución 2025

El caso reúne las perspectivas de Ventas, Operaciones, Facturación y Logística. El flujo principal crea un pedido, valida y reserva inventario, realiza el cobro, despacha y confirma la entrega. Los caminos de fallo deben liberar inventario, rechazar el cobro, devolver artículos y generar crédito según corresponda. [P01, p. 2] [P01, p. 3] [P01, p. 4]

### Fronteras propuestas en esa resolución

La lámina visual separa cinco “gestores”: Pedido, Inventario, Cobros, Envíos y Créditos. `Ítem` queda dentro de Pedido porque, en ese modelo, no existe independientemente. [P01, p. 7]

Esta nomenclatura corresponde a la resolución 2025 y no debe mezclarse silenciosamente con el ejemplo 2026 de `T03`, que presenta cuatro contextos candidatos y trata el crédito dentro de Facturación. Ambos sirven como decisiones de modelado situadas, no como una partición universal. [P01, p. 7] [T03, p. 39]

El lenguaje definido en la resolución distingue Pedido e Ítem, Inventario, Envío, Cobro y Crédito. Pedido conserva comprador, dirección, tarjeta e ítems; Envío representa el paquete en tránsito; Cobro representa el intento de cobrar; y Crédito, el saldo a favor del cliente. [P01, p. 8] [P01, p. 9]

### EventStorming de la resolución

En el camino feliz, `PedidoCreado` abre en paralelo la validación de inventario y la creación del cobro pendiente. `InventarioDisponible` y `CobroRealizado` habilitan `EnvioCreado`; luego ocurren `EnvioDespachado` y `EnvioEntregado`. Esta secuencia se verificó sobre el diagrama original. [P01, p. 10]

El diagrama completo agrega tres bifurcaciones principales:

- inventario insuficiente termina el flujo e informa a Ventas;
- cobro rechazado informa a Ventas y dispara la liberación de inventario;
- envío fallido informa a Ventas, devuelve inventario y solicita un crédito, que luego se crea. [P01, p. 11]

Las vistas por dominio hacen explícitos commands, eventos, datos que cruzan el borde, colas y sistemas externos. Inventario reacciona a pedido creado y a cancelaciones/devoluciones; Cobros integra el sistema de cobro y publica resultados; Envíos reacciona al cobro y coordina fallos; Créditos atiende solicitudes y notifica a Ventas. Esta lectura también se verificó visualmente porque el texto extraído no conserva la topología ni los colores del modelo. [P01, p. 12] [P01, p. 13] [P01, p. 14] [P01, p. 15]

## Huella: ejercicio abierto 2026

Huella es una red de clínicas veterinarias cuyo diferencial declarado es el seguimiento preventivo del Plan Huella; el dueño atribuye a ese plan el 70 % de la facturación recurrente. El ejercicio pide diseñar el dominio a partir de entrevistas contradictorias, sin convertir el resultado en un esquema de base de datos o despliegue. [P02, p. 1]

### Señales de frontera

El material introduce conflictos deliberados de lenguaje y reglas:

- “alta” puede significar alta de una persona/mascota, alta médica o activación de afiliación; [P02, p. 2] [P02, p. 3] [P02, p. 4]
- paciente, cliente, titular y animal cambian de significado entre Clínica, Recepción y Administración; [P02, p. 2] [P02, p. 3] [P02, p. 4]
- una vacuna es acto médico para Clínica, lote y trazabilidad para Depósito, y práctica tarifada para Administración; [P02, p. 3] [P02, p. 4]
- la historia clínica debe persistir entre dueño y sucursal, mientras el legacy mezcla humano y animal y contiene duplicados; [P02, p. 3] [P02, p. 4]
- VetSoft, la pasarela de pagos y la guardia Zoonos poseen modelos ajenos que requieren una decisión explícita de traducción o conformidad. [P02, p. 4] [P02, p. 5]

Estos son datos del ejercicio, no una solución. La consigna exige que el grupo haga visibles las traducciones en vez de producir un glosario global que borre los desacuerdos. [P02, p. 1] [P02, p. 6]

### Entregables del taller

| ID | Resultado esperado |
|---|---|
| E1 | Tabla de lenguaje conflictivo con al menos cuatro términos por área. |
| E2 | Subdominios clasificados como Core, Supporting o Generic y justificados. |
| E3 | Entre tres y cinco bounded contexts con responsabilidad, lenguaje y exclusiones. |
| E4 | Context map con ACL, OHS + Published Language, Customer/Supplier y Conformist o Separate Ways; además, señalar dónde evitar un Shared Kernel. |
| E5 | Modelo táctico dentro del Core: root, entidades, value objects, invariante y aquello que queda con consistencia eventual. |
| E6 | Por cada contexto, commands aceptados, eventos publicados/consumidos, eventos de fallo y distinción entre internos e integración. |
| E7 | EventStorming del flujo de vacunación con eventos, commands, policies, read model, sistema externo y hotspots. |

Los requisitos detallados de E1–E5 están en las páginas 6–7 y los de E6–E7 en las páginas 7–8. [P02, p. 6] [P02, p. 7] [P02, p. 8]

La rúbrica favorece un lenguaje local con traducción explícita, fronteras por modelo y regla, relaciones tipadas, aggregates justificados por invariantes y eventos de negocio distinguidos entre internos y públicos. Penaliza cortar por entidad, compartir base, imponer un `Paciente` global, contaminar el Core con el vocabulario de terceros y publicar eventos CRUD. [P02, p. 9] [P02, p. 10]

Para preparar el ejercicio, conviene repasar [DDD estratégico](ddd-estrategico.md) y [DDD táctico y EventStorming](ddd-tactico-y-eventstorming.md).

