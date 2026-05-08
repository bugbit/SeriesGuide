# Base de datos de la app SeriesGuide

Este documento resume las bases de datos locales usadas por la app Android, su esquema principal y cómo se relacionan las tablas. La base principal es `SgRoomDatabase`, pero la app también tiene bases separadas para Billing y diagnósticos.

## Resumen de bases de datos

| Base de datos | Nombre de fichero | Versión | Propósito | Código |
| --- | --- | --- | --- | --- |
| `SgRoomDatabase` | `seriesdatabase` | 54 | Datos funcionales de la app: series, temporadas, episodios, películas, listas, jobs, actividad y proveedores de streaming. | `app/src/main/java/com/battlelancer/seriesguide/provider/SgRoomDatabase.kt` |
| `LocalBillingDb` | `purchase_db` | 4 | Caché local de compras/entitlements de Play Billing y estado global de desbloqueo. | `app/src/main/java/com/battlelancer/seriesguide/billing/localdb/LocalBillingDb.kt` |
| `DebugLogDatabase` | `seriesguide_debug_log` | 1 | Logs de diagnóstico guardados localmente. | `app/src/main/java/com/battlelancer/seriesguide/diagnostics/DebugLogDatabase.kt` |

Los esquemas exportados por Room están en:

- `app/schemas/com.battlelancer.seriesguide.provider.SgRoomDatabase/`
- `app/schemas/com.battlelancer.seriesguide.billing.localdb.LocalBillingDb/`
- `app/schemas/com.battlelancer.seriesguide.diagnostics.DebugLogDatabase/`

## Arquitectura de acceso

### Room como acceso principal

`SgRoomDatabase` declara entidades Room para:

- tablas legacy de series: `series`, `seasons`, `episodes`
- tablas actuales de series: `sg_show`, `sg_season`, `sg_episode`
- listas: `lists`, `listitems`
- películas: `movies`
- actividad: `activity`
- cola de trabajos: `jobs`
- proveedores de streaming: `sg_watch_provider`, `sg_watch_provider_show_mappings`

Los DAOs principales expuestos por `SgRoomDatabase` son:

| DAO | Uso principal |
| --- | --- |
| `SgShow2Helper` | Consultas y actualizaciones de series actuales (`sg_show`). |
| `SgSeason2Helper` | Temporadas actuales (`sg_season`). |
| `SgEpisode2Helper` | Episodios actuales (`sg_episode`). |
| `SgActivityHelper` | Historial de actividad de episodios. |
| `SgListHelper` | Listas personalizadas. |
| `MovieHelper` | Películas. |
| `SgWatchProviderHelper` | Proveedores de streaming y mappings. |

### ContentProvider legacy

`SeriesGuideProvider` sigue existiendo como ContentProvider legacy. El propio código lo describe como legacy y recomienda que el código nuevo use Room. Actualmente expone rutas principalmente para:

- `lists`
- `listitems`
- `movies`
- `jobs`

### FTS de episodios

Además de las tablas Room, se crea manualmente una tabla virtual FTS4 llamada `searchtable` porque no está soportada por Room en este proyecto. La tabla usa `sg_episode` como contenido externo y solo indexa:

- `episode_title`
- `episode_description`

Se recrea y repuebla con `SeriesGuideDatabase.rebuildFtsTable(...)`, por ejemplo tras añadir series, importar datos o terminar sincronizaciones relevantes.

## Esquema principal (`SgRoomDatabase`)

### Vista general de relaciones

```text
sg_show 1 ── n sg_season
sg_show 1 ── n sg_episode
sg_season 1 ── n sg_episode  (índice por season_id, aunque la FK Room actual solo referencia show)

lists 1 ── n listitems

movies es independiente y se identifica funcionalmente por TMDB ID.
activity guarda actividad usando IDs estables TVDB/TMDB.
jobs guarda trabajos pendientes serializados.
sg_watch_provider se relaciona con sg_show mediante sg_watch_provider_show_mappings.
```

> Nota: las tablas legacy `series`, `seasons` y `episodes` siguen en el esquema como respaldo/compatibilidad después de la migración a tablas `sg_*` basada en TMDB IDs.

## Tablas actuales de series

### `sg_show`

Entidad: `SgShow2`.

Representa una serie actual. Usa `_id` autogenerado como clave primaria local y mantiene IDs externos para sincronizar y enlazar datos.

Campos principales:

| Grupo | Columnas |
| --- | --- |
| Identidad | `_id`, `series_tmdb_id`, `series_tvdb_id`, `series_slug`, `series_trakt_id`, `series_imdbid` |
| Texto/metadatos | `series_title`, `series_title_noarticle`, `series_overview`, `series_genres`, `series_network`, `series_contentrating`, `series_language` |
| Emisión | `series_airstime`, `series_airsdayofweek`, `series_country`, `series_timezone`, `series_firstaired`, `series_runtime`, `series_status` |
| Ratings | `series_rating_tmdb`, `series_rating_tmdb_votes`, `series_rating`, `series_rating_votes`, `series_rating_user` |
| Imágenes | `series_poster`, `series_poster_small` |
| Estado calculado | `series_next`, `series_nextairdate`, `series_nexttext`, `series_lastwatchedid`, `series_lastwatched_ms`, `series_unwatched_count` |
| Estado de usuario | `series_favorite`, `series_hidden`, `series_notify`, `series_syncenabled` |
| Release personalizado | `series_custom_release_time`, `series_custom_day_offset`, `series_custom_timezone` |
| Notas | `series_user_note`, `series_user_note_trakt_id` |
| Sincronización | `series_lastupdate`, `series_lastedit` |

Índices:

- `series_tmdb_id`
- `series_tvdb_id`

Uso:

- TMDB aporta la mayoría de metadatos, IDs externos, ratings TMDB y póster.
- Trakt aporta ID Trakt, horario complementario, ratings Trakt, ratings de usuario y notas.
- Hexagon sincroniza estados como favorito, oculto, notificaciones, idioma, hora personalizada y notas.

### `sg_season`

Entidad: `SgSeason2`.

Representa una temporada de una serie actual.

Campos principales:

- `_id`
- `series_id` como referencia a `sg_show._id`
- `season_tmdb_id`
- `season_tvdb_id`
- `season_number`
- `season_name`
- `season_order`
- campos legacy/deprecados de contadores: `season_watchcount`, `season_willaircount`, `season_noairdatecount`, `season_totalcount`, `season_tags`

Relación e índices:

- FK `series_id -> sg_show._id`
- índice `series_id`

Uso:

- Se crea a partir de la lista `seasons` devuelta por TMDB al descargar detalles de una serie.
- Los contadores antiguos se mantienen por compatibilidad; ahora varias estadísticas se calculan dinámicamente.

### `sg_episode`

Entidad: `SgEpisode2`.

Representa un episodio actual.

Campos principales:

| Grupo | Columnas |
| --- | --- |
| Identidad | `_id`, `episode_tmdb_id`, `episode_tvdb_id`, `episode_imdbid` |
| Relaciones | `season_id`, `series_id` |
| Orden/numeración | `episode_number`, `episode_absolute_number`, `episode_season_number`, `episode_order`, `episode_dvd_number` |
| Texto/metadatos | `episode_title`, `episode_description`, `episode_directors`, `episode_gueststars`, `episode_writers` |
| Imagen | `episode_image` |
| Emisión | `episode_firstairedms` |
| Estado de usuario | `episode_watched`, `episode_plays`, `episode_collected`, `episode_rating_user` |
| Ratings externos | `episode_rating_tmdb`, `episode_rating_tmdb_votes`, `episode_rating`, `episode_rating_votes` |
| Sincronización | `episode_lastedit`, `episode_lastupdate` |

Relación e índices:

- FK `series_id -> sg_show._id`
- índice `season_id`
- índice `series_id`

Uso:

- TMDB aporta título, descripción, numeración, still, fecha de emisión, crew/guest stars y ratings TMDB.
- Trakt y Hexagon sincronizan watched/plays/collected.
- Trakt importa ratings de usuario.
- La tabla `searchtable` indexa título y descripción para búsqueda de episodios.

## Tablas legacy de series

### `series`, `seasons`, `episodes`

Estas tablas corresponden al modelo antiguo previo a la migración completa a `sg_show`, `sg_season` y `sg_episode`.

- `series`: series antiguas con `_id` basado históricamente en TVDB ID.
- `seasons`: temporadas antiguas referenciando `series._id`.
- `episodes`: episodios antiguos referenciando `series._id` y `seasons._id`.

Uso actual:

- Se conservan por compatibilidad y respaldo de migraciones.
- La lógica nueva debe usar las tablas `sg_*`.
- Algunas consultas y migraciones aún conocen IDs TVDB legacy para convertirlos a TMDB ID cuando es posible.

## Listas personalizadas

### `lists`

Entidad: `SgList`.

Campos:

- `_id`: PK autogenerada.
- `list_id`: identificador estable único.
- `list_name`: nombre visible.
- `list_order`: orden manual.

Índice:

- `list_id` único.

Dato inicial:

- En `onCreate`, la base crea una primera lista con el nombre localizado `first_list`.

### `listitems`

Entidad: `SgListItem`.

Campos:

- `_id`: PK autogenerada.
- `list_item_id`: identificador estable único.
- `item_ref_id`: ID del elemento referenciado, guardado como texto.
- `item_type`: tipo de elemento.
- `list_id`: referencia a `lists.list_id`.

Relación e índices:

- FK `list_id -> lists.list_id`
- índice único `list_item_id`
- índice `list_id`

Notas:

- `SeriesGuideDatabase.Tables.LIST_ITEMS_WITH_DETAILS` construye una vista SQL por `UNION` para resolver elementos de listas contra shows TMDB, shows TVDB legacy, temporadas legacy y episodios legacy.
- Las listas se sincronizan con Hexagon.

## Películas

### `movies`

Entidad: `SgMovie`.

Representa películas por TMDB ID.

Campos principales:

| Grupo | Columnas |
| --- | --- |
| Identidad | `_id`, `movies_tmdbid`, `movies_imdbid` |
| Texto/metadatos | `movies_title`, `movies_title_noarticle`, `movies_genres`, `movies_overview`, `movies_certification` |
| Fechas/duración | `movies_released`, `movies_runtime`, `movies_last_updated` |
| Imagen/tráiler | `movies_poster`, `movies_trailer` |
| Estado de usuario | `movies_incollection`, `movies_inwatchlist`, `movies_watched`, `movies_plays`, `movies_rating_user` |
| Ratings externos | `movies_rating_tmdb`, `movies_rating_votes_tmdb`, `movies_rating_trakt`, `movies_rating_votes_trakt` |

Índice:

- `movies_tmdbid` único.

Uso:

- TMDB aporta metadatos, póster, fecha de estreno por región y ratings TMDB.
- Trakt sincroniza colección, watchlist, watched/plays y ratings de usuario.
- Hexagon sincroniza colección, watchlist, watched/plays.
- Las películas sin estado útil pueden eliminarse con `MovieTools.deleteUnusedMovies(...)`.

## Actividad

### `activity`

Entidad: `SgActivity`.

Guarda actividad de episodios vistos usando IDs estables para que funcione aunque una serie se elimine y se vuelva a añadir.

Campos:

- `_id`
- `activity_episode`: ID TVDB o TMDB del episodio.
- `activity_show`: ID TVDB o TMDB de la serie.
- `activity_time`: timestamp en milisegundos.
- `activity_type`: tipo de ID (`TVDB_ID` o `TMDB_ID`).

Índice:

- único compuesto por `activity_episode` y `activity_type`.

## Cola de trabajos

### `jobs`

Entidad: `SgJob`.

Guarda trabajos pendientes de red/sincronización.

Campos:

- `_id`
- `job_created_at`
- `job_type`
- `job_extras` como BLOB.

Índice:

- `job_created_at` único.

Uso:

- Persiste acciones que deben ejecutarse más tarde, por ejemplo cambios de episodios o películas que deben enviarse a Trakt o Hexagon.

## Proveedores de streaming

### `sg_watch_provider`

Entidad: `SgWatchProvider`.

Campos:

- `_id`
- `provider_id`: ID del proveedor en TMDB.
- `provider_name`
- `display_priority`
- `logo_path`
- `type`: `1` para series, `2` para películas.
- `enabled`: usado para filtros de discovery.
- `filter_local`: usado para filtrar biblioteca local.

Índices:

- único compuesto por `provider_id` y `type`
- `provider_name`
- `display_priority`
- `enabled`
- `type`

### `sg_watch_provider_show_mappings`

Entidad: `SgWatchProviderShowMapping`.

Tabla puente entre proveedores y series locales.

Campos:

- `provider_id`
- `show_id`

Clave primaria:

- compuesta por `provider_id` y `show_id`.

Uso:

- Permite filtrar series locales por proveedores disponibles.

## Base de datos de Billing (`LocalBillingDb`)

Nombre: `purchase_db`.

Versión actual: 4.

Tablas:

| Tabla | Entidad | Propósito |
| --- | --- | --- |
| `purchase_table` | `CachedPurchase` | Caché de compras activas de Google Play Billing. Guarda el JSON original y firma mediante `PurchaseTypeConverter`. |
| `gold_status` | `PlayUnlockState` | Estado de desbloqueo obtenido por compra o suscripción de Play Billing. |
| `unlock_state` | `UnlockStateDb` | Estado global de desbloqueo, último desbloqueo y si se debe notificar expiración. |

Notas de migración:

- Versión 2 añadió `purchaseToken` para suscripciones.
- Versión 3 eliminó `AugmentedSkuDetails`.
- Versión 4 añadió `unlock_state` y `last_updated_ms` en `gold_status`.
- Las migraciones desde versiones 1 y 2 son destructivas; de 3 a 4 hay migración manual.

## Base de datos de diagnósticos (`DebugLogDatabase`)

Nombre: `seriesguide_debug_log`.

Versión actual: 1.

Tabla:

### `debug_log`

Entidad: `DbDebugLogEntry`.

Campos:

- `id`: PK autogenerada.
- `priority`: prioridad del log.
- `tag`: etiqueta.
- `message`: mensaje.
- `created_at`: timestamp en milisegundos.

Operaciones DAO:

- insertar entrada.
- leer todas las entradas.
- borrar entradas anteriores a un timestamp.
- recortar a un máximo de 3000 filas conservando las más recientes.

## Migraciones principales de `SgRoomDatabase`

La versión actual es 54. Los hitos recientes más importantes son:

| Versión | Cambio |
| --- | --- |
| 43 | Introducción/validación Room desde esquema legacy. |
| 44 | Recreación de tablas legacy `series` y `episodes` para validación Room. |
| 45 | Recreación de `seasons`. |
| 46 | Añade slug de serie. |
| 47 | Añade póster pequeño. |
| 48 | Añade plays de episodios. |
| 49 | Migración grande a IDs autogenerados y tablas `sg_show`, `sg_season`, `sg_episode` con TMDB ID. |
| 50 | Añade `sg_watch_provider`. |
| 51 | Añade release time/day/timezone personalizados en series. |
| 52 | Añade mappings de proveedores de streaming y `filter_local`. |
| 53 | Añade ratings TMDB en series y episodios. |
| 54 | Añade notas de usuario en series y Trakt note ID. |

## Reglas prácticas para tocar la base de datos

- Si se modifica una entidad Room, actualizar versión y añadir migración o auto-migración según corresponda.
- Mantener los esquemas exportados en `app/schemas/...` sincronizados.
- Preferir DAOs Room para código nuevo; evitar ampliar el ContentProvider legacy salvo compatibilidad.
- Revisar índices al añadir FKs o consultas frecuentes.
- Recordar que varios booleanos se almacenan como `INTEGER` en SQLite.
- Tras cambios que afecten título/descripción de episodios, considerar reconstruir `searchtable`.
- Al tocar series/episodios, tener en cuenta la convivencia entre IDs TMDB actuales e IDs TVDB legacy.
