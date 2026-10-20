# Imágenes de contenedores y Dockerfile

## Imagen, capas y dependencias de ejecución

Una imagen reúne metadata de ejecución y archivos organizados en capas; las capas inmutables pueden reutilizarse entre imágenes, tanto en disco como durante la transferencia. [T06, p. 7]

Una base `scratch` está vacía. Por eso un binario estático puede ser la totalidad de la imagen, mientras que un binario enlazado dinámicamente necesita que el filesystem incluya su *dynamic linker* y sus bibliotecas, como `libc`; que el ejecutable exista no garantiza que pueda arrancar. [T06, p. 9] [T06, p. 10] [T06, p. 11]

## Instrucciones esenciales de Dockerfile

| Instrucción | Responsabilidad |
|---|---|
| `FROM` | Elige la imagen base; `scratch` representa una base vacía. [T06, p. 13] |
| `COPY` | Incorpora archivos desde el *build context*. [T06, p. 13] |
| `RUN` | Ejecuta durante el build y deja su resultado en la imagen. [T06, p. 13] |
| `ENTRYPOINT` | Fija el ejecutable; los argumentos escritos después de la imagen en `docker run` se aplican sobre él. [T06, p. 14] |
| `CMD` | Proporciona el comando o los argumentos predeterminados, reemplazables al ejecutar. [T06, p. 14] [T06, p. 15] |
| `ARG` | Parametriza únicamente el build. [T06, p. 16] |
| `ENV` | Define una variable disponible en los `RUN` posteriores y en el contenedor. [T06, p. 16] |
| `WORKDIR` | Cambia el directorio de trabajo para las instrucciones posteriores y la ejecución. [T06, p. 17] |
| `USER` | Define el usuario de los `RUN`, `ENTRYPOINT` y `CMD` posteriores; si se omite, el contenedor corre como root por defecto. [T06, p. 17] |

`EXPOSE` documenta el puerto de la aplicación, pero no lo publica en el host. `LABEL` agrega metadata inspeccionable y `HEALTHCHECK` define una prueba de salud intrínseca a la imagen. [T06, p. 18]

## Cache y build context

La cache se invalida por contenido en `COPY`/`ADD`, no por *mtime*, y por coincidencia exacta del comando en `RUN`. Un *miss* invalida esa capa y todas las posteriores; por eso conviene copiar e instalar primero las dependencias y recién después el código que cambia con mayor frecuencia. [T06, p. 20] [T06, p. 21] [T06, p. 22]

El *build context* es el conjunto de archivos enviado por el cliente al engine antes de ejecutar el Dockerfile; el build aislado no puede leer archivos que quedaron fuera. Un `.dockerignore` reduce transferencia e invalidaciones y evita incluir por accidente archivos como `.env`, `.git` o logs. [T06, p. 23] [T06, p. 24]

BuildKit representa el build como un grafo, puede paralelizar etapas independientes y ejecutar solo el target solicitado. Sus mounts de cache conservan descargas entre builds, mientras que los mounts de secret o SSH permiten usar credenciales sin grabarlas en las capas. [T06, p. 25]

## Imágenes pequeñas y seguras

En un build multi-stage, la etapa de compilación contiene toolchain y dependencias de desarrollo; la etapa de runtime recibe solo los artefactos copiados explícitamente. Para terminar en `scratch`, el ejecutable debe ser autocontenido, como el binario Go compilado con `CGO_ENABLED=0` del ejemplo. [T06, p. 27] [T06, p. 28]

Las imágenes *distroless* eliminan shell, package manager y utilidades generales para reducir superficie de ataque, pero también eliminan herramientas útiles para diagnosticar dentro del contenedor. Reducir paquetes no reemplaza mantener actualizada la base: la comparación de CVEs de la clase es una medición puntual con Trivy 0.74 y base del 21 de agosto de 2026, no una propiedad permanente de cada familia de imágenes. [T06, p. 29] [T06, p. 30] [T06, p. 31]

Alpine usa `musl` en lugar de `glibc`; eso puede obligar a compilar dependencias que no publican binarios compatibles y producir diferencias de resolución DNS. Su menor tamaño no garantiza por sí solo builds más rápidos ni compatibilidad equivalente. [T06, p. 32]

Las prácticas que la cátedra reúne son: multi-stage, base mínima y actualizada, `.dockerignore`, ejecución sin root, y secretos fuera de `ARG`/`ENV` y de las capas. [T06, p. 24] [T06, p. 27] [T06, p. 31] [T06, p. 37]

## Registry, tags y digest

Una referencia de imagen combina registry, repositorio o nombre y tag; si se omite el registry, se asume Docker Hub. Los registries mantienen el catálogo y pueden ser públicos o privados. [T06, p. 34]

Un tag como `latest` es un puntero mutable: puede pasar a señalar otro contenido y generar drift entre nodos o impedir reconstruir con certeza un rollback. Para trazabilidad convienen tags que no se sobrescriban, como una versión exacta o commit, y el despliegue por digest `sha256`, que identifica el contenido del manifiesto. Las políticas de inmutabilidad del registry complementan esa práctica impidiendo reemplazar tags publicados. [T06, p. 35] [T06, p. 36]

## Laboratorio: de un Dockerfile mínimo a uno endurecido

`G02` parte de un Dockerfile de cuatro líneas para un servicio FastAPI (`FROM python:3.12`, `COPY . .`, `RUN pip install -r requirements.txt`, `CMD` con uvicorn) y lo mejora respondiendo preguntas que deja abiertas en el camino. [G02] `G02-SOLUCION` reúne el resultado final; ver [Laboratorios de la cátedra](laboratorios-de-la-catedra.md).

| Mejora | Por qué, según la guía |
|---|---|
| Imagen base `python:3.12` | No hay restricción de espacio que justifique `-slim`, y `-alpine` puede traer problemas de dependencias; multi-stage se descarta porque no hay compilación. [G02] |
| `WORKDIR /usr/app` | Sin él, los comandos y las copias se hacen desde `/`, con riesgo de pisar partes del sistema. [G02] |
| `.dockerignore` por contexto | El cliente empaqueta el contexto entero antes del build: cuesta tiempo y, con `COPY . .`, un `.env` queda en una capa, y borrarlo en un `RUN` posterior no lo saca. [G02] |
| Dependencias pinneadas | Sin versión, `pip` resuelve la última al momento del build: dos integrantes obtienen versiones distintas y el cache pierde sentido. Para el TP, pinnear las directas es el mínimo; para las transitivas hace falta un lock (`pip freeze`, `pip-tools` o `uv`). [G02] |
| `COPY requirements.txt` antes de `COPY . .` | Cambiar código no invalida la capa de dependencias, y servicios con los mismos requisitos la reutilizan (`CACHED`). [G02] |
| venv y usuario no root | Crea el usuario 999, le da `/usr/app`, copia con `--chown` y usa `USER 999`; `PIP_NO_CACHE_DIR` evita el error de un usuario sin home. [G02] |
| `LABEL`, `EXPOSE` y `HEALTHCHECK` | Documentan la imagen; `EXPOSE` habilita `docker run -P`, y el healthcheck usa `python` para no instalar `curl`. [G02] |

Son las prácticas que la teoría reúne: `.dockerignore`, ejecución sin root y secretos fuera de las capas. [T06, p. 24] [T06, p. 27] [T06, p. 31]

La demo de teoría `G02-TEORIA` construye una imagen `FROM scratch` con un binario estático. Go lo produce por defecto (`CGO_ENABLED=0`) y C requiere `-static`; la discusión compara el peso con el de `python:3.12`, de alrededor de 1 GB. [G02-TEORIA]
