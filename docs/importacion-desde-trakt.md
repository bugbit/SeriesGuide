# Proceso de importación y sincronización desde Trakt

Este documento describe cómo SeriesGuide importa y sincroniza datos desde Trakt hacia la base local. Aunque el código usa el término “sync”, desde el punto de vista de la app hay dos comportamientos distintos:

1. **Importación inicial / mezcla inicial**: primera sincronización con una cuenta Trakt, o tras reiniciar el estado de sincronización. Se descargan datos de Trakt y se conservan datos locales subiéndolos a Trakt si faltan allí.
2. **Sincronización posterior**: Trakt pasa a ser la fuente de verdad para los estados que se sincronizan; si algo ya no está en Trakt, se elimina o desmarca localmente.

## Archivos principales

| Archivo | Responsabilidad |
| --- | --- |
| `TraktSync.kt` | Orquesta la sincronización general con Trakt. |
| `TraktEpisodeSync.kt` | Importa/sincroniza episodios vistos y en colección. |
| `TraktMovieSync.kt` | Importa/sincroniza películas en colección, watchlist y vistas. |
| `TraktRatingsSync.kt` | Importa ratings de usuario para series, episodios y películas. |
| `TraktNotesSync.kt` | Importa/sincroniza notas de usuario para series. |
| `TraktTools4.kt` | Descarga datos paginados de Trakt y los mapea por TMDB ID. |
| `TraktSettings.kt` | Guarda flags de importación inicial y marcas temporales de última actividad. |
| `AddShowTask.kt` / `AddUpdateShowTools.kt` | Caso especial: al añadir una serie manualmente, restaura watched/collection desde Trakt si Hexagon no está activo. |

## Prerrequisitos y activación

La sincronización con Trakt solo se ejecuta si hay credenciales Trakt guardadas. Durante una sincronización múltiple, `SgSyncAdapter` ejecuta Trakt después de TMDB, Hexagon y migración de listas legacy.

Si Hexagon está activo, Trakt se ejecuta en modo `onlyRatings = true`: se importan ratings, pero no estados de episodios, películas ni notas para evitar conflictos con Hexagon. Si Hexagon no está activo, Trakt sincroniza episodios, películas, ratings y notas.

Antes de importar, `TraktSync` descarga `GET /sync/last_activities`. Esas fechas determinan si cada sección cambió desde la última sincronización local y permiten saltar trabajo cuando no hay cambios.

## Endpoints de Trakt usados

| Endpoint Trakt | Uso |
| --- | --- |
| `GET /sync/last_activities` | Saber qué secciones cambiaron y decidir qué importar. |
| `GET /sync/watched/shows` | Importar episodios vistos, agrupados por serie/temporada/episodio. |
| `GET /sync/collection/shows` | Importar episodios en colección. |
| `GET /sync/watched/movies` | Importar películas vistas y número de reproducciones. |
| `GET /sync/collection/movies` | Importar colección de películas. |
| `GET /sync/watchlist/movies` | Importar watchlist de películas. |
| `GET /sync/ratings/shows` | Importar ratings de usuario para series. |
| `GET /sync/ratings/episodes` | Importar ratings de usuario para episodios. |
| `GET /sync/ratings/movies` | Importar ratings de usuario para películas. |
| `GET /users/me/notes/shows` | Importar notas de usuario para series. |
| `POST /sync/history` | En importación inicial, subir episodios/películas vistos que existen localmente pero no en Trakt. |
| `POST /sync/collection` | En importación inicial, subir episodios/películas en colección que existen localmente pero no en Trakt. |
| `POST /sync/watchlist` | En importación inicial, subir películas en watchlist que existen localmente pero no en Trakt. |
| `POST /notes` | En importación inicial, subir notas locales de series que no existen en Trakt. |

## Estado persistido localmente

`TraktSettings` guarda dos tipos de datos:

- **Flags de importación inicial completada**:
  - `KEY_HAS_MERGED_EPISODES`
  - `KEY_HAS_MERGED_MOVIES`
  - `KEY_HAS_MERGED_SHOW_NOTES`
- **Marcas temporales de última actividad importada**:
  - episodios vistos y en colección
  - ratings de series, episodios y películas
  - películas en colección, watchlist y vistas
  - notas actualizadas

`resetToInitialSync(...)` vuelve a marcar episodios, películas y notas como no mezclados y resetea timestamps importantes para forzar una nueva importación amplia.

## Flujo general de `TraktSync`

1. Comprueba conectividad.
2. Descarga `last_activities` desde Trakt.
3. Si hay series locales:
   1. Si no está en modo `onlyRatings`, sincroniza episodios vistos y en colección.
   2. Importa ratings de episodios.
   3. Importa ratings de series.
   4. Si no está en modo `onlyRatings`, sincroniza notas de series.
4. Para películas:
   1. Si no está en modo `onlyRatings`, sincroniza colección, watchlist y vistas.
   2. Elimina películas locales que ya no son útiles si no están vistas ni en listas.
   3. Importa ratings de películas.
5. Si alguna sección falla, marca la sincronización como incompleta para reintentar más adelante.

## Importación inicial frente a sincronización posterior

### Importación inicial

Durante la primera sincronización de una cuenta Trakt, SeriesGuide intenta **mezclar** datos:

- Si Trakt tiene un episodio/película marcado y localmente no, se marca localmente.
- Si localmente hay episodios/películas marcados que no están en Trakt, se suben a Trakt.
- Para notas de series, las notas de Trakt sobrescriben las locales si existen; las notas locales que no existan en Trakt se suben.
- Al completarse correctamente, se guardan los flags `setInitialSyncEpisodesCompleted`, `setInitialSyncMoviesCompleted` o `setInitialSyncShowNotesCompleted`.

### Sincronización posterior

Después de la importación inicial, Trakt se trata como fuente de verdad para esos datos:

- Episodios vistos en Trakt se marcan vistos localmente; episodios vistos localmente pero no en Trakt se desmarcan, excepto episodios “skipped”.
- Episodios en colección en Trakt se marcan en colección localmente; si ya no están en Trakt, se desmarcan.
- Películas existentes localmente reflejan colección, watchlist, visto y plays de Trakt.
- Películas que quedan sin visto, colección ni watchlist pueden eliminarse con `MovieTools.deleteUnusedMovies(...)`.
- Notas de series que desaparecen de Trakt se borran localmente.

## Episodios vistos y en colección

`TraktEpisodeSync` procesa solo series ya existentes localmente. No autoañade series nuevas desde Trakt durante la sincronización general; compara las series locales por TMDB ID con los mapas descargados de Trakt.

### Episodios vistos

1. `syncWatched(...)` comprueba `lastActivity.episodes.watched_at` contra `KEY_LAST_EPISODES_WATCHED_AT`.
2. Si hay cambios o es importación inicial, descarga `GET /sync/watched/shows` mediante `TraktTools4.getWatchedShowsByTmdbId(...)`.
3. Para cada serie local:
   - Si Trakt devuelve datos de esa serie, compara temporadas y episodios locales.
   - Si un episodio está visto en Trakt, lo marca como `WATCHED` localmente y guarda `plays` (mínimo 1).
   - Si el episodio está visto localmente pero no en Trakt:
     - En importación inicial, lo agenda para subirlo a Trakt.
     - En sincronización posterior, lo desmarca y pone `plays = 0`.
   - Los episodios “skipped” no se desmarcan al limpiar watched.
4. Si Trakt incluye `last_watched_at`, actualiza el último visto de la serie local.
5. Guarda `KEY_LAST_EPISODES_WATCHED_AT` si todo termina bien.

### Episodios en colección

1. `syncCollected(...)` compara `lastActivity.episodes.collected_at` con `KEY_LAST_EPISODES_COLLECTED_AT`.
2. Si hay cambios o es importación inicial, descarga `GET /sync/collection/shows`.
3. Para cada episodio:
   - Si está en la colección de Trakt y localmente no, se marca como colección.
   - Si localmente está en colección pero no en Trakt:
     - En importación inicial, se agenda para subirlo a Trakt.
     - En sincronización posterior, se quita de colección.
4. Para temporadas completas usa operaciones por temporada cuando todos los episodios cambian al mismo estado.
5. Guarda `KEY_LAST_EPISODES_COLLECTED_AT` si todo termina bien.

### Subida durante la importación inicial

Cuando hay diferencias locales que Trakt no tiene:

- Los episodios vistos se suben con `POST /sync/history`.
- Los episodios en colección se suben con `POST /sync/collection`.
- Para episodios vistos, se añade un `SyncEpisode` por cada play local para que Trakt cree reproducciones separadas.

## Películas: colección, watchlist y vistas

`TraktMovieSync.syncLists(...)` importa tres listas de estado:

- colección (`GET /sync/collection/movies`)
- watchlist (`GET /sync/watchlist/movies`)
- vistas con plays (`GET /sync/watched/movies`)

El flujo es:

1. Valida `collected_at`, `watchlisted_at` y `watched_at` desde `last_activities`.
2. Si no es importación inicial y esas marcas no cambiaron, no hace nada.
3. Descarga los tres estados desde Trakt, mapeados por TMDB ID.
4. Recorre películas locales:
   - En importación inicial, conserva estado local y añade localmente cualquier estado que venga de Trakt. Lo que localmente exista pero falte en Trakt se agenda para subida.
   - En sincronización posterior, actualiza la película local para reflejar exactamente colección, watchlist, watched y plays de Trakt.
5. Tras procesar locales, los elementos que quedan en los mapas descargados son películas que existen en Trakt pero no localmente.
6. Esas películas se añaden a la base local con `MovieTools.addMovies(...)`, que descarga metadatos de TMDB.
7. Guarda timestamps de películas y resetea `KEY_LAST_MOVIES_RATED_AT` si se añadieron películas nuevas.

Durante la importación inicial, si hay estado local que no existe en Trakt, se sube así:

- colección: `POST /sync/collection`
- watchlist: `POST /sync/watchlist`
- vistas: `POST /sync/history`

## Ratings de usuario

`TraktRatingsSync` importa ratings de usuario de forma incremental:

1. Compara `rated_at` de `last_activities` con el último timestamp local.
2. Si hay cambios, descarga la lista correspondiente:
   - `GET /sync/ratings/shows`
   - `GET /sync/ratings/episodes`
   - `GET /sync/ratings/movies`
3. Trakt devuelve resultados de más reciente a más antiguo; el código deja de procesar cuando llega a ratings anteriores al umbral local, con una tolerancia de 5 minutos.
4. Mapea por TMDB ID y actualiza la base local:
   - series: `sgShow2Helper().updateUserRatings(...)`
   - episodios: `sgEpisode2Helper().updateUserRatings(...)`
   - películas: operaciones sobre `Movies.RATING_USER`
5. Guarda el nuevo timestamp de rating si todo termina bien.

Los ratings sí se importan aunque Hexagon esté activo, porque `onlyRatings = true` permite esta parte del flujo.

## Notas de series

`TraktNotesSync` sincroniza solo notas de series:

1. Compara `lastActivity.notes.updated_at` con `KEY_LAST_NOTES_UPDATED_AT`.
2. Si hay cambios o es importación inicial, descarga páginas de `GET /users/me/notes/shows`.
3. Para cada nota recibida:
   - Requiere TMDB ID de la serie, texto de la nota y Trakt note ID.
   - Si la serie existe localmente, actualiza el texto y el Trakt note ID si han cambiado.
4. Mantiene una lista de shows locales con nota que no aparecieron en Trakt:
   - En importación inicial, intenta subir esas notas con `POST /notes`.
   - En sincronización posterior, borra la nota local y su Trakt note ID.
5. Guarda `KEY_HAS_MERGED_SHOW_NOTES` y `KEY_LAST_NOTES_UPDATED_AT` si termina bien.

Si Trakt devuelve límite de cuenta al subir notas, la sincronización marca un error importante para informar al usuario.

## Caso especial: añadir una serie manualmente

Cuando el usuario añade una serie y Hexagon no está activo, `AddShowTask` puede importar el estado de episodios desde Trakt en ese momento:

1. Antes de añadir la serie, descarga mapas de colección y vistos con `GET /sync/collection/shows` y `GET /sync/watched/shows`.
2. `AddUpdateShowTools.addShow(...)` crea la serie, temporadas y episodios desde TMDB.
3. Después de insertar los episodios, `TraktEpisodeSync.storeEpisodeFlags(...)` aplica watched y collected solo para esa serie recién añadida.

Este flujo no importa series nuevas desde Trakt automáticamente; solo restaura estado de episodios para una serie que el usuario ya decidió añadir.

## Qué no importa desde Trakt

- No autoañade series completas desde la watchlist o el historial de Trakt durante `TraktSync`.
- No descarga metadatos principales desde Trakt; los metadatos de series y películas se descargan de TMDB.
- No sincroniza notas de películas; el flujo de notas documentado solo cubre series.
- Si Hexagon está activo, no importa desde Trakt estados de episodios, películas ni notas, solo ratings.

## Resumen rápido

| Área | Importa desde Trakt | Crea entidades locales | Fuente de metadatos al crear |
| --- | --- | --- | --- |
| Episodios vistos | Sí | No crea series nuevas | TMDB ya debió crear la serie/episodios |
| Episodios en colección | Sí | No crea series nuevas | TMDB ya debió crear la serie/episodios |
| Películas colección/watchlist/vistas | Sí | Sí, añade películas faltantes | TMDB movie summary + release dates |
| Ratings | Sí | No | N/A |
| Notas de series | Sí | No crea series nuevas | N/A |
| Al añadir una serie | Restaura watched/collection de esa serie | Sí, por acción del usuario | TMDB show + seasons |
