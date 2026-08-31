# Docker Compose

## Orquestación local declarativa

Docker Compose describe en YAML una aplicación con uno o más contenedores y permite declarar servicios, redes, volúmenes, variables, política de reinicio y dependencias. La cátedra lo presenta como una forma simplificada de orquestación local. [T07, p. 3] [T07, p. 4]

| Elemento | Función |
|---|---|
| `services` | Declara los contenedores de la aplicación y la imagen o build de cada uno. [T07, p. 5] [T07, p. 6] |
| `restart` | Indica qué hacer cuando el contenedor deja de ejecutarse; se muestran `no`, `always`, `on-failure` y `unless-stopped`. [T07, p. 6] |
| `ports` | Publica un puerto con correspondencia host:contenedor. [T07, p. 8] |
| `expose` | Declara el puerto interno sin publicarlo en el host. [T07, p. 8] |
| `volumes` | Monta almacenamiento y permite declarar volúmenes globales para persistencia. [T07, p. 8] |
| `networks` | Asocia servicios a redes; en una red compartida se resuelven por nombre de servicio. Una red `internal` deshabilita la salida fuera del host. [T07, p. 9] |

## Configuración, secretos y build

`environment` admite mapa `VARIABLE: valor` o lista `VARIABLE=valor`, y la interpolación `${VARIABLE-default}` ofrece un valor predeterminado. En el ejemplo de precedencia, `environment` pisa los valores de `env_file` y, entre archivos listados, el posterior pisa al anterior. [T07, p. 10]

No corresponde hardcodear contraseñas en el YAML versionado. Compose permite declarar secretos externos o leerlos desde un archivo que no se incorpora a Git; cada servicio que referencia el secreto lo recibe como `/run/secrets/<nombre>`. [T07, p. 7] [T07, p. 11]

Un servicio puede usar `build` en vez de descargar una imagen. Se puede indicar context, Dockerfile, argumentos y target multi-stage, y `docker compose up --build` reconstruye antes de iniciar. [T07, p. 12]

`docker compose watch` sincroniza archivos, reconstruye o sincroniza y reinicia según las reglas declaradas; solo aplica a servicios construidos mediante `build`. [T07, p. 13]

## Arranque y salud

Que un contenedor esté iniciado no significa que el servicio esté listo para aceptar trabajo. Un `depends_on` sin prueba de salud puede iniciar al consumidor demasiado pronto; la combinación de `healthcheck` y `condition: service_healthy` mantiene al dependiente esperando hasta que la dependencia esté sana. [T07, p. 15] [T07, p. 16]

El healthcheck es un comando periódico dentro del contenedor. Comienza en `starting`, pasa a `healthy` con código de salida 0 y a `unhealthy` con otro código; por sí mismo informa estado y no reinicia el contenedor. [T07, p. 17]

La distinción conceptual es:

- **readiness:** el proceso puede aceptar trabajo; los consumidores deberían esperar si falla;
- **liveness:** el proceso sigue vivo; una plataforma que use esa señal puede decidir reiniciarlo.

El healthcheck de Compose usado por `service_healthy` se aproxima a readiness, porque coordina el acceso pero no reinicia por un fallo de la prueba. [T07, p. 18]

Si la prueba es intrínseca a la imagen se define con `HEALTHCHECK` en el Dockerfile; si depende del entorno, en Compose. `CMD` ejecuta directamente y `CMD-SHELL` pasa por el shell. La prueba debe ser rápida, local, contar con sus herramientas dentro de la imagen y dar un `start_period` acorde al calentamiento real. [T07, p. 19]

## Lectura del ejemplo completo

El stack ilustrado conecta un proxy Nginx, un backend y MariaDB: el cliente entra por el puerto publicado del proxy; proxy y backend se comunican por HTTP; el backend espera la salud de la base y accede por SQL; un volumen conserva los datos y un secreto externo monta la contraseña en backend y base. [T07, p. 21]

La configuración de MariaDB combina healthcheck, secreto, volumen y variables; el backend usa `depends_on: db: condition: service_healthy`, mientras el proxy depende del backend y monta su configuración como bind read-only. [T07, p. 22] [T07, p. 23]
