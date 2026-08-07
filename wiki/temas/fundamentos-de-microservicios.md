# Fundamentos de microservicios

## Idea central

Una arquitectura de microservicios organiza una aplicación como servicios que se desarrollan y ejecutan de manera independiente, coordinados mediante mecanismos de comunicación simples. La división busca obtener unidades funcionales independientes en lugar de replicar siempre una aplicación completa. [T02, p. 4] [T02, p. 5]

Un servicio es un componente reemplazable o actualizable en forma independiente y se ejecuta en un proceso separado. Esto lo distingue de una librería, que comparte proceso, memoria y CPU con su consumidor y cuyo reemplazo suele exigir reinicio o recompilación. [T02, p. 12]

La independencia se apoya en interfaces bien definidas: una implementación puede cambiar sin afectar a sus consumidores mientras conserve el contrato. También permite elegir tecnologías y mecanismos de persistencia por servicio, aunque esa libertad agrega costos operativos. [T02, p. 11] [T02, p. 18]

## Monolito, SOA y microservicios

| Dimensión | Monolito | SOA según la comparación de la cátedra | Microservicios |
|---|---|---|---|
| Unidad de ejecución | La funcionalidad vive en un único proceso | Servicios normalmente más grandes | Cada capacidad se ejecuta como servicio separado |
| Escalado | Se replica la aplicación completa | No especificado en la comparación | Se distribuyen y replican solo los servicios necesarios |
| Integración | Llamadas internas al proceso | Middleware centralizado o ESB | HTTP o mensajería directa y simple |
| Gobierno | Centralizado en la aplicación | Centralizado, asociado a SOAP y WS-* | Descentralizado por equipo |
| Datos | Habitualmente una base de la aplicación | Base compartida | Base propia por servicio |
| Equipos | Frecuentemente separados por especialidad | Un equipo puede atender varios servicios | Equipo multifuncional dueño de punta a punta |

La representación de ejecución y escalado proviene del diagrama monolito/microservicios. [T02, p. 8] La comparación con SOA es la propuesta explícita de la cátedra. [T02, p. 9]

## Propiedades organizativas y técnicas

- **Equipos alrededor de capacidades.** El modelo tradicional de equipos funcionales favorece arquitecturas en silos; la alternativa agrupa perfiles multifuncionales alrededor de un servicio o capacidad. [T02, p. 13] [T02, p. 14]
- **Producto en lugar de proyecto.** Un equipo estable asume diseño, construcción, operación y evolución del producto, resumido en la idea “you build it, you run it”. [T02, p. 15]
- **Datos descentralizados.** La base compartida del monolito se reemplaza por almacenamiento bajo control de cada servicio; compartir datos entre servicios vuelve a introducir acoplamiento. [T02, p. 16] [T02, p. 32]
- **Entrega automatizada.** El pipeline recorre compilación y pruebas funcionales, aceptación, integración, aceptación de usuario y rendimiento antes de producción. [T02, p. 17]

## Beneficios y costos

Los beneficios señalados son fronteras modulares fuertes, despliegue independiente y diversidad tecnológica. Los costos inherentes son la distribución, la consistencia eventual y la complejidad operativa. [T02, p. 18]

Los casos de adopción presentados vinculan los beneficios con escalado y disponibilidad independientes, incorporación de servicios sin reconfiguración masiva, autonomía de equipos, iniciativas paralelas y aislamiento de fallos. [T02, p. 6] [T02, p. 7]

## Cuándo tiene sentido

La justificación debe ser valor para el negocio, no la popularidad de la arquitectura. Son variables relevantes la frecuencia de releases, la necesidad de escalar partes distintas, la distribución del negocio y el tamaño o autonomía de los equipos. [T02, p. 29] [T02, p. 30] [T02, p. 31]

**Inferencia de estudio:** una heurística de decisión basada en esos criterios —no una respuesta oficial única para los casos— es:

- muchos equipos autónomos, cambios frecuentes o cargas muy desiguales favorecen evaluar microservicios;
- un equipo pequeño, pocos cambios y una carga uniforme favorecen comenzar con un monolito;
- el resultado no es automático: debe justificarse contra el contexto del negocio. [T02, p. 30]

## Señales de alerta

- Separar despliegues sin separar datos ni diseño produce un **monolito distribuido**. [T02, p. 32]
- Los sistemas en tiempo real o con coordinación crítica son candidatos problemáticos para esta arquitectura. [T02, p. 32]
- La falta de coordinación de contratos y velocidades de entrega entre equipos elimina parte de la ventaja del despliegue independiente. [T02, p. 33] [T02, p. 34]
- Cortar por tabla o entidad traslada el esquema relacional a la red y no descubre límites reales del dominio. [T03, p. 6] [T03, p. 7]

Para profundizar en cómo definir esos límites, seguí con [Diseño de servicios](diseno-de-servicios.md) y [DDD estratégico](ddd-estrategico.md).
