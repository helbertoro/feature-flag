# Spec 02 - Configuración de testing

PRD de referencia: [docs/pdrs/feature-flags-mvp.md](../pdrs/feature-flags-mvp.md)

## 1. Objetivo

Configurar Vitest por paquete con una configuración compartida, un script de test raíz que ejecute toda la suite, y al menos un test verde de ejemplo. Esto habilita que todas las specs posteriores escriban criterios de aceptación verificables mediante tests automatizados (base para los CA-xx del PRD).

## 2. Contexto y dependencias

- **Specs previas requeridas (bloqueantes):** `01-monorepo-setup` (los 4 paquetes y `tsconfig.base.json`).
- **Spec siguiente:** `03-database-schema-and-seed` y todas las demás usarán Vitest para sus criterios de aceptación.
- **Paralelismo:** una vez lista `01`, puede ejecutarse en paralelo con `03` y `05`. El resto de specs solo necesita `02` para *ejecutar* sus tests, no para implementarse. La numeración 01–09 es orden sugerido de lectura, NO una cadena estricta.
- **Archivos/paquetes que toca:** raíz (`package.json` script `test`, posible `vitest.workspace.ts`), y cada paquete (`vitest.config.ts` + carpeta de tests).
- **Requerimientos del PRD cubiertos:** no cubre un RF/RNF específico; provee la infraestructura de verificación para los criterios de aceptación CA-01..CA-13 del PRD y para los acceptance de las specs `03`–`09`.

## 3. Alcance (In / Out)

### In
- Instalar `vitest` (y `@vitest/coverage-v8` opcional) como dependencia de desarrollo.
- Configuración compartida de Vitest reutilizable por paquete.
- `vitest.config.ts` en `packages/domain`, `packages/db`, `apps/api` (y `apps/web` si aplica).
- Script raíz `pnpm test` que ejecute los tests de todos los paquetes.
- Un test verde de ejemplo en `packages/domain` (función trivial pura).

### Out
- Tests de negocio reales (los aportan las specs correspondientes).
- Cobertura mínima obligatoria / gates de CI (fuera de alcance del MVP).
- E2E de navegador.

## 4. Tareas en orden

1. Añadir `vitest` a las `devDependencies` de cada paquete que tendrá tests (`packages/domain`, `packages/db`, `apps/api`), o a la raíz si se centraliza.
2. Crear una configuración compartida. Opción recomendada: un archivo `vitest.shared.ts` en la raíz con defaults (entorno `node`, `globals: true`) que cada paquete importe y extienda con `mergeConfig`.
3. Crear `packages/domain/vitest.config.ts` extendiendo la config compartida.
4. Crear `packages/domain/src/example.ts` con una función pura simple, p. ej. `export const sum = (a: number, b: number) => a + b;`.
5. Crear `packages/domain/test/example.test.ts` que importe `sum` y verifique `expect(sum(2, 3)).toBe(5)`.
6. Añadir el script `"test": "vitest run"` al `package.json` de cada paquete con tests.
7. Asegurar que el script raíz `pnpm test` (definido como `pnpm -r test`) recorre todos los paquetes.
8. (Opcional) Crear `vitest.workspace.ts` en la raíz para ejecutar todo con un solo proceso de Vitest.

## 5. Criterios de aceptación verificables

- [ ] `pnpm test` desde la raíz ejecuta los tests de todos los paquetes y termina con código 0.
- [ ] `pnpm --filter @ff/domain test` corre el test de ejemplo y reporta 1 passed.
- [ ] El test de ejemplo `test/example.test.ts` está verde: `expect(sum(2, 3)).toBe(5)`.
- [ ] Cada paquete con tests tiene su `vitest.config.ts` y un script `test` en su `package.json`.
- [ ] Ejecutar `pnpm test` no requiere servicios externos (consistente con RNF-03 de la spec 01).

## 6. Notas técnicas

### Config compartida (sugerida)

`vitest.shared.ts` (raíz):

```ts
import { defineConfig } from "vitest/config";

export const sharedConfig = defineConfig({
  test: {
    environment: "node",
    globals: true,
    include: ["test/**/*.test.ts", "src/**/*.test.ts"],
  },
});
```

`packages/domain/vitest.config.ts`:

```ts
import { mergeConfig } from "vitest/config";
import { sharedConfig } from "../../vitest.shared";

export default mergeConfig(sharedConfig, {});
```

### Test de ejemplo

```ts
// packages/domain/test/example.test.ts
import { describe, it, expect } from "vitest";
import { sum } from "../src/example";

describe("sum", () => {
  it("suma dos números", () => {
    expect(sum(2, 3)).toBe(5);
  });
});
```

### Notas
- Vitest se ejecuta por paquete para mantener aislamiento; el evaluador puro de `packages/domain` (spec `09`) se prueba como función pura sin I/O.
- Para `apps/api`, los tests pueden levantar la app Hono en memoria usando `app.request(...)` sin abrir puertos.
