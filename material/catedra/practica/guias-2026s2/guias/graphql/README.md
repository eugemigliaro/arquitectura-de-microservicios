# GraphQL Microservicio - Guía Práctica

## Objetivo
Este proyecto está pensado como una guía práctica para que puedas comprender y aplicar los conceptos fundamentales de GraphQL en el contexto de microservicios, utilizando Apollo Server. La idea es que puedas experimentar con Queries, Mutations y Subscriptions de manera sencilla y directa.

## Descripción general
La aplicación implementa una API GraphQL básica sobre Node.js, usando Apollo Server. Permite explorar las capacidades de GraphQL para consultar, modificar y suscribirse a datos en tiempo real, facilitando la comparación con arquitecturas REST tradicionales.

### Características principales
- **Queries:** Consultá información estructurada y precisa desde el servidor.
- **Mutations:** Modificá datos en el backend de forma controlada.
- **Subscriptions:** Recibí actualizaciones en tiempo real cuando haya cambios en los datos.

## Uso y despliegue

### Requisitos previos
- Tener Docker y Docker Compose instalados.
- Node.js (opcional, si preferís ejecutarlo sin Docker).

### Despliegue con Docker
1. Cloná este repositorio.
2. Ubicate en la raíz del proyecto y ejecutá:
   ```sh
   docker-compose up --build
   ```
3. Accedé a  [http://localhost:4000/graphql](http://localhost:4000/graphql) para interactuar con el playground de Apollo Server.

### Ejecución local (sin Docker)
1. Instalá las dependencias:
   ```sh
   npm install
   ```
2. Iniciá el servidor:
   ```sh
   node index.js
   ```
3. Accedé a [http://localhost:4000/graphql](http://localhost:4000/graphql) para interactuar con el playground de Apollo Server.

## Ejemplos de uso

### Query
```graphql
query {
  // Tu consulta aquí
}
```

### Mutation
```graphql
mutation {
  // Tu mutación aquí
}
```

### Subscription
```graphql
subscription {
  // Tu suscripción aquí
}
```

## Esquema GraphQL y base de datos mock

El proyecto utiliza un esquema GraphQL simple que modela un sistema de restaurantes, menús y pedidos. Los datos se mantienen en memoria a través del archivo `_db.js`, lo que facilita la experimentación y las pruebas sin depender de una base de datos externa.

### Estructura del esquema (`schema.js`)
- **Restaurant:** Representa un restaurante, con los campos `id`, `name`, `location` y una lista de ítems de menú (`menu`).
- **MenuItem:** Representa un plato del menú, con `id`, `name`, `price` y `description`.
- **Order:** Representa un pedido, incluyendo el restaurante, los ítems pedidos, el total y el estado.
- **Query:** Permite consultar restaurantes, menús y pedidos.
- **Mutation:** Permite crear nuevos pedidos.
- **Subscription:** Permite suscribirse a la creación de nuevos pedidos en tiempo real.

### Base de datos mock (`_db.js`)
- **restaurants:** Lista de restaurantes, cada uno con su menú (referencias a `menuItems`).
- **menuItems:** Lista de platos disponibles en los menús.
- **orders:** Lista de pedidos realizados, con referencias a restaurantes y platos.

Podés modificar estos archivos para agregar más restaurantes, platos o pedidos y así adaptar el ejercicio a tus necesidades. Esto permite practicar cómo se relacionan los datos y cómo se resuelven las referencias en GraphQL.

## Sugerencias para probar la API

- Consultar los restaurantes disponibles
- Consultar el menú de un restaurante
- Realizar un pedido
- Suscribirse a actualizaciones de pedidos

Podés hacerlo desde el menú interactivo de Apollo Server o usando cualquier cliente GraphQL.

## Notas adicionales
- El proyecto está pensado para ser fácilmente extensible y didáctico.
- Podés modificar el esquema en `schema.js` y la lógica en `index.js`.
- Se puede usar cualquier cliente GraphQL para interactuar con la API, como Postman, Insomnia o GraphQL Playground. Se recomienda el que viene integrado con Apollo Server para facilitar las pruebas.