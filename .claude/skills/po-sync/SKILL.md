---
name: po-sync
description: Sincroniza lo que el Product Owner (o el Project Manager, mismo alcance de API) redacta en este repo -- documentos en lenguaje natural, no un formato rígido -- con las Historias de Usuario y los Requerimientos de un proyecto en Scrum Master AI. Lee docs/historias-po.md para crear/editar/borrar Historias de Usuario. Lee docs/requerimientos-po.md para editar o borrar Requerimientos ya existentes colgados de esas Historias -- crear un Requerimiento nuevo es territorio de /scrum-sync (developer/Project Manager), este skill sólo reporta los que faltan. Nunca toca tests, tiempos, ramas ni estados de ejecución. Usar cuando el usuario pide "sincronizar requerimientos", "cargar historias de usuario", "cargar lo que escribí como Product Owner", "actualizar el backlog", "borrar una historia/requerimiento", o corre /po-sync explícitamente.
user-invocable: true
allowed-tools:
  - Read
  - Grep
  - Glob
  - Bash(curl *)
  - Write
---

# /po-sync — Sincronizar Historias de Usuario y Requerimientos redactados por el Product Owner (o Project Manager)

Lee documentos que el Product Owner (o el Project Manager, si es su cuenta la dueña de la
key) mantiene en este repo, en lenguaje natural (no un formato rígido): uno para Historias
de Usuario (`docs/historias-po.md`) y otro para Requerimientos (`docs/requerimientos-po.md`).
Los compara contra lo que ya existe en Scrum Master AI y sincroniza ambos niveles vía
`/api/v1/*` — para que no haya que entrar a la web a cargarlos a mano.

**Alcance**: este skill crea/edita/borra Historias de Usuario (nombre, descripción,
criterios de aceptación, detalle técnico) y edita/borra Requerimientos ya existentes
colgados de ellas (nombre, descripción, tipo) -- **crear un Requerimiento nuevo no es
territorio de este skill**, eso lo hace developer/Project Manager con `/scrum-sync`; acá
sólo se reporta al final cuál falta crear y bajo qué Historia. Nunca toca status,
tiempos, asignado, dependencias, tests ni ramas — eso es territorio de developer (la API
lo rechaza con 403 si una cuenta Product Owner lo intenta; una cuenta Project Manager sí
podría técnicamente, pero este skill nunca lo hace por diseño, no por límite de la API).

**Borrar es distinto a crear/editar: pedí confirmación explícita antes de cada DELETE**,
incluso si el documento parece indicarlo con claridad — a diferencia de crear/editar (que
son reversibles con otra corrida), borrar no lo es. Borrar una Historia de Usuario borra
en cascada todos sus Requerimientos: pedí una confirmación todavía más explícita en ese
caso (ver paso 2.5).

Argumentos: `$ARGUMENTS` — opcionalmente una ruta al documento de Requerimientos a leer
(paso 3). Si no se pasa, default a `docs/requerimientos-po.md`. El documento de Historias
(`docs/historias-po.md`, paso 2.5) no se parametriza por argumento -- si no existe, ese
paso simplemente se saltea.

---

## 0. Identidad y credenciales

- `SCRUM_API_KEY` — variable de entorno. Si no está seteada: explicar que si el usuario
  es Product Owner, el Project Manager le genera la key desde "Usuarios Activos" en la
  app; si es Project Manager, la genera él mismo desde esa misma pantalla ("Tu API Key").
  Hay que exportarla (`export SCRUM_API_KEY=sk_...`). Parar acá si falta. **Nunca**
  escribir esta key a ningún archivo del repo.
- `SCRUM_API_URL` — la base de la instancia (ej. `https://scrum.tudominio.com`). **No es
  secreta y no hay que preguntarla todavía**: se resuelve en el paso 1 leyendo el
  manifest del repo. Sólo si el manifest tampoco la tiene se llega a pedírsela al usuario.

Todas las llamadas llevan `-H "Authorization: Bearer $SCRUM_API_KEY"`. Si `$SCRUM_API_URL`
tiene un `/` final, quitarlo antes de concatenar rutas.

## 1. Leer o inicializar el manifest

Leer `docs/po-manifest.json` en la raíz del repo (es propio de este skill, no el mismo que
usa `/scrum-sync` para developer — ese guarda referencias a código; este guarda
referencias a secciones de este documento). Si no existe, crear:

```json
{
  "apiUrl": null,
  "projectId": null,
  "lastSyncAt": null,
  "userStoryMappings": [],
  "mappings": []
}
```

- **`apiUrl`**: no es secreta — vive commiteada en el repo. Si el archivo ya la trae,
  usarla tal cual y no volver a preguntar. Si falta, antes de preguntarle nada al usuario,
  revisar si existe `docs/SCRUM_MASTER_AI.md` — el Project Manager la publica ahí ya
  resuelta (el servidor la saca sola de su propia URL pública); si está, usar ese valor.
  Sólo si ninguna de las dos fuentes la tiene, preguntarle la URL al usuario. En cualquier
  caso, guardarla en el manifest para que el resto del equipo no tenga que repetirla.
- **`projectId`**: si falta, no preguntarlo a ciegas todavía — se resuelve en el paso 1.5
  contra los proyectos que devuelve `/api/v1/me`.

Cada entrada de `mappings` es `{ requirementId, userStoryId, sourceRef }`, donde
`sourceRef` es una referencia corta a la sección del documento que originó ese
Requerimiento (ej. `docs/requerimientos-po.md#Alta de turno`) — sirve para que una
relectura futura sepa si ya se sincronizó algo y actualizarlo en vez de duplicarlo.
Recomendarle al usuario commitear este archivo (no tiene secretos, sólo IDs y la URL).

## 1.5. Confirmar identidad y rol

```bash
curl -s "$SCRUM_API_URL/api/v1/me" -H "Authorization: Bearer $SCRUM_API_KEY"
```

- `401` → la key es inválida o fue revocada. Avisar que se la pidan de nuevo al Project
  Manager, y parar.
- `200` → `{ id, username, email, role, projects: [{ id, name }, ...] }`. Esto es lo que
  determina el rol de verdad — **nunca preguntarle al usuario "qué rol sos" ni asumirlo**;
  el rol de la cuenta dueña de la key es el único que importa (y la API lo vuelve a
  validar en cada llamada de todos modos).
  - Si `role` no es `product_owner` ni `project_manager`, avisar que esta key no
    corresponde a este skill (`/scrum-sync` es para `developer`, `/qa-sync` para
    `tester`/`qa`) y sugerir el correcto en vez de seguir adelante.
  - Si el manifest no tenía `projectId`: con un solo elemento en `projects`, usar ese `id`
    directo; con varios, listarlos y preguntar cuál; vacío, avisar que el Project Manager
    todavía no agregó a este usuario a ningún proyecto, y parar. Guardar el `projectId`
    elegido en el manifest.

## 2. Traer Historias de Usuario y Requerimientos existentes

```bash
curl -s "$SCRUM_API_URL/api/v1/projects/$PROJECT_ID/user-stories" \
  -H "Authorization: Bearer $SCRUM_API_KEY"
curl -s "$SCRUM_API_URL/api/v1/projects/$PROJECT_ID/requirements" \
  -H "Authorization: Bearer $SCRUM_API_KEY"
```

- `401` → key inválida o revocada. Avisar que se la pidan de nuevo al Project Manager, y
  parar.
- `403` → key válida pero el usuario no pertenece a ese proyecto. Avisar que confirme el
  `projectId`, y parar.
- `200` en ambas → guardar en memoria. Historias de Usuario trae `{ id, code, name,
  description, acceptanceCriteria, technicalDetail, ... }`; Requerimientos trae `{ id,
  code, userStoryId, name, description, type, status, ... }`. Son la base contra la que
  se razona en los pasos siguientes.

## 2.5. Sincronizar Historias de Usuario (`docs/historias-po.md`, opcional)

Si el repo tiene `docs/historias-po.md`, es el documento donde el Product Owner redacta
sus Historias de Usuario en lenguaje natural (nombre, descripción tipo "Como... quiero...
para...", criterios de aceptación) — a diferencia de `docs/requerimientos-po.md` (paso 3),
que es sobre Requerimientos técnicos. Si no existe, saltar directo al paso 3.

Para cada Historia que describe el documento:

1. **Decidí si ya existe** (por el manifest, campo `userStoryMappings` — ver abajo — o
   comparando contenido contra la lista del paso 2) o si es nueva.
2. **Si ya existe**, actualizar sólo lo que cambió:
   ```bash
   curl -s -X PATCH "$SCRUM_API_URL/api/v1/user-stories/$USER_STORY_ID" \
     -H "Authorization: Bearer $SCRUM_API_KEY" -H "Content-Type: application/json" \
     -d '{"name":"...","description":"...","acceptanceCriteria":"...","technicalDetail":"..."}'
   ```
3. **Si es nueva**, crearla:
   ```bash
   curl -s -X POST "$SCRUM_API_URL/api/v1/projects/$PROJECT_ID/user-stories" \
     -H "Authorization: Bearer $SCRUM_API_KEY" -H "Content-Type: application/json" \
     -d '{"name":"...","description":"...","acceptanceCriteria":"...","technicalDetail":"..."}'
   ```
   El `code` (`HU-01`, `HU-02`, ...) lo asigna la API sola por orden de creación — nunca
   lo mandes en el body, se ignora.
4. Guardar/actualizar en el manifest la entrada `{ userStoryId, sourceRef }` dentro de un
   array nuevo `userStoryMappings` (separado de `mappings`, que es sólo de Requerimientos).
5. Agregar cada Historia nueva o actualizada a la lista en memoria del paso 2, para que el
   paso 3 (Requerimientos) ya pueda colgar cosas de una Historia recién creada en esta
   misma corrida.

Borrar una Historia (`DELETE /api/v1/user-stories/$USER_STORY_ID`) borra en cascada todos
sus Requerimientos — mismo criterio que el paso 4.5 (con Requerimientos): sólo si el
documento la marca explícitamente como eliminada o el usuario lo pide directo, y **nunca
sin mostrarle antes al usuario qué Historia (con cuántos Requerimientos colgando, si los
hay) va a borrar y esperar confirmación explícita** — acá el costo de equivocarse es mayor
que borrar un Requerimiento suelto.

## 3. Leer el documento del Product Owner

Leer `$ARGUMENTS` si se pasó una ruta explícita (si no existe, avisar y no asumir el
default en su lugar); si no hay argumento, leer `docs/requerimientos-po.md`. Si tampoco
existe, explicarle al usuario que tiene que escribir ahí (o donde prefiera, pasando la
ruta como argumento) qué funcionalidad quiere pedir, en el lenguaje que le resulte natural
— no hace falta ningún formato rígido tipo RF-01 — y parar.

**Esto es un trabajo de criterio, no de matching de texto.** Leé el documento como lo
haría un Project Manager familiarizado con el proyecto: para cada cosa que el Product
Owner describe,

1. **Decidí a qué Historia de Usuario pertenece**, comparando contra `name` +
   `description` + `acceptanceCriteria` de las Historias traídas en el paso 2 (nunca por
   coincidencia literal de texto). Si el documento ya agrupa el contenido bajo encabezados
   que nombran la Historia, usá eso como pista fuerte, pero confirmá con criterio que el
   contenido efectivamente corresponde. Si no hay ninguna Historia razonable a la que
   colgarlo, **no la inventes ni la crees** — dejalo afuera y reportalo al final como
   "sin Historia de Usuario clara".
2. **Decidí si ya existe un Requerimiento equivalente** (en el manifest por `sourceRef`
   de una corrida anterior, o comparando contenido contra la lista de Requerimientos del
   paso 2 aunque nunca se haya sincronizado desde acá) o si es genuinamente nuevo.
3. **Decidí el tipo** (`funcional` o `no_funcional`) por el contenido — si describe una
   restricción transversal (seguridad, performance, disponibilidad) es `no_funcional`;
   si describe una funcionalidad concreta del sistema, `funcional`.

Si tenés dudas razonables sobre a qué Historia corresponde algo, es preferible dejarlo
afuera (reportarlo en el resumen final) a inventar una relación.

## 4. Actualizar cada Requerimiento existente

**Este skill sólo actualiza Requerimientos que ya existen -- nunca crea uno nuevo.** Si
en el paso 3 detectaste algo que no matchea ningún Requerimiento del paso 2, no lo crees
acá: reportalo en el resumen final como "Requerimiento nuevo pendiente de crear vía
`/scrum-sync`" (con la Historia de Usuario resuelta en el paso 3 y una descripción clara)
-- crear Requerimientos nuevos es territorio de developer/Project Manager con
`/scrum-sync` (paso 4.5 de ese skill), no de este.

Para cada Requerimiento con match, actualizar sólo lo que cambió:

```bash
curl -s -X PATCH "$SCRUM_API_URL/api/v1/requirements/$REQUIREMENT_ID" \
  -H "Authorization: Bearer $SCRUM_API_KEY" -H "Content-Type: application/json" \
  -d '{"name":"...","description":"...","type":"funcional"}'
```

Mandar sólo los campos que realmente cambiaron respecto de lo que ya trajo el paso 2 (no
reescribir todo con lo mismo). Esta llamada rechaza con 403 cualquier intento de tocar
status/tiempos/asignado/dependencias — ese rechazo es correcto, no es un bug: no
reintentar con otros campos, simplemente no son territorio del Product Owner.

Guardar/actualizar en el manifest la entrada `{ requirementId, userStoryId, sourceRef }`
(crear si es la primera vez, actualizar `sourceRef` si ya existía y cambió la sección del
documento que lo originó).

## 4.5. Borrar un Requerimiento (con confirmación explícita)

Dos disparadores posibles:
- El documento marca algo existente como eliminado/descartado (ej. una sección tachada,
  o una nota tipo "esto ya no va").
- El usuario pide directamente, en la conversación, borrar un Requerimiento puntual (por
  código o nombre), fuera del flujo de sincronización del documento.

En cualquiera de los dos casos: **antes de llamar a la API, confirmarle al usuario cuál
Requerimiento vas a borrar (código + nombre) y esperar una confirmación explícita** — no
asumas que "marcado como eliminado en el doc" ya es luz verde. Recién con esa confirmación:

```bash
curl -s -X DELETE "$SCRUM_API_URL/api/v1/requirements/$REQUIREMENT_ID" \
  -H "Authorization: Bearer $SCRUM_API_KEY"
```

- `200` → `{"ok":true}`. Sacar también la entrada correspondiente de `mappings` en el
  manifest (si existía).
- `403` → la cuenta no tiene permiso para borrar (sólo Product Owner y Project Manager
  pueden; Developer no). Avisar tal cual y no reintentar.
- `404` → ya no existe (alguien más lo borró, o el `requirementId` del manifest quedó
  desactualizado) — sacarlo igual del manifest y seguir.

## 5. Guardar el manifest actualizado

Reescribir `docs/po-manifest.json` con `lastSyncAt` en la fecha/hora actual (ISO), todas
las entradas de `mappings` (Requerimientos, viejas + nuevas, sin las que se borraron) y de
`userStoryMappings` (Historias, ídem).

## 6. Resumen final

Reportarle al usuario, en texto, no en JSON crudo:
- Cuántas Historias de Usuario se crearon, actualizaron o borraron (paso 2.5).
- Cuántos Requerimientos existentes se actualizaron, y qué campos cambiaron.
- Cuántos Requerimientos se borraron, y cuáles (código + nombre).
- Qué Requerimientos nuevos quedaron pendientes de crear (no se crean acá), con la
  Historia de Usuario a la que corresponden — para que developer/Project Manager los
  cree con `/scrum-sync`.
- Qué partes del documento quedaron "sin Historia de Usuario clara" (para que se sepa qué
  revisar o crear esa Historia primero).

---

## Notas de implementación

- Crear/editar Historias de Usuario es territorio de este skill desde `docs/historias-po.md`
  (paso 2.5) -- no asumir que son de sólo lectura, sí lo eran en una versión anterior de
  este skill.
- **Nunca crear un Requerimiento nuevo desde este skill** -- sí se creaban en una versión
  anterior; ahora ese alta es exclusivamente de `/scrum-sync` (developer/Project Manager,
  paso 4.5 de ese skill). Acá sólo se actualizan/borran los que ya existen y se reporta al
  final cuáles faltarían crear.
- Nunca borrar una Historia de Usuario o un Requerimiento sin haberle mostrado antes al
  usuario cuál es (código + nombre, y para Historias cuántos Requerimientos se van con
  ella) y haber recibido una confirmación explícita — a diferencia de crear/editar, no es
  una operación que se pueda deshacer con otra corrida del skill.
- Nunca reintentar con otro shape de body si la API devuelve 403 al tocar un campo de
  ejecución — ese rechazo es intencional (ver `EXECUTION_FIELDS` en
  `api/v1/requirements/[id]/route.ts`), no un error a esquivar.
- Todas las respuestas de error de la API vienen como `{"error": "..."}` — mostrar ese
  mensaje tal cual, no reinterpretarlo.
- `docs/po-manifest.json` es propio de este skill — no confundirlo ni fusionarlo con
  `docs/scrum-manifest.json` (ese lo usa `/scrum-sync` del lado del developer, con otro
  significado de `sourceRef`).
