# Patrón Saga Coreografiado: Un Enfoque para Orquestar Transacciones Distribuidas

## Fuente

El siguiente ejemplo fue adaptado de: [choreography-saga-pattern](https://github.com/tmvolpato/choreography-saga-pattern)

## Instrucciones

1. Ejecutar el siguiente comando en un terminal:

```bash
docker compose up --build
```

2. En un nuevo terminal, ejecutar el siguiente comando para realizar una venta. Observar en los logs del terminal donde inició el docker compose el flujo exitoso. Repetir hasta superar los 10 artículos vendidos para observar el flujo de error:

```bash
curl -X POST 'http://localhost:8081/api/sales' \
-H 'Content-Type: application/json' \
-d '{
    "productId": 1,
    "userId": 1,
    "value": 10.00,
    "quantity": 1
}'
```

## Introducción

La gestión de transacciones distribuidas en sistemas complejos es un desafío común que enfrentan muchas organizaciones. La necesidad de mantener la consistencia de los datos y garantizar la fiabilidad de las transacciones en entornos distribuidos ha llevado al desarrollo de varios enfoques. Uno de estos enfoques es el Patrón Saga Coreografiado, que proporciona una solución eficiente para orquestar transacciones distribuidas. En este texto técnico, discutiremos el concepto del Patrón Saga Coreografiado, sus características, así como sus fortalezas y debilidades.

## Definición del Patrón Saga Coreografiado

El Patrón Saga Coreografiado es un patrón de diseño utilizado para manejar transacciones distribuidas en sistemas distribuidos. Permite que un conjunto de servicios interactúen de manera coordinada para ejecutar transacciones complejas mientras mantienen la consistencia de los datos a lo largo del proceso. A diferencia del Patrón Saga Orquestado, donde hay un componente centralizado responsable de la orquestación de la transacción, el Patrón Saga Coreografiado implica la colaboración directa entre los servicios participantes.

## Características del Patrón Saga Coreografiado

1. **Autonomía de servicios**: Cada servicio involucrado en la transacción distribuida tiene su propia lógica de compensación, asegurando una mayor autonomía e independencia entre los servicios.
2. **Colaboración directa**: Los servicios colaboran directamente entre sí enviando eventos y mensajes para coordinar los pasos de la transacción. Esto elimina la necesidad de un componente de orquestación centralizado.
3. **Tolerancia a fallos**: El Patrón Saga Coreografiado está diseñado para manejar fallos en entornos distribuidos. Si algún servicio falla durante la transacción, los servicios participantes pueden ejecutar los pasos de compensación correspondientes para restaurar la consistencia del sistema.

## Ventajas del Patrón Saga Coreografiado

1. **Desacoplamiento**: El patrón promueve el desacoplamiento entre servicios, permitiéndoles evolucionar independientemente. Esto facilita el mantenimiento y la escalabilidad del sistema.
2. **Escalabilidad horizontal**: El Patrón Saga Coreografiado permite la adición de nuevos servicios participantes sin impactar directamente a los servicios existentes. Esto permite que el sistema se escale horizontalmente para manejar un mayor volumen de transacciones.
3. **Consistencia eventual**: Aunque el Patrón Saga Coreografiado no garantiza la consistencia inmediata de la transacción, proporciona consistencia eventual al ejecutar los pasos de compensación necesarios para corregir errores o fallos.

## Desventajas del Patrón Saga Coreografiado

1. **Complejidad de implementación**: La colaboración directa entre servicios puede llevar a una mayor complejidad de implementación, requiriendo una buena comprensión de la arquitectura del sistema y las interacciones entre servicios.
2. **Mensajería asíncrona**: La comunicación entre servicios se basa en el intercambio de mensajes asíncronos, lo que puede aumentar la complejidad de seguimiento y depuración de las transacciones.
3. **Racing conditions**: En escenarios de alta concurrencia, pueden surgir condiciones de carrera entre servicios, resultando en comportamientos inesperados.

## Flujo de Éxito

La imagen representa un flujo exitoso de una transacción distribuida. Esta imagen simboliza la finalización de los pasos de la transacción, donde todos los servicios involucrados han ejecutado sus operaciones correctamente, resultando en un estado del sistema consistente y fiable. La presencia de la imagen de éxito en el Patrón Saga Coreografiado es un indicador visual importante para verificar la finalización exitosa de una transacción distribuida.

![Flujo de éxito](./img/success_flow.png)

![Postman](./img/postman.png)

![Tabla de ventas](./img/case_success/sale_table.png)

![Tabla de inventarios](./img/case_success/inventories_table.png)

![Tabla de pagos](./img/case_success/payment_table.png)

![Tabla de ventas 2](./img/case_success/sale_table_2.png)

![Estado de venta](./img/case_success/sale_status.png)

## Flujo de Fallo

En el Patrón Saga Coreografiado, el flujo de fallo representa un escenario donde ocurrió un problema durante la ejecución de una transacción distribuida. Este flujo indica la interrupción o fallo de uno o más pasos de la transacción, señalando que la transacción no se completó con éxito. En el contexto del Patrón Saga Coreografiado, identificar el flujo de fallo es crucial para que los servicios participantes reconozcan cualquier problema o error que pueda haber surgido durante el proceso de transacción distribuida. Esto les permite tomar las medidas necesarias para abordar la situación y restaurar la consistencia del sistema.

![Flujo de fallo](./img/fault_flow.png)

### Rollback de inventario - Fuera de stock

![Inicio DB inventarios](./img/case_fail/start_db_inventories.png)

![Inicio DB usuario](./img/case_fail/start_db_user.png)

![Inicio ventas](./img/case_fail/start_sales.png)

![DB ventas cantidad 20](./img/case_fail/db_sales_quantity_20.png)

![DB usuario débito](./img/case_fail/db_user_debit.png)

![Inicio DB pago](./img/case_fail/start_db_payment.png)

![Débito DB inventarios](./img/case_fail/debit_db_inventories.png)

![Fuera de stock](./img/case_fail/out_of_stock.png)

![Error fuera de stock](./img/case_fail/error_out_of_stock.png)

![Venta cancelada](./img/case_fail/sale_canceled.png)

![Estado cancelado](./img/case_fail/status_canceled.png)


[URL del repositorio original redactada en esta copia: contenía una credencial de acceso embebida]
