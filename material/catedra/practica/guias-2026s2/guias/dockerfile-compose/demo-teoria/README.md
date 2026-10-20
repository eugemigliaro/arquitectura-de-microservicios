# Demo para Teoría: Imagen Mínima con FROM scratch

Ejemplo para mostrar en clase cómo construir una imagen Docker desde cero (`FROM scratch`) usando un binario estáticamente compilado.

## Archivos

- `hw.go` — Hello World en Go
- `hw.c` — Hello World en C
- `Dockerfile` — Imagen FROM scratch que copia un binario compilado

## Uso

### Go (compilación estática por defecto)

```bash
# Compilar el binario
CGO_ENABLED=0 go build -o hw hw.go

# Build de la imagen
docker build -t hw-go .
# (modificar el Dockerfile para copiar 'hw' en vez de 'hwc')

docker run --rm hw-go
```

### C (requiere compilación estática explícita)

```bash
# Compilar estáticamente
gcc -static -o hwc hw.c

# Build de la imagen
docker build -t hw-c .

docker run --rm hw-c
```

## Punto de Discusión

- ¿Por qué Go produce un binario estático por defecto y C no?
- ¿Qué pasa si intentamos correr el binario de C compilado dinámicamente en una imagen `scratch`?
- ¿Cuánto pesa cada imagen resultante? Comparar con `python:3.12` (~1GB).
