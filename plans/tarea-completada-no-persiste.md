# Bug: Una tarea completada aparece sin marcar al recargar la página

## Encargo y resultado esperado

Encargo (no hay issue en GitHub porque las issues están desactivadas en el repositorio): «cuando se marca una tarea como completada y se recarga la pagina, aparece sin marcar».

Reproducción del usuario: abrir `/`, pulsar el círculo de una tarea pendiente para completarla y recargar.

- **Esperado:** tras recargar, la tarea sigue completada. Lo mismo al reabrirla: sigue pendiente tras recargar.
- **Observado:** la tarea vuelve a aparecer sin marcar.

## Evidencia y causa raíz

### Hechos observados

1. La ruta de cambio de estado solo acepta `PUT` (`actions/app.go`):
   `app.PUT("/tasks/{task_id}/toggle", TasksToggle)`.
2. El formulario de cada tarea (`templates/tasks/index.plush.html`) es `method="POST"` con `_method=PUT` y `authenticity_token` como campos ocultos, y está marcado con `data-task-toggle`. Enviado de forma nativa funciona: lo cubre `Test_TasksToggle_CompletesAndReopens` en `actions/tasks_test.go`.
3. `public/assets/app.js` intercepta el `submit` de los formularios `[data-task-toggle]`. Llama a `event.preventDefault()`, cambia la UI de forma optimista (clase `task-completed`, `aria-pressed`, etiquetas) y termina con:
   ```js
   fetch(form.action);
   ```
   `fetch` sin opciones hace un **GET** sin cuerpo: no envía `_method`, ni el token CSRF, ni usa POST.
4. Reproducción contra el entorno local (app en Docker, puerto 3000), con una tarea pendiente:
   ```bash
   curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/tasks/095bfb69-202a-4b25-a23b-4d65c6f0b3c7/toggle
   ```
   → `404`. Después, `select completed from tasks where id='095bfb69-…'` sigue devolviendo `f`.
5. La respuesta de `fetch` no se comprueba, así que el 404 pasa desapercibido y la UI muestra un estado que el servidor nunca guardó.

### Causa raíz

El toggle optimista de `app.js` nunca envía la petición correcta al servidor. Hace un GET a una ruta que solo acepta PUT y obtiene un 404 en silencio. La UI cambia y la base de datos no, así que al recargar se ve el estado real: sin completar.

### Incertidumbres

- No hay infraestructura de tests de JavaScript en el repositorio (no hay `package.json` ni runner). El comportamiento en navegador se verifica manualmente. El test automático propuesto protege el contrato del asset, no ejecuta el JS.
- Buffalo aplica el method override (`_method`) a peticiones POST con cuerpo de formulario. Lo confirma el test de acción existente, que envía `PUT` mediante el helper, y el formulario nativo. Hay que verificar en el navegador que el `fetch` con `FormData` (multipart) también se interpreta como PUT. Si no fuera así, enviar `URLSearchParams(new FormData(form))` (urlencoded), que es lo mismo que envía el formulario nativo.

### Reproducción mínima

1. `docker compose up -d` (app y db sanas).
2. En `http://localhost:3000/`, pulsar el círculo de una tarea pendiente.
3. En las herramientas de red: `GET /tasks/<id>/toggle` → 404.
4. Recargar: la tarea aparece sin completar.

## Alcance

**Incluido**

- Corregir el envío del toggle en `public/assets/app.js`: POST con el cuerpo del propio formulario (`_method=PUT`, `authenticity_token`) y la cookie de sesión.
- Comprobar la respuesta y, si falla, recargar la página para mostrar el estado real del servidor en vez de dejar la UI desincronizada.
- Subir la versión de caché del script en `templates/application.plush.html` (`app.js?v=2` → `app.js?v=3`), para que los navegadores no sigan usando el script roto.
- Un test de acción que proteja el contrato del asset servido.

**Excluido**

- Cambios en rutas, acciones, modelo o plantilla de la lista. El servidor funciona correctamente.
- Eliminar la actualización optimista o rediseñar la interacción.
- El comportamiento de `data-confirm` del mismo listener.
- Añadir infraestructura de tests de JavaScript.

## Archivos relevantes

| Ruta | Responsabilidad |
|---|---|
| `public/assets/app.js` | Listener de `submit`: toggle optimista y petición al servidor. **Se corrige aquí.** |
| `templates/application.plush.html` | Carga `app.js?v=2`. Se sube la versión de caché. |
| `actions/app.go` | Rutas; `PUT /tasks/{task_id}/toggle`. Referencia, no cambia. |
| `actions/tasks.go` | `TasksToggle` alterna `Completed` y redirige 303 a `/`. Referencia, no cambia. |
| `templates/tasks/index.plush.html` | Formulario `data-task-toggle` con `_method=PUT` y token. Referencia, no cambia. |
| `actions/tasks_test.go` | Test de regresión nuevo. |

## Implementación paso a paso

1. En `public/assets/app.js`, sustituye la última línea `fetch(form.action);` por:
   ```js
   fetch(form.action, {
     method: "POST",
     body: new URLSearchParams(new FormData(form)),
     credentials: "same-origin",
   }).then(function (response) {
     if (!response.ok) {
       window.location.reload();
     }
   }, function () {
     window.location.reload();
   });
   ```
   - `form.action` ya es la URL de toggle. El cuerpo urlencoded replica el envío nativo del formulario (`authenticity_token`, `_method=PUT`).
   - `TasksToggle` responde `303` a `/`; `fetch` sigue la redirección y obtiene `200`, así que `response.ok` es verdadero en el caso correcto.
   - Si el servidor falla o hay un error de red, recargar muestra el estado real.
2. En `templates/application.plush.html`, cambia `/assets/app.js?v=2` por `/assets/app.js?v=3`.
3. Añade el test de regresión de *Tests y validación*.
4. Ejecuta las validaciones y la verificación manual en el navegador.

## Tests y validación

### Test de regresión (`actions/tasks_test.go`)

`Test_AppJS_TogglePersistsWithPost`: pide el asset servido y comprueba que el toggle envía un POST con el cuerpo del formulario.

```go
func (as *ActionSuite) Test_AppJS_TogglePersistsWithPost() {
	res := as.HTML("/assets/app.js").Get()
	as.Equal(200, res.Code)

	body := res.Body.String()
	as.Contains(body, `method: "POST"`)
	as.Contains(body, "new FormData(form)")
	as.NotContains(body, "fetch(form.action);")
}
```

Antes del arreglo falla, porque el asset contiene `fetch(form.action);` y no tiene `method: "POST"`. Después pasa. Es una guarda de contrato del asset, no una ejecución del JS (ver *Incertidumbres*). `Test_TasksToggle_CompletesAndReopens` ya cubre la persistencia del lado servidor y debe seguir en verde.

### Comandos

Requieren PostgreSQL en `127.0.0.1:5432` y `todo_test` migrada (`GO_ENV=test soda migrate -e test`, como en `.github/workflows/ci.yaml`).

```bash
gofmt -l .
```

```bash
go vet ./...
```

```bash
GO_ENV=test go test -p 1 ./...
```

```bash
go build ./...
```

### Verificación manual (navegador)

1. `docker compose up --build -d` y esperar a que `app` esté sana.
2. En `/`, completar una tarea pendiente. En la pestaña de red debe verse `POST /tasks/<id>/toggle` → `303` y luego `GET /` → `200`.
3. Recargar: la tarea sigue completada. Comprobar en la BD:
   `docker compose exec -T db psql -U postgres -d todo_development -tAc "select completed from tasks where id='<id>'"` → `t`.
4. Reabrirla y recargar: sigue pendiente (`f`).
5. Dejar la tarea en su estado original al terminar.

## Riesgos y recuperación

- **Method override con `fetch`.** Si Buffalo no interpretase `_method` en el cuerpo, la petición daría 404 o 405 y el código recargaría la página. El fallo sería visible, ya no silencioso. Se detecta en la verificación manual del paso 2.
- **CSRF.** El token va en el cuerpo, igual que en el envío nativo; `credentials: "same-origin"` asegura la cookie de sesión. Si se rechazase (403), la página se recarga con el estado real.
- **Clics rápidos repetidos.** Cada clic envía su propio toggle y el servidor alterna en orden de llegada. Dos peticiones simultáneas podrían llegar desordenadas, pero el resultado neto de dos toggles es el mismo. Riesgo bajo y fuera de alcance.
- **Caché del navegador.** Sin subir `?v=`, los usuarios seguirían con el script roto. Por eso se incluye el cambio de versión.
- **Recuperación.** Revertir el commit restaura el comportamiento anterior; no hay datos ni migraciones implicados.
