# Feature: color principal de acción en índigo

## Encargo y criterios de aceptación

Encargo (issue #6): "Quiero que el color principal de accion sea indigo".

Criterios de aceptación:

- El color de fondo de los botones de acción principal (`button-primary`) pasa de rojo/granate (`#b73526`) a un tono índigo, tanto en estado normal como en `:hover`.
- Los demás elementos que usan la variable `--color-primary` (el punto del logo "todo." y el icono "+" de añadir tarea) cambian también a índigo, por ser el mismo color semántico en todo el sitio.
- El resto de colores (foco, peligro, prioridades, texto, bordes) no cambian.
- El cambio es visible tras recargar la aplicación en el navegador.

Inferencia explícita: "índigo" no especifica un valor hexadecimal exacto. Se usa la paleta Tailwind Indigo como referencia estándar: `indigo-600` (`#4f46e5`) para el estado normal y `indigo-700` (`#4338ca`) para `:hover`, manteniendo el mismo patrón de dos tonos (normal/hover) que ya existe en el CSS actual.

## Comportamiento actual y propuesta

Actual: `public/assets/app.css` define `--color-primary: #b73526;` y `--color-primary-hover: #992b1e;` en el bloque `:root` (líneas 22-23). Estas variables alimentan `.button-primary` (líneas 411-420), `.brand-dot` (línea 73), `.add-mark` (línea 303) y el fallback de `.task-toggle-button:hover .completion-icon` (línea 207).

Propuesta: sustituir únicamente los valores hexadecimales de `--color-primary` y `--color-primary-hover` por los tonos índigo. No se toca ninguna otra regla ni variable; al estar centralizado en dos variables CSS, el cambio se propaga automáticamente a todos los usos.

## Alcance

Incluido:
- Cambiar los valores de `--color-primary` y `--color-primary-hover` en `public/assets/app.css`.
- Bump del parámetro de cache-busting `?v=6` a `?v=7` en `templates/application.plush.html` (sigue el patrón ya usado por el propio repo al modificar el CSS), para que los navegadores no sirvan la hoja de estilos cacheada con los colores antiguos.

Excluido:
- Cambiar `--color-focus` (azul, usado solo para el anillo de foco de accesibilidad) o cualquier otro color (`--color-danger`, prioridades, etc.).
- Añadir Tailwind, variables adicionales o un sistema de theming nuevo.
- Tests automatizados de regresión visual (no existen en el repo y no hay infraestructura para ellos).

## Diseño por capas

Solo capa de presentación (CSS estático servido por Go/Buffalo vía `//go:embed`), no hay cambios de modelo, persistencia, acción ni ruta.

## Archivos relevantes

- `public/assets/app.css:22-23` — definición de las variables `--color-primary` y `--color-primary-hover`; único lugar donde se edita el valor del color.
- `templates/application.plush.html:7` — enlace `<link>` al CSS con el parámetro `?v=N` de cache-busting; se incrementa para invalidar la caché del navegador.
- `public/embed.go` — embebe `public/assets/app.css` en el binario; no requiere cambios, solo recompilar.

## Implementación paso a paso

1. En `public/assets/app.css`, cambiar:
   - `--color-primary: #b73526;` → `--color-primary: #4f46e5;`
   - `--color-primary-hover: #992b1e;` → `--color-primary-hover: #4338ca;`
2. En `templates/application.plush.html:7`, cambiar `href="/assets/app.css?v=6"` a `href="/assets/app.css?v=7"`.
3. Verificar visualmente en el navegador (botón "Añadir" en `/`, botón "Guardar cambios" en la edición de una tarea, el punto del logo y el icono "+") que el color mostrado es índigo y que el resto de la interfaz no cambió.

## Tests y validación

No hay tests automatizados de Go afectados (no existe código Go ni plantillas que referencien el valor del color), así que no se requieren tests nuevos. Validación:

- `gofmt -l .` (sin salida) y `go vet ./...` — para confirmar que no se ha roto nada en el código Go, aunque este cambio no lo toca.
- `go build ./...` — confirma que el binario sigue embebiendo correctamente los assets estáticos tras el cambio.
- Verificación manual en navegador: levantar el entorno (`docker compose up`, app en `/`), comprobar que los botones primarios, el punto del logo y el icono "+" se ven en índigo y que el `:hover` usa el tono más oscuro.

## Riesgos y recuperación

- Riesgo bajo: es un cambio de dos valores hexadecimales en un único archivo CSS, sin lógica de aplicación involucrada.
- Si el índigo elegido no satisface al usuario, revertir es trivial: restaurar los dos valores hexadecimales originales (`#b73526` / `#992b1e`) y el `?v=6`.
- Si se olvida el bump de `?v=N`, el navegador podría servir el CSS cacheado y el cambio no se apreciaría hasta forzar recarga; por eso el paso 2 es parte del plan, no opcional.
