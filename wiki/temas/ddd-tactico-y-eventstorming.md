# DDD táctico y EventStorming

## Bloques del modelo

### Entity

Objeto cuya identidad se conserva aunque cambien sus atributos. Dos instancias con el mismo identificador representan la misma entidad; ejemplos: un cliente, un pedido de venta o un envío. [T03, p. 35]

### Value Object

Objeto definido por sus atributos, sin identidad propia y normalmente inmutable. Dinero, dirección, cantidad y SKU son ejemplos; dos valores iguales son intercambiables. [T03, p. 36]

### Aggregate

Conjunto de entidades y value objects tratado como una unidad. Tiene una raíz que es el único punto de acceso externo y forma una frontera de consistencia transaccional: un comando modifica un aggregate a la vez. [T03, p. 37]

### Command y Domain Event

| | Command | Domain Event |
|---|---|---|
| Semántica | Intención en imperativo | Hecho de negocio ya ocurrido |
| Resultado | Puede ser rechazado | Se propaga porque ya sucedió |
| Dirección | Entra al aggregate o contexto | Sale hacia interesados internos o externos |
| Ejemplo | `AprobarPedido` | `InventarioReservado` |

El flujo básico es: comando → el aggregate aplica reglas → emite uno o más eventos de dominio. [T03, p. 38]

## Ejemplo por contextos

No existe un único objeto `Pedido` compartido. Cada contexto modela su aggregate y sus eventos:

| Contexto | Aggregate raíz candidato | Eventos de ejemplo |
|---|---|---|
| Ventas | `PedidoVenta` | `PedidoCreado` |
| Operaciones | `ReservaInventario` | `InventarioReservado`, `InventarioInsuficiente`, `InventarioLiberado` |
| Facturación | `Cobro` | `PagoConfirmado`, `PagoRechazado`, `CreditoEmitido` |
| Logística | `Envio` | `EnvioProgramado`, `PedidoDespachado`, `EntregaConfirmada`, `EntregaFallida` |

La asignación es el modelo de ejemplo de la clase, no una taxonomía universal. [T03, p. 39]

## EventStorming

EventStorming es una técnica colaborativa y visual que reúne a expertos de negocio y desarrollo. Parte de eventos de dominio expresados como hechos pasados, permite recorrer el proceso completo y puede profundizar hasta aggregates y bounded contexts. [T03, p. 41]

La notación mostrada usa colores o tarjetas diferentes para actor/persona, command, domain event, aggregate, policy, sistema externo, read model/vista y hotspot o pregunta abierta. Esta información se verificó sobre la lámina original porque el color es parte de la notación. [T03, p. 42]

## Taller de e-commerce

El relato recorre una compra desde Ventas a Operaciones, Facturación y Logística. El camino feliz crea el pedido, reserva inventario, cobra, programa el envío, despacha y confirma la entrega. Los fallos obligan a liberar inventario, emitir créditos o informar el rechazo a otros contextos. [T03, p. 46] [T03, p. 47]

### Secuencia de modelado usada en clase

1. **Recolectar lenguaje por área sin unificarlo:** “pedido”, “aprobado”, “pagado” y “rechazado” adquieren significados locales. [T03, p. 48]
2. **Proponer bounded contexts y aggregates:** Ventas/PedidoVenta, Inventario/ReservaInventario, Facturación/Cobro y Logística/Envio. La pasarela de pagos queda como sistema genérico externo detrás de una frontera de integración. [T03, p. 49]
3. **Ordenar los eventos del camino feliz:** `PedidoCreado` → `InventarioReservado` → `PagoConfirmado` → `EnvioProgramado` → `PedidoDespachado` → `EntregaConfirmada`. [T03, p. 50]
4. **Agregar policies de fallo y compensación:** por ejemplo, `PagoRechazado` dispara `LiberarInventario`, que produce `InventarioLiberado`; `EntregaFallida` puede requerir actualización de inventario y `CreditoEmitido`. [T03, p. 50]
5. **Dibujar el mapa:** los BC intercambian eventos asíncronos y Facturación usa una ACL frente al proveedor de cobros. [T03, p. 51]
6. **Marcar hotspots:** las preguntas todavía abiertas se conservan en el modelo en vez de esconderse; el ejercicio propone hacerlo para los caminos de fallo. [T03, p. 52]

### Criterio de salida

**Síntesis de estudio:** el resultado útil no es la cantidad de notas adhesivas, sino un lenguaje explícito por contexto, aggregates que contienen invariantes, eventos que narran hechos y un mapa de integraciones. El material resume esta relación como “Command in / Event out” y usa los BC como candidatos —no equivalencias obligatorias— a microservicios. [T03, p. 54] [T03, p. 31]

Los eventos y fronteras descubiertos preparan los temas siguientes: comunicación síncrona/asíncrona y sagas; Kafka y garantías; gateways y contratos; resiliencia, compensación y consistencia eventual. [T03, p. 56]

## Materiales de práctica relacionados

La resolución 2025 de Pedidos contiene diagramas de camino feliz, fallos y detalle por dominio; su partición separa Créditos de Cobros y por eso no debe reemplazar silenciosamente el modelo 2026 de esta página. [P01, p. 7] [P01, p. 10] [P01, p. 11]

El ejercicio Huella no trae solución: exige detectar lenguaje conflictivo, clasificar subdominios, proponer bounded contexts, dibujar un context map, modelar el Core y construir commands, eventos y un EventStorming mínimo. [P02, p. 6] [P02, p. 7] [P02, p. 8] El detalle de ambos materiales está en [Práctica de DDD](practica-ddd.md).
