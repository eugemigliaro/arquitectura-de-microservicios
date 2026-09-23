# Testing de microservicios

## La pirámide y sus dimensiones

La pirámide presentada ordena unit, integration, component y end-to-end. Al subir aumenta el alcance, la fragilidad, la duración, el costo y la intervención manual, mientras disminuye la cantidad razonable de pruebas. La lámina representa exploratory por encima pero separado y aclara que la pirámide clásica solo cubre pruebas automatizadas de desarrollo: quedan fuera el testing exploratorio, la usabilidad, la performance y la seguridad. [T10, p. 3]

En un microservicio hay más de un eje posible para clasificar una prueba. El alcance distingue unit, integration, component y end-to-end; la intención funcional pregunta si se cumple el requerimiento y se opone a preocupaciones no funcionales como performance, seguridad o usabilidad. Por eso `component` y `functional` no son escalones mutuamente excluyentes: responden preguntas diferentes. [T10, p. 9]

## Alcances de prueba

| Tipo | Alcance y propósito | Costo o precaución |
|---|---|---|
| Unit | Una función o método; valida entradas y resultados y da feedback temprano | La dificultad para aislarlo puede revelar un módulo demasiado acoplado; mockear de más termina probando los mocks [T10, p. 5] [T10, p. 6] |
| Integration | La interacción mínima entre servicios o entre el servicio y un límite externo, como HTTP o persistencia | Conviene aislar el resto con stubs o mocks y verificar configuración, ida y vuelta, timeouts y transacciones [T10, p. 7] [T10, p. 8] |
| Component | El servicio entero aislado, con base efímera y vecinos reemplazados por stubs | Define alcance, no intención funcional [T10, p. 9] |
| Contract | La interfaz y los mensajes acordados entre consumidor y proveedor, verificables por separado | Simplifica el ambiente y soporta Consumer-Driven Contracts, pero no recorre el sistema completo [T10, p. 10] [T10, p. 11] |
| End-to-end | Toda o gran parte de la infraestructura y las integraciones desde la perspectiva del usuario | Es costoso, frágil y poco concurrente; se recomienda cubrir pocas historias importantes e independientes [T10, p. 12] [T10, p. 14] |

La figura de contract testing muestra que el consumidor prueba contra un provider mock y publica expectativas que un verificador ejecuta luego contra el proveedor real. Así ambos lados pueden validar el mismo contrato sin levantar todo el entorno. [T10, p. 11]

### Nota terminológica de clase

La cátedra también indicó oralmente que a `integration test` se lo llama `functional testing`. [N-2026-09-11-integration-test-functional-testing]

Para evitar una equivalencia ambigua, esta wiki conserva ambos contextos: la nota registra el uso terminológico de clase, mientras el material escrito define `integration` por alcance y `functional` por intención, y afirma que alcance e intención son preguntas distintas. [T10, p. 7] [T10, p. 9] La tensión está registrada en [Dudas y conflictos](../dudas-y-conflictos.md).

## Dobles y estructura de una prueba

Un **stub** devuelve una respuesta fija sin verificar interacciones; un **mock** además comprueba que una interacción ocurrió; un **fake** es una implementación real pero simplificada, como una base en memoria. Un unit test puede ser solitary, con todo lo externo simulado, o sociable, usando algunas dependencias reales. [T10, p. 6]

La estructura de cuatro fases es:

1. **Arrange:** prepara datos y contexto.
2. **Act:** ejecuta una acción.
3. **Assert:** compara el resultado con lo esperado.
4. **Clean-up:** restaura o destruye el ambiente para que la siguiente prueba sea independiente.

Un arrange desproporcionado sugiere demasiado alcance; un act con varios pasos prueba un flujo; y omitir el clean-up produce pruebas intermitentes. Si una prueba depende de que otra se ejecute antes, no son independientes. [T10, p. 16]

Given–When–Then expresa la misma estructura en lenguaje de negocio: estado previo, acción y resultado esperado. Su lectura business-facing permite que una persona de producto valide lo que codifica la prueba. [T10, p. 17]

## Cobertura, seguridad y test-first

La cobertura mide qué código fue ejecutado, no si se probaron los comportamientos correctos. Una cobertura del 100 % no garantiza pruebas efectivas; sigue siendo útil como detector de huecos, no como medida suficiente de calidad. [T10, p. 18] [T10, p. 19]

| Técnica | Qué inspecciona | Momento y ejemplo |
|---|---|---|
| Linter | Convenciones y patrones peligrosos | Al escribir; ESLint, Checkstyle o Hadolint [T10, p. 20] |
| SAST | Vulnerabilidades en el código propio | Commit; SonarQube o Semgrep [T10, p. 20] |
| SCA | Vulnerabilidades en dependencias | Build; Trivy [T10, p. 20] |
| DAST | La aplicación ejecutándose, atacada desde afuera | Ambiente de prueba; OWASP ZAP [T10, p. 20] |

Estas técnicas no verifican que el sistema haga lo correcto: complementan las pruebas funcionales. [T10, p. 20]

El ciclo test-first propuesto es escribir una prueba que falle, implementar lo mínimo para que pase y limpiar. El valor está en el requerimiento que codifica; generar simultáneamente implementación y prueba puede hacer que ambas compartan el mismo error. [T10, p. 21]

El ejemplo NextDate descompone “calcular el día siguiente” en casos de entrada inválida, meses de 30 y 31 días, febrero, años bisiestos y cambio de año. Ilustra que una historia pequeña todavía requiere explorar particiones y bordes del dominio. [T10, p. 22] [T10, p. 23]
