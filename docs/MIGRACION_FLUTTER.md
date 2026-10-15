# Plan de migración: Berkelio v2 → Flutter

> Estado: **planeado** · Origen: rama `feature/nuevo-diseno` (= `main` = `develop`, el rediseño con API v2) · No se migra `produccion` (la de tiendas, API legacy).

## 1. Objetivo y alcance

- **Migrar la v2** (el rediseño que ya habla con `/api/v2/`), no la app publicada.
- **Backend intacto**: mismo API v2. El contrato está congelado en `escandio/movil/contrato_v2.py` + `openapi_v2.yaml`. Flutter es un cliente nuevo del mismo servidor.
- **Meta de la fase 1: paridad funcional 1:1** con la v2 de RN. Las mejoras vienen después, no durante (evita mezclar "reescribir" con "rediseñar").
- **Superficie**: 14 pantallas en 6 features.

| Feature | Pantallas | Rol |
|---|---|---|
| `auth` | 5 (login, register, forgot-password, welcome, pending-approval) | Login + refresh, cuenta sin acceso |
| `home` | 1 (dashboard) | Inicio: agrega stats de órdenes activas |
| `visita` | 3 (visitas, entregas-detalle, entrega-form) + ruta-mapa | **El corazón.** Órdenes, entregas, sync, mapa |
| `novedad` | 2 (novedad-form, solucion-form) + lista | Novedades por orden |
| `settings` | 3 (settings, profile, about) | Ajustes, perfil, dev-mode |
| `vertical` | 0 | Config/lookup (solo repos) |

## 2. Arquitectura destino: Clean Architecture (Reso Coder) + Riverpod

Se toma la **estructura** de capas del curso Reso Coder (Flutter Clean Architecture), con **Riverpod** en la capa de presentación en lugar del BLoC original. **No habrá pruebas unitarias.** La justificación de Clean aquí es separación de responsabilidades y mantenibilidad (del curso se toma solo la estructura, no la disciplina de tests).

```
lib/
├── core/
│   ├── error/          # Failure (base), *Failure, Exception
│   ├── usecases/       # UseCase<Type, Params> base (call() → Future<Either<Failure, Type>>)
│   ├── network/        # NetworkInfo (wrap connectivity_plus)
│   ├── api/            # cliente Dio + interceptor de token (single-flight)
│   └── util/
└── features/
    └── <feature>/
        ├── data/
        │   ├── datasources/   # remote (Dio/OpenAPI) + local (Isar/secure storage)
        │   ├── models/        # DTOs: extienden entity, fromJson/toJson
        │   └── repositories/  # <Feature>RepositoryImpl
        ├── domain/
        │   ├── entities/      # objetos puros (Equatable/Freezed)
        │   ├── repositories/  # contrato abstracto
        │   └── usecases/      # uno por caso de uso
        └── presentation/
            ├── providers/     # Riverpod Notifier/AsyncNotifier (reemplaza BLoC)
            ├── pages/
            └── widgets/
```

**Reglas Reso Coder que se conservan:**
- **Regla de dependencia**: presentation → domain ← data. El domain no conoce a nadie.
- **Entities vs Models**: entity puro en domain; model (DTO) en data extiende la entity y hace JSON. Los modelos generados desde OpenAPI **no cruzan** al domain: el datasource los mapea a entities.
- **Either<Failure, T>** con **`fpdart`** (sucesor mantenido de `dartz`; misma semántica de `Either`).
- **Exceptions → Failures**: los datasources lanzan Exceptions; el repository las atrapa y devuelve `Left(Failure)`.
- **Riverpod como DI + estado**: los providers cablean datasource → repository → usecase → notifier. Los `override` sirven para intercambiar implementaciones (fakes, config dev/prod). (Alternativa: `get_it` + `injectable` como en el curso, si prefieres DI explícita fuera de presentation.)

### Reglas de arquitectura (dónde va la lógica)

Dos principios que juntos dicen **"la lógica de negocio NO va en presentation ni en los widgets"**:

1. **Regla de Dependencia** (Uncle Bob): las dependencias apuntan hacia adentro; el `domain` no conoce Flutter/Riverpod/widgets → las reglas no pueden vivir afuera.
2. **Humble Object / vista humilde** (*dumb widgets*): el widget se deja lo más tonto posible; solo pinta estado y reenvía eventos.

Hay **dos tipos de "lógica"** y se confunden:

| Tipo | Ejemplo en berkelio | Dónde VA | Dónde NO |
|---|---|---|---|
| **Reglas de negocio** | no re-entregar una visita ya entregada; **sellar la hora al registrar, no al sincronizar**; calcular estado de orden; drenar cola offline | `domain` → **use case** / entity | ❌ widget · ❌ Notifier |
| **Lógica de presentación** | formatear fecha, mapear `Failure`→mensaje, habilitar botón, qué chip mostrar | `presentation` → **Notifier/Provider** | ❌ widget (idealmente) |
| **UI puro** | layout, colores, render de la lista | **widget** | — |

El Notifier **sí** puede tener lógica, pero solo **de presentación**; las **reglas de negocio** van en el use case.

**Olores (violaciones):**
- `onPressed:` que decide una regla (valida entrega, arma payload, decide si sincroniza).
- Widget que llama directo a un repository o a `Dio`.
- Fechas/estados calculados dentro del `build()`.
- Notifier con `if` de negocio en vez de llamar un use case.

**Flujo correcto (siempre el mismo):**
```
widget   → notifier.registrarEntrega(...)                    // solo dispara
notifier → await ref.read(registrarEntregaProvider)(params)  // orquesta
useCase  → aplica la regla (sella la hora) → Either<Failure, Entrega>
notifier → result.fold(err → AsyncError, ok → AsyncData)     // mapea a estado
widget   → pinta el estado                                   // humilde
```

**Mnemotecnia:** el widget es tonto, el Notifier orquesta, y las reglas viven en el use case (domain).

## 3. Mapeo RN → Clean

| Concepto en RN (v2 actual) | Destino en Flutter Clean |
|---|---|
| `repositories/*.repository.ts` (HTTP, extiende HttpBase) | `data/datasources/*_remote_data_source.dart` + `data/repositories/*_repository_impl.dart` + contrato en `domain/repositories/` |
| `interfaces/` + `models/` | `domain/entities/` (puro) + `data/models/` (DTO con JSON) |
| `view-models/*.view-model.ts` (React Query) | `presentation/providers/*` (AsyncNotifier) — el rol de cache/estado servidor |
| `services/*.service.ts` (lógica de negocio) | `domain/usecases/*` (reglas) — p.ej. `VisitaProcessingService` → `ProcesarEntrega`, `SincronizarLote` |
| Redux `slice/selector/thunk` (persistido) | `presentation/providers` (Notifier) + `data/datasources/*_local_data_source.dart` (Isar) para lo persistido |
| `core/repositories/http-base.repository.ts` | `core/api/` (Dio + interceptores) |
| `core/interceptors/error.interceptor.ts` | `core/error/` (mapeo Exception→Failure) + interceptor Dio |
| `config/environment.ts` (dev/prod runtime) | provider de entorno + `--dart-define`/flavors |

## 4. Stack (pubspec)

| Necesidad | Paquete |
|---|---|
| HTTP | `dio` (+ cliente generado desde openapi) |
| Estado + DI | `flutter_riverpod` + `riverpod_annotation` + `riverpod_generator` |
| Modelos inmutables + JSON | `freezed` + `json_serializable` |
| Programación funcional | `fpdart` (Either/Option) |
| Persistencia / offline | `drift` + `drift_flutter` (SQL, mantenido, compatible Dart 3.12; Isar quedó descartado — su generador está clavado en source_gen 1.x) |
| Tokens | `flutter_secure_storage`; flags simples `shared_preferences` |
| **Geolocalización** | `flutter_background_geolocation` + `background_fetch` (**Transistor, mismo SDK que RN**) |
| Cámara / foto | `camera` + `flutter_image_compress` + `image` (rotación) |
| Escáner código | `mobile_scanner` (reemplaza el scanner de vision-camera) |
| Firma | `signature` |
| Mapa | `google_maps_flutter` (nativo; mejor que el WebView actual) |
| Navegación | `go_router` |
| Conectividad | `connectivity_plus` (envuelto en `NetworkInfo`) |
| Permisos | `permission_handler` |
| Monitoreo | `sentry_flutter` |
| Iconos | `ionicons` |
| Tests | `mocktail`, overrides de Riverpod, `integration_test`/`patrol` |
| Codegen | `build_runner` |

## 5. Los 4 flujos de riesgo (spikes ANTES que las pantallas)

### 5.1 Background geolocation — `background-geolocation.service.ts` (1100 líneas)
El flujo actual mapea 1:1 al plugin de Flutter (mismo vendor):
- `ready()` una sola vez → `BackgroundGeolocation.ready(Config(...))` + registrar `onLocation/onProviderChange/onHeartbeat/onGeofence/onEnabledChange/onAuthorization`.
- `startTracking(config)` / `stopTracking()` por orden → `.start()` / `.stop()`.
- `restoreTrackingIfNeeded()` (reanudar tras app terminada) → leer estado + `.start()` en bootstrap. **Crítico**: es el valor central de la app.
- `cleanup()` (preserva listeners) vs `fullCleanup()` (logout) → mismo patrón; con `flutter_background_geolocation` los listeners se sueltan con las suscripciones.
- `sendLocationToServer()` → puede seguir manual, o delegarse al `url`/`autoSync` nativo del plugin (evalúa en el spike).
- Permisos Android "Always" + notificación (Android 13+) → `permission_handler` + config del plugin.
> **Decisión de spike**: manual vs `autoSync` nativo para enviar ubicaciones. Además envuelve el SDK en un `LocationTrackingDataSource` (data layer) para no acoplar el domain al plugin.

### 5.2 Sync offline-first — `VisitaProcessingService` + `visita.slice` + `visita.thunk`
- Regla de dominio a preservar: **la hora de entrega se sella al registrar, no al sincronizar** (commit `1be4b2f`).
- `procesarVisitaIndividual` / `procesarVisitasEnLote` → usecases `RegistrarEntrega`, `SincronizarEntregasPendientes`.
- Multi-orden con memoria por despacho (`snapshotsByDespacho`) → colección Isar + provider.
- Cola offline: entregas se guardan local (Isar) y se drenan cuando `NetworkInfo.isConnected`.

### 5.3 Token refresh single-flight — `http-base.repository.ts` (318 líneas)
- `isRefreshingToken` + cola (`processQueue`) en 401 → en Dio: `QueuedInterceptorsWrapper` (serializa refresh, reintenta cola con token nuevo, o dispara logout).
- Base URL dinámica (`configBaseUrl || environment.apiBase`) → provider de entorno.
- `postMultipart` (subida de fotos) → `dio` con `FormData`.

### 5.4 Captura (foto, firma, escáner)
- Foto: `photo-capture` + rotación (`image-rotation.util`) + storage (`photo-storage.util`) → `camera` + `image` + `flutter_image_compress`, guardado en `path_provider`.
- Firma (prueba de entrega): `react-native-signature-canvas` → `signature`.
- Escáner "Cargar por código": scanner vision-camera → `mobile_scanner`.

## 6. Roadmap por fases

- **Fase 0 — Spikes de riesgo (1º)**: proyecto Flutter + los 4 flujos de §5 probados en un cascarón. Aquí se resuelve la **licencia Transistor** y la subida de ubicación. *Nada de UI final.*
- **Fase 1 — Core**: `core/` (error, network, api/Dio+interceptor, usecase base), cliente OpenAPI generado, DI con Riverpod, entorno dev/prod, Sentry.
- **Fase 2 — Auth**: login + refresh + cuenta sin acceso (welcome/pending-approval). Cierra el ciclo de sesión completo.
- **Fase 3 — Visita (el corazón)**: Mis Órdenes, detalle/entregas, entrega-form (foto+firma), ruta-mapa, sync offline, tracking por orden. Es ~40-50% del esfuerzo.
- **Fase 4 — Home + Novedad + Settings + Vertical**: dashboard con stats, novedades por orden, ajustes/perfil/dev-mode.
- **Fase 5 — Endurecer + publicar**: QA en dispositivo, bundle id/versión, tiendas.

## 7. Cliente API v2 desde OpenAPI

- Generar DTOs + cliente desde `escandio/movil/openapi_v2.yaml` (`openapi-generator` dio, o `swagger_dart_code_generator`).
- Vive en **data layer**; el `remote_data_source` mapea DTO generado → entity de domain (no filtrar tipos generados al domain).
- Regenerable cuando el contrato v2 evolucione (es aditivo sobre legacy).

## 8. Publicación

- **Bundle id**: decidir el definitivo (iOS RN quedó en `lutencio`). La licencia de Transistor va atada al bundle id → **fijarlo antes del spike de geo**.
- **versionCode**: la app de tiendas está en `58`. La nueva debe subir **por encima de 58** (Play rechaza duplicados). Considerar reiniciar `versionName` a `2.0.0` por ser reescritura.
- **Licencia Transistor**: confirmar que cubre el plugin de Flutter + el bundle id elegido (único ítem con costo/bloqueo real).
- Reusar listings/store presence existentes (misma app para el usuario final).

## 9. Estimación (rough)

- 14 pantallas, geo de 1100 líneas, sync offline, ~26.5k líneas TS (mucho es estilo que se reduce en Flutter).
- Un dev enfocado: **~8–12 semanas a paridad**. Fase 0+3 (geo+sync+visita) son ~la mitad y van primero por riesgo.
- Supuestos: contrato v2 estable (lo es), licencia Transistor resuelta, sin cambios de alcance durante la migración.

## 10. Decisiones abiertas

1. **Bundle id** definitivo (bloquea licencia Transistor). 
2. **Ubicación al server**: manual vs `autoSync` nativo del plugin (se decide en Fase 0).
3. ~~**Persistencia**: `isar` vs `drift`.~~ **RESUELTO en Fase 0 = `drift`** (Isar incompatible con el codegen moderno).
4. **DI**: Riverpod-como-DI (recomendado) vs `get_it`+`injectable` (como el curso).
5. **Repo**: ¿Flutter en repo nuevo o carpeta dentro del actual? (recomendado: repo nuevo `berkelio-flutter`).
