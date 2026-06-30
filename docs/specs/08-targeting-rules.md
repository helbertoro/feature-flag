# Spec 08 - Reglas de targeting (master toggle, rollout %, overrides)

PRD de referencia: [docs/pdrs/feature-flags-mvp.md](../pdrs/feature-flags-mvp.md)

## 1. Objetivo

Implementar la API y la UI para gestionar el targeting de una flag por ambiente: el toggle master on/off por ambiente, el rollout porcentual (0–100, entero) por ambiente, y los overrides por empresa (`company_id` + `on/off`) por ambiente, incluyendo crear y eliminar overrides. La UI organiza todo en la vista de detalle de la flag, lado a lado por ambiente.

## 2. Contexto y dependencias

- **Specs previas requeridas (bloqueantes):** `01-monorepo-setup` (`apps/api`, `apps/web`, `packages/domain`), `03-database-schema-and-seed` (tablas `environment_config`, `company_overrides`), `05-basic-login` (guard de sesión para proteger los endpoints).
- **Spec siguiente:** `09-flag-evaluator` consume estos datos (`master_enabled`, `rollout_percentage`, overrides) para evaluar.
- **Dependencias no bloqueantes:** `04-api-feature-flags` (la vista de detalle puede reusar `GET /api/v1/flags/:key`, pero los endpoints de targeting montan su propio router y leen las tablas de `03` directamente) y `06-dashboard-list` (solo enlace desde la lista). Puede ejecutarse en paralelo con `04`, `06`, `07` y `09`.
- **Archivos/paquetes que toca:** `apps/api/src/routes/targeting.ts`, validación Zod en `packages/domain`, `apps/web/app/flags/[key]/page.tsx` (vista de detalle).
- **Requerimientos del PRD cubiertos:**
  - **RF-11:** activar/desactivar globalmente una flag en un ambiente (toggle master on/off por ambiente).
  - **RF-12:** cuando el master de un ambiente está off, la evaluación devuelve `false` para ese ambiente independientemente de overrides o rollout % (la API de targeting persiste el estado; la regla la aplica la spec `09`).
  - **RF-13:** configurar el rollout porcentual (0–100, entero) por ambiente.
  - **RF-14:** añadir un override por empresa en un ambiente: `company_id` + valor `on` u `off`.
  - **RF-15:** eliminar un override de empresa existente.
  - **RF-16:** un override por empresa en un ambiente tiene precedencia sobre el rollout % de ese ambiente (pero no sobre el master off — ver RF-12).
  - **RF-27:** la UI permite gestionar targeting (master toggle, rollout %, overrides) desde la vista de detalle, organizado por ambiente.

## 3. Alcance (In / Out)

### In
- Endpoints para actualizar `environment_config` (master toggle y rollout %) por `(flag, environment)`.
- Endpoints para crear/eliminar overrides en `company_overrides`.
- Validación: `environment` ∈ {`dev`, `staging`, `prod`}; `rollout_percentage` entero 0–100; `enabled` booleano; `company_id` no vacío.
- Actualización de `updated_at` en cada mutación (auditoría mínima del PRD).
- Vista de detalle en `apps/web` con secciones por ambiente: switch master, input/slider de rollout, tabla de overrides con alta y baja.
- Todos los endpoints protegidos por el guard de sesión (spec `05`).

### Out
- La lógica de precedencia/evaluación en runtime (spec `09`); aquí solo se persiste configuración. La nota de precedencia se incluye para contexto.
- CRUD de la flag en sí (specs `04`/`07`).

## 4. Tareas en orden

1. En `packages/domain`, añadir schemas Zod: `updateEnvConfigSchema` (`masterEnabled?`, `rolloutPercentage?` entero 0–100), `createOverrideSchema` (`companyId`, `enabled`), y un validador `isValidEnvironment(env)`.
2. Crear `apps/api/src/routes/targeting.ts` con los endpoints (ver tabla), montado bajo `/api/v1/flags/:key` y protegido por `requireSession`.
3. `PATCH /api/v1/flags/:key/environments/:env`: validar `env` (si inválido `400`); actualizar `masterEnabled` y/o `rolloutPercentage` en `environment_config` para esa flag+ambiente; actualizar `updated_at`; `404` si la flag/ambiente no existe.
4. `GET /api/v1/flags/:key/environments/:env/overrides`: listar overrides de ese ambiente.
5. `POST /api/v1/flags/:key/environments/:env/overrides`: crear override (`company_id`, `enabled`); si ya existe override para ese `company_id` en ese ambiente, actualizarlo (upsert) o devolver `409` (decidir; recomendado upsert). Devolver `201`/`200`.
6. `DELETE /api/v1/flags/:key/environments/:env/overrides/:companyId`: eliminar el override; `204`; `404` si no existe.
7. En `apps/web`, construir la vista de detalle `/flags/:key` que muestra las tres secciones por ambiente (`dev`, `staging`, `prod`) lado a lado (RF-27): switch master (RF-11), control de rollout 0–100 (RF-13), tabla de overrides con formulario de alta (RF-14) y botón de baja (RF-15).
8. Conectar la UI a los endpoints; reflejar cambios inmediatamente (consistente con RNF-04: visibles en evaluación en < 1 s sin reinicio).
9. Escribir tests Vitest de la API de targeting.

## 5. Criterios de aceptación verificables

- [ ] **RF-11 / CA-04:** `PATCH /api/v1/flags/:key/environments/staging` con `{ "masterEnabled": true }` persiste master on en `staging`.
- [ ] **RF-13:** `PATCH .../environments/dev` con `{ "rolloutPercentage": 50 }` persiste 50; valores fuera de 0–100 o no enteros devuelven `400`.
- [ ] **RF-20 / validación de ambiente:** `PATCH .../environments/qa` (ambiente inválido) devuelve `400`.
- [ ] **RF-14 / CA-06:** `POST .../environments/prod/overrides` con `{ "companyId": "acme", "enabled": true }` crea el override; aparece al listar.
- [ ] **RF-15:** `DELETE .../environments/prod/overrides/acme` devuelve `204` y el override desaparece del listado.
- [ ] **RF-27:** la vista de detalle muestra master, rollout y overrides para `dev`, `staging` y `prod`.
- [ ] **RNF-04 / CA-13:** un cambio de rollout desde la UI se refleja en `GET /api/v1/flags/:key` (y en la evaluación de la spec `09`) sin reinicio.
- [ ] Todos los endpoints de targeting requieren sesión válida (sin cookie -> `401`).

## 6. Notas técnicas

### Endpoints

| Método | Ruta | Descripción | Éxito | Errores |
|--------|------|-------------|-------|---------|
| PATCH  | `/api/v1/flags/:key/environments/:env` | Actualizar master y/o rollout | 200 | 400 (env/rollout), 404 |
| GET    | `/api/v1/flags/:key/environments/:env/overrides` | Listar overrides | 200 | 404 |
| POST   | `/api/v1/flags/:key/environments/:env/overrides` | Crear/actualizar override | 201/200 | 400, 404 |
| DELETE | `/api/v1/flags/:key/environments/:env/overrides/:companyId` | Eliminar override | 204 | 404 |

### Validación

```ts
// packages/domain/src/targeting.ts
import { z } from "zod";

export const ENVIRONMENTS = ["dev", "staging", "prod"] as const;
export type Environment = (typeof ENVIRONMENTS)[number];
export const isValidEnvironment = (e: string): e is Environment =>
  (ENVIRONMENTS as readonly string[]).includes(e);

export const updateEnvConfigSchema = z.object({
  masterEnabled: z.boolean().optional(),
  rolloutPercentage: z.number().int().min(0).max(100).optional(),
});

export const createOverrideSchema = z.object({
  companyId: z.string().min(1),
  enabled: z.boolean(),
});
```

### Contexto de precedencia (la aplica la spec 09)
Orden de evaluación: `flag_not_found -> master_off -> company_override -> default_off -> rollout`. Es decir, un override de empresa tiene precedencia sobre el rollout % (RF-16), pero el master off del ambiente gana sobre todo (RF-12). Esta spec solo persiste la configuración; la evaluación vive en la spec `09`.

### Notas
- Ambientes válidos exactamente: `dev`, `staging`, `prod`.
- Cada mutación actualiza `updated_at` (auditoría mínima requerida por el PRD).
- La UI organiza el targeting por ambiente lado a lado para evitar drift entre ambientes (mitiga el riesgo R-04 del PRD).
