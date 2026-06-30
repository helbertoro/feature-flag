# Feature Flags Dashboard

Herramienta interna para activar/desactivar features por **ambiente**, **empresa** y **rollout porcentual**, sin redeploy. Persistencia local en SQLite. Acceso vía login demo (sin OAuth ni roles).

Fuente de verdad del producto: [docs/pdrs/feature-flags-mvp.md](docs/pdrs/feature-flags-mvp.md).

## Stack

Monorepo pnpm + TypeScript: Next.js + Tailwind (web), Hono (API), Drizzle ORM + libSQL/SQLite (persistencia), Zod (validación), Vitest (tests).

## Estructura del monorepo

```
apps/web        # Next.js + Tailwind: UI admin
apps/api        # Hono: API REST (flags, targeting, login, evaluación)
packages/db     # Drizzle + libSQL: schema, migraciones, seed
packages/domain # Tipos, validación Zod y evaluador puro (sin I/O)
docs/specs      # Specs de implementación
docs/pdrs       # PRD
```

Responsabilidades y dirección de imports: ver la regla `monorepo-architecture`.

