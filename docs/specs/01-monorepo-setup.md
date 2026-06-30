# Spec 01 - Configuración del monorepo

PRD de referencia: [docs/pdrs/feature-flags-mvp.md](../pdrs/feature-flags-mvp.md)

## 1. Objetivo

Crear la estructura base del monorepo con pnpm workspaces y los cuatro paquetes (`apps/web`, `apps/api`, `packages/db`, `packages/domain`), TypeScript compartido, Tailwind CSS en la app web y scripts raíz de orquestación. Esta spec deja el repo listo para que las siguientes specs implementen funcionalidad.

## 2. Contexto y dependencias

- **Specs previas requeridas (bloqueantes):** ninguna. Es la base del monorepo; el resto de specs dependen de ella directa o transitivamente.
- **Spec siguiente:** `02-testing-setup` depende de que esta spec deje los paquetes y `tsconfig` base creados.
- **Estado del repo:** greenfield. Solo existe `docs/`. No hay `git` inicializado todavía; esta spec puede inicializarlo si se desea, pero no es obligatorio.
- **Archivos/paquetes que toca:** raíz del repo (`package.json`, `pnpm-workspace.yaml`, `tsconfig.base.json`, `.gitignore`, `.env.example`), `apps/web`, `apps/api`, `packages/db`, `packages/domain`.
- **Requerimientos del PRD cubiertos:**
  - **RNF-02 (Disponibilidad local):** la herramienta debe arrancar como proceso único (admin + API) con un solo comando documentado.
  - **RNF-03 (Portabilidad):** debe ejecutarse en Linux/macOS con dependencias mínimas (sin servicios externos obligatorios).

## 3. Alcance (In / Out)

### In
- `pnpm-workspace.yaml` declarando `apps/*` y `packages/*`.
- `package.json` raíz con scripts de orquestación basados en pnpm filters.
- `tsconfig.base.json` compartido y un `tsconfig.json` por paquete que lo extiende.
- Scaffolding mínimo de los 4 paquetes con su `package.json` y un punto de entrada vacío/placeholder.
- `apps/web`: Next.js (App Router) + Tailwind CSS configurado (no se construyen pantallas reales aquí).
- `apps/api`: Hono con un endpoint de salud `GET /health`.
- `.gitignore` y `.env.example` con las variables de entorno del proyecto.

### Out
- Tests (los define la spec `02-testing-setup`).
- Schema de base de datos y migraciones (spec `03`).
- Endpoints de negocio, login, UI de flags, evaluador (specs `04`–`09`).
- Turborepo: opcional, NO requerido. La orquestación se hace con pnpm filters.

## 4. Tareas en orden

1. Crear `pnpm-workspace.yaml` en la raíz:
   ```yaml
   packages:
     - "apps/*"
     - "packages/*"
   ```
2. Crear `package.json` raíz (privado) con `packageManager` pnpm y scripts (ver Notas técnicas).
3. Crear `tsconfig.base.json` con opciones estrictas compartidas (ver Notas técnicas).
4. Crear `.gitignore` incluyendo `node_modules`, `.env`, `data/`, `.next`, `dist`, `*.db`.
5. Crear `.env.example` con: `DATABASE_PATH=./data/flags.db`, `DEMO_USER=admin`, `DEMO_PASSWORD=admin`, `SESSION_SECRET=change-me`.
6. Crear `packages/domain` con `package.json` (nombre `@ff/domain`), `tsconfig.json` que extiende la base, y `src/index.ts` placeholder que exporta `export {}`.
7. Crear `packages/db` con `package.json` (nombre `@ff/db`), dependencias `drizzle-orm` y `@libsql/client`, `tsconfig.json`, y `src/index.ts` placeholder.
8. Crear `apps/api` con `package.json` (nombre `@ff/api`), dependencia `hono` y `@hono/node-server`, `tsconfig.json`, y `src/index.ts` que levante un servidor Hono con `GET /health` devolviendo `{ "status": "ok" }`.
9. Crear `apps/web` con Next.js + Tailwind: `package.json` (nombre `@ff/web`), `next.config.js`, `tailwind.config.ts`, `postcss.config.js`, `app/layout.tsx`, `app/page.tsx`, y `app/globals.css` con las directivas de Tailwind.
10. Instalar dependencias con `pnpm install` desde la raíz.
11. Documentar en un `README.md` raíz el comando único de arranque (RNF-02).

## 5. Criterios de aceptación verificables

- [ ] `pnpm install` en la raíz completa sin errores.
- [ ] `pnpm -r exec tsc --noEmit` (o `pnpm typecheck`) compila los 4 paquetes sin errores de tipos.
- [ ] `pnpm --filter @ff/api dev` levanta la API y `curl http://localhost:3001/health` devuelve `{"status":"ok"}`.
- [ ] `pnpm --filter @ff/web dev` levanta Next.js y la página raíz responde HTTP 200 con estilos Tailwind aplicados (una clase utilitaria visible).
- [ ] Existen los directorios `apps/web`, `apps/api`, `packages/db`, `packages/domain`, cada uno con su `package.json` y `tsconfig.json`.
- [ ] **RNF-02:** un único script raíz documentado (`pnpm dev`) arranca admin + API.
- [ ] **RNF-03:** no se requiere ningún servicio externo (sin Docker/Postgres/Redis) para arrancar.
- [ ] `.env` está en `.gitignore` y NO se commitea; solo se versiona `.env.example`.

## 6. Notas técnicas

### Estructura objetivo

```
feature-flags/
  apps/
    web/        # Next.js + Tailwind (UI admin)
    api/        # Hono (API REST: flags, evaluación, login)
  packages/
    db/         # Drizzle ORM + @libsql/client (schema, migraciones, seed)
    domain/     # Tipos TS, validación Zod, evaluador puro (sin I/O)
  docs/
  pnpm-workspace.yaml
  package.json
  tsconfig.base.json
  .env.example
```

### `tsconfig.base.json` (sugerido)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "declaration": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true
  }
}
```

### Scripts raíz (sugeridos en `package.json`)

```json
{
  "private": true,
  "packageManager": "pnpm@9",
  "scripts": {
    "dev": "pnpm --parallel -r dev",
    "build": "pnpm -r build",
    "typecheck": "pnpm -r exec tsc --noEmit",
    "test": "pnpm -r test"
  }
}
```

### Decisiones de stack (válidas para todas las specs)
- **Gestor de paquetes:** pnpm workspaces; orquestación con pnpm filters. Turborepo opcional.
- **Lenguaje:** TypeScript en todos los paquetes.
- **Web:** Next.js (App Router) + Tailwind CSS.
- **API:** Hono (puerto sugerido 3001).
- **Persistencia:** `drizzle-orm` + `@libsql/client`, archivo local vía `DATABASE_PATH` (default `./data/flags.db`).
- **Variables de entorno del proyecto:** `DATABASE_PATH`, `DEMO_USER`, `DEMO_PASSWORD`, `SESSION_SECRET`.

### Ambientes del dominio (constante usada en specs 03, 06, 08, 09)
Los ambientes son exactamente: `dev`, `staging`, `prod`.
