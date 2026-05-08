# Endpoints usados por el frontend Android

Este documento enumera los endpoints que usa el frontend Android de SeriesGuide y explica qué proceso dispara cada llamada. En el código muchas llamadas no aparecen como rutas literales porque se realizan mediante clientes generados o librerías (`tmdb2`, `trakt5`, clientes Discovery de Hexagon/Firebase), así que las rutas se muestran en formato REST equivalente.

## Clientes HTTP del frontend

| Cliente | Base lógica | Código | Uso |
| --- | --- | --- | --- |
| TMDB API v3 | `https://api.themoviedb.org/3/` | `SgTmdb`, `TmdbModule`, `TmdbTools*` | Metadatos de series/películas, búsqueda, discovery, créditos, proveedores y configuración de imágenes. |
| TMDB Image API | `https://image.tmdb.org/t/p/` o `secure_base_url` de `/configuration` | `TmdbSettings`, `TmdbTools`, `ImageTools` | Carga de pósters, stills, backdrops y perfiles. |
| Trakt API v2 | `https://api.trakt.tv/` | `SgTrakt`, `TraktModule`, `TraktTools*` | Sincronización, ratings, comentarios, historial, check-in y OAuth. |
| Hexagon / SeriesGuide Cloud | `https://optical-hexagon-364.appspot.com/_ah/api/` | `HexagonTools`, `Hexagon*Sync`, jobs Hexagon | Sincronización Cloud propia de SeriesGuide. |
| Firebase/Google Auth | SDK Firebase/FirebaseUI | `HexagonTools`, `CloudSetupFragment` | Identidad de usuario para Hexagon. |
| TheTVDB legacy images | `https://artworks.thetvdb.com/banners/`, `https://www.thetvdb.com/banners/` | `ImageTools` | Compatibilidad con rutas antiguas de imágenes. |
| SeriesGuide image cache | `BuildConfig.IMAGE_CACHE_URL` | `ImageTools` | Proxy/cache opcional de imágenes firmado con HMAC-SHA256. |

## Flujo general del frontend

1. **Arranque/sync programado**: `SgSyncAdapter` actualiza configuración TMDB, series/películas, Hexagon y Trakt según credenciales.
2. **Pantallas de búsqueda/discovery**: llaman a TMDB para buscar o descubrir series y películas.
3. **Añadir o actualizar una entidad**: descarga metadatos de TMDB y completa estado de usuario desde Hexagon o Trakt.
4. **Pantallas de detalle**: pueden pedir créditos, tráilers, proveedores, recomendaciones, comentarios o ratings.
5. **Acciones del usuario**: se guardan localmente y se encolan/suben a Trakt o Hexagon según configuración.
6. **Imágenes**: la UI transforma rutas guardadas (`poster_path`, `still_path`, etc.) en URLs finales y las carga con Picasso.

## TMDB API

### Configuración e imágenes

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET /configuration` | Sincronización TMDB (`TmdbSync.updateConfiguration`). | Descarga `secure_base_url` de imágenes y lo guarda en `TmdbSettings`; las URLs de imágenes posteriores se construyen a partir de este valor. |

### Series

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET /search/tv` | Búsqueda remota de series. | Busca series por texto, idioma, año de primera emisión y página. Los resultados se muestran para que el usuario elija una serie. |
| `GET /discover/tv` | Discovery de series populares o con episodios nuevos. | Lista series según filtros de idioma, año, idioma original, región y proveedores de streaming. |
| `GET /tv/{tv_id}` | Resolución puntual de detalles de serie o póster diferido. | Obtiene detalles básicos de una serie; `SgPicassoRequestHandler` puede usarlo para obtener `poster_path` si solo tiene TMDB ID. |
| `GET /tv/{tv_id}?append_to_response=external_ids` | Alta/actualización de serie. | Descarga metadatos de la serie y IDs externos como TVDB e IMDb para mapearlos a `sg_show`. |
| `GET /tv/{tv_id}/season/{season_number}` | Alta/actualización de episodios. | Descarga episodios de una temporada, incluyendo títulos, descripciones, número, still, ratings y créditos básicos. |
| `GET /tv/{tv_id}/videos` | Tráiler de serie. | Busca vídeos y elige un tráiler de YouTube si existe. |
| `GET /tv/{tv_id}/recommendations` | Pantalla de series similares. | Carga recomendaciones/similares para la serie actual. |
| `GET /find/{external_id}?external_source=tvdb_id` | Migración legacy TVDB → TMDB. | Convierte IDs TVDB heredados a TMDB ID para listas, Cloud o series antiguas. |
| `GET /tv/{tv_id}/aggregate_credits` | Pantallas de reparto/equipo de una serie. | Descarga cast y crew agregados; la UI los muestra agrupados/ordenados. |
| `GET /tv/{tv_id}/season/{season_number}/episode/{episode_number}/external_ids` | Resolución de ID externo de episodio. | Obtiene IMDb ID de un episodio cuando se necesita abrir/enlazar ese identificador. |

### Películas

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET /search/movie` | Búsqueda remota de películas. | Busca películas por texto, idioma, región y año. |
| `GET /discover/movie` | Discovery de películas. | Lista películas populares, digitales, en cines o próximas con filtros de idioma, región, proveedores y fechas de estreno. |
| `GET /movie/{movie_id}` | Detalles de película o fallback de idioma. | Descarga resumen de una película; también se usa como fallback si falta overview localizado. |
| `GET /movie/{movie_id}?append_to_response=release_dates` | Alta/actualización de película. | Descarga metadatos y fechas de estreno por región para guardar/actualizar `movies`. |
| `GET /movie/{movie_id}/videos` | Tráiler de película. | Busca vídeos y elige un tráiler de YouTube si existe. |
| `GET /movie/{movie_id}/recommendations` | Pantalla de películas similares. | Carga recomendaciones/similares de la película actual. |
| `GET /movie/{movie_id}/credits` | Pantallas de reparto/equipo de película. | Descarga cast y crew para mostrar personas relacionadas. |
| `GET /collection/{collection_id}` | Colecciones de películas. | Descarga información de una colección cuando la UI necesita datos de la saga/colección. |

### Personas y proveedores de streaming

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET /person/{person_id}` | Pantalla/ficha de persona. | Descarga resumen de una persona de TMDB. |
| `GET /watch/providers/tv` | Configuración/filtros de proveedores de series. | Descarga catálogo de proveedores de streaming por idioma y región. |
| `GET /watch/providers/movie` | Configuración/filtros de proveedores de películas. | Descarga catálogo de proveedores de películas por idioma y región. |
| `GET /tv/{tv_id}/watch/providers` | Detalle de serie. | Obtiene dónde ver una serie en una región. |
| `GET /tv/{tv_id}/season/{season_number}/watch/providers` | Detalle de temporada. | Obtiene dónde ver una temporada concreta. |
| `GET /movie/{movie_id}/watch/providers` | Detalle de película. | Obtiene dónde ver una película en una región. |

## Endpoints de imágenes

| URL/patrón | Proceso frontend | Qué hace |
| --- | --- | --- |
| `{secure_base_url}{size}{file_path}` | Carga normal de imágenes TMDB. | Construye URLs para pósters, stills, backdrops y perfiles. `size` puede ser `w154`, `w342`, `w780`, `w45`, `w185`, `h632` u `original`. |
| `https://artworks.thetvdb.com/banners/{path}` | Carga de imágenes legacy. | Carga imágenes antiguas cuando la ruta no empieza por `/` y no es de TMDB. |
| `https://www.thetvdb.com/banners/{path}` | Carga de imágenes legacy con `_cache/`. | Usa el host legacy porque redirige a miniaturas compatibles. |
| `https://seriesgui.de/demo/...` | Modo demo. | Sustituye imágenes reales por assets demo estables. |
| `{BuildConfig.IMAGE_CACHE_URL}/s{firma}/{url_original}` | Proxy/cache opcional. | Firma la URL original con HMAC-SHA256 y la envuelve para pasar por el servidor de caché. |
| `showtmdb://{id}?language={lang}` | Resolución diferida de póster de serie. | Esquema interno de Picasso: obtiene `poster_path` por TMDB ID y luego carga la imagen real. |
| `movietmdb://{id}` | Resolución diferida de póster de película. | Esquema interno de Picasso: obtiene `poster_path` con TMDB y luego carga el póster grande. |

## Trakt API

### Autenticación y perfil

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `POST /oauth/token` | Conexión o renovación de cuenta Trakt. | Intercambia el código OAuth por tokens o refresca access token. |
| `GET /users/settings` | Post-login Trakt. | Obtiene ajustes del usuario conectado tras autenticar. |
| `GET /users/me/friends` | Historial de amigos. | Descarga amigos del usuario para luego consultar su último episodio/película visto. |

### Búsqueda por IDs externos

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET /search/tmdb/{id}?type=show&extended=full` | Alta/actualización de series. | Obtiene Trakt ID, horario, país, zona horaria, rating y votos de una serie. |
| `GET /search/tmdb/{id}?type=movie` | Películas, comentarios e historial. | Convierte TMDB ID de película a Trakt ID cuando un endpoint Trakt requiere ID Trakt. |

### Sincronización/importación

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET /sync/last_activities` | Inicio de sync Trakt. | Descarga timestamps para saber qué secciones cambiaron. |
| `GET /sync/watched/shows` | Importación de episodios vistos. | Descarga series/temporadas/episodios vistos y plays, mapeados por TMDB ID. |
| `GET /sync/collection/shows` | Importación de colección de episodios. | Descarga episodios en colección. |
| `GET /sync/watchlist/shows` | Watchlist de series. | Descarga series en watchlist cuando el flujo lo necesita. |
| `GET /sync/watched/movies` | Importación de películas vistas. | Descarga películas vistas y número de plays por TMDB ID. |
| `GET /sync/collection/movies` | Importación de colección de películas. | Descarga TMDB IDs de películas en colección. |
| `GET /sync/watchlist/movies` | Importación de watchlist de películas. | Descarga TMDB IDs de películas en watchlist. |
| `GET /sync/ratings/shows` | Ratings de series. | Importa ratings propios de series. |
| `GET /sync/ratings/episodes` | Ratings de episodios. | Importa ratings propios de episodios. |
| `GET /sync/ratings/movies` | Ratings de películas. | Importa ratings propios de películas. |
| `POST /sync/collection` | Subida de colección. | Sube episodios o películas que el usuario marcó como en colección. |
| `POST /sync/collection/remove` | Borrado de colección. | Elimina episodios o películas de la colección de Trakt. |
| `POST /sync/watchlist` | Subida de watchlist. | Añade series o películas a watchlist. |
| `POST /sync/watchlist/remove` | Borrado de watchlist. | Elimina series o películas de watchlist. |
| `POST /sync/history` | Subida de vistos. | Añade episodios o películas al historial de vistos. |
| `POST /sync/history/remove` | Borrado de vistos. | Elimina episodios o películas del historial de vistos. |
| `POST /sync/ratings` | Subida de rating. | Sube rating de usuario para serie, episodio o película. |
| `POST /sync/ratings/remove` | Borrado de rating. | Elimina rating de usuario en Trakt. |

### Historial, comentarios y check-in

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET /users/me/history/episodes` | Pantallas de historial reciente de episodios. | Carga últimos episodios vistos por el usuario. |
| `GET /users/me/history/movies` | Pantallas de historial reciente de películas. | Carga últimas películas vistas por el usuario. |
| `GET /users/{slug}/history/episodes` | Historial de amigos. | Carga el último episodio visto por cada amigo. |
| `GET /users/{slug}/history/movies` | Historial de amigos. | Carga la última película vista por cada amigo. |
| `GET /users/me/history/{type}/{id}` | Evitar duplicados al subir watched. | Comprueba si ya existe una entrada watched en un instante concreto antes de reintentar una subida. |
| `GET /movies/{id}/comments` | Comentarios de película. | Carga comentarios de una película; antes puede resolver TMDB ID a Trakt ID. |
| `GET /shows/{id}/comments` | Comentarios de serie. | Carga comentarios de una serie usando Trakt ID local. |
| `GET /shows/{id}/seasons/{season}/episodes/{episode}/comments` | Comentarios de episodio. | Carga comentarios de un episodio usando Trakt ID de la serie y número de temporada/episodio. |
| `POST /checkin` | Check-in de episodio o película. | Envía check-in rápido o con mensaje a Trakt. |

### Ratings globales y notas

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET /movies/{id}/ratings` | Detalle/actualización de película. | Descarga rating global de Trakt para una película. |
| `GET /shows/{id}/seasons/{season}/episodes/{episode}/ratings` | Detalle de episodio. | Descarga rating global de Trakt para un episodio. |
| `GET /users/me/notes/shows` | Sincronización de notas. | Descarga notas de usuario para series. |
| `POST /notes` | Subida de notas. | Crea o actualiza una nota de serie. |
| `DELETE /notes/{id}` | Borrado de notas. | Elimina una nota de Trakt. |

## Hexagon / SeriesGuide Cloud

Hexagon sincroniza estado propio de SeriesGuide. El frontend usa clientes generados desde `backend/src/endpoints/*-v2-rest.discovery`.

### Account

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET account/v2/deleteData` | Borrar datos Cloud. | El usuario confirma eliminar datos de la cuenta; el frontend pide al backend borrar los datos asociados. |

### Shows

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET shows/v2/getSgShows` | Sync Cloud de series actuales. | Descarga series por TMDB ID con favoritos, ocultos, notificaciones, idioma, hora personalizada, nota e indicador de eliminada. |
| `GET shows/v2/getSgShow` | Añadir una serie. | Obtiene estado Cloud de una serie concreta para restaurar propiedades al añadirla localmente. |
| `PUT shows/v2/saveSgShows` | Subida de estado de series. | Sube cambios de series actuales a Cloud, en lotes. |
| `GET shows/v2/get` | Migración/mezcla legacy. | Descarga series antiguas por TVDB ID durante una primera mezcla. |
| `GET shows/v2/getShow` | Fallback legacy al añadir serie. | Busca una serie legacy concreta por TVDB ID si no hay estado por TMDB ID. |
| `PUT shows/v2/save` | Legacy. | Guarda series legacy; el flujo actual usa principalmente `saveSgShows`. |

### Episodes

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET episodes/v2/getSgEpisodes` | Sync Cloud de episodios actuales. | Descarga watched/skipped, plays y colección por TMDB ID o por cambios desde `updatedSince`. |
| `PUT episodes/v2/saveSgEpisodes` | Subida de flags de episodios. | Sube watched/skipped, plays y colección por serie TMDB. |
| `GET episodes/v2/get` | Fallback legacy. | Descarga flags antiguos por TVDB ID si no hay datos por TMDB. |
| `PUT episodes/v2/save` | Legacy. | Guarda flags legacy; el flujo actual usa principalmente `saveSgEpisodes`. |

### Movies

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET movies/v2/get` | Sync Cloud de películas. | Descarga estado de colección, watchlist, watched y plays de películas por TMDB ID. |
| `PUT movies/v2/save` | Subida de estado de películas. | Sube estado de películas a Cloud desde jobs o sync completa. |

### Lists

| Endpoint | Proceso frontend | Qué hace |
| --- | --- | --- |
| `GET lists/v2/get` | Sync Cloud de listas. | Descarga listas personalizadas, orden y elementos. |
| `GET lists/v2/getIds` | Poda de listas locales. | Descarga solo IDs para eliminar localmente listas que ya no existen en Cloud. |
| `PUT lists/v2/save` | Crear/actualizar listas. | Sube listas, orden y elementos. |
| `PUT lists/v2/removeItems` | Quitar elementos de listas. | Elimina elementos concretos de una o varias listas. |
| `DELETE lists/v2/remove` | Eliminar lista. | Borra una lista personalizada de Cloud. |

## Firebase/Google Auth usado por Hexagon

| Endpoint/servicio | Proceso frontend | Qué hace |
| --- | --- | --- |
| FirebaseUI Email/Google sign-in | Conectar Cloud. | Muestra login por email o Google y obtiene un usuario Firebase. |
| Firebase ID token | Llamadas a Hexagon. | `FirebaseHttpRequestInitializer` adjunta token para que el backend valide identidad. |

## Procesos completos por pantalla o acción

### Buscar y añadir serie

1. La UI busca con `GET /search/tv` o muestra discovery con `GET /discover/tv`.
2. Al añadir, descarga `GET /tv/{tv_id}?append_to_response=external_ids`.
3. Por cada temporada descarga `GET /tv/{tv_id}/season/{season_number}`.
4. Completa horario/rating Trakt con `GET /search/tmdb/{id}?type=show&extended=full`.
5. Si Hexagon está activo, restaura Cloud con `GET shows/v2/getSgShow` y episodios con `GET episodes/v2/getSgEpisodes`; después sube estado con `PUT shows/v2/saveSgShows`.
6. Si Hexagon no está activo y hay Trakt, aplica watched/collection de Trakt usando `GET /sync/watched/shows` y `GET /sync/collection/shows`.

### Buscar y añadir película

1. La UI busca con `GET /search/movie` o descubre con `GET /discover/movie`.
2. Al añadir una película a colección/watchlist/vistas descarga `GET /movie/{movie_id}?append_to_response=release_dates`.
3. Si falta descripción localizada, usa `GET /movie/{movie_id}` como fallback.
4. Si necesita rating Trakt, resuelve Trakt ID con `GET /search/tmdb/{id}?type=movie` y puede usar `GET /movies/{id}/ratings`.
5. El estado se sube a Trakt (`POST /sync/collection`, `POST /sync/watchlist`, `POST /sync/history`) o Hexagon (`PUT movies/v2/save`) según configuración/acción.

### Sincronización periódica

1. TMDB: `GET /configuration`; después actualiza películas actuales con `GET /movie/{movie_id}?append_to_response=release_dates`.
2. Hexagon: descarga/sube shows, episodes, movies y lists con los endpoints `*/v2/*`.
3. Trakt: descarga `GET /sync/last_activities`; según cambios importa episodios, películas, ratings y notas; durante la primera mezcla puede subir lo local que falte en Trakt.
4. Recalcula FTS, próximos episodios y limpia películas sin estado útil cuando procede.

### Cargar imágenes en pantalla

1. La base local guarda rutas (`poster_path`, `still_path`, etc.), no la imagen binaria.
2. `ImageTools` transforma la ruta en URL TMDB, TheTVDB legacy, demo o URL de caché.
3. Picasso descarga usando red si está permitida; si la preferencia obliga a Wi‑Fi y la red no es válida, usa `NetworkPolicy.OFFLINE` para intentar caché local.

## Notas de implementación

- El frontend no llama a endpoints con strings manuales en muchos casos: las librerías `tmdb2`, `trakt5` y los clientes Discovery construyen las URLs.
- Trakt usa paginación en varios endpoints; `TraktTools4.fetchAllPages(...)` descarga todas las páginas usando cabeceras de conteo.
- Hexagon usa `cursor` para paginación y `updatedSince` para sincronización incremental.
- Las imágenes pueden pasar por un proxy/cache firmado, por lo que la URL final de red puede no ser la URL original de TMDB/TheTVDB.
- Si Hexagon está habilitado, Trakt se limita a ratings para evitar conflictos de estado.
