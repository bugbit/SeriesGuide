# Cómo obtiene SeriesGuide los datos y las imágenes de series y películas

Este documento resume el flujo actual de la app para descargar, guardar y mostrar información de series, episodios y películas. Está pensado como guía de lectura del código, no como documentación de API pública.

## Fuentes externas principales

SeriesGuide combina varias fuentes:

- **TMDB**: fuente principal de metadatos de series, temporadas, episodios y películas. También proporciona las rutas relativas de pósteres, imágenes de episodios, fondos, vídeos, créditos y proveedores de streaming.
- **Trakt**: fuente complementaria para ratings, identificadores Trakt, horarios de emisión de series y estados de usuario cuando no se usa Hexagon.
- **Hexagon**: servicio de sincronización de SeriesGuide para restaurar/propagar estado del usuario, como favoritos, ocultos, notas, flags de episodios y ajustes personalizados.
- **TheTVDB legacy**: solo aparece en el flujo de imágenes para compatibilidad con rutas antiguas que no son de TMDB.

La integración con TMDB se inyecta mediante `TmdbModule`: crea servicios de configuración, películas, personas, búsqueda y TV a partir de `SgTmdb`, usando `BuildConfig.TMDB_API_KEY`.


## APIs y endpoints usados

> Los endpoints se muestran en formato REST de alto nivel. En el código muchos no aparecen como strings porque se invocan a través de las librerías `com.uwetrottmann.tmdb2` y `com.uwetrottmann.trakt5`.

### TMDB API

Base lógica: API v3 de TMDB. La app configura el cliente en `SgTmdb` y obtiene servicios desde `TmdbModule`.

| Método | Endpoint | Uso en SeriesGuide | Código principal |
| --- | --- | --- | --- |
| `GET` | `/configuration` | Obtener `secure_base_url` para construir URLs de imágenes TMDB. | `TmdbSync.updateConfiguration()` |
| `GET` | `/search/tv` | Buscar series remotas por texto, año e idioma. | `TmdbTools2.searchShows()` |
| `GET` | `/discover/tv` | Descubrir series populares o con episodios nuevos; admite filtros de idioma, año, idioma original, proveedores y región. | `TmdbTools2.getPopularShows()`, `TmdbTools2.getShowsWithNewEpisodes()` |
| `GET` | `/tv/{tv_id}` | Descargar detalles de una serie. | `TmdbTools2.getShowDetails()` |
| `GET` | `/tv/{tv_id}?append_to_response=external_ids` | Descargar detalles de serie junto con IDs externos como TVDB e IMDb. | `TmdbTools3.getShowAndExternalIds()` |
| `GET` | `/tv/{tv_id}/season/{season_number}` | Descargar episodios de una temporada. | `TmdbTools3.getSeason()` |
| `GET` | `/tv/{tv_id}/videos` | Buscar tráiler de serie en YouTube. | `TmdbTools3.getShowTrailerYoutubeId()` |
| `GET` | `/tv/{tv_id}/recommendations` | Cargar series recomendadas/similares. | `SimilarShowsViewModel` |
| `GET` | `/find/{external_id}?external_source=tvdb_id` | Migrar/relacionar series legacy buscando TMDB ID desde TVDB ID. | `TmdbTools3.findShowTmdbId()` |
| `GET` | `/watch/providers/tv` | Descargar catálogo de proveedores de streaming para series por idioma/región. | `WatchProvidersService.tv()`, `TmdbTools2.getShowWatchProviders()` |
| `GET` | `/tv/{tv_id}/watch/providers` | Descargar proveedores de streaming disponibles para una serie concreta. | `TmdbTools2.getWatchProvidersForShow()` |
| `GET` | `/tv/{tv_id}/season/{season_number}/watch/providers` | Descargar proveedores de streaming de una temporada. | `TmdbTools2.getWatchProvidersForSeason()` |
| `GET` | `/tv/{tv_id}/aggregate_credits` | Descargar reparto y equipo agregado de una serie. | `TmdbTools2.getCreditsForShow()` |
| `GET` | `/tv/{tv_id}/season/{season_number}/episode/{episode_number}/external_ids` | Obtener IDs externos, especialmente IMDb ID, de un episodio. | `TmdbTools2.getImdbIdForEpisode()` |
| `GET` | `/search/movie` | Buscar películas remotas por texto, idioma, región y año. | `TmdbMoviesDataSource.load()` |
| `GET` | `/discover/movie` | Descubrir películas populares, digitales, en cines o próximas; admite filtros de estreno, idioma, región, proveedores y fechas. | `TmdbMoviesDataSource.buildMovieListCall()` |
| `GET` | `/movie/{movie_id}` | Descargar detalles de una película. | `TmdbTools4.getMovieSummary()` |
| `GET` | `/movie/{movie_id}?append_to_response=release_dates` | Descargar película con fechas de estreno por región. | `MovieTools.getEnhancedMovieFromTmdb()` |
| `GET` | `/movie/{movie_id}/videos` | Buscar tráiler de película en YouTube. | `TmdbTools2.getMovieTrailerYoutubeId()` |
| `GET` | `/movie/{movie_id}/recommendations` | Cargar películas recomendadas/similares. | `SimilarMoviesViewModel` |
| `GET` | `/watch/providers/movie` | Descargar catálogo de proveedores de streaming para películas por idioma/región. | `WatchProvidersService.movie()`, `TmdbTools2.getMovieWatchProviders()` |
| `GET` | `/movie/{movie_id}/watch/providers` | Descargar proveedores de streaming de una película concreta. | `TmdbTools2.getWatchProvidersForMovie()` |
| `GET` | `/movie/{movie_id}/credits` | Descargar reparto y equipo de una película. | `TmdbTools2.getCreditsForMovie()` |
| `GET` | `/person/{person_id}` | Descargar ficha de una persona. | `TmdbTools2.getPerson()` |
| `GET` | `/collection/{collection_id}` | Descargar información de una colección de películas. | `TmdbTools2.getMovieCollection()` |

### TMDB Image API y espejos de imágenes

| Tipo | URL o patrón | Uso |
| --- | --- | --- |
| TMDB imágenes | `{secure_base_url}{size}{file_path}` | Patrón general para pósteres, stills, backdrops y perfiles. `secure_base_url` viene de `/configuration`. |
| Póster pequeño | `{secure_base_url}w154{poster_path}` o `{secure_base_url}w342{poster_path}` | Póster de serie/película según densidad de pantalla. |
| Póster grande | `{secure_base_url}w342{poster_path}` | Póster resuelto para película o vistas que requieren más tamaño. |
| Imagen original | `{secure_base_url}original{file_path}` | Imagen a tamaño original cuando se solicita `originalSize`. |
| Still/backdrop | `{secure_base_url}w780{still_path|backdrop_path}` | Imagen de episodio o fondo. |
| Perfil | `{secure_base_url}{w45|w185|h632|original}{profile_path}` | Fotos de personas. |
| TheTVDB legacy | `https://artworks.thetvdb.com/banners/{path}` | Imágenes antiguas que no empiezan por `/`. |
| TheTVDB legacy cache | `https://www.thetvdb.com/banners/{path}` | Rutas antiguas con `_cache/`, usando redirección del sitio legacy. |
| Demo | `https://seriesgui.de/demo/...` | Imágenes fijas cuando está activo el modo demo. |
| Caché SeriesGuide | `{BuildConfig.IMAGE_CACHE_URL}/s{firma}/{url_original}` | Proxy/cache opcional de imágenes firmado con HMAC-SHA256. |

### Trakt API

Base lógica: Trakt API v2. `SgTrakt` extiende `TraktV2`, añade credenciales OAuth y usa los servicios `episodes`, `movies`, `shows`, `search`, `sync`, `users`, `checkin` y `notes`.

| Método | Endpoint | Uso en SeriesGuide | Código principal |
| --- | --- | --- | --- |
| `POST` | `/oauth/token` | Intercambiar código OAuth y refrescar access token. | `TraktAuthActivityModel`, `TraktCredentials` |
| `GET` | `/users/settings` | Obtener ajustes del usuario conectado. | `TraktAuthActivityModel` |
| `GET` | `/search/tmdb/{id}?type=show&extended=full` | Buscar serie de Trakt por TMDB ID para ID Trakt, horario, país, zona horaria y ratings. | `TraktTools3.getShowByTmdbId()` |
| `GET` | `/search/tmdb/{id}?type=movie` | Buscar película de Trakt por TMDB ID para obtener Trakt ID. | `TraktTools.lookupMovieTraktId()` |
| `GET` | `/sync/last_activities` | Saber qué partes de Trakt han cambiado antes de sincronizar. | `TraktTools3.getLastActivity()` |
| `GET` | `/sync/watched/shows` | Descargar series vistas y episodios vistos. | `TraktTools4.getWatchedShows()` |
| `GET` | `/sync/collection/shows` | Descargar colección de episodios/series. | `TraktTools4.getCollectedShows()` |
| `GET` | `/sync/watchlist/shows` | Descargar watchlist de series. | `TraktTools4.getShowsOnWatchlist()` |
| `GET` | `/sync/watched/movies` | Descargar películas vistas y número de reproducciones. | `TraktTools4.getWatchedMoviesByTmdbId()` |
| `GET` | `/sync/collection/movies` | Descargar colección de películas. | `TraktTools4.getCollectedMoviesByTmdbId()` |
| `GET` | `/sync/watchlist/movies` | Descargar watchlist de películas. | `TraktTools4.getMoviesOnWatchlistByTmdbId()` |
| `POST` | `/sync/collection` | Añadir películas o episodios a colección. | `TraktMovieJob`, `TraktEpisodeJob` |
| `POST` | `/sync/collection/remove` | Eliminar películas o episodios de colección. | `TraktMovieJob`, `TraktEpisodeJob` |
| `POST` | `/sync/watchlist` | Añadir películas o series a watchlist. | `TraktMovieJob`, `AddShowToWatchlistTask` |
| `POST` | `/sync/watchlist/remove` | Eliminar películas o series de watchlist. | `TraktMovieJob`, `RemoveShowFromWatchlistTask` |
| `POST` | `/sync/history` | Añadir películas o episodios al historial de vistos. | `TraktMovieJob`, `TraktEpisodeJob` |
| `POST` | `/sync/history/remove` | Eliminar películas o episodios del historial. | `TraktMovieJob`, `TraktEpisodeJob` |
| `GET` | `/users/me/history/{type}/{id}` | Comprobar entradas ya existentes en historial para evitar duplicados. | `TraktMovieJob`, `TraktEpisodeJob` |
| `GET` | `/sync/ratings/shows` | Descargar ratings propios de series. | `TraktRatingsSync` |
| `GET` | `/sync/ratings/episodes` | Descargar ratings propios de episodios. | `TraktRatingsSync` |
| `GET` | `/sync/ratings/movies` | Descargar ratings propios de películas. | `TraktRatingsSync` |
| `POST` | `/sync/ratings` | Enviar rating de usuario para serie, episodio o película. | `BaseRateItemTask` |
| `POST` | `/sync/ratings/remove` | Eliminar rating de usuario. | `BaseRateItemTask` |
| `GET` | `/movies/{id}/ratings` | Obtener rating global de Trakt para película. | `MovieTools.loadRatingsFromTrakt()` |
| `GET` | `/shows/{id}/seasons/{season}/episodes/{episode}/ratings` | Obtener rating global de Trakt para episodio. | `TraktTools2.getEpisodeRatings()` |
| `GET` | `/movies/{id}/comments/{sort}` | Descargar comentarios de película. | `TraktCommentsLoader` |
| `GET` | `/shows/{id}/comments/{sort}` | Descargar comentarios de serie. | `TraktCommentsLoader` |
| `GET` | `/shows/{id}/seasons/{season}/episodes/{episode}/comments/{sort}` | Descargar comentarios de episodio. | `TraktCommentsLoader` |
| `POST` | `/checkin` | Hacer check-in de episodio o película. | `TraktTask` |
| `POST` | `/notes` | Crear o actualizar nota de usuario para una serie. | `TraktTools4.saveNoteForShow()` |
| `DELETE` | `/notes/{id}` | Eliminar nota de usuario. | `TraktTools4.deleteNote()` |

### Hexagon / SeriesGuide Cloud

Los clientes de Hexagon se generan a partir de los ficheros `backend/src/endpoints/*-v2-rest.discovery`. Se usan para sincronizar datos propios de SeriesGuide, no para descargar metadatos públicos de TMDB.

| Servicio | Método | Endpoint | Uso |
| --- | --- | --- | --- |
| account v2 | `GET` | `deleteData` | Borrar datos de la cuenta Cloud. |
| shows v2 | `GET` | `get` | Obtener shows legacy. |
| shows v2 | `GET` | `getShow` | Obtener una serie legacy concreta. |
| shows v2 | `GET` | `getSgShow` | Obtener una serie Cloud por TMDB/TVDB. |
| shows v2 | `GET` | `getSgShows` | Descargar lista de series Cloud. |
| shows v2 | `PUT` | `save` | Guardar shows legacy. |
| shows v2 | `PUT` | `saveSgShows` | Guardar series Cloud actuales. |
| episodes v2 | `GET` | `get` | Obtener episodios legacy. |
| episodes v2 | `GET` | `getSgEpisodes` | Descargar episodios Cloud por serie. |
| episodes v2 | `PUT` | `save` | Guardar episodios legacy. |
| episodes v2 | `PUT` | `saveSgEpisodes` | Guardar episodios Cloud actuales. |
| movies v2 | `GET` | `get` | Descargar estado Cloud de películas. |
| movies v2 | `PUT` | `save` | Guardar estado Cloud de películas. |
| lists v2 | `GET` | `get` | Descargar listas personalizadas. |
| lists v2 | `GET` | `getIds` | Descargar identificadores de listas. |
| lists v2 | `PUT` | `save` | Guardar listas personalizadas. |
| lists v2 | `PUT` | `removeItems` | Eliminar elementos de listas. |
| lists v2 | `DELETE` | `remove` | Eliminar listas. |


## Series: descarga de datos

### Búsqueda y descubrimiento

Para resultados remotos de series se usa `TmdbTools2`:

1. `searchShows(...)` llama al servicio de búsqueda de TMDB (`searchService().tv(...)`) con query, idioma, año de primera emisión y página.
2. `getPopularShows(...)` y `getShowsWithNewEpisodes(...)` usan el builder de descubrimiento de TV de TMDB, con idioma, página, año, idioma original y filtros de proveedores/región cuando existen.

> Nota: `ShowSearchViewModel` no busca en TMDB. Ese ViewModel filtra la base local (`SG_SHOW`) para buscar entre las series ya guardadas.

### Alta o actualización de una serie

El flujo principal está en `AddUpdateShowTools` y `GetShowTools`.

1. `AddShowTask` valida conectividad, prepara datos opcionales de Trakt o Hexagon y llama a `AddUpdateShowTools.addShow(...)`.
2. `addShow(...)` comprueba si la serie ya existe por TMDB ID o, tras descargar detalles, por TVDB ID.
3. `GetShowTools.getShowDetails(...)` descarga los detalles de la serie con `TmdbTools3.getShowAndExternalIds(...)`.
4. La petición de detalles de serie usa TMDB TV details e incluye `EXTERNAL_IDS` para obtener identificadores externos como TVDB e IMDb.
5. Si la descripción no existe en el idioma deseado, se hace una segunda petición con el idioma fallback configurado.
6. En paralelo, `TraktTools3.getShowByTmdbId(...)` se usa para completar información que TMDB no cubre igual, como Trakt ID, horario/día de emisión, país, zona horaria, primera emisión y rating/votos de Trakt.
7. La app mapea el resultado a `SgShow2` o `SgShow2Update`. Entre los campos guardados están título, descripción, género, cadena, runtime, estado, ratings TMDB/Trakt, IMDb ID, TVDB ID y `poster_path`.
8. Después de insertar la serie, se insertan sus temporadas a partir de la lista `seasons` incluida en la respuesta de detalles de TMDB.
9. Para cada temporada se descarga su lista de episodios con `TmdbTools3.getSeason(...)`, que llama al servicio de temporadas de TV de TMDB.
10. Cada episodio se mapea a `SgEpisode2` o `SgEpisode2Update`, guardando título, descripción, número, fecha/hora calculada de emisión, directores, guionistas, estrellas invitadas, ratings TMDB y `still_path` como imagen del episodio.
11. Tras guardar datos base, se restauran flags de usuario desde Hexagon o desde Trakt, según configuración.

### Idiomas y fallback en series

- Para series, el idioma deseado viene de los ajustes de series.
- Si falta descripción de serie en ese idioma, se consulta un idioma fallback y se antepone un aviso de “sin traducción”.
- Para episodios, si falta título o descripción en la respuesta principal, se descarga la misma temporada en el idioma fallback y se toma solo el campo faltante del episodio equivalente.

## Películas: descarga de datos

### Búsqueda y descubrimiento

La clase `TmdbMoviesDataSource` pagina resultados desde TMDB:

1. Si hay texto de búsqueda, llama a `searchService().movie(...)` con query, idioma, región y año.
2. Si no hay query pero sí un enlace de descubrimiento, construye llamadas de discover para películas populares, digitales, en cines o próximas.
3. Los filtros pueden incluir idioma, región, año, idioma original, proveedores de streaming y tipos de estreno.

### Alta o actualización de una película

El flujo principal está en `MovieTools` y `MovieDetails`.

1. Al añadir una película a colección, watchlist o vistas, `MovieTools.addToList(...)` comprueba si ya existe en la base local.
2. Si no existe, `addMovie(...)` llama a `getMovieDetailsWithDefaults(...)`.
3. `getMovieDetails(...)` descarga primero datos TMDB con `getEnhancedMovieFromTmdb(...)`.
4. `getEnhancedMovieFromTmdb(...)` usa `TmdbTools4.getMovieSummary(...)`, que llama a `MoviesService.summary(...)`; puede incluir `RELEASE_DATES` para elegir fecha de estreno según la región configurada.
5. Si TMDB devuelve 404, se marca como “no encontrada” y se evita insertar datos corruptos.
6. Si falta overview en el idioma configurado, se hace otra petición en idioma por defecto y se añade una nota de falta de traducción.
7. Opcionalmente, si se necesitan ratings de Trakt, se busca el Trakt ID a partir del TMDB ID y se descargan ratings de Trakt.
8. `MovieDetails.toContentValuesInsert()` y `toContentValuesUpdate()` convierten el objeto TMDB/Trakt a columnas locales: IMDb ID, título, descripción, póster, duración, ratings, votos y fecha de estreno.

## Imágenes: rutas, URLs y caché

### Qué guarda la base de datos

SeriesGuide no descarga imágenes durante el guardado de metadatos. Normalmente guarda **rutas relativas** devueltas por TMDB:

- Series: `poster_path` se guarda como póster de la serie.
- Episodios: `still_path` se guarda como imagen del episodio.
- Películas: `poster_path` se guarda como póster de película.

Las rutas TMDB suelen empezar por `/`, por ejemplo `/abc123.jpg`. La URL final se construye solo cuando la UI necesita mostrar la imagen.

### Construcción de URLs TMDB

`TmdbTools` centraliza las URLs base y tamaños:

- `getPosterBaseUrl(...)`: póster pequeño, `w154` o `w342` según densidad de pantalla.
- `buildLargePosterUrl(...)`: póster `w342`.
- `buildOriginalSizeImageUrl(...)`: imagen `original`.
- `buildBackdropUrl(...)`: fondo/still con tamaño `w780`.
- `buildProfileImageUrl(...)`: imágenes de personas con tamaños `w45`, `w185`, `h632` u `original`.

La URL base de imágenes procede de `TmdbSettings.getImageBaseUrl(context)`, por lo que el código no codifica directamente `https://image.tmdb.org/t/p/` en cada llamada.

### ImageTools y servidor de caché

`ImageTools` decide cómo transformar una ruta en una URL cargable:

1. Si la ruta está vacía, devuelve `null`.
2. Si la app está en modo demo, devuelve imágenes demo de `seriesgui.de`.
3. Si la ruta contiene el prefijo legacy `_cache/`, crea una URL legacy de TheTVDB usando `https://www.thetvdb.com/banners/` para aprovechar redirecciones.
4. Si la ruta empieza por `/`, se considera imagen TMDB y se construye una URL con el tamaño apropiado.
5. Si no empieza por `/`, se trata como ruta legacy de TheTVDB bajo `https://artworks.thetvdb.com/banners/`.
6. La URL resultante puede envolverse con el servidor de caché de imágenes configurado en `BuildConfig.IMAGE_CACHE_URL`.
7. Si hay servidor de caché, la URL se firma con HMAC-SHA256 usando `BuildConfig.IMAGE_CACHE_SECRET` y se genera una ruta del estilo `IMAGE_CACHE_URL/s<firma>/<url_original>`.

### Carga en la UI con Picasso

La app carga imágenes con Picasso mediante `ImageTools.loadWithPicasso(...)`:

- Si el usuario permite descargar datos grandes en la conexión actual, Picasso puede usar red normalmente.
- Si la conexión está restringida (por ejemplo, solo Wi‑Fi para imágenes), se fuerza `NetworkPolicy.OFFLINE` para usar caché local y no descargar.

### Resolución diferida de pósteres

Hay casos en los que una lista solo conoce el TMDB ID, pero no el `poster_path`. Para eso existe `SgPicassoRequestHandler` con esquemas internos:

- `showtmdb://<id>?language=<idioma>`: descarga detalles mínimos de la serie para obtener `poster_path`, construye URL de imagen y la carga.
- `movietmdb://<id>`: llama a `MovieTools.getMoviePosterPath(...)`, obtiene el `poster_path`, construye URL de póster grande y la carga.

Esto permite mostrar un póster aunque la entidad local todavía no tenga guardada la ruta.

## Resumen rápido por tipo de dato

| Tipo | Fuente principal | Código principal | Qué se guarda |
| --- | --- | --- | --- |
| Serie | TMDB TV details + external IDs | `GetShowTools`, `TmdbTools3` | TMDB ID, TVDB ID, Trakt ID, título, overview, géneros, red, estado, runtime, ratings, póster |
| Temporadas | TMDB TV details (`seasons`) | `AddUpdateShowTools` | TMDB ID, número, orden, nombre |
| Episodios | TMDB season details | `AddUpdateShowTools`, `TmdbTools3` | TMDB ID, título, overview, número, fecha calculada, créditos, ratings, still |
| Película | TMDB movie summary + release dates | `MovieTools`, `TmdbTools4` | TMDB ID, IMDb ID, título, overview, póster, runtime, ratings, fecha de estreno |
| Ratings/horarios serie | Trakt | `GetShowTools`, `TraktTools3` | Trakt ID, rating/votos, país, día/hora/zona, primera emisión |
| Estado usuario | Hexagon o Trakt | `AddUpdateShowTools`, sync helpers | vistos, colección, favoritos, ocultos, notas, flags |
| Imágenes | TMDB image paths o TheTVDB legacy | `ImageTools`, `TmdbTools`, `SgPicassoRequestHandler` | Se guardan rutas; la URL final se construye al mostrar |

## Archivos clave para seguir el flujo

- `app/src/main/java/com/battlelancer/seriesguide/modules/TmdbModule.kt`: creación de servicios TMDB.
- `app/src/main/java/com/battlelancer/seriesguide/tmdbapi/TmdbTools.kt`: construcción de URLs y tamaños de imágenes TMDB.
- `app/src/main/java/com/battlelancer/seriesguide/tmdbapi/TmdbTools2.kt`: búsquedas, discovery, tráilers, créditos y proveedores.
- `app/src/main/java/com/battlelancer/seriesguide/tmdbapi/TmdbTools3.kt`: llamadas TMDB para series, temporadas y episodios con manejo de errores.
- `app/src/main/java/com/battlelancer/seriesguide/tmdbapi/TmdbTools4.kt`: patrón nuevo de llamadas TMDB para películas.
- `app/src/main/java/com/battlelancer/seriesguide/shows/tools/GetShowTools.kt`: mapeo de detalles de serie desde TMDB/Trakt a modelos locales.
- `app/src/main/java/com/battlelancer/seriesguide/shows/tools/AddUpdateShowTools.kt`: alta/actualización de series, temporadas y episodios.
- `app/src/main/java/com/battlelancer/seriesguide/movies/TmdbMoviesDataSource.kt`: búsqueda y discovery paginado de películas.
- `app/src/main/java/com/battlelancer/seriesguide/movies/tools/MovieTools.kt`: alta/actualización de películas y ratings de Trakt.
- `app/src/main/java/com/battlelancer/seriesguide/movies/details/MovieDetails.java`: conversión de película a valores de base de datos.
- `app/src/main/java/com/battlelancer/seriesguide/util/ImageTools.kt`: resolución de rutas, servidor de caché y carga con Picasso.
- `app/src/main/java/com/battlelancer/seriesguide/util/SgPicassoRequestHandler.kt`: resolución diferida de póster por TMDB ID.
