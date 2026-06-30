# PRD: Herramienta interna de Feature Flags

**Versión:** 1.0  
**Estado:** Decisiones bloqueadas  
**Fecha:** 2026-06-29

---

## 1) Contexto y problema

Hoy, activar o desactivar una funcionalidad para una empresa concreta, un ambiente específico o un subconjunto de tráfico requiere un **deploy de código o configuración**. Eso genera:

- Ciclos de release lentos para cambios de bajo riesgo.
- Imposibilidad de apagar rápidamente una feature en producción ante un incidente.
- Rollouts graduales implementados ad hoc (if/else en código, variables de entorno por ambiente).
- Falta de visibilidad centralizada sobre qué está activo, dónde y para quién.

Se necesita una **herramienta interna** que permita gestionar flags de forma independiente al pipeline de deploy, con persistencia local y un modelo de targeting explícito.

---

## 2) Objetivo

Entregar un **MVP funcional** que permita:

1. Crear y gestionar feature flags booleanas desde una UI admin interna.
2. Configurar targeting por **ambiente**, **empresa** y **rollout porcentual**.
3. Evaluar flags en runtime vía API server-side, sin redeploy al cambiar un toggle.
4. Persistir toda la configuración en **SQLite local**.

**Métrica de éxito del MVP:** un operador puede activar una flag para el 10% de empresas en staging, verificar el comportamiento con la API de evaluación y desactivarla en menos de 5 minutos, sin tocar código ni hacer deploy.

---

## 3) Público objetivo y usuarios

| Actor | Descripción | Necesidad principal |
|-------|-------------|---------------------|
| **Operador interno (demo)** | Desarrollador o PM con acceso a la herramienta vía login demo | Crear flags, configurar targeting y togglear sin deploy |
| **Aplicación consumidora** | Servicio backend que integra el SDK/API de evaluación | Obtener `true/false` por `(flag_key, environment, company_id)` en runtime |
| **Equipo de plataforma** | Mantenedor de convenciones y estándares | Namespaces consistentes, evaluación server-side, documentación de uso |

**Modelo de acceso:** un único **usuario demo** (credenciales fijas o variables de entorno). No hay roles, permisos diferenciados ni OAuth.

---

## 4) Alcance

### In scope

- Login demo (usuario/contraseña fijos o configurables por env var).
- CRUD de feature flags booleanas con convención de namespace (`equipo.feature`).
- Targeting por **ambiente** (`dev`, `staging`, `prod`).
- Override por **empresa** (`company_id`) dentro de un ambiente.
- **Rollout porcentual** por ambiente (0–100%).
- Regla de precedencia: **ambiente → override por empresa → % global del ambiente**.
- API REST de evaluación server-side.
- UI admin para gestión visual de flags y reglas.
- Persistencia en **SQLite** (schema versionado, migraciones básicas).
- Rollout sticky por identidad (`company_id` + `flag_key`).
- Auditoría mínima: `created_at`, `updated_at` por flag y regla.

### Out of scope (explícito)

- OAuth, SSO, RBAC y permisos avanzados.
- Workflow de aprobación para cambios en producción.
- Flags multivariante (A/B con payloads JSON, strings, números).
- Catálogo integrado de empresas (la app consumidora provee `company_id`).
- Infraestructura distribuida (Redis, Postgres, multi-región, LaunchDarkly-style).
- SDK cliente oficial más allá de ejemplos/documentación de la API.
- Métricas, analytics de exposición y experimentación estadística.
- Historial de cambios / audit log detallado por operador.
- Alta disponibilidad, sharding o replicación de SQLite.

---

## 5) Conceptos de dominio

### Feature Flag

Entidad booleana identificada por una **`key` única** (ej. `billing.new-checkout`). Representa una funcionalidad que puede estar activa o inactiva según reglas de targeting. Valor por defecto: **`false` (off)**.

Atributos mínimos: `key`, `name`, `description`, `namespace`, `created_at`, `updated_at`.

### Targeting Rule

Conjunto de condiciones que determinan si una flag está **on** para un contexto de evaluación dado. Tres dimensiones:

| Dimensión | Descripción | Ejemplo |
|-----------|-------------|---------|
| **Ambiente** | Contexto de despliegue | `prod`, `staging`, `dev` |
| **Empresa** | Override explícito para un tenant | `company_id = "acme-123"` → on/off forzado |
| **Rollout %** | Porcentaje de identidades expuestas en el ambiente | `25%` en `staging` |

**Precedencia de evaluación** (de mayor a menor prioridad):

1. Si existe override por empresa en el ambiente solicitado → usar ese valor.
2. Si no, evaluar rollout % del ambiente (hash sticky).
3. Si el rollout no aplica o es 0% → `false`.

### Evaluador

Componente server-side que recibe un contexto de evaluación y devuelve el estado booleano de una flag:

```
Input:  flag_key, environment, company_id (opcional)
Output: { enabled: boolean, reason: string }
```

- Calcula el hash sticky: `hash(company_id + flag_key) % 100 < rollout_percentage`.
- Al **subir** el porcentaje, entran más identidades.
- Al **bajar** el porcentaje, las identidades ya expuestas **mantienen acceso** hasta que la flag se desactive explícitamente (`enabled: false` global o override off).
- Es la **única fuente de verdad**; cualquier cache cliente es secundario.

---

## 6) Requerimientos funcionales

### Autenticación y acceso

**RF-01.** El sistema debe presentar una pantalla de login accesible en `/login`.

**RF-02.** El login debe aceptar credenciales demo configurables vía variables de entorno (`DEMO_USER`, `DEMO_PASSWORD`) con valores por defecto documentados.

**RF-03.** Tras login exitoso, el operador debe acceder al panel admin; sesiones no autenticadas deben redirigir a `/login`.

**RF-04.** El endpoint de logout debe invalidar la sesión demo y redirigir a `/login`.

### Gestión de flags

**RF-05.** El operador debe poder **crear** una flag con: `key` (única, formato `namespace.name`), `name`, `description`.

**RF-06.** El sistema debe rechazar la creación si `key` ya existe o no cumple el formato `namespace.name` (solo minúsculas, números, guiones y puntos).

**RF-07.** El operador debe poder **listar** todas las flags con su estado resumido por ambiente.

**RF-08.** El operador debe poder **editar** `name` y `description` de una flag existente.

**RF-09.** El operador debe poder **eliminar** una flag; la eliminación debe borrar también sus reglas de targeting asociadas.

**RF-10.** Al crear una flag, el sistema debe registrar automáticamente configuración inicial en los tres ambientes (`dev`, `staging`, `prod`) con rollout `0%` y sin overrides de empresa.

### Targeting por ambiente

**RF-11.** El operador debe poder activar o desactivar globalmente una flag en un ambiente específico (toggle master on/off para ese ambiente).

**RF-12.** Cuando el toggle master de un ambiente está **off**, la evaluación debe devolver `false` para ese ambiente independientemente de overrides o rollout %.

**RF-13.** El operador debe poder configurar el **rollout porcentual** (0–100, entero) de una flag por ambiente.

### Targeting por empresa

**RF-14.** El operador debe poder añadir un **override por empresa** en un ambiente: `company_id` + valor `on` u `off`.

**RF-15.** El operador debe poder eliminar un override de empresa existente.

**RF-16.** Un override por empresa en un ambiente debe tener precedencia sobre el rollout % de ese ambiente (pero no sobre el toggle master off del ambiente — ver RF-12).

### API de evaluación

**RF-17.** El sistema debe exponer `GET /api/v1/evaluate` (o equivalente) con parámetros: `flag` (key), `environment`, `company_id` (opcional).

**RF-18.** La respuesta debe incluir `{ "enabled": boolean, "reason": string }` donde `reason` indica la regla aplicada (`master_off`, `company_override`, `rollout`, `default_off`).

**RF-19.** Si `flag` no existe, la API debe devolver `{ "enabled": false, "reason": "flag_not_found" }` con HTTP 200 (fail-safe off).

**RF-20.** Si `environment` no es válido, la API debe devolver HTTP 400 con mensaje de error descriptivo.

**RF-21.** La evaluación de rollout % debe ser **determinista y sticky**: la misma combinación `(company_id, flag_key)` debe producir el mismo resultado mientras el porcentaje no cambie.

**RF-22.** Si `company_id` no se provee, el evaluador debe tratar el rollout como no aplicable y devolver `false` salvo toggle master on sin reglas adicionales (documentar comportamiento: default off).

### Persistencia

**RF-23.** Toda la configuración de flags y reglas debe persistirse en una base de datos **SQLite** local.

**RF-24.** El path del archivo SQLite debe ser configurable vía variable de entorno (`DATABASE_PATH`) con default `./data/flags.db`.

**RF-25.** El sistema debe ejecutar migraciones de schema al arrancar si la versión almacenada es inferior a la esperada.

### UI Admin

**RF-26.** La UI debe mostrar un listado de flags con búsqueda/filtro por namespace.

**RF-27.** La UI debe permitir gestionar targeting (toggle master, rollout %, overrides por empresa) desde la vista de detalle de una flag, organizado por ambiente.

**RF-28.** La UI debe mostrar feedback visual del estado actual de cada flag por ambiente (on/off/parcial %).

---

## 7) Requerimientos no funcionales

**RNF-01. Latencia de evaluación:** la API `/evaluate` debe responder en **< 50 ms p95** con SQLite local y hasta 1 000 flags registradas.

**RNF-02. Disponibilidad local:** la herramienta debe arrancar como proceso único (admin + API) con un solo comando documentado.

**RNF-03. Portabilidad:** debe ejecutarse en Linux/macOS con dependencias mínimas (sin servicios externos obligatorios).

**RNF-04. Consistencia:** cambios en la UI deben ser visibles en la API de evaluación en **< 1 segundo** sin reinicio.

**RNF-05. Seguridad demo:** las credenciales demo no deben committearse al repositorio; deben cargarse solo vía env vars o `.env` ignorado por git.

**RNF-06. Schema versionado:** migraciones SQLite deben ser idempotentes y reversibles manualmente (script down documentado).

**RNF-07. Observabilidad mínima:** logs estructurados en evaluación con `flag_key`, `environment`, `company_id`, `enabled`, `reason`.

**RNF-08. Convención de keys:** documentar formato `namespace.feature-name`; rechazar keys que no cumplan en creación (RF-06).

---

## 8) Criterios de aceptación del MVP

| # | Escenario | Resultado esperado |
|---|-----------|-------------------|
| CA-01 | Login con credenciales demo correctas | Acceso al panel admin |
| CA-02 | Login con credenciales incorrectas | Mensaje de error, sin acceso al panel |
| CA-03 | Crear flag `billing.new-checkout` | Flag visible en listado; existe en dev, staging y prod con 0% |
| CA-04 | Activar toggle master en `staging` | `GET /evaluate?flag=billing.new-checkout&environment=staging&company_id=any` → `enabled: true` |
| CA-05 | Desactivar toggle master en `staging` | Misma llamada → `enabled: false`, `reason: master_off` |
| CA-06 | Override on para `company_id=acme` en `prod` con rollout 0% | Evaluación con `company_id=acme` → `enabled: true`; con otro id → `enabled: false` |
| CA-07 | Rollout 50% en `dev`, sin overrides | ~50% de company_ids distintos evalúan `true` de forma consistente en llamadas repetidas |
| CA-08 | Subir rollout de 20% a 60% | Company_ids ya en el 20% siguen en `true`; nuevas entran hasta ~60% total |
| CA-09 | Bajar rollout de 60% a 30% | Company_ids previamente expuestos siguen en `true` (sticky); no hay flickering |
| CA-10 | Desactivar flag explícitamente (master off) | Todos los company_ids, incluidos los expuestos por rollout, evalúan `false` |
| CA-11 | Evaluar flag inexistente | `enabled: false`, `reason: flag_not_found`, HTTP 200 |
| CA-12 | Reiniciar el proceso | Configuración intacta (persistida en SQLite) |
| CA-13 | Cambiar rollout en UI | API refleja el cambio sin redeploy ni reinicio |

---

## 9) Riesgos y supuestos

### Supuestos

- **S-01:** Las aplicaciones consumidoras ya disponen de `company_id` confiable en el contexto de request.
- **S-02:** El MVP es para uso interno/demo; no se requiere seguridad de producción en autenticación.
- **S-03:** Un único proceso con SQLite es suficiente para la carga esperada del MVP (< 100 req/s de evaluación).
- **S-04:** Los tres ambientes (`dev`, `staging`, `prod`) son suficientes; no se necesitan ambientes dinámicos.
- **S-05:** Los operadores respetarán la convención de namespace sin enforcement automatizado por equipo.

### Riesgos

| Riesgo | Impacto | Mitigación |
|--------|---------|------------|
| **R-01:** SQLite no escala con concurrencia alta de escrituras | Medio | Documentar límite; evaluación es read-heavy; writes solo desde admin |
| **R-02:** Login demo expuesto en red interna | Alto | Documentar que no debe exponerse a internet; env vars obligatorias fuera de local |
| **R-03:** Ambigüedad en sticky rollout al bajar % | Medio | Comportamiento documentado en RF-21 y CA-09; tests de regresión |
| **R-04:** Drift entre ambientes por error humano | Bajo | UI muestra estado side-by-side por ambiente; defaults sincronizados al crear |
| **R-05:** Flags huérfanas en código sin registro en la herramienta | Medio | Convención: toda flag en código debe existir en la herramienta; evaluación fail-safe off |
| **R-06:** Pérdida del archivo SQLite | Alto | Documentar backup manual; path configurable; script de export JSON (nice-to-have post-MVP) |

---

## Apéndice: Modelo de datos (referencia)

```
flags
  id, key, name, description, namespace, created_at, updated_at

environment_config
  id, flag_id, environment, master_enabled, rollout_percentage, created_at, updated_at

company_overrides
  id, flag_id, environment, company_id, enabled, created_at, updated_at
```

## Apéndice: Flujo de evaluación

```
1. ¿Existe la flag?          → No  → false (flag_not_found)
2. ¿Master off en ambiente?  → Sí  → false (master_off)
3. ¿Override para company?  → Sí  → valor del override (company_override)
4. ¿Rollout % > 0?           → No  → false (default_off)
5. hash(company_id + key) % 100 < rollout% ?
                               → Sí  → true (rollout)
                               → No  → false (default_off)
```
