# Spec 09 - Evaluador de flags (puro + API)

PRD de referencia: [docs/pdrs/feature-flags-mvp.md](../pdrs/feature-flags-mvp.md)

## 1. Objetivo

Implementar el evaluador puro (sin I/O) en `packages/domain` que decide si una flag está `enabled` dado un contexto `(flag, environment, company_id?)` siguiendo la precedencia exacta del PRD, y exponerlo vía `GET /api/v1/evaluate` en `apps/api` con validación de ambiente, comportamiento fail-safe off, rollout sticky determinista y logs estructurados.

## 2. Contexto y dependencias

- **Specs previas requeridas (bloqueantes):** `01-monorepo-setup` (`apps/api`, `packages/domain`), `03-database-schema-and-seed` (lectura de `flags`, `environment_config`, `company_overrides`).
- **Spec siguiente:** ninguna; es la última de la cadena.
- **Dependencias no bloqueantes:** `02-testing-setup` solo para *ejecutar* tests; `04-api-feature-flags` y `08-targeting-rules` **no** son bloqueantes: el evaluador lee las tablas de `03` directamente y sus tests insertan la configuración por filas, así que `09` puede implementarse y testearse en paralelo con `04` y `08`.
- **Archivos/paquetes que toca:** `packages/domain/src/evaluator.ts` (función pura), `apps/api/src/routes/evaluate.ts` (endpoint + lectura de DB + logs).
- **Requerimientos del PRD cubiertos:**
  - **RF-17:** exponer `GET /api/v1/evaluate` con parámetros `flag` (key), `environment`, `company_id` (opcional).
  - **RF-18:** respuesta `{ "enabled": boolean, "reason": string }` con `reason` ∈ {`master_off`, `company_override`, `rollout`, `default_off`, `flag_not_found`}.
  - **RF-19:** si `flag` no existe -> `{ "enabled": false, "reason": "flag_not_found" }` con HTTP 200 (fail-safe off).
  - **RF-20:** si `environment` no es válido -> HTTP 400 con mensaje descriptivo.
  - **RF-21:** evaluación de rollout determinista y sticky: misma `(company_id, flag_key)` produce el mismo resultado mientras el porcentaje no cambie.
  - **RF-22:** si `company_id` no se provee, el rollout es no aplicable -> default off.
  - **RNF-01:** `/evaluate` responde en < 50 ms p95 con SQLite local y hasta 1 000 flags.
  - **RNF-07:** logs estructurados en evaluación con `flag_key`, `environment`, `company_id`, `enabled`, `reason`.

## 3. Alcance (In / Out)

### In
- Función pura `evaluate(input)` en `packages/domain` sin acceso a DB ni a red.
- Función de hash sticky determinista.
- Endpoint `GET /api/v1/evaluate` que lee la configuración de la DB y delega en la función pura.
- Validación de `environment` (400 si inválido).
- Fail-safe off para flag inexistente (200, `flag_not_found`).
- Logs estructurados por evaluación.
- Tests unitarios del evaluador puro y tests de integración del endpoint cubriendo CA-04..CA-11.

### Out
- UI (no hay UI nueva en esta spec).
- Mutaciones de configuración (specs `04`/`08`).
- Autenticación: `/api/v1/evaluate` es de uso server-to-server y NO requiere sesión demo (consistente con spec `05`).

## 4. Tareas en orden

1. En `packages/domain`, definir tipos del contexto y resultado:
   - Input del evaluador: `{ flagExists: boolean; masterEnabled: boolean; rolloutPercentage: number; override?: { enabled: boolean }; companyId?: string; flagKey: string }`.
   - Output: `{ enabled: boolean; reason: "master_off" | "company_override" | "rollout" | "default_off" | "flag_not_found" }`.
2. Implementar `hashRollout(companyId, flagKey)` determinista que retorne un entero 0–99 (ver Notas técnicas).
3. Implementar `evaluate(input)` aplicando la precedencia exacta (ver Notas técnicas), retornando `enabled` y `reason`.
4. Escribir tests unitarios puros del evaluador para cada rama de la precedencia y para la propiedad sticky (CA-07/08/09).
5. En `apps/api`, crear `GET /api/v1/evaluate` que:
   - Lea `flag`, `environment`, `company_id` de la query.
   - Valide `environment` ∈ {`dev`, `staging`, `prod`}; si no, `400` con mensaje (RF-20).
   - Busque la flag por `key`; si no existe, responder `200 { enabled:false, reason:"flag_not_found" }` (RF-19) sin consultar más.
   - Si existe, leer `environment_config` (master, rollout) y el override de `company_id` (si se proveyó) para ese ambiente.
   - Construir el input y llamar `evaluate(...)`.
   - Emitir un log estructurado con `flag_key`, `environment`, `company_id`, `enabled`, `reason` (RNF-07).
   - Responder `200` con el resultado.
6. Montar el endpoint sin el middleware de sesión.
7. Escribir tests de integración del endpoint con `app.request(...)` cubriendo CA-04, CA-05, CA-06, CA-10, CA-11.

## 5. Criterios de aceptación verificables

- [ ] **RF-19 / CA-11:** `GET /api/v1/evaluate?flag=no.existe&environment=dev` -> HTTP 200 con `{ "enabled": false, "reason": "flag_not_found" }`.
- [ ] **RF-20:** `GET /api/v1/evaluate?flag=billing.new-checkout&environment=qa` -> HTTP 400 con mensaje descriptivo.
- [ ] **CA-04:** con master on en `staging`, `...&environment=staging&company_id=any` -> `enabled: true` (rollout 100) o `reason: rollout` según config; con rollout 100 y sin override -> `enabled:true`.
- [ ] **RF-12 / CA-05:** con master off en `staging`, misma llamada -> `{ "enabled": false, "reason": "master_off" }`, independientemente de override o rollout.
- [ ] **RF-16 / CA-06:** override `on` para `company_id=acme` en `prod` con rollout 0% -> con `company_id=acme` `{ enabled:true, reason:"company_override" }`; con otro id `{ enabled:false, reason:"default_off" }`.
- [ ] **CA-10:** con master off, todos los `company_id` (incluidos los que el rollout expondría) -> `enabled:false, reason:"master_off"`.
- [ ] **RF-21 / CA-07:** rollout 50% en `dev` sin overrides: ~50% de `company_id` distintos dan `true`, y la misma combinación da siempre el mismo resultado en llamadas repetidas.
- [ ] **RF-21 / CA-08:** subir rollout 20%→60%: los `company_id` que estaban en el 20% siguen en `true`.
- [ ] **RF-21 / CA-09:** bajar rollout 60%→30%: `company_id` previamente expuestos por hash < 30 siguen en `true`; no hay flickering (sticky por hash, no por estado almacenado).
- [ ] **RF-22:** sin `company_id`, con rollout < 100 -> `enabled:false, reason:"default_off"`.
- [ ] **RNF-07:** cada evaluación emite un log con `flag_key`, `environment`, `company_id`, `enabled`, `reason`.

## 6. Notas técnicas

### Endpoint

`GET /api/v1/evaluate`

| Parámetro | Requerido | Notas |
|-----------|-----------|-------|
| `flag` | sí | key de la flag (`namespace.name`) |
| `environment` | sí | uno de `dev`, `staging`, `prod`; otro valor -> HTTP 400 |
| `company_id` | no | si falta, rollout no aplica -> default off (RF-22) |

Respuesta (HTTP 200 salvo ambiente inválido):

```json
{ "enabled": true, "reason": "rollout" }
```

### Precedencia de evaluación (literal del apéndice "Flujo de evaluación" del PRD)

Orden: `flag_not_found -> master_off -> company_override -> default_off -> rollout`.

```
1. ¿Existe la flag?            → No  → { enabled:false, reason:"flag_not_found" }  (HTTP 200)
2. ¿Master off en ambiente?   → Sí  → { enabled:false, reason:"master_off" }
3. ¿Override para company?     → Sí  → { enabled: override.enabled, reason:"company_override" }
4. ¿Rollout % > 0 y aplica?    → No  → { enabled:false, reason:"default_off" }
5. hash(company_id + flag_key) % 100 < rollout_percentage
                               → Sí  → { enabled:true,  reason:"rollout" }
                               → No  → { enabled:false, reason:"default_off" }
```

Notas sobre los pasos:
- Paso 3: el override aplica solo si se proveyó `company_id` y existe un override para ese `company_id` en ese ambiente. Su valor (`on`/`off`) se devuelve tal cual con `reason: "company_override"` (precede al rollout, RF-16, pero no al master off, RF-12).
- Paso 4: el rollout "no aplica" si `company_id` no se proveyó (RF-22) o si `rollout_percentage <= 0`. En ambos casos -> `default_off`.
- `reason` permitido exactamente: `master_off`, `company_override`, `rollout`, `default_off`, `flag_not_found`.

### Evaluador puro (sugerido)

```ts
// packages/domain/src/evaluator.ts
export type Reason =
  | "master_off"
  | "company_override"
  | "rollout"
  | "default_off"
  | "flag_not_found";

export interface EvalInput {
  flagExists: boolean;
  masterEnabled: boolean;
  rolloutPercentage: number; // 0-100
  override?: { enabled: boolean };
  companyId?: string;
  flagKey: string;
}

export interface EvalResult {
  enabled: boolean;
  reason: Reason;
}

export function evaluate(input: EvalInput): EvalResult {
  if (!input.flagExists) return { enabled: false, reason: "flag_not_found" };
  if (!input.masterEnabled) return { enabled: false, reason: "master_off" };
  if (input.override) {
    return { enabled: input.override.enabled, reason: "company_override" };
  }
  if (!input.companyId || input.rolloutPercentage <= 0) {
    return { enabled: false, reason: "default_off" };
  }
  const bucket = hashRollout(input.companyId, input.flagKey) % 100;
  if (bucket < input.rolloutPercentage) {
    return { enabled: true, reason: "rollout" };
  }
  return { enabled: false, reason: "default_off" };
}
```

### Hash sticky determinista (RF-21)

La condición es `hash(company_id + flag_key) % 100 < rollout_percentage`. El hash debe ser determinista y estable entre reinicios (no usar `Math.random` ni el hash no estable de objetos). Sugerencia: un hash simple tipo FNV-1a o djb2 sobre la cadena `company_id + flag_key`.

```ts
export function hashRollout(companyId: string, flagKey: string): number {
  const s = companyId + flag_key_safe(flagKey);
  let h = 2166136261; // FNV-1a 32-bit offset basis
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i);
    h = Math.imul(h, 16777619);
  }
  return (h >>> 0); // entero sin signo; el caller aplica % 100
}
const flag_key_safe = (k: string) => k;
```

La propiedad sticky surge de que el bucket depende solo de `(company_id, flag_key)`: al subir el % entran más buckets pero los previos siguen incluidos; al bajar el %, un `company_id` con `bucket < nuevo%` sigue en `true` y los que estaban entre el nuevo% y el anterior% salen, pero el resultado es estable y reproducible (sin flickering en llamadas repetidas con el mismo %).

### Rendimiento (RNF-01)
Para `< 50 ms p95` con hasta 1 000 flags: la evaluación lee como máximo la flag por `key` (índice único), su fila de `environment_config` por `(flag_id, environment)` y, si hay `company_id`, su override por `(flag_id, environment, company_id)`. Asegurar índices adecuados; la función pura es O(longitud de la cadena del hash).

### Observabilidad (RNF-07)
Emitir un log estructurado (JSON) por evaluación, p. ej.:

```json
{ "flag_key": "billing.new-checkout", "environment": "staging", "company_id": "acme", "enabled": true, "reason": "rollout" }
```

### Notas
- Ambientes válidos exactamente: `dev`, `staging`, `prod`.
- El endpoint NO requiere sesión demo (uso server-to-server).
- La función pura no realiza I/O; toda lectura de DB ocurre en `apps/api` antes de invocarla, lo que la hace trivialmente testeable (specs de `02`).
