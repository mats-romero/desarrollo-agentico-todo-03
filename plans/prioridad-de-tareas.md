# Feature: Prioridad de las tareas (alta, media y baja)

## Encargo y criterios de aceptación

Encargo original: «incluye un método de gestión de prioridad de las tareas: alta, media y baja».

Requisitos explícitos:

1. Cada tarea tiene una prioridad con uno de tres valores: **alta**, **media** o **baja**.
2. La prioridad se puede gestionar: el usuario la elige al crear la tarea y la cambia al editarla.

Inferencias (decisiones de este plan, no pedidas literalmente):

3. Si no se indica prioridad, la tarea es **media**. Las tareas que ya existen pasan a **media** con la migración.
4. La lista muestra la prioridad de cada tarea con una etiqueta visible ("Alta", "Media", "Baja").
5. La lista se ordena por prioridad (alta → media → baja) y, dentro de cada prioridad, por el orden actual (`created_at asc, id asc`).
6. Un valor de prioridad distinto de los tres permitidos se rechaza con `422` y el mensaje «La prioridad no es válida.», sin persistir cambios.
7. Editar una tarea sin enviar `priority` conserva la prioridad existente (el contrato actual de `TasksUpdate` de no tocar campos no enviados se mantiene).

Criterios de aceptación observables:

- `POST /tasks` con `title=X&priority=alta` crea la tarea con prioridad `alta` y redirige `303` a `/`.
- `POST /tasks` sin `priority` crea la tarea con prioridad `media`.
- `POST /tasks` con `priority=urgente` responde `422`, muestra «La prioridad no es válida.» y no crea la tarea.
- `GET /` muestra, en cada tarea, la etiqueta de su prioridad y lista primero las de prioridad alta, luego media y luego baja.
- El formulario de creación de `GET /` ofrece un selector de prioridad con «Media» preseleccionada.
- `GET /tasks/{id}/edit` muestra el selector con la prioridad actual de la tarea preseleccionada.
- `PUT /tasks/{id}` con `priority=baja` cambia la prioridad, conserva `completed`, `id` y `created_at`, y redirige `303`.
- `PUT /tasks/{id}` sin `priority` conserva la prioridad; con un valor inválido responde `422` y no modifica la tarea.
- Completar/reabrir una tarea (`PUT /tasks/{id}/toggle`) no cambia su prioridad.

## Comportamiento actual y propuesta

**Actual.** `models.Task` (`models/task.go`) solo tiene `ID`, `Title`, `Completed`, `CreatedAt` y `UpdatedAt`. La tabla `tasks` se crea en `migrations/20260901000000_create_tasks.up.fizz`. `TasksIndex` ordena por `created_at asc, id asc`. `TasksCreate` hace `c.Bind(task)` sobre `models.Task` (solo `title` tiene etiqueta `form`). `TasksUpdate` hace bind a un struct anónimo con `Title` únicamente, para no sobrescribir otros campos. Las plantillas `templates/tasks/index.plush.html` y `edit.plush.html` solo tienen el campo de título. No existe ningún concepto de prioridad.

**Propuesta.** Cambio vertical mínimo siguiendo los patrones existentes:

- Nueva columna `priority` (string, no nula, por defecto `'media'`, con un CHECK sobre los tres valores) mediante una migración fizz nueva.
- Campo `Priority` en `models.Task` con constantes, validación de pertenencia y valor por defecto `media`.
- `TasksCreate` recoge `priority` del formulario. `TasksUpdate` añade `Priority` a su struct de entrada y solo la aplica si viene informada. `TasksIndex` ordena por prioridad.
- Selector `<select name="priority">` en los formularios de creación y edición, y una etiqueta de prioridad en cada fila de la lista.
- Estilos mínimos en `public/assets/app.css` para el selector y la etiqueta.

## Alcance

**Incluido**

- Migración que añade `tasks.priority`.
- Modelo: constantes, validación, valor por defecto y etiqueta legible.
- Acciones `TasksIndex`, `TasksCreate` y `TasksUpdate`.
- Plantillas de lista/creación y de edición.
- CSS para el selector y la etiqueta.
- Fixture `tres tareas` con prioridades explícitas.
- Tests de modelo y de acciones.

**Excluido**

- Filtrar la lista por prioridad o cambiar la prioridad directamente desde la lista (sin pasar por edición).
- Cambios en rutas: no hacen falta rutas nuevas.
- Cambios en `public/assets/app.js`. El toggle optimista no toca la prioridad. Hay una rareza ajena: termina con `fetch(form.action)`, que hace un GET sobre la ruta de toggle. Queda fuera de alcance.
- Modificar la migración de seed existente (`20260901000001_seed_tasks.up.sql`): sus filas reciben `media` por el DEFAULT de la columna.
- API JSON, internacionalización o configuración de valores de prioridad.

## Diseño por capas

### Persistencia y migraciones

Nueva migración `migrations/20261014000000_add_priority_to_tasks.up.fizz`, con un timestamp posterior a las existentes para que se aplique después del seed:

```fizz
add_column("tasks", "priority", "string", {"size": 10, "default": "media"})
sql("ALTER TABLE tasks ADD CONSTRAINT tasks_priority_check CHECK (priority IN ('alta', 'media', 'baja'))")
```

En fizz las columnas son `NOT NULL` salvo `{"null": true}`; con el DEFAULT, PostgreSQL rellena con `media` las filas existentes.

Down, `migrations/20261014000000_add_priority_to_tasks.down.fizz`:

```fizz
drop_column("tasks", "priority")
```

(La restricción CHECK desaparece con la columna.)

`migrations/embed.go` ya incrusta `*.fizz`, así que no cambia. `cmd/app/main.go` ejecuta `models.Migrate()` al arrancar, así que el entorno de desarrollo aplica la migración solo.

### Modelo (`models/task.go`)

- Constantes:
  ```go
  const (
      PriorityHigh   = "alta"
      PriorityMedium = "media"
      PriorityLow    = "baja"
  )
  ```
- Campo nuevo en `Task`, entre `Completed` y `CreatedAt`:
  `Priority string `json:"priority" db:"priority" form:"priority"``
- `Validate`: si `t.Priority` está vacía, se acepta (se resolverá a `media`). Si no está vacía y no es uno de los tres valores, se añade `verrs.Add("priority", "La prioridad no es válida.")`. La comparación es exacta: no se normalizan mayúsculas. El formulario siempre envía minúsculas.
- Callback `BeforeSave(tx *pop.Connection) error`: si `t.Priority == ""`, asigna `PriorityMedium`. pop lo invoca en `Create`/`Update` (también en `ValidateAndCreate`/`ValidateAndUpdate` tras validar), así que el struct en memoria queda coherente con la BD. `Test_Task_Create_Valid` sigue pasando sin indicar prioridad.
- Método de presentación `func (t Task) PriorityLabel() string` que devuelve «Alta», «Media» o «Baja» (y `""` para valores desconocidos), para usarlo desde Plush.

### Acciones (`actions/tasks.go`)

- Helper privado con la ordenación, para no duplicarla en `TasksIndex` y en la rama de error de `TasksCreate`:
  ```go
  const tasksOrder = "CASE priority WHEN 'alta' THEN 0 WHEN 'media' THEN 1 ELSE 2 END, created_at asc, id asc"
  ```
  Sustituye las dos apariciones de `"created_at asc, id asc"` por `tasksOrder`.
- `TasksIndex`: `c.Set("task", &models.Task{Priority: models.PriorityMedium})` para que el selector salga con «Media» preseleccionada.
- `TasksCreate`: no cambia la lógica. `c.Bind(task)` recoge `priority` gracias a la etiqueta `form`. Aplica `strings.TrimSpace` a `task.Priority`, igual que al título. Si la validación falla, se re-renderiza `index` con `422` como ahora.
- `TasksUpdate`: el struct de entrada pasa a ser `{ Title string `form:"title"`; Priority string `form:"priority"` }`. Tras recortar espacios: `if input.Priority != "" { task.Priority = input.Priority }`. Si una prioridad inválida produce error de validación, se re-renderiza `edit` con `422`. La tarea no se persiste porque `ValidateAndUpdate` no escribe si hay errores. Actualizar el comentario de la función: «actualiza el título y la prioridad…».
- `TasksToggle` y `TasksDestroy` no cambian. `tx.Update` conserva la prioridad cargada.

### Rutas

Sin cambios en `actions/app.go`.

### Renderizado (Plush)

`templates/tasks/index.plush.html`:

- En cada `<li class="task">`, envolver el título en un contenedor para no romper la rejilla de tres columnas de `.task`:
  ```html
  <div class="task-main">
    <span class="task-title"><%= t.Title %></span>
    <span class="priority-badge priority-<%= t.Priority %>">
      <span class="sr-only">Prioridad </span><%= t.PriorityLabel() %>
    </span>
  </div>
  ```
  `.task-title` debe seguir siendo un `span` con ese nombre de clase, porque `app.js` lee su `textContent` para el `aria-label`. El `<span class="task-status">` y `.task-actions` se quedan como hermanos directos del `<li>`, porque `app.js` inserta el estado antes de `.task-actions`.
- En el formulario de creación, dentro de `.new-task-row` y entre el input y el botón «Añadir»:
  ```html
  <label for="task-priority" class="sr-only">Prioridad</label>
  <select id="task-priority" name="priority" <%= if (len(errors.Get("priority")) > 0) { %>aria-invalid="true" aria-describedby="priority-error"<% } %>>
    <option value="alta" <%= if (task.Priority == "alta") { %>selected<% } %>>Alta</option>
    <option value="media" <%= if (task.Priority == "media") { %>selected<% } %>>Media</option>
    <option value="baja" <%= if (task.Priority == "baja") { %>selected<% } %>>Baja</option>
  </select>
  ```
- Errores: separar los bloques de error por campo. El de título se muestra si `len(errors.Get("title")) > 0` (y el `aria-invalid`/`autofocus` del input de título pasa a depender de esa misma condición en vez de `errors.HasAny()`). Se añade `<p id="priority-error" class="field-error" role="alert">` con los mensajes de `errors.Get("priority")` cuando existan.

`templates/tasks/edit.plush.html`:

- Tras el bloque del título (hint + error), añadir `<label for="task-priority">Prioridad</label>` y el mismo `<select>` con la opción actual de `task.Priority` preseleccionada. Añadir su bloque `priority-error`.
- Cambiar el `aria-invalid`/`aria-describedby` del input de título para que dependa de `len(errors.Get("title")) > 0` en vez de `errors.HasAny()`. Así sigue cumpliéndose `Test_TasksUpdate_EmptyTitle`.
- Ajustar el texto de ayuda `#title-help` a «Hasta 200 caracteres.», porque ya no es cierto que «los demás datos no cambian» desde este formulario.

### CSS (`public/assets/app.css`)

- `.task-main`: `display: flex; flex-wrap: wrap; align-items: center; gap: 0.5rem; min-width: 0; align-self: center;`. `.task-title` pierde `align-self: center` porque lo hereda del contenedor y mantiene `overflow-wrap`.
- `.priority-badge`: pastilla pequeña (`font-size: 0.75rem; font-weight: 600; padding: 0.1rem 0.5rem; border-radius: 999px; white-space: nowrap;`).
- Variantes con contraste AA sobre fondo claro, usando tokens existentes cuando encajen:
  - `.priority-alta` → color `var(--color-danger)` sobre `#fbeceb`.
  - `.priority-media` → `#7a5a00` sobre `#fbf3dc`.
  - `.priority-baja` → `var(--color-muted)` sobre `var(--color-sidebar)`.
- `.task-completed .priority-badge { opacity: 0.6; }` para que no compita con las tareas pendientes.
- `select`: mismas reglas base que `input[type="text"]` (alto mínimo 44px, borde, radio, `font: inherit`), añadiendo `select` a ese selector, y `select[aria-invalid="true"]` al de error. En `.new-task-row select { flex: 0 0 auto; width: auto; }`. En el bloque `@media (max-width: 380px)` existente el wrap ya cubre el reparto; comprobarlo visualmente.

## Archivos relevantes

| Ruta | Responsabilidad |
|---|---|
| `migrations/20261014000000_add_priority_to_tasks.up.fizz` | **Nuevo.** Añade la columna `priority` con DEFAULT y CHECK. |
| `migrations/20261014000000_add_priority_to_tasks.down.fizz` | **Nuevo.** Elimina la columna. |
| `models/task.go` | Campo `Priority`, constantes, validación, `BeforeSave` y `PriorityLabel`. |
| `models/task_test.go` | Tests de modelo de prioridad. |
| `actions/tasks.go` | Orden por prioridad, valor por defecto del formulario, bind en create/update. |
| `actions/tasks_test.go` | Tests de acciones de prioridad. |
| `templates/tasks/index.plush.html` | Etiqueta en la lista y selector en el formulario de creación. |
| `templates/tasks/edit.plush.html` | Selector en el formulario de edición. |
| `public/assets/app.css` | Estilos de la etiqueta y del selector. |
| `fixtures/tasks.toml` | Prioridades explícitas en el escenario `tres tareas`. |
| `migrations/embed.go`, `actions/app.go`, `public/assets/app.js` | Referencia; **no cambian**. |

## Implementación paso a paso

1. Crear las dos migraciones fizz descritas en *Persistencia y migraciones*.
2. En `models/task.go`, añadir constantes, campo, validación, `BeforeSave` y `PriorityLabel`.
3. En `fixtures/tasks.toml`, añadir `priority` a las tres filas de `tres tareas`: «Comprar material de oficina» → `media`, «Llamar al gestor» → `alta`, «Publicar la oferta de empleo» → `baja`.
4. En `actions/tasks.go`, añadir `tasksOrder` y usarlo en los dos sitios. Preseleccionar `media` en `TasksIndex`, recortar `Priority` en `TasksCreate` y ampliar el input de `TasksUpdate` con la regla «solo si viene informada». Actualizar el comentario.
5. Actualizar `templates/tasks/index.plush.html` y `templates/tasks/edit.plush.html` según *Renderizado*.
6. Añadir los estilos a `public/assets/app.css`.
7. Escribir los tests de *Tests y validación*.
8. Preparar la BD de test y ejecutar las validaciones.
9. Reconstruir el entorno (`docker compose up --build -d`) y comprobar en el navegador `http://localhost:3000`: crear con cada prioridad, editar, completar y ver el orden y las etiquetas a ancho de escritorio y a 375px.

## Tests y validación

### Tests de modelo (`models/task_test.go`)

- `Test_Task_Priority_DefaultsToMedium`: crear `Task{Title: "x"}` → `task.Priority == PriorityMedium` y, tras `Find`, también en BD.
- `Test_Task_Priority_AcceptsAllowedValues`: `alta`, `media` y `baja` validan sin errores.
- `Test_Task_Priority_Invalid`: `Priority: "urgente"` → `verrs.Get("priority")` contiene «La prioridad no es válida.» y el recuento es 0.
- `Test_Task_PriorityLabel`: devuelve «Alta», «Media» y «Baja».

### Tests de acciones (`actions/tasks_test.go`)

- `Test_TasksCreate_WithPriority`: `Post(&models.Task{Title: "x", Priority: "alta"})` → 303 y la prioridad persistida es `alta`.
- `Test_TasksCreate_DefaultPriority`: el `Test_TasksCreate_Valid` existente envía `priority=""`; añadir `as.Equal(models.PriorityMedium, task.Priority)` en un test nuevo, o extender ese.
- `Test_TasksCreate_InvalidPriority`: `priority=urgente` → 422, el cuerpo contiene «La prioridad no es válida.» y el recuento es 0.
- `Test_TasksIndex_ShowsPriority`: con el fixture, el cuerpo contiene `priority-alta` y «Alta», y el selector de creación tiene `<option value="media" selected>`.
- `Test_TasksIndex_OrdersByPriority`: con el fixture, el índice de «Llamar al gestor» (alta) < «Comprar material de oficina» (media) < «Publicar la oferta de empleo» (baja), usando `strings.Index` sobre el cuerpo.
- `Test_TasksEdit_PreselectsPriority`: tarea `baja` → el cuerpo contiene `<option value="baja" selected>`.
- `Test_TasksUpdate_ChangesPriority`: tarea completada con prioridad media, `PUT` con `title` y `priority=alta` → 303, prioridad `alta` y `Completed` sigue `true`.
- `Test_TasksUpdate_WithoutPriorityKeepsIt`: tarea `alta`, `PUT` solo con `title` → la prioridad sigue `alta`.
- `Test_TasksUpdate_InvalidPriority`: `priority=urgente` → 422, el cuerpo contiene el mensaje y la tarea queda sin cambios (título y prioridad).
- `Test_TasksToggle_KeepsPriority`: tarea `baja`, toggle → la prioridad sigue `baja`.

Los tests existentes deben seguir pasando sin cambios, en particular `Test_TasksUpdate_PersistsTitleOnly` y `Test_TasksUpdate_EmptyTitle`.

> Nota de implementación: comprobar con el HTML real cómo Plush serializa el atributo `selected` (por ejemplo `value="media" selected>` frente a un espacio extra) y ajustar la aserción a la salida exacta, sin cambiar el comportamiento.

### Preparación de la BD de test

Los tests necesitan PostgreSQL en `127.0.0.1:5432` (el servicio `db` de compose ya está levantado) y la base `todo_test` migrada, igual que en CI (`.github/workflows/ci.yaml`):

```bash
docker compose exec db createdb -U postgres todo_test
```

(Solo si no existe; si ya existe, el comando falla sin efectos.)

```bash
go install github.com/gobuffalo/pop/v6/soda@v6.4.1
```

```bash
GO_ENV=test soda migrate -e test
```

### Comandos de validación

```bash
gofmt -l .
```

(debe salir vacío)

```bash
go vet ./...
```

```bash
GO_ENV=test go test -p 1 ./...
```

```bash
go build ./...
```

Además, `soda migrate down -e test` seguido de `soda migrate -e test` para comprobar que la migración es reversible.

## Riesgos y recuperación

- **Migración sobre datos existentes.** El DEFAULT `'media'` rellena las filas existentes; el CHECK no puede fallar sobre ellas. Si algo falla en desarrollo, `soda migrate down` elimina la columna sin tocar los demás datos. No hay pérdida de datos previos.
- **Volumen de desarrollo.** La app aplica la migración al arrancar. Tras reconstruir, la BD `todo_development` del volumen `db-data` queda con la columna nueva. Para volver atrás hay que ejecutar la migración down **antes** de volver a una versión de código sin el campo (pop no falla por columnas extra en `SELECT` explícito, pero conviene mantenerlas alineadas). No eliminar el volumen.
- **Mutación en `BeforeSave`.** Si en el futuro se usa `tx.Update` con un `Task` construido a mano sin prioridad, quedaría `media` en lugar de conservar la existente. Hoy todas las rutas cargan la tarea con `Find` antes de actualizarla, así que no ocurre.
- **Orden de la lista.** Pasar a ordenar por prioridad es una inferencia que cambia el orden visible actual. Si no se desea, basta con revertir `tasksOrder` a `"created_at asc, id asc"` y eliminar `Test_TasksIndex_OrdersByPriority`. El resto de la feature es independiente.
- **Rejilla de la fila.** Envolver el título en `.task-main` puede afectar al alineado y al toggle optimista de `app.js`. Mitigación: mantener `.task-title` como `span` y `.task-status`/`.task-actions` como hijos directos de `.task`, y comprobar visualmente el toggle y los anchos 760px y 380px.
- **`errors.HasAny()` en plantillas.** Cambiar la condición a errores por campo evita marcar el título como inválido por un error de prioridad. Riesgo bajo, cubierto por `Test_TasksUpdate_EmptyTitle` y `Test_TasksCreate_EmptyTitle_Invalid`.
