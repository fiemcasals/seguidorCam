# Este repo está conectado a Scrum Master AI

_Generado automáticamente el 2026-08-05T23:36:21.014Z -- no editar a mano, se sobreescribe en cada publicación._

Si el usuario te pidió leer la documentación de este proyecto, o arrancó una
conversación sobre "qué sigue", "cargar requerimientos", "sincronizar tests",
"reportar avance" o similar, seguí estos pasos antes de hacer nada más.

## 1. Conseguir la API key

Preguntale al usuario **sólo** por `SCRUM_API_KEY` (si no la tenés ya como variable
de entorno) -- se la genera su Project Manager desde "Usuarios Activos" en la app.
Si no la tiene, explicáselo y pedile que la consiga antes de seguir. **Nunca** la
escribas a ningún archivo del repo, sólo se usa desde la variable de entorno.

## 2. La URL de la instancia ya la sabés -- no la preguntes

`SCRUM_API_URL` es `https://scrum.misitiowebpersonal.com.ar`. Se generó sola a partir de la URL con la que tu
Project Manager entra a la app, así que es un dato fijo de este repo, no algo que
tengas que pedirle a nadie. Usala directo en las llamadas de los pasos siguientes.
Si vas a crear/leer `docs/scrum-manifest.json` o `docs/po-manifest.json` (se usan
de acá en adelante para no repetir preguntas), guardala ahí en el campo `apiUrl`
si todavía no está.

## 3. Identificar el rol -- nunca preguntarlo

Con la key y la URL, llamá:

```bash
curl -s "$SCRUM_API_URL/api/v1/me" -H "Authorization: Bearer $SCRUM_API_KEY"
```

La respuesta trae `{ role, projects, ... }`. El rol de la cuenta dueña de la key es el
único dato que importa (la API lo vuelve a validar en cada llamada de todos modos) --
**no le preguntes al usuario "cuál es tu rol"**, ni le creas si te lo dice: usá el
`role` que devolvió `/api/v1/me`. `401` significa key inválida o revocada -- avisar y
parar.

## 4. Resolver el proyecto y dar una bienvenida orientadora (no un resumen técnico)

Antes de preguntarle "qué querés hacer" a ciegas, armale un pantallazo real de en qué
está parado. Esto reemplaza cualquier mensaje tipo "repositorio clonado, autenticación
verificada" -- ese resumen de infraestructura no le sirve de nada al usuario, lo que
necesita es entender qué hay en el proyecto y qué puede pedir a partir de acá.

1. **Resolver `projectId`**: buscarlo en `docs/scrum-manifest.json` o
   `docs/po-manifest.json` (el que exista en este repo). Si ninguno lo tiene, usar
   `projects` de la respuesta del paso 3: un solo elemento → usarlo directo; varios →
   listarlos y preguntar cuál; vacío → avisar que el Project Manager todavía no lo
   agregó a ningún proyecto, y parar.
2. **Traer el estado actual**:
   ```bash
   curl -s "$SCRUM_API_URL/api/v1/projects/$PROJECT_ID/user-stories" \
     -H "Authorization: Bearer $SCRUM_API_KEY"
   curl -s "$SCRUM_API_URL/api/v1/projects/$PROJECT_ID/requirements" \
     -H "Authorization: Bearer $SCRUM_API_KEY"
   ```
3. **Con eso, escribile un mensaje de bienvenida en texto natural** (nunca una lista
   de pasos técnicos tipo "clonación/autenticación/manifest"), que incluya:
   - Nombre del proyecto, cuántas Historias de Usuario tiene, y cuántos Requerimientos
     desglosados por estado (`to_do`/`doing`/`testing`/`done`/`blocked`) -- así ya sabe
     si el proyecto recién arranca o viene con trabajo en curso.
   - Qué puede pedirte a partir de acá, en casos de uso concretos según su `role`, no
     un listado técnico de comandos:
     - `developer`: reportar avance de lo que ya implementó, preguntar "qué sigue" y
       que le dejes la rama lista, o pedir que se cree un Requerimiento que faltaba.
     - `product_owner`: cargar o editar Historias de Usuario y Requerimientos con sus
       propias palabras, sin formato rígido.
     - `tester` / `qa`: armar o sincronizar tests de un endpoint puntual.
     - `project_manager`: cualquiera de los casos de developer y product_owner --
       preguntale directamente cuál quiere hacer ahora si no es obvio por contexto.
   - Para qué sirve `docs/scrum-manifest.json` (o `po-manifest.json`) que se acaba de
     leer o crear: guarda `projectId` y `apiUrl` para no volver a preguntarlos nunca
     más en este repo, no tiene secretos, conviene commitearlo.
   Cerrar preguntándole qué quiere hacer ahora -- no asumas un flujo por tu cuenta.

## 5. Seguir el skill correspondiente, de forma interactiva

Una vez que el usuario contestó qué quiere hacer, este repo ya trae los tres skills
publicados en `.claude/skills/` -- no hace falta que escriba el slash command a mano,
podés seguir las instrucciones del archivo correspondiente vos directamente, según el
`role` del paso 3:

- **`developer`** → `.claude/skills/scrum-sync/SKILL.md`
- **`product_owner`** → `.claude/skills/po-sync/SKILL.md`
- **`tester` / `qa`** → `.claude/skills/qa-sync/SKILL.md`
- **`project_manager`** → depende de qué pidió: para reportar avance, decidir qué
  Requerimiento sigue, manejar ramas, o crear un Requerimiento nuevo,
  `.claude/skills/scrum-sync/SKILL.md`; para crear/editar/borrar Historias de Usuario,
  o editar/borrar Requerimientos ya existentes, `.claude/skills/po-sync/SKILL.md` --
  ambos aceptan su key, preguntale si no es obvio por el contexto cuál de las dos
  cosas quiere.

Cada uno explica qué leer del proyecto y qué llamadas hacer contra
`$SCRUM_API_URL/api/v1/*` -- ya podés saltear sus propios pasos de credenciales/rol/
proyecto, porque los pasos 1-4 de acá ya los cubrieron.
