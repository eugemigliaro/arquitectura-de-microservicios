# Solución

Acá se encuentran los Dockerfiles y el docker-compose.yml con todas las mejoras descritas en el README del laboratorio.

## Cómo ejecutar

```bash
# 1. Copiar los archivos de ejemplo de variables de entorno
cp gateway.env.example gateway.env
cp trips.env.example trips.env
cp airports.env.example airports.env

# 2. Levantar todos los servicios
docker compose up --build

# 3. Verificar que todo funcione
curl http://localhost:8000/ping
curl http://localhost:8000/trip-ping
curl http://localhost:8000/airport-ping
```

## Modo desarrollo con Watch

Docker Compose Watch permite que los cambios en el código se reflejen automáticamente en los contenedores sin tener que hacer rebuild manual. Es la forma recomendada de trabajar en desarrollo local.

```bash
# Levantar en modo watch
docker compose watch
```

### ¿Qué hace?

| Cambio | Acción | Qué pasa |
|---|---|---|
| Editar un `.py` | `sync+restart` | Sincroniza el archivo al contenedor y reinicia el servicio |
| Editar `requirements.txt` | `rebuild` | Reconstruye la imagen completa (necesita reinstalar dependencias) |

### Probarlo

1. Levantar con `docker compose watch`
2. Hacer un `curl http://localhost:8000/ping` → responde "Pong!"
3. Editar `gateway/gateway.py`, cambiar el return de `ping` a `"Pong! (modified)"`
4. Esperar unos segundos — Compose detecta el cambio, sincroniza y reinicia
5. Volver a hacer `curl http://localhost:8000/ping` → responde "Pong! (modified)"

> **Nota:** `sync+restart` es necesario porque uvicorn, tal como lo arrancamos, no recarga el código: `sync` copiaría el archivo al contenedor pero el proceso seguiría sirviendo la versión vieja. Si se arranca con `uvicorn --reload`, alcanza con `sync` solo — ver el desafío extra del README principal.

## Qué incluye

- **Dockerfiles** con venv, non-root user, cache de layers, healthcheck (gateway)
- **docker-compose.yml** con healthchecks, depends_on, redes segregadas, env_file y watch
- **`.env.example`** con las variables necesarias para cada servicio
