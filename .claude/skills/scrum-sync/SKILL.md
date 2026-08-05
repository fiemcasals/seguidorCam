---
name: scrum-sync
description: Sincroniza lo implementado en este repo con los Requerimientos de un proyecto en Scrum Master AI — lee la documentación local, decide con criterio qué requerimientos quedaron cubiertos, y actualiza esos Requerimientos y sus Tests vía la API. Si algo documentado no matchea ningún Requerimiento existente, puede crear uno nuevo (developer o Project Manager, con confirmación explícita) colgado de la Historia de Usuario que corresponda. También puede decidir sola cuál es el siguiente Requerimiento a encarar leyendo el plan publicado en la rama principal, y dejar la rama/commit/push listos (el tiempo real trabajado se calcula solo, a partir de ahí: arranca con ese primer push y se congela al abrir el Pull Request — este skill no lo controla). Usar cuando el usuario pide "sincronizar con scrum", "reportar al scrum master", "actualizar requerimientos", "crear un requerimiento", "avisarle al scrum lo que hice", "qué sigue", o corre /scrum-sync explícitamente.
user-invocable: true
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash(curl *)
  - Bash(git *)
  - Write
---

# /scrum-sync — Reportar avance a Scrum Master AI

Lee lo que este repo ya documenta como implementado, lo compara contra los Requerimientos
del proyecto en Scrum Master AI, y actualiza esos mismos Requerimientos/Tests vía la API
`/api/v1/*` — para que el programador no tenga que reportar nada a mano. El Requerimiento
es la unidad atómica (RF-01, RNF-01, etc.): no hay ningún nivel intermedio tipo
"Funcionalidad". La mayoría de las veces se actualiza un Requerimiento que ya existe (lo
cargó el Product Owner); si algo documentado no matchea ninguno, este skill también puede
crear el que falta (ver paso 4.5), colgado de la Historia de Usuario que corresponda, con
confirmación explícita del usuario antes de cada alta. También puede decidir cuál es el
siguiente Requerimiento a encarar.

**El tiempo real trabajado (`real_time`) no lo controla este skill ni ningún botón manual.**
Se calcula exclusivamente a partir de git: arranca en el primer push a la rama del
Requerimiento y se congela al abrir el Pull Request hacia la rama base — automático vía
webhook o GitHub Action, según el proveedor del proyecto. No existe (ni acá ni en la app)
una forma de arrancar/parar el cronómetro a mano.

Argumentos: `$ARGUMENTS`. Dos formas:
- **(vacío) o una ruta** → sincronizar documentación (comportamiento por defecto, ver
  sección "Sincronizar documentación" más abajo). Si se pasa una ruta, se usa esa en vez
  de `docs/`.
- **`siguiente`** → decidir cuál Requerimiento encarar ahora, crear/retomar su rama y
  dejar el primer commit (o el de retoma) pusheado.

---

## 0. Identidad y credenciales (aplica a las dos formas)

- `SCRUM_API_KEY` — variable de entorno. Si no está seteada: explicar que hay que
  pedírsela al admin del proyecto (la genera desde "Gestión de Accesos" en la app) y
  exportarla en el shell (`export SCRUM_API_KEY=sk_...`), y parar acá. **Nunca** escribir
  esta key a ningún archivo del repo.
- `SCRUM_API_URL` — la base de la instancia (ej. `https://scrum.tudominio.com`). **No
  es secreta y no hay que preguntarla todavía**: se resuelve en el paso 1 leyendo el
  manifest del repo. Sólo si el manifest tampoco la tiene se llega a pedírsela al
  usuario.

Todas las llamadas a la API llevan `-H "Authorization: Bearer $SCRUM_API_KEY"`.

## 1. Leer o inicializar el manifest (aplica a las dos formas)

Leer `docs/scrum-manifest.json` en la raíz del repo. Si no existe, crearlo:

```json
{
  "apiUrl": null,
  "projectId": null,
  "lastSyncAt": null,
  "mappings": []
}
```

- **`apiUrl`**: a diferencia de la key, no es secreta — vive commiteada en el repo.
  Si el archivo ya la trae, usarla tal cual y no volver a preguntar. Si falta, antes de
  preguntarle nada al usuario, revisar si existe `docs/SCRUM_MASTER_AI.md` — el Project
  Manager la publica ahí ya resuelta (`SCRUM_API_URL es <url>`) porque el servidor la saca
  sola de su propia URL pública; si está, usar ese valor. Sólo si ninguna de las dos
  fuentes la tiene, preguntarle la URL al usuario. En cualquier caso, guardarla en el
  manifest para que el resto del equipo (developer/PO/QA que clonen este mismo repo) no
  tenga que repetirla nunca.
- **`projectId`**: si falta, no preguntarlo a ciegas todavía — se resuelve en el paso 1.5
  contra los proyectos que devuelve `/api/v1/me`.

Este archivo es el estado de trazabilidad — no decide qué existe (eso ya lo sabe la API,
los Requerimientos los crea el Project Manager), pero guarda `sourceRef` para que una
relectura futura entienda por qué se marcó cubierto cada Requerimiento. Recomendarle al
usuario commitearlo al repo (no tiene secretos, sólo IDs y la URL).

## 1.5. Confirmar identidad y rol (aplica a las dos formas)

```bash
curl -s "$SCRUM_API_URL/api/v1/me" -H "Authorization: Bearer $SCRUM_API_KEY"
```

- `401` → la key es inválida o fue revocada. Avisar al usuario que le pida al admin que
  le genere una nueva, y parar.
- `200` → `{ id, username, email, role, projects: [{ id, name }, ...] }`. Esto es lo que
  determina el rol de verdad — **nunca preguntarle al usuario "qué rol sos" ni asumirlo**,
  el rol de la cuenta dueña de la key es el único que importa (y la API lo vuelve a
  validar en cada llamada de todos modos, así que confiar en otra cosa acá no cambiaría
  nada salvo dar un error más tarde y más confuso).
  - Si `role` no es `developer` ni `project_manager`, avisar que esta key no corresponde a
    este skill (`/po-sync` es para `product_owner`, `/qa-sync` para `tester`/`qa`) y
    sugerir el skill correcto en vez de seguir adelante.
  - Si el manifest no tenía `projectId`: si `projects` trae un solo elemento, usar ese
    `id` directamente sin preguntar; si trae varios, listarlos y preguntar cuál; si viene
    vacío, avisar que el Project Manager todavía no agregó a este usuario a ningún
    proyecto, y parar. Guardar el `projectId` elegido en el manifest.

---

## Dispatch según `$ARGUMENTS`

### Sin argumentos, o una ruta — sincronizar documentación

#### 2. Traer las Historias de Usuario y los Requerimientos del proyecto

```bash
curl -s "$SCRUM_API_URL/api/v1/projects/$PROJECT_ID/user-stories" \
  -H "Authorization: Bearer $SCRUM_API_KEY"
curl -s "$SCRUM_API_URL/api/v1/projects/$PROJECT_ID/requirements" \
  -H "Authorization: Bearer $SCRUM_API_KEY"
```

- `401` → la key es inválida o fue revocada. Avisar al usuario que le pida al admin
  que le genere una nueva, y parar.
- `403` → la key es válida pero el usuario dueño no pertenece a ese proyecto. Avisar
  que confirme el `projectId` con el admin, y parar.
- `200` en ambas → Historias de Usuario trae `{ id, code, name, description,
  acceptanceCriteria, ... }` (hace falta para el paso 4.5, si hay que crear un
  Requerimiento nuevo). Requerimientos trae `{ id, code, userStoryId, name, description,
  type, status, ... }` (`type` es `funcional` o `no_funcional`; `status` es el estado
  Kanban actual: `to_do`/`doing`/`testing`/`done`/`blocked`). Guardar ambas listas en
  memoria, son la base contra la que se va a razonar en el paso siguiente.

#### 3. Leer la documentación local y decidir qué requerimiento cubre cada cosa

Leer `$ARGUMENTS` si se pasó una ruta explícita (si no existe, avisar y no asumir
`docs/` en su lugar); si no hay argumento, leer todo `docs/`; si `docs/` no existe,
leer `README.md`.

**Esto es el corazón del skill y es un trabajo de criterio, no de matching de texto.**
No busques que el nombre del requerimiento aparezca literal en el doc. Leé la
documentación como lo haría un humano familiarizado con el proyecto: entendé qué
funcionalidad describe cada sección/endpoint/feature documentado, y decidí — comparando
contra la `description` de cada requerimiento, no sólo el `name` — cuáles quedaron
cubiertos. Si tenés dudas razonables sobre un match, es preferible dejarlo afuera
(reportarlo como "sin cobertura clara" en el resumen final) a inventar una relación —
distinto es cuando estás razonablemente seguro de que no matchea ningún Requerimiento
existente porque genuinamente no estaba trackeado: eso es candidato al paso 4.5, no
"cobertura dudosa".

Para cada match, quedate con una referencia corta a la fuente (`sourceRef`, ej.
`docs/api.md#POST /login` o `README.md#Autenticación`) — se guarda en el manifest y
sirve para que una relectura futura entienda por qué se marcó cubierto.

#### 4. Actualizar el Requerimiento cubierto

Para cada requerimiento con match, actualizarlo directamente (ya existe, normalmente
lo cargó el Product Owner):

```bash
curl -s -X PATCH "$SCRUM_API_URL/api/v1/requirements/$REQUIREMENT_ID" \
  -H "Authorization: Bearer $SCRUM_API_KEY" -H "Content-Type: application/json" \
  -d '{"status":"done","observations":"<qué se encontró implementado y dónde (sourceRef)>"}'
```

- No fuerces `"status":"done"` si el Requerimiento está `to_do`/`doing` sin rama todavía
  — recién puede pasar a `done` si genuinamente ya está mergeado; si sólo encontraste
  evidencia parcial, dejá el `status` afuera del PATCH (no lo toques) y volcá el detalle
  en `observations` nomás.
- Guardar en el manifest una entrada `{ requirementId, sourceRef, testIds }` (crear si
  es la primera vez que se matchea ese Requerimiento, actualizar `sourceRef` si ya
  existía).
- **Sólo si encontrás evidencia real de tests en el repo** (archivos de test existentes
  que cubren ese requerimiento — nunca inventar esto) crear un Test:
  ```bash
  curl -s -X POST "$SCRUM_API_URL/api/v1/requirements/$REQUIREMENT_ID/tests" \
    -H "Authorization: Bearer $SCRUM_API_KEY" -H "Content-Type: application/json" \
    -d '{"title":"<nombre del test>","isAutoGenerated":true,"status":"Aprobado"}'
  ```
  Guardar el `id` (`TEST-...`) en `testIds` de esa entrada del manifest.
  Si el repo tiene `docs/tests-manifest.json` (ver paso 5), no crear tests sueltos acá
  para lo que ya esté cubierto por ese archivo — dejarle el trabajo al paso 5, que es
  más rico (guarda los pasos de verificación, no sólo el nombre).

#### 4.5. Crear un Requerimiento que no existe todavía (con confirmación explícita)

Dos disparadores posibles:
- Algo que la documentación describe con claridad no matchea ningún Requerimiento del
  paso 2 -- genuinamente no estaba trackeado, no es un caso dudoso de cobertura.
- El usuario pide directamente, en la conversación, crear un Requerimiento puntual (por
  nombre/descripción), sin pasar por el flujo de sincronización de documentación.

En cualquiera de los dos casos, antes de crear nada:

1. **Resolver a qué Historia de Usuario cuelga**, comparando contra `name` +
   `description` + `acceptanceCriteria` de las Historias del paso 2 (nunca por
   coincidencia literal de texto). Si no hay ninguna Historia razonable, **no la
   inventes** — avisar que hace falta que el Product Owner (o vos mismo, si tenés
   permiso) cargue esa Historia primero, y no crear el Requerimiento suelto.
2. **Confirmarle al usuario, antes de llamar a la API**: nombre propuesto, tipo
   (`funcional`/`no_funcional`) y bajo qué Historia va a quedar. Esperar confirmación
   explícita -- a diferencia de actualizar un Requerimiento existente (reversible con
   otra corrida), crear uno de más ensucia el backlog y hay que borrarlo a mano después.
3. Con la confirmación:
   ```bash
   curl -s -X POST "$SCRUM_API_URL/api/v1/user-stories/$USER_STORY_ID/requirements" \
     -H "Authorization: Bearer $SCRUM_API_KEY" -H "Content-Type: application/json" \
     -d '{"name":"...","description":"...","type":"funcional"}'
   ```
   - `403` → la cuenta no tiene permiso para crear (developer, product_owner y project
     manager sí pueden; tester/qa no). Avisar tal cual y no reintentar.
   - `201` → guardar en el manifest una entrada `{ requirementId, sourceRef, testIds: [] }`
     igual que en el paso 4, para que una relectura futura no lo vuelva a crear.

#### 5. Sincronizar `docs/tests-manifest.json` (tests de endpoint)

Si el repo tiene `docs/tests-manifest.json`, es la fuente de tests de endpoint que el
programador mantiene junto con su código — un archivo por proyecto, con un test por
endpoint/flujo, pensado para que QA los corra desde "Verificación en vivo" en la app sin
tener que tipear URLs a mano. Formato:

```json
{
  "tests": [
    {
      "requirementCode": "RF-03",
      "title": "Alta de profesional",
      "type": "Integración",
      "preconditions": "Usuario autenticado con rol admin",
      "description": "Autentica como admin, da de alta un profesional nuevo y confirma que se puede volver a leer.",
      "expectedResult": "Devuelve 201 y el profesional creado con id",
      "steps": [
        { "name": "crear", "method": "POST", "url": "{{baseUrl}}/api/professionals", "body": "{\"name\":\"Juan\"}", "expectedStatus": 201 },
        { "name": "leer", "method": "GET", "url": "{{baseUrl}}/api/professionals/{{crear.id}}", "expectedStatus": 200 }
      ]
    }
  ]
}
```

- `requirementCode` se resuelve contra la lista de Requerimientos traída en el paso 2
  (por `code`, ej. `RF-03`, nunca por nombre). Si no matchea ninguno, dejarlo afuera y
  avisar en el resumen final — no crear el Requerimiento ni adivinar cuál es.
- `type` usa los mismos valores que ya existen en la app: `Unitario`, `Integración`, `E2E`.
- `description`: en prosa, el flujo que describe el test (qué hace y en qué orden) — es
  el campo que la app muestra como "Descripción / Flujo de Ejecución" en cada test.
  **Siempre completarlo** — no se infiere de `steps`, y si falta queda en blanco en la app.
- `steps` es una lista de llamadas HTTP en orden (`name`, `method`, `url`, `body`
  opcional, `expectedStatus` opcional). Un paso puede reusar el resultado de uno anterior
  con `{{nombreDelPaso.campo}}` (o `{{nombreDelPaso.status}}`), y `{{now}}` da un valor
  único por corrida. **No resolver `{{baseUrl}}` acá ni pedirle la URL real al usuario**
  — lo resuelve la app cuando QA corre el test, contra la Base URL de verificación que el
  Project Manager ya configuró para el proyecto (este skill no necesita conocerla).
- No confiar sólo en el `testId` de `docs/scrum-manifest.json` para saber si el test ya
  existe — ese archivo puede faltar, no estar commiteado, o venir de otro clon. Antes de
  crear, traer los tests que la API ya tiene registrados para este Requerimiento y
  matchear por `title` exacto:
  ```bash
  curl -s "$SCRUM_API_URL/api/v1/requirements/$REQUIREMENT_ID/tests" \
    -H "Authorization: Bearer $SCRUM_API_KEY"
  ```
- Si ya existe (por el manifest o por esa respuesta), actualizarlo en vez de crear uno
  nuevo — con esto también se puede refrescar el contenido, no sólo los pasos, por si el
  endpoint cambió desde la última corrida:
  ```bash
  curl -s -X PATCH "$SCRUM_API_URL/api/v1/tests/$TEST_ID" \
    -H "Authorization: Bearer $SCRUM_API_KEY" -H "Content-Type: application/json" \
    -d '{"preconditions":"...","description":"...","expectedResult":"...","verification":{"steps":[...]}}'
  ```
- Si no existe en ninguna de las dos, crearlo con una sola llamada (test + pasos juntos,
  **incluyendo siempre `description`**):
  ```bash
  curl -s -X POST "$SCRUM_API_URL/api/v1/requirements/$REQUIREMENT_ID/tests" \
    -H "Authorization: Bearer $SCRUM_API_KEY" -H "Content-Type: application/json" \
    -d '{"title":"...","type":"Integración","preconditions":"...","description":"...","expectedResult":"...","isAutoGenerated":true,"verification":{"steps":[...]}}'
  ```
  y guardar el `id` devuelto (o el que salió de la reconciliación) en `testIds` de esa
  entrada del manifest de trazabilidad.
- **Nunca mandar `status` en `Aprobado`/`Fallido` desde acá** — crear/actualizar deja el
  test en `Pendiente`; que alguien haya escrito los pasos no significa que ya se
  corrieron y se revisaron. Eso lo decide QA corriéndolos desde la app.
- Si la reconciliación contra la API encuentra más de un test con el mismo `title` para el
  mismo Requerimiento (duplicados de corridas viejas), no elegir uno a ciegas: reportarlo
  en el resumen final para que QA decida cuál borrar
  (`DELETE $SCRUM_API_URL/api/v1/tests/$TEST_ID`).

#### 6. Guardar el manifest actualizado

Reescribir `docs/scrum-manifest.json` con `lastSyncAt` en la fecha/hora actual (ISO) y
todas las entradas de `mappings` (viejas + nuevas, incluyendo los `testIds` del paso 5).

#### 7. Resumen final

Reportarle al usuario, en texto, no en JSON crudo:
- Cuántos Requerimientos se actualizaron (y a qué estado, si cambió).
- Cuántos Requerimientos se crearon (paso 4.5), y bajo qué Historia de Usuario cada uno.
- Cuántos Tests se crearon o actualizaron desde `docs/tests-manifest.json`.
- Qué Requerimientos quedaron sin cobertura clara (para que sepa qué falta implementar o
  documentar mejor).

### `siguiente` — decidir qué Requerimiento encarar y dejarlo listo para trabajar

Este modo asume el flujo: pull a la rama principal → leer el plan → decidir con
criterio cuál sigue → asegurar la rama → commit + push. El push dispara solo (webhook o
GitHub Action, según el proveedor del proyecto) el pase de "Hacer" a "Haciendo" en el
Kanban, y arranca ahí mismo el cómputo de tiempo real trabajado — no hace falta tocar
nada a mano en la app.

1. **Traer el plan actualizado**: correr `git pull` sobre la rama principal del repo (si
   el working tree tiene cambios sin commitear, avisar y parar — no pisar trabajo en
   curso). Leer `docs/scrum-plan.md`. Si no existe, avisar que el Project Manager
   todavía no publicó el plan desde la app ("Publicar Plan") y parar.
2. **Elegir el Requerimiento**: el archivo trae una tabla ya ordenada por dependencias
   (columna "Orden") con columnas Código/Estado/Desarrollador/Depende de/Rechazos. Con
   criterio, elegir la primera fila que cumpla:
   - Estado no es `Hecho ✓ Visado` ni `Hecho` (ya está en revisión, no hay nada para
     arrancar).
   - Todos los Requerimientos listados en "Depende de" ya están en `Hecho ✓ Visado`.
   - Si la columna "Desarrollador" tiene nombres cargados, preferir uno asignado al
     usuario actual (`git config user.name` o preguntar) antes que uno sin asignar o de
     otra persona.
   - A igualdad de las condiciones anteriores, preferir una fila con "Rechazos" > 0
     (retrabajo pendiente: ya se encaró antes y volvió con observaciones) por sobre una
     que nunca se tocó — es la más urgente de resolver.
   Si hay empate real entre varias candidatas razonables, preguntarle al usuario cuál
   prefiere — no adivinar.
3. **Asegurar la rama** (idempotente, se puede llamar aunque ya exista). Primero
   consultar `GET $SCRUM_API_URL/api/v1/projects/$PROJECT_ID` para saber `vcsProvider`
   del proyecto (`github` o `gitlab`), y pegarle al endpoint que corresponda:
   ```bash
   curl -s -X POST "$SCRUM_API_URL/api/v1/requirements/$REQUIREMENT_ID/github/branch" \
     -H "Authorization: Bearer $SCRUM_API_KEY"
   # o, si vcsProvider es 'gitlab':
   curl -s -X POST "$SCRUM_API_URL/api/v1/requirements/$REQUIREMENT_ID/gitlab/branch" \
     -H "Authorization: Bearer $SCRUM_API_KEY"
   ```
   - `{"alreadyExists":false,...,"checkoutCommand":"git fetch && git checkout <rama>"}`
     → rama nueva, recién creada en el repo a partir de la rama base del proyecto.
   - `{"alreadyExists":true,...}` → ya existía (se está retomando trabajo previo).
   - `409` → el proyecto no tiene el repositorio configurado; avisar y parar.
   Ejecutar el `checkoutCommand` devuelto tal cual.
4. **Commit + push**:
   - Si la rama era nueva: hacer un commit inicial marcador (ej. mensaje
     `"Inicio de trabajo en $REQUIREMENT_ID: <nombre>"`, aunque sea vacío con
     `git commit --allow-empty` si todavía no hay cambios de código) y
     `git push -u origin <rama>`. Ese primer push es lo que arranca el cómputo de
     tiempo real trabajado en el servidor.
   - Si ya existía: si hay cambios locales sin commitear, commitearlos con un mensaje
     tipo `"Retomo $REQUIREMENT_ID: <nombre>"`; si no hay nada para commitear, hacer un
     commit vacío con el mismo mensaje. Después `git push`.
   - **Si el Requerimiento no tiene código asociado** (ej. una No Funcional de
     configuración/política, como "qué tipo de seguridad se adoptó"): el commit igual
     tiene que llevar algo tangible — un archivo de documentación en el propio repo
     describiendo la decisión tomada (no un commit vacío sin explicación). Usar criterio
     sobre dónde documentarlo (`docs/`, un README de la carpeta relevante, etc.).
5. Confirmarle al usuario qué Requerimiento quedó eligiendo, en qué rama, y que el push
   ya debería haber movido la tarjeta a "Haciendo" en la app y arrancado el cómputo de
   tiempo (no hace falta verificarlo, es automático vía webhook/Action). Recordarle que
   ese tiempo se congela solo cuando abra el Pull Request hacia la rama base — no hay
   nada que tenga que "parar" a mano.

---

## Notas de implementación

- Nunca crear un Requerimiento (paso 4.5) sin haberle confirmado antes al usuario nombre,
  tipo e Historia de Usuario destino, y sin haber recibido una confirmación explícita --
  a diferencia de actualizar uno existente, crear de más ensucia el backlog.
- Nunca inventar una Historia de Usuario para colgar un Requerimiento nuevo -- si no hay
  ninguna razonable, avisar y no crear el Requerimiento suelto.
- Nunca marcar `isAutoGenerated`/crear un Test sin evidencia real de que existe en el
  repo — es preferible no reportar cobertura de tests a inventarla.
- Nunca fuerces `status` a `done` sólo porque encontraste documentación que lo describe
  — `done` implica que el Pull Request ya se mergeó (lo mueve solo el webhook/Action).
  Si la evidencia es de código ya escrito pero no mergeado, dejá que el flujo de git
  mueva el estado y usá `observations` para dejar constancia de lo que se encontró.
- Si `$SCRUM_API_URL` tiene un `/` final, quitarlo antes de concatenar rutas.
- Todas las respuestas de error de la API vienen como `{"error": "..."}` — mostrar ese
  mensaje tal cual, no reinterpretarlo.
- `docs/scrum-plan.md` lo publica el Project Manager desde la app ("Publicar Plan") —
  este skill sólo lo lee, nunca lo escribe ni lo edita.
- `docs/tests-manifest.json` es al revés: lo escribe el programador (o este skill en su
  nombre) en el repo del proyecto, y este skill lo lee para sincronizar. Nunca inventar
  entradas ahí — sólo reflejar endpoints que realmente existen en el código.
