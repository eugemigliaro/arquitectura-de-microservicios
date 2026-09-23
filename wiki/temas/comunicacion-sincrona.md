# Comunicación síncrona

## Encuadre

En un microservicio, una llamada cruza la red y por eso incorpora latencia, fallos y particiones temporales. En la comunicación síncrona el cliente envía un request, queda esperando la respuesta y depende de que el servidor esté disponible; ese ciclo produce acoplamiento temporal. En la comunicación asíncrona el productor continúa y una respuesta, si existe, llega después. La elección es un trade-off, no una jerarquía absoluta. [T09, p. 4] [T09, p. 5] [T09, p. 7]

La comparación de la cátedra caracteriza el flujo síncrono como lineal y de respuesta inmediata, pero más difícil de escalar bajo carga; el asíncrono agrega broker y máquinas de estado o callbacks, libera al productor durante la espera y permite escalar workers. [T09, p. 6]

## REST, gRPC y GraphQL

| Opción | Contrato y transporte | Fortaleza | Uso sugerido por la cátedra |
|---|---|---|---|
| REST | JSON sobre HTTP; OpenAPI como documentación del contrato | Simplicidad, universalidad y tooling maduro | APIs públicas y externas [T09, p. 8] [T09, p. 17] |
| gRPC | Protocol Buffers sobre HTTP/2; el `.proto` genera código | Serialización compacta, multiplexing y streaming | Comunicación interna service-to-service [T09, p. 11] [T09, p. 17] |
| GraphQL | Queries JSON sobre HTTP y schema SDL tipado | El cliente pide exactamente los campos que necesita | BFF y frontends con datos complejos [T09, p. 14] [T09, p. 17] |

No corresponde adoptar un único protocolo para todos los bordes. La arquitectura propuesta combina REST para clientes externos, gRPC o ConnectRPC para baja latencia interna, colas para eventos y procesamiento asíncrono, y GraphQL para frontends complejos. [T09, p. 34] [T09, p. 35]

### Prácticas y fallos frecuentes

- En REST: versionar, paginar, hacer idempotentes las mutaciones, usar correctamente los estados HTTP y documentar errores. Deben evitarse los verbos en la URL, los errores escondidos dentro de `200 OK` y las APIs excesivamente conversadoras. [T09, p. 9] [T09, p. 10]
- En gRPC: centralizar y versionar los `.proto`, reservar los números de campos retirados, definir deadlines, usar `UNKNOWN = 0` en enums y propagar trazas como metadata. Omitir deadlines o copiar contratos entre repositorios favorece fallos en cascada y divergencia. [T09, p. 12] [T09, p. 13]
- En GraphQL: usar DataLoader contra N+1, limitar profundidad y complejidad, preferir persisted queries y mantener delgados los resolvers. Para CRUD simple o llamadas internas de alta performance puede resultar más complejo que REST o gRPC. [T09, p. 15] [T09, p. 16]

ConnectRPC conserva el contrato `.proto` y permite que un mismo servidor atienda Connect con HTTP/1.1 y JSON, gRPC con HTTP/2 y Protobuf, y gRPC-Web para browsers. Resuelve fricciones de acceso desde navegadores y depuración con `curl`, pero no reemplaza automáticamente a gRPC. [T09, p. 20] [T09, p. 36]

## Capacidad, sobrecarga y destino del trabajo

La demo distingue tasa de llegada `λ` y capacidad de terminación `μ`. Si `λ < μ`, la espera es transitoria; si `λ > μ`, el trabajo se acumula y el sistema debe rechazarlo, perderlo o hacerlo esperar. [T09, p. 22]

El experimento usa 20 clientes closed-loop, dos workers, medio core por worker, una capacidad aproximada de 4 requests/s, demanda cercana a 6,7 requests/s, timeout de 3 segundos y fallos tardíos del 20 %. [T09, p. 23]

| Variante | Resultado principal | Costo o límite |
|---|---|---|
| Síncrono naive | Solo 6 de 100 pedidos vuelven a tiempo; los demás vencen, aunque el servidor sigue procesándolos | Trabajo huérfano y pico de 93 MiB [T09, p. 25] [T09, p. 29] |
| Bulkhead | Acota ejecución y espera, y rechaza rápido con 503 | Protege el servicio, pero no completa lo rechazado [T09, p. 26] [T09, p. 27] |
| Bulkhead + retry | Recupera 41 pedidos a tiempo | Genera 311 intentos y mantiene ocupado al cliente; retry sin bulkhead empeora la sobrecarga [T09, p. 27] [T09, p. 29] |
| Cola | Completa 99 y envía 1 a DLQ | Solo 4 terminan antes de 3 s; agrega broker y eleva la latencia mediana a 17,5 s [T09, p. 29] |

La conclusión no es que la cola agregue capacidad: mantiene `μ` y transforma pérdida en latencia. El hardware adicional también puede resolver un pico, pero tiene costo sostenido; la decisión depende de qué debe ocurrir con el exceso de trabajo. [T09, p. 30] [T09, p. 31]

Ante fallos, el diseño combina reintentos con backoff y jitter, idempotencia, DLQ y circuit breaker. En síncrono decide principalmente el cliente y su reloj; con cola, el broker administra intentos. [T09, p. 32]

## Límites de la demostración

La cola de la demo está en memoria, no sobrevive a un crash y puede entregar duplicados. Además, si `λ > μ` de forma sostenida, cualquier backlog finito termina saturándose y reaparece el rechazo. Por eso la demo no prueba durabilidad ni garantías de entrega. [T09, p. 33]
