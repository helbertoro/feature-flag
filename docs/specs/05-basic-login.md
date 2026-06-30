# Spec 05 - Login demo y sesión

PRD de referencia: [docs/pdrs/feature-flags-mvp.md](../pdrs/feature-flags-mvp.md)

## 1. Objetivo

Implementar autenticación demo de un único usuario: pantalla de login en `/login`, validación de credenciales contra variables de entorno (`DEMO_USER`, `DEMO_PASSWORD`), sesión basada en cookie firmada gestionada por `apps/api`, guard de rutas del panel admin (redirige a `/login` si no hay sesión) y logout que invalida la sesión. Las credenciales no se committean.

## 2. Contexto y dependencias

- **Specs previas requeridas (bloqueantes):** `01-monorepo-setup` (`apps/web` Next.js, `apps/api` Hono, `.env.example`).
- **Spec siguiente:** `06-dashboard-list` (y `07`/`08`) viven detrás del guard definido aquí.
- **Dependencias no bloqueantes:** `02-testing-setup` solo para *ejecutar* tests. **No** depende de `04`: el middleware `requireSession` se define aquí y se aplica a las rutas de gestión cuando existan (`04`/`08`), así que `05` puede implementarse en paralelo con `03` y `04` (solo necesita `01`).
- **Archivos/paquetes que toca:** `apps/api/src/routes/auth.ts`, middleware de sesión en `apps/api`, `apps/web/app/login/page.tsx`, guard en `apps/web` (middleware o layout), `.env.example`.
- **Requerimientos del PRD cubiertos:**
  - **RF-01:** pantalla de login accesible en `/login`.
  - **RF-02:** login acepta credenciales demo configurables vía `DEMO_USER` y `DEMO_PASSWORD` con valores por defecto documentados.
  - **RF-03:** tras login exitoso, acceso al panel admin; sesiones no autenticadas redirigen a `/login`.
  - **RF-04:** endpoint de logout invalida la sesión demo y redirige a `/login`.
  - **RNF-05:** las credenciales demo no se committean; se cargan solo vía env vars o `.env` ignorado por git.

## 3. Alcance (In / Out)

### In
- Endpoint `POST /api/v1/auth/login` que valida `DEMO_USER`/`DEMO_PASSWORD` y emite cookie de sesión firmada.
- Endpoint `POST /api/v1/auth/logout` que limpia la cookie.
- Endpoint `GET /api/v1/auth/session` para verificar sesión.
- Middleware de Hono que protege las rutas de gestión (`/api/v1/flags`, targeting) exigiendo sesión válida.
- Página `/login` en `apps/web` con formulario y manejo de error.
- Guard en `apps/web` que redirige a `/login` cuando no hay sesión.
- Logout desde la UI.

### Out
- OAuth, SSO, RBAC, múltiples usuarios o roles (fuera de alcance del PRD).
- Protección del endpoint público de evaluación `/api/v1/evaluate` (spec `09`): la evaluación es server-to-server y NO requiere sesión demo.

## 4. Tareas en orden

1. Añadir `SESSION_SECRET`, `DEMO_USER`, `DEMO_PASSWORD` a `.env.example` con defaults documentados (`DEMO_USER=admin`, `DEMO_PASSWORD=admin`, `SESSION_SECRET=change-me`). Confirmar que `.env` está en `.gitignore` (RNF-05).
2. En `apps/api`, implementar utilidades de cookie firmada (usar `hono/cookie` `setSignedCookie` / `getSignedCookie` con `SESSION_SECRET`).
3. Crear `POST /api/v1/auth/login`: comparar credenciales del body contra `process.env.DEMO_USER`/`DEMO_PASSWORD`; si coinciden, setear cookie firmada `ff_session` (`httpOnly`, `sameSite=Lax`, `path=/`); responder `200`. Si no, responder `401` con mensaje de error.
4. Crear `POST /api/v1/auth/logout`: borrar la cookie `ff_session`; responder `200` (la redirección a `/login` la realiza la UI).
5. Crear `GET /api/v1/auth/session`: `200 { authenticated: true }` si la cookie firmada es válida; `401` si no.
6. Crear middleware `requireSession` y aplicarlo a las rutas de gestión (`/api/v1/flags` de la spec `04` y las de targeting de la spec `08`). NO aplicarlo a `/api/v1/evaluate`.
7. En `apps/web`, crear `app/login/page.tsx` con formulario (usuario, contraseña), que haga `POST` al login y, en éxito, redirija al dashboard; en error muestre mensaje (RF-02 / CA-02).
8. Implementar guard en `apps/web` (Next.js middleware o verificación en el layout del área admin) que, sin sesión válida, redirija a `/login` (RF-03).
9. Añadir acción de logout en la UI que llame al endpoint y redirija a `/login` (RF-04).
10. Escribir tests Vitest del flujo de login/logout/guard en la API.

## 5. Criterios de aceptación verificables

- [ ] **RF-01:** `/login` responde HTTP 200 y muestra el formulario.
- [ ] **RF-02 / CA-01:** `POST /api/v1/auth/login` con `DEMO_USER`/`DEMO_PASSWORD` correctos devuelve `200` y setea la cookie `ff_session`.
- [ ] **CA-02:** `POST /api/v1/auth/login` con credenciales incorrectas devuelve `401` y la UI muestra mensaje de error, sin acceso al panel.
- [ ] **RF-03:** una request a una ruta de gestión (p. ej. `GET /api/v1/flags`) sin cookie válida devuelve `401`; en la UI, navegar al dashboard sin sesión redirige a `/login`.
- [ ] **RF-04:** `POST /api/v1/auth/logout` limpia la cookie; tras logout, `GET /api/v1/auth/session` devuelve `401` y la UI redirige a `/login`.
- [ ] **RNF-05:** `.env` no está versionado (sí `.env.example`); no hay credenciales en el código fuente.
- [ ] La cookie es `httpOnly` y está firmada (manipularla invalida la sesión -> `401`).

## 6. Notas técnicas

### Variables de entorno (RF-02, RNF-05)

```
DEMO_USER=admin
DEMO_PASSWORD=admin
SESSION_SECRET=change-me
```

Valores por defecto documentados solo para desarrollo local. En cualquier entorno compartido deben proveerse por env var; `.env` está en `.gitignore`.

### Cookie de sesión firmada (Hono)

```ts
import { setSignedCookie, getSignedCookie, deleteCookie } from "hono/cookie";

const SECRET = process.env.SESSION_SECRET ?? "change-me";

// login
await setSignedCookie(c, "ff_session", "demo", SECRET, {
  httpOnly: true,
  sameSite: "Lax",
  path: "/",
  maxAge: 60 * 60 * 8,
});

// verificación (middleware requireSession)
const value = await getSignedCookie(c, SECRET, "ff_session");
if (!value) return c.json({ error: "no autenticado" }, 401);

// logout
deleteCookie(c, "ff_session", { path: "/" });
```

### Endpoints de auth

| Método | Ruta | Descripción | Éxito | Error |
|--------|------|-------------|-------|-------|
| POST | `/api/v1/auth/login` | Validar credenciales y emitir cookie | 200 | 401 |
| POST | `/api/v1/auth/logout` | Invalidar sesión | 200 | - |
| GET  | `/api/v1/auth/session` | Verificar sesión | 200 | 401 |

### Notas
- El guard del lado web puede consultar `GET /api/v1/auth/session` o verificar la cookie en un middleware de Next.js.
- El endpoint de evaluación `/api/v1/evaluate` (spec `09`) NO se protege con este middleware: es de uso server-to-server.
- No exponer la herramienta a internet con credenciales por defecto (riesgo R-02 del PRD).
