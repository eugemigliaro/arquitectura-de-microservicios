# Programa y organización

## Propósito

La materia busca dar herramientas para diseñar e implementar arquitecturas basadas en microservicios. El recorrido previsto incluye contenedores, balanceo y alta disponibilidad, CI/CD, observabilidad y seguridad. [T01, p. 6]

## Recorrido temático

1. **Fundamentos, diseño y construcción:** DDD y EventStorming; contenedores, imágenes y seguridad; Docker Compose y pruebas de interfaces; pipelines y versionado. [T01, p. 10] [T01, p. 17]
2. **Comunicación y sistemas distribuidos:** comunicación síncrona y asíncrona, arquitectura orientada a eventos, sagas, Kafka, CQRS, garantías de entrega, caché y estado distribuido. [T01, p. 11] [T01, p. 18]
3. **Orquestación con Kubernetes:** arquitectura del clúster, objetos fundamentales y service discovery; volúmenes, StatefulSets, Helm, ingress, secretos y ConfigMaps; GitOps y entrega continua. [T01, p. 12] [T01, p. 19]
4. **Operaciones y evolución:** logs, métricas y trazas; escalabilidad y resiliencia; service mesh, gestión de tráfico y mTLS; serverless y comparación entre contenedores y FaaS. [T01, p. 13] [T01, p. 20]

El programa también anticipa una aplicación serverless con AWS Lambda, incluida la organización y aislamiento de funciones y una coreografía de microservicios. [T01, p. 21]

## Recursos y bibliografía

Las guías prácticas reducidas se presentan como actividades recomendadas pero no evaluadas; también se mencionan lecturas complementarias y uso de IA para análisis y generación de código. [T01, p. 8]

La bibliografía principal mostrada en la presentación —verificada sobre la lámina original porque los títulos aparecen como portadas— comprende:

- *Building Microservices: Designing Fine-Grained Systems*, 2.ª edición, de Sam Newman.
- *Microservices from Day One*, de Cloves Carneiro Jr. y Tim Schmelmer.
- *Microservice Architecture: Aligning Principles, Practices, and Culture*, de Irakli Nadareishvili, Ronnie Mitra, Matt McLarty y Mike Amundsen. [T01, p. 14]

## Evaluación: versiones incompatibles

La presentación contiene dos esquemas que no pueden reconciliarse sin una confirmación posterior:

| Aspecto | Versión de la página 7 | Versión de las páginas 23–24 |
|---|---|---|
| Cursada | Parciales teórico-prácticos que cierran módulos; tres avances del trabajo final | Un parcial del módulo de diseño; entregas parciales; asistencia y participación |
| Ponderación | Parciales 60 %, entregas 40 % | Parcial 45 %, entregas 45 %, concepto 10 % |
| Aprobación de cursada | Cada parcial con 5 | Nota mínima 5 |
| Final | Trabajo integral, nota mínima 7 | Trabajo práctico final con al menos 70 % de requerimientos, nota mínima 5 |

Evidencia: [T01, p. 7] [T01, p. 23] [T01, p. 24]. Hasta confirmar cuál es vigente, esta wiki no debe usarse como autoridad administrativa; el caso queda registrado en [Dudas y conflictos](../dudas-y-conflictos.md).

La consigna específica del Álbum agrega otra diferencia relevante: la versión 1.1 declara obligatorios todos sus requerimientos, establece que los RNF son un gate sin el cual la entrega no se evalúa y exige que toda la funcionalidad sea demostrable. Esto no coincide de manera evidente con el umbral general de 70 % de requerimientos mencionado en `T01`; ambas versiones se conservan y el nuevo conflicto también queda registrado. [T01, p. 24] [E01, p. 1] [E01, p. 6] [E01, p. 7]

## Cómo usar este mapa

Los temas ya desarrollados son [Fundamentos de microservicios](fundamentos-de-microservicios.md), [Diseño de servicios](diseno-de-servicios.md), [DDD estratégico](ddd-estrategico.md), [DDD táctico y EventStorming](ddd-tactico-y-eventstorming.md), [Práctica de DDD](practica-ddd.md) y [Contenedores sobre Linux](contenedores-linux.md). La [entrega del Álbum 2026](entrega-album-2026.md) tiene su propia página de requisitos. El resto del recorrido funciona por ahora como índice del programa, no como contenido ya dictado o documentado en profundidad.
