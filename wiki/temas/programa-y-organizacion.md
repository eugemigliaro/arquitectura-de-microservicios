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

La [consigna general del trabajo práctico](consigna-general-album-2026.md) suma una tercera formulación. Separa lo que se implementa de lo que solo se diseña, trata CI/CD con despliegue en AKS como condición previa y usa una rúbrica propia. [E03, p. 8] [E03, p. 13] Para el Álbum, un anuncio de la cátedra fijó que ante la duda rige la consigna funcional. [N-2026-10-04-criterio-consignas-tp-album] El régimen general de evaluación sigue sin confirmar.

## Cómo usar este mapa

Los temas ya desarrollados son [Fundamentos de microservicios](fundamentos-de-microservicios.md), [Diseño de servicios](diseno-de-servicios.md), [DDD estratégico](ddd-estrategico.md), [DDD táctico y EventStorming](ddd-tactico-y-eventstorming.md), [Práctica de DDD](practica-ddd.md), [Contenedores sobre Linux](contenedores-linux.md), [Comunicación síncrona](comunicacion-sincrona.md), [Comunicación asincrónica y Kafka](comunicacion-asincrona-y-kafka.md), [Testing de microservicios](testing.md), [Sistemas distribuidos](sistemas-distribuidos.md), [Estado distribuido](estado-distribuido.md), [DevOps y CI/CD](devops-y-ci-cd.md), [Del monolito a microservicios](del-monolito-a-microservicios.md), [Kubernetes — Parte 1](kubernetes-parte-1.md), [Kubernetes — Parte 2](kubernetes-parte-2.md), [Programar microservicios para Kubernetes](programar-microservicios-para-kubernetes.md), [Helm](helm.md), [GitOps y entrega continua](gitops-y-entrega-continua.md), [Resiliencia y escalabilidad](resiliencia-y-escalabilidad.md), [Feature flagging](feature-flagging.md), [Observabilidad](observabilidad.md), [API Gateway](api-gateway.md) y [Service mesh](service-mesh.md). La clase de migración no figura como tal en el cronograma de `T01`, que menciona microservicios frente a monolitos en la primera clase. [T01, p. 10] [T15, p. 2] El trabajo final del Álbum 2026 tiene páginas propias: [consigna general](consigna-general-album-2026.md), [requerimientos funcionales](entrega-album-2026.md), [primera entrega](primera-entrega-album-ddd.md), [segunda entrega](segunda-entrega-album-arquitectura.md) y [tercera entrega](tercera-entrega-album-cicd-aks.md). `T21` cierra anunciando Service Mesh como próxima clase, coherente con el orden del módulo IV de `T01`. [T21, p. 55] [T01, p. 13] De ese módulo ya están incorporadas la observabilidad (`T23`), el feature flagging (`T22`) y el service mesh (`T25`), además del API gateway (`T24`), que el cronograma ubicaba en el módulo de Kubernetes. [T01, p. 12] [T01, p. 13] Queda sin material la clase de serverless y cierre, que sigue funcionando como índice del programa. [T01, p. 13]
