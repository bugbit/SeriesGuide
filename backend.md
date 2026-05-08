# Backend de SeriesGuide: endpoints, uso y dependencias de terceros

Este documento describe los endpoints del backend **Hexagon / SeriesGuide Cloud** definidos en `backend/src/endpoints/*-v2-rest.discovery` y cómo los usa la app Android. Además, para cada endpoint se indican los endpoints de terceros que se usan en el flujo cliente asociado, si los hay.

> Importante: los endpoints de Hexagon sincronizan **estado de usuario** (series añadidas, favoritos, episodios vistos, películas en listas, listas personalizadas, etc.). Los metadatos públicos (títulos, sinopsis, temporadas, episodios, pósters, fechas de estreno) se obtienen de TMDB y algunos datos complementarios de Trakt.

## Información común

- **Base URL:** `https://optical-hexagon-364.appspot.com/_ah/api/`
- **Autenticación:** todos los servicios Discovery declaran OAuth2 con scope `https://www.googleapis.com/auth/userinfo.email`.
- **Cliente Android:** `HexagonTools` prepara los clientes generados y les asigna la cuenta/autenticación actual.
- **Paginación:** varios endpoints devuelven un `cursor`; el cliente repite la llamada con ese cursor hasta que no haya más páginas.
- **Filtro incremental:** varios endpoints aceptan `updatedSince` para descargar solo cambios desde la última sincronización guardada en `HexagonSettings`.
- **Endpoints de terceros:** Hexagon no descarga metadatos de TMDB/Trakt por sí mismo en el cliente. Cuando un endpoint de Hexagon devuelve IDs o estado que requieren crear/actualizar entidades locales, la app llama a TMDB/Trakt alrededor de ese endpoint.

## Resumen de servicios

| Servicio | Service path | Archivo Discovery |
| --- | --- | --- |
| Account | `account/v2/` | `backend/src/endpoints/account-v2-rest.discovery` |
| Shows | `shows/v2/` | `backend/src/endpoints/shows-v2-rest.discovery` |
| Episodes | `episodes/v2/` | `backend/src/endpoints/episodes-v2-rest.discovery` |
| Movies | `movies/v2/` | `backend/src/endpoints/movies-v2-rest.discovery` |
| Lists | `lists/v2/` | `backend/src/endpoints/lists-v2-rest.discovery` |

## Endpoints de terceros más relevantes

Estos son los endpoints externos que aparecen en los flujos asociados a la sincronización Cloud:

| Proveedor | Endpoint | Uso |
| --- | --- | --- |
| TMDB | `GET /find/{external_id}?external_source=tvdb_id` | Convertir shows legacy guardados por TVDB ID a TMDB ID. |
| TMDB | `GET /tv/{tv_id}?append_to_response=external_ids` | Descargar detalles de serie e IDs externos antes de insertar o actualizar una serie local. |
| TMDB | `GET /tv/{tv_id}/season/{season_number}` | Descargar episodios de cada temporada al añadir/actualizar una serie. |
| TMDB | `GET /movie/{movie_id}?append_to_response=release_dates` | Descargar detalles de película y fechas de estreno por región al crear una película local desde Cloud. |
| TMDB | `GET /movie/{movie_id}` | Fallback para overview de película si falta la traducción solicitada. |
| Trakt | `GET /search/tmdb/{id}?type=show&extended=full` | Completar series con Trakt ID, horario, país, zona horaria y rating/votos de Trakt. |
| Trakt | `GET /search/tmdb/{id}?type=movie` | Obtener Trakt ID de película en flujos que necesitan ratings o historial de Trakt. |
| Firebase/Google | OAuth / identidad de usuario | Autenticación usada por los clientes de Hexagon. |

## Account v2

### `GET account/v2/deleteData`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/account/v2/deleteData`
- **Parámetros:** ninguno.
- **Cuerpo:** ninguno.
- **Respuesta:** sin modelo específico.
- **Uso en app:** `RemoveCloudAccountDialogFragment`.
- **Resumen:** borra los datos Cloud asociados a la cuenta autenticada. Se usa cuando el usuario confirma que quiere eliminar su cuenta/datos de SeriesGuide Cloud.
- **Endpoints de terceros usados en el flujo:**
  - Firebase/Google OAuth para identificar la cuenta autenticada.
  - No usa TMDB ni Trakt en este flujo.

## Shows v2

### `GET shows/v2/get`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/shows/v2/get`
- **Parámetros query:**
  - `updatedSince` (`string`, opcional): fecha/hora desde la que descargar cambios.
  - `cursor` (`string`, opcional): cursor de paginación.
  - `limit` (`integer`, opcional): límite de resultados.
- **Respuesta:** `ShowList` con `shows[]` y `cursor`.
- **Modelo principal:** `Show` legacy con `tvdbId`, `isFavorite`, `notify`, `isHidden`, `isRemoved`, `language`, `note`, `createdAt`, `updatedAt`.
- **Uso en app:** `HexagonShowSync.downloadLegacyShows()` durante la primera mezcla Cloud para datos antiguos basados en TVDB ID.
- **Resumen:** descarga series legacy guardadas en Cloud con TVDB ID. El cliente intenta migrarlas a TMDB ID y convertirlas a `SgCloudShow` para aplicar estado local o añadir la serie.
- **Endpoints de terceros usados en el flujo:**
  - TMDB `GET /find/{external_id}?external_source=tvdb_id` para mapear `tvdbId` a `tmdbId`.
  - Si la serie no existe localmente y se agenda su alta: TMDB `GET /tv/{tv_id}?append_to_response=external_ids`, TMDB `GET /tv/{tv_id}/season/{season_number}` y Trakt `GET /search/tmdb/{id}?type=show&extended=full`.

### `GET shows/v2/getShow`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/shows/v2/getShow`
- **Parámetros query:**
  - `showTvdbId` (`integer`, opcional): TVDB ID de la serie legacy.
- **Respuesta:** `Show` legacy.
- **Uso en app:** fallback de `HexagonTools.getShow(showTmdbId, showTvdbId)` si `getSgShow` no devuelve nada y hay TVDB ID.
- **Resumen:** obtiene una única serie legacy por TVDB ID para restaurar propiedades antiguas como favorita, oculta o notificaciones al añadir una serie.
- **Endpoints de terceros usados en el flujo:**
  - Normalmente se invoca dentro de un alta que ya ha consultado TMDB `GET /tv/{tv_id}?append_to_response=external_ids` para conocer IDs externos.
  - El endpoint en sí no requiere TMDB ni Trakt directamente.

### `GET shows/v2/getSgShow`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/shows/v2/getSgShow`
- **Parámetros query:**
  - `showTmdbId` (`integer`, opcional): TMDB ID de la serie.
- **Respuesta:** `SgCloudShow`.
- **Modelo principal:** `SgCloudShow` con `tmdbId`, `isFavorite`, `notify`, `isHidden`, `isRemoved`, `language`, `customReleaseTime`, `customReleaseDayOffset`, `customReleaseTimeZone`, `note`, `createdAt`, `updatedAt`.
- **Uso en app:** `HexagonTools.getShow()` al añadir una serie para restaurar estado Cloud antes de insertarla localmente.
- **Resumen:** obtiene el estado Cloud de una serie concreta por TMDB ID. Permite restaurar favoritos, notificaciones, ocultación, idioma, hora personalizada y nota.
- **Endpoints de terceros usados en el flujo:**
  - TMDB `GET /tv/{tv_id}?append_to_response=external_ids` para construir la serie local.
  - TMDB `GET /tv/{tv_id}/season/{season_number}` para poblar episodios.
  - Trakt `GET /search/tmdb/{id}?type=show&extended=full` para completar horario y ratings de Trakt.

### `GET shows/v2/getSgShows`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/shows/v2/getSgShows`
- **Parámetros query:**
  - `updatedSince` (`string`, opcional): fecha/hora desde la que descargar cambios.
  - `cursor` (`string`, opcional): cursor de paginación.
  - `limit` (`integer`, opcional): límite de resultados.
- **Respuesta:** `SgCloudShowList` con `shows[]` y `cursor`.
- **Uso en app:** `HexagonShowSync.downloadShows()` en sincronización normal e inicial.
- **Resumen:** descarga cambios de estado de series actuales por TMDB ID. Para series existentes actualiza propiedades Cloud; para series no existentes agenda su alta local si no están marcadas como eliminadas.
- **Endpoints de terceros usados en el flujo:**
  - Para shows nuevos agendados: TMDB `GET /tv/{tv_id}?append_to_response=external_ids`, TMDB `GET /tv/{tv_id}/season/{season_number}` y Trakt `GET /search/tmdb/{id}?type=show&extended=full`.
  - Si solo actualiza propiedades de una serie local existente, no necesita endpoints de terceros.

### `PUT shows/v2/save`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/shows/v2/save`
- **Cuerpo:** `ShowList` legacy.
- **Respuesta:** sin modelo específico.
- **Uso en app:** endpoint legacy generado; el flujo actual usa principalmente `saveSgShows`.
- **Resumen:** guarda una lista de series legacy basadas en TVDB ID.
- **Endpoints de terceros usados en el flujo:** ninguno en el flujo actual.

### `PUT shows/v2/saveSgShows`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/shows/v2/saveSgShows`
- **Cuerpo:** `SgCloudShowList`.
- **Respuesta:** sin modelo específico.
- **Uso en app:** `HexagonShowSync.upload()`, `HexagonShowSync.uploadAll()` y alta de series en `AddUpdateShowTools`.
- **Resumen:** sube a Cloud estado de series por TMDB ID: favorita, oculta, notificaciones, idioma, hora personalizada, nota y marca de eliminada.
- **Endpoints de terceros usados en el flujo:**
  - El upload no usa terceros directamente.
  - Si se llama después de añadir una serie, antes se han usado TMDB `GET /tv/{tv_id}?append_to_response=external_ids`, TMDB `GET /tv/{tv_id}/season/{season_number}` y Trakt `GET /search/tmdb/{id}?type=show&extended=full` para crear la entidad local.

## Episodes v2

### `GET episodes/v2/get`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/episodes/v2/get`
- **Parámetros query:**
  - `updatedSince` (`string`, opcional): fecha/hora desde la que descargar cambios.
  - `showTvdbId` (`integer`, opcional): TVDB ID de la serie legacy.
  - `cursor` (`string`, opcional): cursor de paginación.
  - `limit` (`integer`, opcional): límite de resultados.
- **Respuesta:** `EpisodeList` con `episodes[]`, `showTvdbId` y `cursor`.
- **Modelo principal:** `Episode` legacy con `showTvdbId`, `seasonNumber`, `episodeNumber`, `watchedFlag`, `plays`, `isInCollection`, `createdAt`, `updatedAt`.
- **Uso en app:** fallback legacy en `HexagonEpisodeSync.downloadFlags()` si no hay datos por TMDB ID.
- **Resumen:** descarga flags legacy de episodios vinculados a una serie por TVDB ID.
- **Endpoints de terceros usados en el flujo:**
  - No usa terceros directamente.
  - En migraciones de series legacy puede estar relacionado con TMDB `GET /find/{external_id}?external_source=tvdb_id` para convertir la serie a TMDB ID.

### `GET episodes/v2/getSgEpisodes`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/episodes/v2/getSgEpisodes`
- **Parámetros query:**
  - `updatedSince` (`string`, opcional): fecha/hora desde la que descargar cambios.
  - `showTmdbId` (`integer`, opcional): TMDB ID de la serie.
  - `cursor` (`string`, opcional): cursor de paginación.
  - `limit` (`integer`, opcional): límite de resultados.
- **Respuesta:** `SgCloudEpisodeList` con `episodes[]`, `showTmdbId` y `cursor`.
- **Modelo principal:** `SgCloudEpisode` con `showTmdbId`, `seasonNumber`, `episodeNumber`, `watchedFlag`, `plays`, `isInCollection`, `createdAt`, `updatedAt`.
- **Uso en app:** `HexagonEpisodeSync.downloadChangedFlags()` para cambios globales y `downloadFlagsByTmdbId()` para restaurar flags de una serie concreta.
- **Resumen:** descarga estado Cloud de episodios por TMDB ID o desde una fecha: visto/no visto/saltado, reproducciones y colección.
- **Endpoints de terceros usados en el flujo:**
  - No usa terceros directamente.
  - Para que existan episodios locales donde aplicar los flags, el alta/actualización de serie usa TMDB `GET /tv/{tv_id}/season/{season_number}`.

### `PUT episodes/v2/save`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/episodes/v2/save`
- **Cuerpo:** `EpisodeList` legacy.
- **Respuesta:** sin modelo específico.
- **Uso en app:** endpoint legacy generado; el flujo actual usa principalmente `saveSgEpisodes`.
- **Resumen:** guarda flags legacy de episodios asociados a TVDB ID.
- **Endpoints de terceros usados en el flujo:** ninguno en el flujo actual.

### `PUT episodes/v2/saveSgEpisodes`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/episodes/v2/saveSgEpisodes`
- **Cuerpo:** `SgCloudEpisodeList`.
- **Respuesta:** sin modelo específico.
- **Uso en app:** `HexagonEpisodeSync.uploadFlags()` y `HexagonEpisodeJob`.
- **Resumen:** sube flags de episodios por TMDB ID: visto/saltado, reproducciones y colección.
- **Endpoints de terceros usados en el flujo:** ninguno directamente. El endpoint solo refleja estado local ya calculado o modificado por el usuario.

## Movies v2

### `GET movies/v2/get`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/movies/v2/get`
- **Parámetros query:**
  - `updatedSince` (`string`, opcional): fecha/hora desde la que descargar cambios.
  - `cursor` (`string`, opcional): cursor de paginación.
  - `limit` (`integer`, opcional): límite de resultados.
- **Respuesta:** `MovieList` con `movies[]` y `cursor`.
- **Modelo principal:** `Movie` con `tmdbId`, `isInCollection`, `isInWatchlist`, `isWatched`, `plays`, `createdAt`, `updatedAt`.
- **Uso en app:** `HexagonMovieSync.download()`.
- **Resumen:** descarga estado Cloud de películas. Actualiza películas existentes, elimina las que ya no están en ninguna lista/estado y agenda altas locales para películas nuevas.
- **Endpoints de terceros usados en el flujo:**
  - Para películas nuevas: TMDB `GET /movie/{movie_id}?append_to_response=release_dates`.
  - Si falta overview localizado: TMDB `GET /movie/{movie_id}` como fallback.
  - No usa Trakt en `MovieTools.addMovies(...)` porque se llama con `getTraktRating = false` en este flujo.

### `PUT movies/v2/save`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/movies/v2/save`
- **Cuerpo:** `MovieList`.
- **Respuesta:** sin modelo específico.
- **Uso en app:** `HexagonMovieSync.uploadAll()` y `HexagonMovieJob`.
- **Resumen:** sube estado de películas por TMDB ID: colección, watchlist, vista/no vista y número de reproducciones.
- **Endpoints de terceros usados en el flujo:** ninguno directamente. El endpoint sube estado local; los metadatos de la película se obtienen de TMDB cuando la película se crea o actualiza localmente.

## Lists v2

### `GET lists/v2/get`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/lists/v2/get`
- **Parámetros query:**
  - `updatedSince` (`string`, opcional): fecha/hora desde la que descargar cambios.
  - `cursor` (`string`, opcional): cursor de paginación.
  - `limit` (`integer`, opcional): límite de resultados.
- **Respuesta:** `SgListList` con `lists[]` y `cursor`.
- **Modelo principal:** `SgList` con `listId`, `name`, `order`, `listItems[]`, `createdAt`, `updatedAt`; cada `SgListItem` tiene `listItemId`.
- **Uso en app:** `HexagonListsSync.download()`.
- **Resumen:** descarga listas personalizadas y sus elementos para crearlas, actualizarlas o eliminar elementos locales que ya no están en Cloud.
- **Endpoints de terceros usados en el flujo:** ninguno directamente. Los `listItemId` referencian elementos ya conocidos por la app; si falta la entidad local, su creación depende de los flujos de series o películas descritos arriba.

### `GET lists/v2/getIds`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/lists/v2/getIds`
- **Parámetros query:**
  - `cursor` (`string`, opcional): cursor de paginación.
  - `limit` (`integer`, opcional): límite de resultados.
- **Respuesta:** `SgListIds` con `listIds[]` y `cursor`.
- **Uso en app:** `HexagonListsSync.pruneRemovedLists()`.
- **Resumen:** descarga solo IDs de listas existentes en Cloud para borrar localmente las listas que ya fueron eliminadas en otros dispositivos.
- **Endpoints de terceros usados en el flujo:** ninguno.

### `DELETE lists/v2/remove`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/lists/v2/remove`
- **Parámetros query:**
  - `listId` (`string`, requerido): ID estable de la lista a eliminar.
- **Respuesta:** sin modelo específico.
- **Uso en app:** `DeleteListTask`.
- **Resumen:** elimina una lista personalizada de Cloud. Localmente, la app también elimina sus elementos para mantener integridad de base de datos.
- **Endpoints de terceros usados en el flujo:** ninguno.

### `PUT lists/v2/removeItems`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/lists/v2/removeItems`
- **Cuerpo:** `SgListList` con listas y elementos a quitar.
- **Respuesta:** sin modelo específico.
- **Uso en app:** `RemoveListItemTask`, `ChangeListItemListsTask` y `ListsTools2`.
- **Resumen:** elimina elementos concretos de una o varias listas en Cloud sin borrar necesariamente la lista completa.
- **Endpoints de terceros usados en el flujo:** ninguno.

### `PUT lists/v2/save`

- **URL completa:** `https://optical-hexagon-364.appspot.com/_ah/api/lists/v2/save`
- **Cuerpo:** `SgListList`.
- **Respuesta:** sin modelo específico.
- **Uso en app:** `HexagonListsSync.uploadAll()`, `AddListTask`, `ChangeListItemListsTask`, `ReorderListsTask` y `ListsTools2`.
- **Resumen:** crea o actualiza listas personalizadas, su orden y sus elementos.
- **Endpoints de terceros usados en el flujo:** ninguno directamente. Si se añade a una lista un elemento que todavía no existe localmente, su alta depende del flujo correspondiente de serie/película.

## Modelos de datos Cloud

| Modelo | Campos principales |
| --- | --- |
| `SgCloudShow` | `tmdbId`, `isFavorite`, `notify`, `isHidden`, `isRemoved`, `language`, `customReleaseTime`, `customReleaseDayOffset`, `customReleaseTimeZone`, `note`, `createdAt`, `updatedAt` |
| `Show` legacy | `tvdbId`, `isFavorite`, `notify`, `isHidden`, `isRemoved`, `language`, `note`, `createdAt`, `updatedAt` |
| `SgCloudEpisode` | `showTmdbId`, `seasonNumber`, `episodeNumber`, `watchedFlag`, `plays`, `isInCollection`, `createdAt`, `updatedAt` |
| `Episode` legacy | `showTvdbId`, `seasonNumber`, `episodeNumber`, `watchedFlag`, `plays`, `isInCollection`, `createdAt`, `updatedAt` |
| `Movie` | `tmdbId`, `isInCollection`, `isInWatchlist`, `isWatched`, `plays`, `createdAt`, `updatedAt` |
| `SgList` | `listId`, `name`, `order`, `listItems[]`, `createdAt`, `updatedAt` |
| `SgListItem` | `listItemId` |

## Lectura rápida del flujo de sincronización

1. `HexagonSync.sync()` coordina episodios, series, películas y listas.
2. Episodios: descarga cambios con `episodes/v2/getSgEpisodes`, aplica flags a episodios locales y sube flags pendientes con `episodes/v2/saveSgEpisodes`.
3. Series: descarga `shows/v2/getSgShows`; si es la primera mezcla también descarga `shows/v2/get` legacy. Las series no existentes se añaden usando TMDB y Trakt.
4. Películas: descarga `movies/v2/get`; las películas no existentes se crean usando TMDB.
5. Listas: descarga `lists/v2/get`, poda listas con `lists/v2/getIds` y sube cambios con `lists/v2/save`, `lists/v2/removeItems` o `lists/v2/remove`.
6. En una primera mezcla, después de descargar, se sube todo el estado local relevante a Cloud para completar la convergencia entre dispositivos.
