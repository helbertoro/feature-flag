# Spec 06 - Dashboard: listado de flags

PRD de referencia: [docs/pdrs/feature-flags-mvp.md](../pdrs/feature-flags-mvp.md)

## 1. Objetivo

Construir en `apps/web` la vista principal del panel admin: un listado de todas las feature flags con filtro/búsqueda por namespace y feedback visual del estado de cada flag por ambiente (`on` / `off` / `parcial %`). La vista está protegida por el guard de sesión de la spec `05`.

## 2. Contexto y dependencias

- **Specs previas requeridas (bloqueantes):** `01-monorepo-setup` (Next.js + Tailwind), `04-api-feature-flags` (`GET /api/v1/flags`, fuente de datos), `05-basic-login` (guard de sesión, para el criterio de protección).
- **Spec siguiente:** `07-create-edit-flag` añade los formularios de creación/edición/eliminación accesibles desde esta lista; `08-targeting-rules` añade la vista de detalle/targeting.
- **Dependencias no bloqueantes:** el render del listado solo necesita `04`; `05` es imprescindible únicamente para el criterio de redirección sin sesión. No depende de `07`/`08`; puede ejecutarse en paralelo con `07`.
- **Archivos/paquetes que toca:** `apps/web/app/(admin)/page.tsx` (o `app/flags/page.tsx`), componentes de listado y filtro, cliente HTTP a la API.
- **Requerimientos del PRD cubiertos:**
  - **RF-07:** listar todas las flags con su estado resumido por ambiente.
  - **RF-26:** la UI debe mostrar un listado de flags con búsqueda/filtro por namespace.
  - **RF-28:** la UI debe mostrar feedback visual del estado actual de cada flag por ambiente (`on`/`off`/`parcial %`).

## 3. Alcance (In / Out)

### In
- Página de listado que consume `GET /api/v1/flags`.
- Filtro/búsqueda por namespace (selector o input de texto que filtra por prefijo de la `key`).
- Indicador de estado por ambiente para cada flag (`dev`, `staging`, `prod`): `off` (master off o 0%), `on` (master on y 100%), `parcial X%` (master on con 0 < rollout < 100).
- Enlace por flag hacia su vista de detalle (implementada en spec `08`).
- Estados de carga y vacío.

### Out
- Crear/editar/eliminar flags (spec `07`).
- Gestión de targeting (master toggle, rollout, overrides) (spec `08`).
- Lógica de evaluación (spec `09`).

## 4. Tareas en orden

1. Crear un cliente HTTP en `apps/web` (p. ej. `lib/api.ts`) con `getFlags()` que llame a `GET /api/v1/flags` enviando credenciales/cookies.
2. Crear la página de listado protegida por el guard de la spec `05`.
3. Renderizar cada flag mostrando `key`, `name`, `namespace` y, para cada ambiente, su estado visual.
4. Implementar el cálculo de estado por ambiente a partir de `{ masterEnabled, rolloutPercentage }` (ver Notas técnicas).
5. Añadir el filtro por namespace: derivar la lista de namespaces presentes y permitir filtrar; alternativamente, input de búsqueda por texto sobre `key`/`name`.
6. Añadir estados de carga, error y vacío ("no hay flags todavía").
7. Enlazar cada fila a `/flags/:key` (detalle, spec `08`).
8. Estilar con Tailwind para una UI clara (tabla o tarjetas con badges de color por estado).

## 5. Criterios de aceptación verificables

- [ ] **RF-07 / RF-28:** con al menos una flag creada (vía seed o API), la lista la muestra con su estado para `dev`, `staging` y `prod`.
- [ ] **RF-28:** una flag con `master_enabled=false` muestra `off`; con `master_enabled=true` y `rollout=100` muestra `on`; con `master_enabled=true` y `rollout=40` muestra `parcial 40%`.
- [ ] **RF-26:** al filtrar por un namespace (p. ej. `billing`), solo se muestran flags cuya `key` empieza por ese namespace.
- [ ] La vista está detrás del guard: sin sesión válida redirige a `/login` (consistente con RF-03 de la spec `05`).
- [ ] Cada flag enlaza a su vista de detalle `/flags/:key`.
- [ ] Estado vacío visible cuando no hay flags.

## 6. Notas técnicas

### Cálculo del estado por ambiente (RF-28)

A partir del estado por ambiente devuelto por `GET /api/v1/flags` (ver spec `04`, cada flag trae `environments: [{ environment, masterEnabled, rolloutPercentage }]`):

```ts
type EnvState = { environment: string; masterEnabled: boolean; rolloutPercentage: number };

function displayState(e: EnvState): string {
  if (!e.masterEnabled) return "off";
  if (e.rolloutPercentage >= 100) return "on";
  if (e.rolloutPercentage <= 0) return "off"; // master on pero 0% -> off efectivo
  return `parcial ${e.rolloutPercentage}%`;
}
```

Sugerencia de color (Tailwind): `off` -> gris, `on` -> verde, `parcial` -> ámbar.

### Filtro por namespace (RF-26)
El namespace es el prefijo de la `key` antes del primer punto (p. ej. `billing.new-checkout` -> `billing`). Construir el conjunto de namespaces a partir de las flags cargadas.

### Notas
- Ambientes mostrados siempre en orden `dev`, `staging`, `prod`.
- Toda mutación (crear/editar/targeting) vive en specs posteriores; esta vista es de solo lectura más navegación.
- La data se obtiene de la API; no se accede a la base de datos directamente desde `apps/web`.
