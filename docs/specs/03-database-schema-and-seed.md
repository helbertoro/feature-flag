# Spec 03 - Schema de base de datos y seed

PRD de referencia: [docs/pdrs/feature-flags-mvp.md](../pdrs/feature-flags-mvp.md)

## 1. Objetivo

Definir el schema de persistencia con Drizzle ORM sobre SQLite (libSQL) para las tablas `flags`, `environment_config` y `company_overrides`, las migraciones, el cliente de base de datos configurable por `DATABASE_PATH`, un seed de datos demo, y un helper que autocrea la configuración por ambiente (`dev`, `staging`, `prod`) con rollout 0% cuando se crea una flag.

## 2. Contexto y dependencias

- **Specs previas requeridas (bloqueantes):** `01-monorepo-setup` (paquete `packages/db` con `drizzle-orm` y `@libsql/client`).
- **Spec siguiente:** `04-api-feature-flags` consume estas tablas y el helper de autocreación; `08` y `09` leen `environment_config` y `company_overrides`.
- **Dependencias no bloqueantes:** `02-testing-setup` solo es necesaria para *ejecutar* los tests de esta spec, no para implementarla. Tras `01`, puede ejecutarse en paralelo con `05` (y con `02`).
- **Archivos/paquetes que toca:** `packages/db` (`src/schema.ts`, `src/client.ts`, `src/seed.ts`, `src/createFlagConfig.ts`, `drizzle.config.ts`, carpeta `migrations/`).
- **Requerimientos del PRD cubiertos:**
  - **RF-10:** al crear una flag, registrar automáticamente configuración inicial en los tres ambientes (`dev`, `staging`, `prod`) con rollout `0%` y sin overrides de empresa.
  - **RF-23:** toda la configuración persiste en SQLite local.
  - **RF-24:** path del archivo SQLite configurable vía `DATABASE_PATH` con default `./data/flags.db`.
  - **RF-25:** ejecutar migraciones de schema al arrancar si la versión almacenada es inferior a la esperada.
  - **RNF-06:** migraciones idempotentes y reversibles manualmente (script down documentado).

## 3. Alcance (In / Out)

### In
- Schema Drizzle de las tres tablas según el apéndice "Modelo de datos" del PRD.
- Cliente libSQL leyendo `DATABASE_PATH` (default `./data/flags.db`), creando el directorio `data/` si no existe.
- Migraciones generadas con drizzle-kit + runner que las aplica al arrancar (RF-25).
- Helper `createFlagWithDefaults` / `seedEnvironmentConfig` que inserta la flag y sus 3 filas de `environment_config` con `rollout_percentage = 0`, `master_enabled = false`.
- Script `seed` con datos demo de ejemplo.
- Documentación de migración `down` manual (RNF-06).

### Out
- Endpoints HTTP (spec `04`).
- Lógica de evaluación (spec `09`).
- Validación de formato de `key` (se define en spec `04`; aquí solo persistencia).

## 4. Tareas en orden

1. En `packages/db`, definir `src/schema.ts` con las tablas `flags`, `environment_config`, `company_overrides` (ver Notas técnicas).
2. Crear `src/client.ts` que construya el cliente `@libsql/client` con `url = file:${DATABASE_PATH ?? "./data/flags.db"}`, creando el directorio padre si no existe, y exporte la instancia `db` de `drizzle(...)`.
3. Crear `drizzle.config.ts` apuntando a `src/schema.ts` y a la carpeta `migrations/`.
4. Generar migración inicial con `pnpm --filter @ff/db exec drizzle-kit generate`.
5. Crear `src/migrate.ts` que ejecute `migrate(db, { migrationsFolder: "./migrations" })` al arrancar (RF-25); exponerlo como función `runMigrations()` y como script `pnpm --filter @ff/db migrate`.
6. Crear `src/createFlagConfig.ts` con `createFlagWithDefaults(input)` que, en una transacción: inserta la flag y luego inserta 3 filas en `environment_config` (una por ambiente `dev`, `staging`, `prod`) con `master_enabled = false` y `rollout_percentage = 0` (RF-10).
7. Crear `src/seed.ts` que aplique migraciones y cree 1-2 flags demo (p. ej. `billing.new-checkout`) usando el helper anterior; script `pnpm --filter @ff/db seed`.
8. Documentar en comentarios o README del paquete el procedimiento `down` manual (RNF-06).
9. Escribir tests Vitest del helper y del seed (ver criterios).

## 5. Criterios de aceptación verificables

- [ ] `pnpm --filter @ff/db migrate` crea el archivo en la ruta de `DATABASE_PATH` (o `./data/flags.db` por defecto) sin error. (RF-24)
- [ ] Cambiar `DATABASE_PATH=./tmp/test.db` hace que la base se cree en esa ruta. (RF-24)
- [ ] **RF-10 / CA-03:** tras crear la flag `billing.new-checkout` con `createFlagWithDefaults`, existen exactamente 3 filas en `environment_config` para esa flag (`dev`, `staging`, `prod`), todas con `master_enabled = false` y `rollout_percentage = 0`, y 0 filas en `company_overrides`.
- [ ] Test Vitest: insertar una flag y verificar que `select` sobre `environment_config` devuelve los 3 ambientes esperados.
- [ ] **RF-25:** llamar `runMigrations()` dos veces seguidas es idempotente (no falla la segunda vez).
- [ ] `pnpm --filter @ff/db seed` deja la base con al menos la flag demo y su config por ambiente.
- [ ] **CA-12:** tras "reiniciar" (reabrir el cliente apuntando al mismo archivo), los datos persisten.

## 6. Notas técnicas

### Modelo de datos (apéndice del PRD)

```
flags
  id, key, name, description, namespace, created_at, updated_at

environment_config
  id, flag_id, environment, master_enabled, rollout_percentage, created_at, updated_at

company_overrides
  id, flag_id, environment, company_id, enabled, created_at, updated_at
```

### Schema Drizzle (sugerido, SQLite)

```ts
// packages/db/src/schema.ts
import { sqliteTable, integer, text } from "drizzle-orm/sqlite-core";
import { sql } from "drizzle-orm";

export const flags = sqliteTable("flags", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  key: text("key").notNull().unique(),
  name: text("name").notNull(),
  description: text("description").notNull().default(""),
  namespace: text("namespace").notNull(),
  createdAt: text("created_at").notNull().default(sql`CURRENT_TIMESTAMP`),
  updatedAt: text("updated_at").notNull().default(sql`CURRENT_TIMESTAMP`),
});

export const environmentConfig = sqliteTable("environment_config", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  flagId: integer("flag_id").notNull().references(() => flags.id, { onDelete: "cascade" }),
  environment: text("environment").notNull(), // 'dev' | 'staging' | 'prod'
  masterEnabled: integer("master_enabled", { mode: "boolean" }).notNull().default(false),
  rolloutPercentage: integer("rollout_percentage").notNull().default(0), // 0-100
  createdAt: text("created_at").notNull().default(sql`CURRENT_TIMESTAMP`),
  updatedAt: text("updated_at").notNull().default(sql`CURRENT_TIMESTAMP`),
});

export const companyOverrides = sqliteTable("company_overrides", {
  id: integer("id").primaryKey({ autoIncrement: true }),
  flagId: integer("flag_id").notNull().references(() => flags.id, { onDelete: "cascade" }),
  environment: text("environment").notNull(), // 'dev' | 'staging' | 'prod'
  companyId: text("company_id").notNull(),
  enabled: integer("enabled", { mode: "boolean" }).notNull(),
  createdAt: text("created_at").notNull().default(sql`CURRENT_TIMESTAMP`),
  updatedAt: text("updated_at").notNull().default(sql`CURRENT_TIMESTAMP`),
});
```

### Cliente (sugerido)

```ts
// packages/db/src/client.ts
import { createClient } from "@libsql/client";
import { drizzle } from "drizzle-orm/libsql";
import { mkdirSync } from "node:fs";
import { dirname } from "node:path";

const path = process.env.DATABASE_PATH ?? "./data/flags.db";
mkdirSync(dirname(path), { recursive: true });
export const client = createClient({ url: `file:${path}` });
export const db = drizzle(client);
```

### Ambientes
Los ambientes son exactamente `dev`, `staging`, `prod`. Definir una constante `export const ENVIRONMENTS = ["dev", "staging", "prod"] as const;` reutilizable.

### Borrado en cascada (RF-09)
`environment_config.flag_id` y `company_overrides.flag_id` referencian `flags.id` con `onDelete: "cascade"`, de modo que eliminar una flag borra sus reglas de targeting asociadas (lo usa la spec `04`/`07`). Habilitar `PRAGMA foreign_keys = ON;` en el cliente si fuese necesario para que SQLite respete el cascade.

### Migración down manual (RNF-06)
Documentar que para revertir la migración inicial se ejecuta el SQL inverso: `DROP TABLE company_overrides; DROP TABLE environment_config; DROP TABLE flags;`. Las migraciones generadas por drizzle-kit son idempotentes vía la tabla de control de migraciones.
