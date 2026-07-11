# Guia para Codex: actualizar un preset del catalogo remoto

## Objetivo

Actualizar un preset ya publicado en el repositorio de catalogo remoto de presets, manteniendo el mismo UUID y publicando una nueva version inmutable del archivo `.scpreset`.

Esta guia esta pensada para una instancia de Codex trabajando dentro del repo de presets, no dentro del repo de la app.

## Contexto

La app consume el catalogo publicado por GitHub Pages:

```text
https://laurasirco.github.io/camera-presets/catalog.json
```

Estructura esperada del repo:

```text
public/
├── catalog.json
└── presets/
    └── <preset-uuid>/
        ├── v1.scpreset
        └── v2.scpreset
```

Regla principal: los payloads son inmutables. Nunca reemplaces un `.scpreset` existente. Para actualizar un preset, añade `v<version-nueva>.scpreset`.

## Entradas que debe pedir Codex

Antes de modificar archivos, pide o confirma:

1. Ruta del `.scpreset` nuevo exportado desde SimpleCamera.
2. UUID del preset que se quiere actualizar.
3. Version nueva esperada, por ejemplo `v3`.
4. Nombre del preset, si debe cambiar en `catalog.json`.

Si falta el UUID, inspecciona `public/catalog.json` y pregunta cual de los presets existentes se quiere actualizar.

## Procedimiento

1. Comprueba el estado del repo:

```bash
git status --short
```

Si hay cambios no relacionados, no los reviertas. Informa al usuario y trabaja solo sobre `public/catalog.json` y el nuevo `.scpreset`.

2. Lee el catalogo:

```bash
cat public/catalog.json
```

Localiza la entrada cuyo `id` coincide con el UUID del preset.

3. Verifica que no se va a sobrescribir una version existente:

```bash
ls public/presets/<preset-uuid>/
```

Si `v<version-nueva>.scpreset` ya existe, detente y pregunta. No reemplaces el archivo.

4. Copia el nuevo preset a su ruta inmutable:

```bash
cp /ruta/al/nuevo-preset.scpreset public/presets/<preset-uuid>/v<version-nueva>.scpreset
```

5. Calcula el SHA-256:

```bash
shasum -a 256 public/presets/<preset-uuid>/v<version-nueva>.scpreset
```

Guarda el hash hexadecimal en minusculas.

6. Actualiza `public/catalog.json`:

- Incrementa `catalog_version` en 1.
- En la entrada del preset actualizado, incrementa `version`.
- Mantiene exactamente el mismo `id`.
- Actualiza `payload_url` para apuntar al nuevo archivo.
- Actualiza `sha256`.
- Mantiene `status: "active"` salvo que el usuario pida retirar el preset.
- Ajusta `name` solo si el usuario lo ha pedido o si el preset exportado representa un cambio de nombre intencional.

Ejemplo de entrada actualizada:

```json
{
  "id": "11111111-2222-3333-4444-555555555555",
  "version": 3,
  "name": "Nombre del preset",
  "order": 0,
  "status": "active",
  "payload_url": "../presets/11111111-2222-3333-4444-555555555555/v3.scpreset",
  "sha256": "hash_sha256_nuevo"
}
```

7. Valida JSON:

```bash
python3 -m json.tool public/catalog.json
```

8. Revisa el diff:

```bash
git diff -- public/catalog.json public/presets/<preset-uuid>/v<version-nueva>.scpreset
```

El diff debe mostrar:

- un `.scpreset` nuevo;
- cambios en `public/catalog.json`;
- ningun `.scpreset` antiguo modificado.

9. Si todo esta bien, prepara el commit:

```bash
git add public/catalog.json public/presets/<preset-uuid>/v<version-nueva>.scpreset
git commit -m "Update preset <nombre-del-preset>"
```

10. Indica al usuario que haga push o hazlo si te lo pide:

```bash
git push
```

Despues del push, GitHub Pages debe ejecutar el workflow de deploy.

## Validacion posterior

Tras el deploy:

1. Abrir:

```text
https://laurasirco.github.io/camera-presets/catalog.json
```

2. Confirmar que:

- `catalog_version` es el nuevo;
- el preset tiene la nueva `version`;
- `payload_url` apunta al nuevo `.scpreset`;
- `sha256` coincide con el archivo publicado.

3. Probar la app.

Nota: en modo debug, la app no sobrescribe presets locales gestionados para proteger trabajo local. Para validar comportamiento real de usuario, usar debug desactivado o TestFlight.

## Reglas de seguridad

- No cambiar el UUID de un preset existente.
- No reemplazar archivos `v1.scpreset`, `v2.scpreset`, etc.
- No bajar `catalog_version`.
- No bajar `version`.
- No cambiar `enabled` a `false` salvo peticion explicita.
- No retirar presets (`status: "retired"`) salvo peticion explicita.
- No tocar el workflow de GitHub Pages salvo que el usuario lo pida.
- No formatear masivamente el catalogo si el diff queda ruidoso.

## Rollback

Para volver al contenido anterior, no reduzcas versiones. Publica una version nueva usando el payload anterior.

Ejemplo: si `v3` falla, copia el contenido de `v2.scpreset` como `v4.scpreset`, calcula su SHA-256, sube `catalog_version` y cambia la entrada del preset a `version: 4`.

