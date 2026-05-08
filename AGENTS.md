# AGENTS.md

## Memoria del proyecto

- Este repositorio contiene la app Android **SeriesGuide**, con código Kotlin y Java organizado principalmente en `app/src/main/java/com/battlelancer/seriesguide`.
- Se añadieron estas guías Markdown en español para orientar a futuros contribuidores:
  - `backend.md`: inventario de endpoints Hexagon/SeriesGuide Cloud, autenticación, paginación y relación con flujos TMDB/Trakt del cliente.
  - `docs/base-de-datos.md`: resumen de las bases Room/locales, tablas principales, DAOs, FTS y migraciones relevantes.
  - `docs/datos-e-imagenes-series-peliculas.md`: fuentes de metadatos e imágenes, endpoints TMDB/Trakt/Hexagon usados, construcción de URLs de imágenes y caché/proxy.
  - `docs/frontend-endpoints.md`: mapa de clientes HTTP/endpoints que usa la app Android y qué pantallas, acciones o procesos disparan cada llamada.
  - `docs/importacion-desde-trakt.md`: diferencias entre importación inicial y sincronización posterior con Trakt, archivos clave y endpoints de estados, ratings y notas.

## Notas para futuros cambios de documentación

- Mantén la documentación técnica en español si el cambio continúa o amplía `docs/datos-e-imagenes-series-peliculas.md`.
- Para cambios de solo Markdown, al menos ejecuta `git diff --check -- <archivo>` antes de confirmar.
- Si se añaden o cambian endpoints, contrasta el documento con los usos reales en `TmdbTools*`, `MovieTools`, `AddUpdateShowTools`, `GetShowTools`, `TraktTools*`, los jobs de Trakt y los ficheros `backend/src/endpoints/*-v2-rest.discovery`.

## Pistas de código relevantes

- Integración TMDB: `app/src/main/java/com/battlelancer/seriesguide/modules/TmdbModule.kt` y `app/src/main/java/com/battlelancer/seriesguide/tmdbapi/`.
- Integración Trakt: `app/src/main/java/com/battlelancer/seriesguide/modules/TraktModule.kt` y `app/src/main/java/com/battlelancer/seriesguide/traktapi/`.
- Series: `app/src/main/java/com/battlelancer/seriesguide/shows/tools/`.
- Películas: `app/src/main/java/com/battlelancer/seriesguide/movies/`.
- Imágenes: `app/src/main/java/com/battlelancer/seriesguide/util/ImageTools.kt` y `app/src/main/java/com/battlelancer/seriesguide/util/SgPicassoRequestHandler.kt`.
- Hexagon/Cloud: `backend/src/endpoints/` y clases relacionadas bajo `app/src/main/java/com/battlelancer/seriesguide/backend/` y `app/src/main/java/com/battlelancer/seriesguide/sync/`.
