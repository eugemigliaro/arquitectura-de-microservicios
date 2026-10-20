# Laboratorio

Código, datasets pequeños, esquemas y entornos reproducibles de la materia. La tecnología se define en `materia.yaml`; este template no presupone motor de base de datos, lenguaje ni runtime.

No versionar credenciales, datos privados, artefactos pesados ni dependencias regenerables.

## Guías de la cátedra

La cátedra publica las guías prácticas en <https://gitlab.com/itba-73-40-microservicios/alumnos/2026s2>. El repositorio se clona en `laboratorio/catedra-2026s2/`, que está excluido de git, y las guías se corren ahí:

```sh
git clone https://gitlab.com/itba-73-40-microservicios/alumnos/2026s2.git laboratorio/catedra-2026s2
git -C laboratorio/catedra-2026s2 pull
```

El clon no se versiona por tres motivos: la cátedra lo actualiza clase a clase, trae binarios compilados de decenas de MB y algunas guías generan credenciales al correrlas (por ejemplo `terraform.tfstate` y `kubeconfig` en la de AKS).

Los documentos de texto de las guías están registrados como fuentes `G00`–`G10`, copiados del commit `36b9189` en `material/catedra/practica/guias-2026s2/`. La síntesis está en `wiki/temas/laboratorios-de-la-catedra.md`.

### Después de cada `git pull`

1. Ver qué documentos cambiaron desde el último commit registrado:

   ```sh
   git -C laboratorio/catedra-2026s2 diff --stat 36b9189..HEAD -- '*.md'
   ```

2. Registrar cada guía nueva con `scripts/registrar_fuente.py` y el siguiente ID `G`. Si cambió un documento ya registrado, registrar la versión nueva con otra ruta (por ejemplo, con el commit en el nombre) y declarar en `estado` qué ID reemplaza. Los originales registrados no se sobrescriben.
3. Revisar que no haya credenciales antes de registrar: la copia de `G06` tuvo que redactarse.
4. Actualizar el commit de referencia en este archivo y en la página de laboratorios de la wiki.

`rg` respeta `.gitignore`, así que no busca dentro del clon: usar `rg --no-ignore ... laboratorio/catedra-2026s2` o buscar desde esa carpeta.

