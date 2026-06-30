# Spec 07 - Crear, editar y eliminar flags (UI)

PRD de referencia: [docs/pdrs/feature-flags-mvp.md](../pdrs/feature-flags-mvp.md)

## 1. Objetivo

Implementar en `apps/web` los formularios y acciones del panel admin para crear una flag (`key`, `name`, `description`), editar `name`/`description` de una flag existente y eliminar una flag (con sus reglas asociadas), consumiendo la API de la spec `04` y mostrando validación de errores (formato de `key` y duplicados).

## 2. Contexto y dependencias

- **Specs previas requeridas (bloqueantes):** `01-monorepo-setup` (Next.js + Tailwind), `04-api-feature-flags` (endpoints CRUD), `05-basic-login` (guard de sesión).
- **Spec siguiente:** `08-targeting-rules` añade la gestión de targeting dentro del detalle de la flag.
- **Dependencias no bloqueantes:** `06-dashboard-list` (solo para navegación y refresco de la lista; los formularios crear/editar/eliminar viven en sus propias rutas y no requieren la lista). Puede ejecutarse en paralelo con `06` y `08`.
- **Archivos/paquetes que toca:** `apps/web/app/flags/new/page.tsx`, `apps/web/app/flags/[key]/edit/page.tsx` (o modales), acciones de borrado, cliente HTTP `lib/api.ts`.
- **Requerimientos del PRD cubiertos:**
  - **RF-05:** crear una flag con `key` (única, formato `namespace.name`), `name`, `description`.
  - **RF-06:** rechazar creación si `key` ya existe o no cumple el formato `namespace.name` (solo minúsculas, números, guiones y puntos) — mostrar el error en la UI.
  - **RF-08:** editar `name` y `description` de una flag existente.
  - **RF-09:** eliminar una flag; la eliminación borra también sus reglas de targeting asociadas.

## 3. Alcance (In / Out)

### In
- Formulario de creación (`key`, `name`, `description`) que hace `POST /api/v1/flags`.
- Mostrar errores de validación devueltos por la API: formato de `key` inválido (`400`) y duplicado (`409`).
- Formulario de edición de `name`/`description` que hace `PATCH /api/v1/flags/:key` (la `key` es inmutable, se muestra de solo lectura).
- Acción de eliminación con confirmación que hace `DELETE /api/v1/flags/:key`.
- Refresco de la lista (spec `06`) tras cada operación.

### Out
- Gestión de targeting: master toggle, rollout %, overrides (spec `08`).
- Validación de formato de `key` server-side (ya definida en spec `04`; la UI muestra el error, no lo reimplementa como fuente de verdad).
- Evaluación (spec `09`).

## 4. Tareas en orden

1. Extender `lib/api.ts` con `createFlag(input)`, `updateFlag(key, input)`, `deleteFlag(key)` apuntando a los endpoints de la spec `04`, enviando cookies de sesión.
2. Crear la página/modal de creación con campos `key`, `name`, `description` y un botón de guardar.
3. Validar mínimamente en cliente (campos requeridos) pero confiar en la API para formato/unicidad; mapear respuestas `400`/`409` a mensajes claros junto al campo `key`.
4. En éxito de creación (`201`), redirigir a la lista o al detalle de la flag y mostrar confirmación.
5. Crear la página/modal de edición que precarga `name`/`description` desde `GET /api/v1/flags/:key`, muestra la `key` de solo lectura, y hace `PATCH` al guardar.
6. Añadir acción de eliminación con diálogo de confirmación; al confirmar, `DELETE`; tras `204`, actualizar la lista.
7. Manejar estados de carga, error y deshabilitar botones durante el envío.
8. Asegurar que todas las vistas están detrás del guard de sesión (spec `05`).

## 5. Criterios de aceptación verificables

- [ ] **RF-05 / CA-03:** crear `billing.new-checkout` desde la UI muestra la flag en el listado; el backend la crea con sus 3 ambientes a 0% (verificable vía `GET /api/v1/flags`).
- [ ] **RF-06:** introducir una `key` inválida (p. ej. `Billing.New`, `billing`, `billing new`) muestra un mensaje de error en la UI y NO crea la flag (la API responde `400`).
- [ ] **RF-06:** introducir una `key` que ya existe muestra un mensaje de "ya existe" en la UI (la API responde `409`).
- [ ] **RF-08:** editar `name`/`description` de una flag existente persiste el cambio (verificable recargando o vía `GET /api/v1/flags/:key`); la `key` no es editable.
- [ ] **RF-09:** eliminar una flag la quita del listado; `GET /api/v1/flags/:key` devuelve `404` y no quedan filas de targeting para esa flag.
- [ ] Las vistas requieren sesión válida (sin sesión, redirigen a `/login`).

## 6. Notas técnicas

### Mapa de operaciones UI -> API (spec 04)

| Acción UI | Endpoint | Éxito | Errores a mostrar |
|-----------|----------|-------|-------------------|
| Crear | `POST /api/v1/flags` | 201 | 400 (formato `key`), 409 (duplicada) |
| Editar | `PATCH /api/v1/flags/:key` | 200 | 404 |
| Eliminar | `DELETE /api/v1/flags/:key` | 204 | 404 |

### Recordatorio de formato de `key` (RF-06, para mensajes de ayuda)
Formato `namespace.name`: solo minúsculas, números, guiones y puntos, con al menos un punto. Ejemplos válidos: `billing.new-checkout`, `team-a.feature.x`. Ejemplos inválidos: `Billing.New` (mayúsculas), `billing` (sin punto), `billing new` (espacio), `billing_.x` (guion bajo).

### Payloads

```jsonc
// POST /api/v1/flags
{ "key": "billing.new-checkout", "name": "New checkout", "description": "Flujo nuevo" }

// PATCH /api/v1/flags/billing.new-checkout
{ "name": "New checkout v2", "description": "Texto actualizado" }
```

### Notas
- La `key` es inmutable tras la creación (la API ignora cambios de `key` en `PATCH`); mostrarla deshabilitada en edición.
- La eliminación en cascada de reglas la garantiza la base de datos (spec `03`) + el endpoint `DELETE` (spec `04`); la UI solo confirma y refresca.
- El `namespace` se deriva automáticamente de la `key` en el backend; no se pide por separado en el formulario.
