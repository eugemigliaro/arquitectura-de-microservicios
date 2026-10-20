# Arquitectura de Microservicios — 2026 S2

Repositorio de guías prácticas de la materia **73.40 — Arquitectura de Microservicios** del ITBA.

## Equipo Docente

- **Federico Castañeda** — Jefe de Cátedra
- **Carlos Maceira García Coni** — Profesor
- **Franco Rupnik** — Profesor Adjunto

## Guías prácticas

Las guías están organizadas **por tema**, no por número de clase: el calendario se reacomoda
entre cuatrimestres y los temas no. Esta tabla es el mapa entre una y otra cosa.

| Clase | Fecha | Tema | Guía |
|---|---|---|---|
| 3 | 21/08 | Linux y Contenedores | [`guias/linux-containers`](guias/linux-containers) |
| 4 | 28/08 | Imágenes y orquestación local | [`guias/dockerfile-compose`](guias/dockerfile-compose) |
| 6 | 11/09 | DevOps — pipelines de CI/CD | [`guias/cicd-pipelines`](guias/cicd-pipelines) |
| 6 | 11/09 | APIs — GraphQL | [`guias/graphql`](guias/graphql) |
| 7 | 18/09 | Estado distribuido — Redis | [`guias/laboratorio-redis-patterns`](guias/laboratorio-redis-patterns) |
| 7 | 18/09 | Estado distribuido — Saga coreografiada | [`guias/laboratorio-coreografia`](guias/laboratorio-coreografia) |
| 7 | 18/09 | Estado distribuido — Saga orquestada | [`guias/laboratorio-orquestacion`](guias/laboratorio-orquestacion) |
| 10 | 09/10 | Kubernetes I — Fundamentos | [`guias/k8s-fundamentos`](guias/k8s-fundamentos) |
| 11 | 16/10 | Kubernetes II — Cluster AKS (base del TP) | [`guias/k8s-aks`](guias/k8s-aks) |

## Demos

Código que se mostró en clase. No son guías con pasos: son experimentos para
correr, tocar y sacar conclusiones propias.

| Clase | Demo | De qué va |
|---|---|---|
| 6 | [`demos/capacidad-sync-async`](demos/capacidad-sync-async) | Qué pasa cuando llegan más pedidos de los que el sistema puede atender, y qué cambia según cómo lo enfrentes: aceptar todo, acotar la admisión, reintentar, o meter una cola en el medio |

## Pre-requisitos generales

- Acceso a una terminal Linux (VM, WSL2 en Windows, o Linux nativo)
- Docker y Docker Compose (a partir de la clase 4)
- Conocimientos básicos de programación y línea de comandos

Cada guía declara sus pre-requisitos propios en su `README.md`.

## Cómo trabajar con las guías

1. Clonar este repositorio.
2. Entrar a la carpeta del tema correspondiente.
3. Leer el `README.md` de la guía — ahí está el orden de los pasos y el setup previo.
4. Seguir los pasos **en orden**: cada uno se apoya en el anterior.

Cada guía es auto-contenida e incluye comandos de verificación en cada paso.
