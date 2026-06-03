# Prode Mundial 2026

App web de pozo de pronósticos (Prode) para la Copa del Mundo 2026.
Es un **único archivo HTML standalone** (`index.html`): todo el HTML, CSS y JS está inline, sin build step.

> Última actualización de este documento: **2026-06-03**

---

## Hosting

> **Plataforma actual: Vercel** (migrado desde Netlify por agotamiento del free tier).
>
> - URL de producción: **https://prode-mundial-2026-gr.vercel.app**
> - Repo GitHub: `https://github.com/gonzarogala/prode-mundial-2026` (Vercel redeploya solo en cada commit)
>
> **Legacy (ya no se usa):** Netlify — `https://prode-mundial2026-grt.netlify.app`

### Cómo se publica en Vercel (vía GitHub, todo por navegador)

Vercel no tiene un "drag & drop" como Netlify Drop. Se conecta un **repositorio de GitHub** y Vercel **redeploya solo** cada vez que se sube un cambio al repo. No hace falta terminal ni Git instalado.

**Setup inicial (una sola vez):**
1. Crear cuenta gratis en https://github.com (si no tenés).
2. **New repository** → nombre `prode-mundial-2026`, visibilidad **Public** → *Create*.
3. En el repo vacío: **uploading an existing file** → arrastrar `~/Desktop/prode-deploy/index.html` → *Commit changes*.
4. Crear cuenta gratis en https://vercel.com (lo más cómodo: **Sign up with GitHub**).
5. En Vercel: **Add New → Project → Import** el repo `prode-mundial-2026`.
   - Framework Preset: **Other** · Build Command: *(vacío)* · Output Directory: *(vacío / raíz)*. Es HTML estático, no hay build.
6. **Deploy.** Vercel te da la URL `*.vercel.app`. Anotala y completala arriba en este README.

**Para publicar un cambio (después del setup):**
1. Editar el fuente `~/Desktop/prode-mundial-2026.html`.
2. Sincronizar la copia de deploy:
   ```bash
   cp ~/Desktop/prode-mundial-2026.html ~/Desktop/prode-deploy/index.html
   ```
3. En GitHub, abrir el repo → `index.html` → ícono del **lápiz** (editar) o **Add file → Upload files** y reemplazarlo → *Commit changes*.
4. Vercel detecta el commit y **redeploya automáticamente** en ~30 segundos.
5. Hard refresh en el navegador: **Ctrl+F5** (Windows) / **Cmd+Shift+R** (Mac).

> 💡 **Vía alternativa (CLI, requiere Node):** la carpeta `~/Desktop/prode-deploy/` ya es un repo Git inicializado. Si instalás Node, podés deployar con:
> ```bash
> cd ~/Desktop/prode-deploy
> npx vercel --prod
> ```

> ⚠️ **Nada del código cambia con la migración.** La app es 100% estática del lado del cliente (Supabase + football-data.org se llaman desde el navegador), así que funciona idéntica en cualquier hosting. Solo cambia *dónde* se sube el archivo.

---

## Archivos del proyecto

| Archivo | Rol |
|---|---|
| `~/Desktop/prode-mundial-2026.html` | **Fuente de trabajo.** Acá se editan los cambios. |
| `~/Desktop/prode-deploy/index.html` | Copia que se publica (se sube al repo de GitHub conectado a Vercel). Sincronizar tras cada cambio. Es además un repo Git ya inicializado. |
| `~/Desktop/prode-deploy/README.md` | Este documento. |

> ⚠️ **Nota iCloud:** el Desktop está sincronizado con iCloud Drive y en algún momento la carpeta "desapareció" temporalmente. Si vuelve a pasar, la última versión publicada siempre se puede recuperar descargando el `index.html` desde la URL de producción (una vez en Vercel, usar la `*.vercel.app`; la de Netlify aún sirve como respaldo mientras siga online):
> ```bash
> curl -o ~/Desktop/prode-mundial-2026.html https://prode-mundial-2026-gr.vercel.app
> # respaldo temporal: https://prode-mundial2026-grt.netlify.app
> ```

### Workflow para cada cambio
```bash
# 1. editar el fuente
#    ~/Desktop/prode-mundial-2026.html
# 2. sincronizar la copia de deploy
cp ~/Desktop/prode-mundial-2026.html ~/Desktop/prode-deploy/index.html
# 3. subir index.html al repo de GitHub (commit) → Vercel redeploya solo
# 4. hard refresh (Ctrl+F5 / Cmd+Shift+R)
```

---

## Stack y servicios

- **Frontend:** HTML + CSS + JS vanilla, todo inline en un solo archivo. Iconos vía Tabler Icons (webfont).
- **Banderas:** [Twemoji](https://github.com/jdecked/twemoji) por CDN — reemplaza los emojis de bandera por `<img>` SVG, porque **Windows no tiene glifos de bandera** y mostraría "MX", "AR", etc. Se llama a `paintTwemoji()` al final de cada render.
- **Base de datos / sync en tiempo real:** [Supabase](https://supabase.com) (free tier).
- **Resultados automáticos (solo fase de grupos):** API de [football-data.org](https://www.football-data.org) (v4), con fallback por proxy CORS. Las eliminatorias se cargan a mano.

### Supabase
- Proyecto: `https://zyblcmwucymlkqcyrjix.supabase.co`
- Tabla única `prode_state` → `(key TEXT PRIMARY KEY, value JSONB)`.
- Todo el estado vive en **una sola fila** con `key = 'prode26'`.
- **RLS abierto** (anon read/insert/update), sin auth. La `anon key` es pública por diseño.
- Sync en tiempo real vía Supabase Realtime (canal `prode-state`, escucha cambios de la fila y re-renderiza).
- SQL de creación (idempotente):
  ```sql
  CREATE TABLE IF NOT EXISTS prode_state (
    key   TEXT PRIMARY KEY,
    value JSONB NOT NULL DEFAULT '{}'::jsonb
  );
  ALTER TABLE prode_state ENABLE ROW LEVEL SECURITY;
  DROP POLICY IF EXISTS "anon all" ON prode_state;
  CREATE POLICY "anon all" ON prode_state
    FOR ALL TO anon USING (true) WITH CHECK (true);
  ```
- Para **resetear** el pozo (borrar todo): `DELETE FROM prode_state WHERE key = 'prode26';`

### Sincronización de datos (cómo se mantienen frescos los datos)
La app sincroniza el estado compartido (Supabase) **automáticamente**, sin que el usuario toque nada. Hay 4 disparadores (función `silentSync()`):

1. **Al cargar la página** — `load()` corre antes de mostrar cualquier pantalla (en `boot()`). Nunca se ven datos viejos al entrar.
2. **Al cambiar de sección** — un click en cualquier pestaña (`.nb`) dispara una sync en background (delegación de eventos).
3. **Polling cada 60 s** — `setInterval` silencioso; re-renderiza la vista actual solo si el server cambió.
4. **Al volver a la pestaña del navegador** — eventos `visibilitychange` + `focus`.

Además, **Supabase Realtime** (canal `prode-state`) da actualizaciones **instantáneas** cuando está habilitado; el polling es el respaldo para que funcione igual si Realtime está apagado.

Detalles de robustez:
- `silentSync()` **no interrumpe** mientras el usuario escribe un marcador/predicción (`isEditing()`), y **no pisa** un guardado local recién hecho (ventana `lastLocalSaveAt`).
- Solo re-renderiza si el snapshot del server (`JSON.stringify` del JSONB) difiere de lo último visto (`_lastRaw`) → no mueve la pantalla sin motivo.
- **Indicador de estado** (badge abajo a la izquierda, `#sync-badge`): 🟢 **Actualizado** / 🔄 **Sincronizando…** / 🔴 **Sin conexión**. Click = forzar refresco (opcional, ya no es necesario).

> 💡 **Para updates instantáneos:** habilitar Realtime para la tabla en Supabase → *Database → Replication* (o *Realtime*), activar `prode_state`. Sin esto, igual sincroniza por polling (hasta 60 s).

### Sincronización de resultados desde la API (solo grupos)
- `syncResultsFromAPI()` consulta football-data.org cada `UPDATE_INTERVAL_MINUTES` (5 min) y **solo** actualiza partidos de **fase de grupos** (`g1`–`g72`), escribiendo únicamente el marcador `{h, a}`.
- **Las eliminatorias (`e1`–`e32`) NO se tocan por API** — las carga el admin a mano (90 min + quién clasificó + cómo se resolvió), porque la API no provee de forma confiable tiempo extra/penales ni el clasificado.
- En **Admin › Resultados** se distingue visualmente: banner + chip `🤖 Auto` en grupos, `👤 Manual` en eliminatorias, con un aviso explicativo arriba.
- El admin igual puede **corregir a mano** un resultado de grupo si la API se equivocó.

### Config en el HTML
Constantes al inicio del `<script>` (buscar por nombre):
- `SUPABASE_URL`
- `SUPABASE_ANON_KEY`
- `STATE_KEY = 'prode26'`
- `ADMIN_PIN = '0801'` (PIN del panel de administración)
- API football-data: `X-Auth-Token` y `CORS_PROXY` (fallback corsproxy.io).

> La API key de football-data vive en el HTML público. Si se quiere ocultar, hay que mover la llamada a una función serverless (p. ej. una Netlify Function) y apuntar `CORS_PROXY` ahí.

---

## Reglas de negocio clave

### Puntajes
Se calculan en `gpts` (grupos) y `epts` (eliminatorias) **siempre sobre el resultado de los 90 minutos**.

### Bonus
Una vez que el admin carga **al menos 1 resultado real**, la sección Bonus queda **bloqueada** (`isBonusLocked()`).

### Partidos eliminatorios definidos por tiempo extra / penales
Regla del prode: se registra **solo el marcador de los 90 minutos**; los goles de tiempo extra y penales **se ignoran** para el puntaje.

El formulario del admin tiene 3 pasos:
1. **Paso 1 — Marcador a los 90 min.**
2. **Paso 2 — ¿Cómo se resolvió?** (⏱️ Tiempo extra / 🥅 Penales) → *solo aparece si los 90 min terminan empatados.*
3. **Paso 3 — ¿Quién clasificó?** → *solo aparece si hay empate.*

Si hay ganador en los 90 min, el clasificado y `resolution='regular'` se autocompletan.

Estructura guardada en `S.results[id]` para una eliminatoria:
```js
{ h: 1, a: 1, adv: '1° Grupo C', resolution: 'extra_time' }
// h/a   = marcador 90 min (lo único que puntúa)
// adv   = etiqueta del lado que clasifica (placeholder, no el nombre real)
// resolution = 'regular' | 'extra_time' | 'penalties'  (SOLO informativo, no afecta puntajes)
```
Indicadores en el bracket: `[90']`, `[AET]` (tiempo extra), `[PEN]` (penales).

Funciones relacionadas: `rCard` (formulario admin de 3 pasos), `setRes` (normaliza el ganador automático), `validateElimResult`, `updateElimUI` (feedback en vivo), `elimBadge`, `officialResultLine`, `renderBracketCard`.

---

## Secciones de la app

- **Login:** ingreso por PIN de jugador (4 dígitos) o PIN admin (`0801`).
- **Predicciones:** carga de pronósticos. Toggle de vista **Por fecha / Por grupo** (preferencia guardada en `localStorage` con clave `prode_pred_view`).
- **Ranking:** tabla de posiciones de los jugadores.
- **Grupos:** tablas de posiciones reales + **diagrama de eliminatorias** (bracket partido izquierda/derecha que se une en la Final, con códigos FIFA de 3 letras y asignación automática de los mejores terceros).
- **Bonus:** premios especiales (campeón, goleador, etc.), se bloquea al cargar el primer resultado.
- **Admin:** Jugadores · Resultados · Grupos · Bonus · API (sync automática de resultados).

---

## Fixture

- `MG` = partidos de fase de grupos (72) · `ME` = partidos de eliminación (32).
- Claves de fase: `r32` (16avos), `qf` (Octavos), `sf` (Cuartos), `semif` (Semis), `tp` (3er puesto), `final`.
- En eliminatorias, `m.h` / `m.a` son **etiquetas placeholder** (`"1° Grupo C"`, `"G. P73"`, `"Gan. SF1"`), que `resolveSlot()` resuelve al equipo real a medida que se cargan resultados.
- `validateFixtureIntegrity()` (IIFE) chequea que cada etiqueta de slot (1°/2° A–L) aparezca exactamente una vez — sirvió para detectar el bug de "Alemania en 2 slots".
- Mejores terceros: `thirdAssignments()` hace una asignación greedy (rankea por Pts/DG/GF y los ubica en los pools de cada slot). Es una **aproximación**, no la tabla oficial de 462 filas de FIFA.

---

## Compatibilidad

Probado para verse correctamente en **Mac y Windows**:
- Banderas vía Twemoji (Windows no tiene glifos nativos).
- `<option>` de los `<select>` con color de fondo explícito (en Windows salían blanco sobre blanco).
- `overflow-x:hidden` en `html/body` y buffer en el bracket para evitar scroll horizontal por el ancho del scrollbar de Windows.

---

## Historial de cambios principales

1. Migración de `localStorage` → **Supabase** con sync en tiempo real e indicador de "Guardando…".
2. Paleta de colores **Adidas Trionda** + fondo oscuro con glassmorphism (imagen de fondo de Messi/Diario Mendoza, overlay 0.70).
3. **Banderas** junto a cada selección (Twemoji para Windows).
4. Partidos ordenados **cronológicamente por jornada**.
5. Sección pública **Grupos** con tablas de posiciones.
6. **Bloqueo del Bonus** al cargar el primer resultado real.
7. Fixture oficial FIFA completo (MG 72 + ME 32) reemplazando los arrays de ejemplo.
8. **Diagrama de eliminatorias** tipo bracket partido (izq/der → Final), con códigos FIFA y mejores terceros automáticos.
9. Fix bug "Alemania en 2 slots" + `validateFixtureIntegrity()`.
10. **Sync automática de resultados** vía football-data.org (con fallback corsproxy.io por CORS).
11. Toggle de vista **Por fecha / Por grupo** en Predicciones.
12. **(2026-05-29)** Carga de eliminatorias por **tiempo extra / penales**: formulario de 3 pasos, validación `validateElimResult`, feedback en vivo, indicadores `[90']`/`[AET]`/`[PEN]` en el bracket. El puntaje se sigue calculando solo sobre los 90 min.
13. **(2026-06-03)** Migración de hosting **Netlify → Vercel** (se agotó el free tier de Netlify). Repo Git inicializado en `prode-deploy/` + repo GitHub `gonzarogala/prode-mundial-2026`. Sin cambios de código: la app es estática y funciona idéntica. URL nueva: `https://prode-mundial-2026-gr.vercel.app`.
14. **(2026-06-03)** **Sincronización automática del storage compartido** (antes había que tocar el botón a mano): `silentSync()` con 4 disparadores (carga inicial, cambio de sección, polling 60 s, foco de pestaña) + Realtime como vía instantánea. Guardas anti-interrupción (`isEditing`, `lastLocalSaveAt`, `_lastRaw`). El badge de abajo a la izquierda se repurposeó para mostrar el estado del storage (🟢/🔄/🔴); la sync manual de football-data quedó en **Admin › API**.
15. **(2026-06-03)** **La sync por API ahora es solo de fase de grupos.** `syncResultsFromAPI()` solo actualiza `g1`–`g72` (marcador `{h,a}`); las eliminatorias nunca se tocan por API (las carga el admin). En **Admin › Resultados**: aviso + banners/chips `🤖 Auto` (grupos) y `👤 Manual` (eliminatorias).
16. **(2026-06-03)** Limpieza de código muerto: se eliminaron `STAGE_MAP` y `apiDateToDt` (solo los usaba la rama de eliminatorias de la API, ya removida).
