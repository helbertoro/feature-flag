# Spec 04 - API de feature flags (CRUD)

PRD de referencia: [docs/pdrs/feature-flags-mvp.md](../pdrs/feature-flags-mvp.md)

## 1. Objetivo

Implementar en `apps/api` (Hono) los endpoints REST CRUD de feature flags: crear, listar, obtener, editar y eliminar, con validación estricta del formato de `key` (`namespace.name`), rechazo de duplicados, y autocreación de la configuración por ambiente al crear (delegando en el helper de la spec `03`).

## 2. Contexto y dependencias

- **Specs previas requeridas (bloqueantes):** `01-monorepo-setup` (Hono en `apps/api`, `packages/domain`), `03-database-schema-and-seed` (tablas y helper `createFlagWithDefaults`).
- **Spec siguiente:** `05-basic-login` protegerá estos endpoints; `06`/`07` los consumen desde la UI; `08` añade endpoints de targeting.
- **Dependencias no bloqueantes:** `02-testing-setup` solo para *ejecutar* tests. No depende de `05`/`06`/`07`/`08`/`09`; tras `03` puede ejecutarse en paralelo con `05`, `08` y `09`.
- **Archivos/paquetes que toca:** `apps/api/src/routes/flags.ts`, `apps/api/src/index.ts`, y validación en `packages/domain` (Zod, schema de `key`).
- **Requerimientos del PRD cubiertos:**
  - **RF-05:** crear flag con `key` (única, formato `namespace.name`), `name`, `description`.
  - **RF-06:** rechazar creación si `key` ya existe o no cumple el formato `namespace.name` (solo minúsculas, números, guiones y puntos).
  - **RF-07:** listar todas las flags con su estado resumido por ambiente.
  - **RF-08:** editar `name` y `description` de una flag existente.
  - **RF-09:** eliminar una flag; la eliminación debe borrar también sus reglas de targeting asociadas.
  - **RF-10:** al crear, registrar configuración inicial en los 3 ambientes con rollout 0% (vía helper de la spec `03`).

## 3. Alcance (In / Out)

### In
- Endpoints HTTP CRUD bajo `/api/v1/flags`.
- Validación de payloads con Zod (compartida en `packages/domain`).
- Validación de formato de `key` y unicidad (RF-06).
- Listado con estado resumido por ambiente (RF-07) leyendo `environment_config`.
- Borrado en cascada de reglas asociadas (RF-09) apoyado en el `onDelete: cascade` de la spec `03`.

### Out
- Autenticación / guard de sesión (lo añade la spec `05`).
- Endpoints de targeting: master toggle, rollout %, overrides (spec `08`).
- Endpoint de evaluación `/api/v1/evaluate` (spec `09`).
- UI (specs `06`/`07`).

## 4. Tareas en orden

1. En `packages/domain`, definir el regex y el schema Zod de `key` y de los payloads (ver Notas técnicas). Exportar `parseKey`, `createFlagSchema`, `updateFlagSchema`.
2. En `apps/api/src/routes/flags.ts`, crear un router Hono con los endpoints (ver tabla de endpoints).
3. `POST /api/v1/flags`: validar body con `createFlagSchema`; validar formato de `key`; derivar `namespace` (texto antes del primer punto); si la `key` ya existe devolver `409`; si el formato es inválido devolver `400`; en éxito llamar `createFlagWithDefaults` (spec `03`) y devolver `201` con la flag creada.
4. `GET /api/v1/flags`: devolver todas las flags, cada una con su estado resumido por ambiente (array con `{ environment, masterEnabled, rolloutPercentage }`).
5. `GET /api/v1/flags/:key`: devolver la flag con su config por ambiente y overrides; `404` si no existe.
6. `PATCH /api/v1/flags/:key`: validar con `updateFlagSchema` (solo `name`, `description`); actualizar `updated_at`; `404` si no existe. La `key` es inmutable.
7. `DELETE /api/v1/flags/:key`: eliminar la flag; el cascade borra `environment_config` y `company_overrides`; `404` si no existe, `204` en éxito.
8. Montar el router en `apps/api/src/index.ts` bajo `/api/v1/flags`.
9. Escribir tests Vitest con `app.request(...)` cubriendo los criterios de aceptación.

## 5. Criterios de aceptación verificables

- [ ] **RF-05 / CA-03:** `POST /api/v1/flags` con `{ "key": "billing.new-checkout", "name": "New checkout", "description": "..." }` devuelve `201`; al listar, la flag aparece con 3 ambientes (`dev`, `staging`, `prod`) a `0%`.
- [ ] **RF-06:** `POST` con `key` ya existente devuelve `409`.
- [ ] **RF-06:** `POST` con `key` inválida (mayúsculas, espacios, sin punto, o caracteres fuera de `[a-z0-9.-]`) devuelve `400`. Ejemplos que deben fallar: `Billing.New`, `billing`, `billing new`, `billing_.x`.
- [ ] **RF-06:** `key` válida de ejemplo que debe pasar: `billing.new-checkout`, `team-a.feature.x`.
- [ ] **RF-07:** `GET /api/v1/flags` devuelve un array donde cada flag incluye su estado por ambiente.
- [ ] **RF-08:** `PATCH /api/v1/flags/billing.new-checkout` con `{ "name": "X" }` actualiza el nombre y devuelve `200`; intentar cambiar `key` no tiene efecto.
- [ ] **RF-09:** `DELETE /api/v1/flags/billing.new-checkout` devuelve `204`; luego `GET` de esa flag devuelve `404` y no quedan filas en `environment_config`/`company_overrides` para esa flag.
- [ ] `PATCH`/`DELETE`/`GET` de una `key` inexistente devuelven `404`.

## 6. Notas técnicas

### Endpoints

| Método | Ruta | Descripción | Éxito | Errores |
|--------|------|-------------|-------|---------|
| POST   | `/api/v1/flags` | Crear flag + config por ambiente | 201 | 400 (formato), 409 (duplicada) |
| GET    | `/api/v1/flags` | Listar flags con estado por ambiente | 200 | - |
| GET    | `/api/v1/flags/:key` | Detalle de flag + targeting | 200 | 404 |
| PATCH  | `/api/v1/flags/:key` | Editar `name`/`description` | 200 | 404 |
| DELETE | `/api/v1/flags/:key` | Eliminar flag (cascade) | 204 | 404 |

### Validación de `key` (RF-06, RNF-08)

Formato `namespace.name`: solo minúsculas, números, guiones y puntos. Debe contener al menos un punto separando namespace y nombre.

```ts
// packages/domain/src/key.ts
import { z } from "zod";

// solo [a-z0-9.-], al menos un punto, sin punto inicial/final ni puntos dobles
export const KEY_REGEX = /^[a-z0-9-]+(\.[a-z0-9-]+)+$/;

export const keySchema = z.string().regex(KEY_REGEX, "key inválida: use formato namespace.name con minúsculas, números, guiones y puntos");

export const createFlagSchema = z.object({
  key: keySchema,
  name: z.string().min(1),
  description: z.string().default(""),
});

export const updateFlagSchema = z.object({
  name: z.string().min(1).optional(),
  description: z.string().optional(),
});

export const namespaceOf = (key: string) => key.split(".")[0];
```

### Forma de respuesta del listado (RF-07)

```json
[
  {
    "key": "billing.new-checkout",
    "name": "New checkout",
    "namespace": "billing",
    "environments": [
      { "environment": "dev", "masterEnabled": false, "rolloutPercentage": 0 },
      { "environment": "staging", "masterEnabled": false, "rolloutPercentage": 0 },
      { "environment": "prod", "masterEnabled": false, "rolloutPercentage": 0 }
    ]
  }
]
```

### Notas
- El `namespace` se deriva del prefijo de la `key` (antes del primer punto) y se persiste en la columna `namespace`.
- La autocreación de config por ambiente (RF-10) NO se reimplementa aquí: se usa `createFlagWithDefaults` de `packages/db` (spec `03`).
- Ambientes válidos: `dev`, `staging`, `prod`.
