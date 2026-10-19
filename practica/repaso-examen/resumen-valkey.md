# Resumen de clase: Estado distribuido y patrones con Valkey

Repaso para Francisco · Segundo parcial · Arquitectura de Microservicios

Fuente: **T12**, [Estado distribuido/Patterns — Patrones con Valkey](../../material/catedra/teoria/Estado%20distribuido_Patterns%20-%20Slidev.pdf). Síntesis del material oficial; las aclaraciones inferidas se identifican explícitamente.

## 1. El problema: varias instancias, un estado compartido

Cada instancia de un servicio tiene memoria propia. Un mutex local solo coordina threads del mismo proceso: no impide que otra instancia acceda simultáneamente al recurso. Dos instancias pueden leer «disponible» y confirmar la misma reserva. La secuencia **consultar, decidir y actualizar** debe preservar la invariante como una operación atómica. Que `GET` y `DECR` sean individualmente atómicos no vuelve atómica su combinación. [T12, p. 2] [T12, p. 3] [T12, p. 8] [T12, p. 9]

Hay que distinguir estado **local y efímero**, **compartido de coordinación** (cupos, leases), **durable de negocio** (pagos, inventario) y **derivado** (caché, rankings). Los datos de negocio requieren persistencia y una fuente de verdad; los derivados deben poder reconstruirse. Valkey es un motor de datos en memoria con estructuras útiles para estos patrones. [T12, p. 5] [T12, p. 7]

## 2. Lock distribuido: una instancia a la vez

Sirve para exclusión mutua sobre una sección crítica compartida. Adquisición:

```text
SET lock:recurso <owner-id> NX PX 30000
```

`NX` crea la clave solo si no existe; `PX` establece una vigencia de 30 segundos; `owner-id` identifica esa adquisición. Para liberar, **comparar propietario y borrar debe ser atómico**: borrar sin verificar podría eliminar un lock que ya adquirió otro proceso. [T12, p. 10]

**El TTL no basta para proteger las escrituras.** A puede pausarse hasta que expire su lease; B adquiere el recurso y escribe; A vuelve y todavía intenta escribir. Un **fencing token** creciente permite que el recurso rechace operaciones con tokens anteriores al mayor observado. El propietario protege la liberación; el fencing token protege el recurso frente a operaciones atrasadas. [T12, p. 11]

## 3. Semáforo: hasta N usuarios simultáneos

Controla una capacidad limitada, en lugar de permitir un único usuario. La lista contiene los permisos disponibles: `BLPOP` retira uno para adquirir el recurso y `LPUSH` lo devuelve al liberarlo. Si no hay permisos, el proceso espera; un timeout distinto de cero limita esa espera y puede devolver `nil`. [T12, p. 12] [T12, p. 13]

**Ejemplo didáctico:** tres permisos permiten que hasta tres procesos usen un recurso. **Inferencia:** si un proceso cae sin devolver su permiso, esta implementación necesita un mecanismo adicional de recuperación; el timeout de espera no devuelve el permiso perdido.

## 4. Cola de trabajo: repartir tareas

Una lista representa una cola. El productor agrega con `LPUSH` y los workers extraen con `BRPOP` o `RPOP`. Los workers compiten por las tareas; no reciben todos una copia. `BRPOP` espera si la cola está vacía y timeout `0` espera indefinidamente. [T12, p. 14]

**Inferencia:** extraer una tarea no demuestra que se completó. Si el worker cae después de retirarla, la lista simple no ofrece por sí sola ACK ni recuperación de pendientes. Para ese seguimiento, la clase presenta Streams y consumer groups. [T12, p. 32] [T12, p. 33]

## 5. Caché: evitar consultas repetidas

En **cache-aside**, la aplicación consulta Valkey con `GET`. Si hay **hit**, devuelve el valor. Si hay **miss**, consulta la base, guarda el resultado con una expiración —por ejemplo, `SET ... EX 60`— y lo devuelve. La clave puede representar una entidad o derivarse de una consulta. [T12, p. 15] [T12, p. 16]

La caché es estado derivado y admite obsolescencia controlada. **Inferencia:** un TTL limita cuánto puede permanecer una entrada, pero no garantiza que cada lectura refleje inmediatamente una actualización en la base. [T12, p. 5]

## 6. Idempotencia: reintentar sin repetir el efecto

Un timeout no permite saber si falló la operación o se perdió su respuesta. También puede haber reentrega cuando un consumidor cae después del commit y antes del ACK. Una clave de idempotencia identifica la operación: puede estar en `PROCESSING`, `COMPLETED` o `FAILED`; si ya terminó, se repite la respuesta guardada. La misma clave con parámetros distintos debe rechazarse. [T12, p. 18] [T12, p. 19]

`SADD` permite adquirir un identificador de forma atómica: devuelve `1` al primero y `0` a los siguientes. **Adquirir no equivale a completar**: hacen falta finalización y recuperación explícitas. [T12, p. 22] [T12, p. 23]

Registrar «procesado» antes del efecto puede perder el efecto si hay una caída; registrarlo después puede duplicarlo. El **inbox transaccional** guarda el identificador único y aplica el cambio de negocio en la misma transacción. Después del commit envía el ACK. Una reentrega encuentra el ID registrado y omite el cambio. [T12, p. 20] [T12, p. 21]

## 7. Pub/Sub, Streams y garantías

| Mecanismo | Idea clave |
|---|---|
| Pub/Sub | Difunde a suscriptores; no conserva mensajes. At-most-once: admite pérdida y no reentrega. [T12, p. 25] |
| Streams | Log ordenado que permite lectura por posición y relectura de entradas conservadas. [T12, p. 29] |
| Consumer group | Reparte trabajo entre consumidores y espera ACK. Los pendientes pueden recuperarse o reclamarse con `XAUTOCLAIM`. [T12, p. 32] [T12, p. 33] |

**ACK antes del efecto:** una caída puede perder el trabajo. **ACK después del commit:** una caída puede causar reentrega; se necesita idempotencia para evitar repetir el efecto. At-least-once admite duplicados; «exactly-once» exige precisar qué efecto y dentro de qué frontera se garantiza. [T12, p. 26] [T12, p. 27]

El orden del log **no garantiza el orden de finalización** de consumidores paralelos. Si una entidad requiere orden, sus mensajes deben compartir una ruta serializada. [T12, p. 28]

## 8. Otros usos y claves para el parcial

Los **sorted sets** sirven para rankings Top N; los **hashes** almacenan atributos bajo una clave, por ejemplo una sesión. [T12, p. 17] [T12, p. 24]

Para resolver un caso, identificá primero la necesidad: **lock → exclusión mutua; semáforo → capacidad N; cola → reparto de tareas; caché → lecturas; idempotencia → efecto único frente a reintentos.** Después preguntá qué ocurre si un proceso cae, expira su lease o se pierde el ACK. [T12, p. 4] [T12, p. 10] [T12, p. 12] [T12, p. 14] [T12, p. 16] [T12, p. 20] [T12, p. 27]
