# Feature Flags Dashboard

Herramienta interna para activar/desactivar features por ambiente, empresa y rollout porcentual.

## Requisitos

- Node.js 20+
- [pnpm](https://pnpm.io/) 9

## Configuración

Copia las variables de entorno de ejemplo:

```bash
cp .env.example .env
```

## Arranque local (admin + API)

Un solo comando levanta la UI (Next.js en `:3000`) y la API (Hono en `:3001`):

```bash
pnpm dev
```

No se requiere Docker, Postgres, Redis ni ningún otro servicio externo.

## Scripts

| Comando | Descripción |
|---------|-------------|
| `pnpm dev` | Arranca `@ff/web` y `@ff/api` en paralelo |
| `pnpm build` | Build de todos los paquetes |
| `pnpm typecheck` | Comprobación de tipos en todos los paquetes |
| `pnpm test` | Tests (placeholder hasta spec 02) |

## Estructura

```
apps/web        Next.js + Tailwind (UI admin)
apps/api        Hono (API REST)
packages/db     Drizzle ORM + libSQL
packages/domain Tipos, validación Zod, evaluador puro
```

Documentación de producto: [docs/pdrs/feature-flags-mvp.md](docs/pdrs/feature-flags-mvp.md).
