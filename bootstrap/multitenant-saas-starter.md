# Multi-tenant SaaS starter — el baseline de Sprint 1

## Por qué este doc

Cuando arranques un SaaS multi-tenant nuevo (uno por cliente = un tenant, datos aislados, signup público o controlado), hay un conjunto de decisiones que se toman antes de escribir la primera ruta de negocio. Si las dejás para "más adelante", terminás reescribiendo módulos enteros — o peor, descubriendo en producción que un cliente vio datos de otro.

Este doc captura el baseline que armamos en Nexus al cierre del Sprint 1. Es el set mínimo para tener: auth funcional, multi-tenancy con RLS validada, onboarding atómico-en-lo-posible, tests E2E aislados y CI verde. **Punto de partida para el siguiente proyecto similar — clonable conceptualmente, no literal.**

Tag de referencia en Nexus: [`v1.0-sprint1-baseline`](https://github.com/Dev-Joseluis/Nexus/releases/tag/v1.0-sprint1-baseline). `git checkout v1.0-sprint1-baseline` te deja en exactamente este estado.

## Cuándo aplica este pattern

**Aplica si:**

- Modelo SaaS B2B donde "tenant" = "organización cliente" (empresa, equipo, unidad operativa). Un user puede pertenecer a 1+ tenants vía membership.
- Aislamiento de datos a nivel fila es requisito (no solo a nivel app). Un cliente que ve datos de otro = incidente de seguridad reportable.
- El stack es Postgres-friendly y el equipo es chico (1-5 devs) — overhead de operar tres bases de datos físicas separadas no se justifica.
- Hay un piloto controlado (10-100 users) antes de escalar — podés aceptar trade-offs pragmáticos hoy y diferir optimizaciones a Pro.

**No aplica si:**

- Cada cliente tiene su propia base de datos física (modelo schema-per-tenant o database-per-tenant). Las decisiones cambian — RLS pasa a ser irrelevante, los costos operativos suben, el aislamiento es trivial.
- Es un consumer SaaS donde "cuenta = persona" sin concepto de organización. No necesitás memberships ni tenant context — un `user_id` en cada query alcanza.
- Compliance impone separación física (HIPAA strict, ciertos contratos enterprise). Volver a evaluar.

## Stack ground rules

Decisiones que tomás antes de escribir código y NO se discuten después sin un ADR:

| Decisión | Default | Por qué |
|---|---|---|
| Lenguaje | TypeScript estricto end-to-end | Compartir tipos entre back y front; Zod schemas reutilizables |
| Backend framework | NestJS (monolito modular) | DI, módulos por capa de negocio, testing infrastructure madura |
| ORM | Drizzle | TypeScript-first, zero codegen, SQL transparente cuando lo necesitás |
| Validación | Zod | Schema único compartido entre API boundary y frontend forms |
| DB | Postgres 16 + extensiones | RLS nativo, JSON columns, range types, gen_random_uuid built-in |
| Auth | better-auth | Email + password + sessions, OAuth pluggable, schema generado vs custom |
| Multi-tenancy | RLS con role NOBYPASSRLS | Defensa en profundidad — ver `multitenant-db/rls-with-nobypassrls-role.md` |
| Frontend | Next.js 15 + Tailwind + shadcn/ui | App Router, RSC, PWA ready |
| Tests | Vitest + supertest + docker-compose.test.yml aislado | Unit + integration + E2E sin tocar dev DB |
| CI | GitHub Actions con servicios postgres + redis embedded | Reproducible, sin self-hosted runners hasta que el costo lo justifique |
| Quality gates | Husky + lint-staged + ESLint flat + Prettier | Bloqueo en pre-commit, no "promesa de revisar después" |

Si tu proyecto se desvía de uno de estos, escribí un ADR ANTES de codear. Tres horas de discusión hoy ahorran tres semanas de refactor en el sprint 6.

## Componentes del baseline (el orden importa)

### 1. Schema de DB con multi-tenancy desde la primera tabla

**Tablas que existen al final del setup:**

- `users` — globales, sin `tenant_id`. Un user puede pertenecer a 0+ tenants.
- `tenants` — raíz del modelo. Sin `tenant_id` (es la fuente). Slug + name + flag is_active.
- `memberships` — N:M entre users y tenants. Lleva `tenant_id` + `role` (enum owner/admin/member/viewer). Tenant-scoped con RLS FORCE.
- `sessions`, `accounts`, `verification_tokens` — tablas internas de better-auth, globales (sin RLS per-tenant).

**Conventions desde día 1:**

- PKs siempre `uuid().defaultRandom()` (NO serial, NO bigint). Evita enumeration attacks y simplifica replicación futura.
- Toda tabla tenant-scoped tiene columna `tenant_id UUID NOT NULL`.
- Toda tabla con `tenant_id` tiene policy RLS desde la migración inicial. **Sin "TODO: agregar RLS después"** — agregarlo después es el camino hacia un leak.
- Helpers compartidos en `_shared.ts`: `primaryId()`, `timestamps`, `softDelete`, `tenantIdColumn()`.

**Tres archivos críticos en `packages/db/`:**

- `src/schema/*.ts` — definiciones Drizzle.
- `src/migrations/custom/001-rls-policies.sql` — `ENABLE` + `FORCE` RLS + policies por tabla.
- `src/migrations/custom/002-create-app-role.sql` — role NOBYPASSRLS con grants.

Detalle del pattern: ver `multitenant-db/rls-with-nobypassrls-role.md`.

### 2. Two-client split (NO usar superuser en runtime)

```typescript
// packages/db/src/client.ts
export function createAdminClient(): DbClient {
  // DATABASE_URL → conecta como superuser. SOLO migrations + admin scripts.
  // BYPASSRLS — saltea policies. Si lo usás en runtime, las 3 capas son decorativas.
}

export function createAppClient(): DbClient {
  // APP_DATABASE_URL → conecta como nexus_app (NOBYPASSRLS).
  // Uso del runtime del producto. Toda query pasa por RLS.
}
```

Red line: **el API en runtime nunca usa `createAdminClient()`**. Si aparece en un controller / service / guard, es bug crítico. Hay UNA excepción documentada en Nexus al cierre de Sprint 1 (queries cross-tenant del propio user, ej: `/me/memberships`) y va a backlog para resolverse vía SECURITY DEFINER function. Reconocer la excepción explícitamente es parte del pattern — los workarounds no documentados se vuelven la regla.

### 3. Auth integration con better-auth

**Pieza que sale del baseline:**

- `apps/api/src/auth/auth.factory.ts` — instancia `betterAuth({...})` con drizzle adapter, mapping de tablas (`schema.users` → `user` model name, etc.), `trustedOrigins`, cookies httpOnly + sameSite=lax + secure si prod.
- `apps/api/src/auth/auth.guard.ts` — `AuthGuard` que valida session via `auth.api.getSession({ headers })` y agrega `user` + `session` al request.
- `apps/api/src/auth/auth.tokens.ts` — `AUTH_TOKEN` string en archivo separado para evitar circular imports.
- `apps/api/src/main.ts` — bootstrap con `bodyParser: false` + handler de better-auth montado ANTES de body parsers + `setGlobalPrefix("api", { exclude: ["health"] })` + CORS dual (better-auth `trustedOrigins` para `/api/auth/*`, NestJS `enableCors` para el resto).

**Gotchas del camino que cuestan medio día si no las sabés:**

- **better-auth genera IDs alfanuméricos no-UUID.** Si tus PKs son `uuid().defaultRandom()`, el INSERT explota con `invalid input syntax for type uuid`. Fix: `advanced.database.generateId: () => randomUUID()`. Crítico, primer hallazgo del primer smoke test si no lo prevés.
- **NestJS body parser entra en conflicto con el body parser interno de better-auth.** Hay que `bodyParser: false` en `NestFactory.create` y montar el handler de better-auth ANTES de `useBodyParser('json')`. El orden importa.
- **`auth.api.getSession({ headers })` espera `Headers` (Web standard), NO el `Record<string, string|string[]>` de Express.** Sin conversión, el guard tira 401 incluso con cookie válida. Helper inline en `auth.guard.ts`.
- **better-auth requiere `Origin` header en POSTs autenticados** (sign-out, account update). signup pre-auth no lo exige. Tests E2E con supertest necesitan `.set("Origin", FRONTEND_URL)`.
- **CORS de NestJS NO aplica a `/api/auth/*`** porque el handler de better-auth termina el request antes de pasar por el pipeline NestJS. Solución: `trustedOrigins` en la config de better-auth (CORS built-in para sus endpoints) + `enableCors()` para el resto. **Dos pipelines distintos**.

Implementación de referencia: PR [#24](https://github.com/Dev-Joseluis/Nexus/pull/24) en Nexus.

### 4. Onboarding endpoint (composite, NO hooks)

El endpoint que crea user + tenant + membership en un solo round-trip. Tres approaches considerados:

- **(a) `databaseHooks.user.create.after`** — better-auth llama nuestro callback después de crear el user. Pero el hook corre en `queueAfterTransactionHook` que es fire-and-forget — el response al cliente puede llegar antes que el tenant se cree. Race window observable, frontend ve `[]` en `/me/memberships` justo después del signup.
- **(b)** **Custom endpoint `POST /api/onboarding/sign-up`** que wrappea `auth.api.signUpEmail({ asResponse: true, headers, body })` y después abre Tx2 propia para tenant + membership. **Lo elegido.** Single round-trip, control explícito de errores, idempotencia clara.
- **(c)** Wrapper custom de adapter de better-auth para una única tx atómica. Pure architectural fantasy en piloto — 3-4 días de trabajo para resolver un problema que con (b) acotás a un orphan window de ~10ms.

**Caveats de (b) que vale documentar como tech debt** (no resolver en piloto):

- **Dos transacciones, no atómicas.** Tx1 (better-auth) y Tx2 (nuestra) corren secuenciales. Si Tx2 falla, el user queda orphan (existe en auth, no tiene tenant). Logueás error con `userId`, devolvés 500 con código `ONBOARDING_TENANT_FAILED`. Reconciliation cron es item de Pro.
- **Sin Origin check en `/api/onboarding/*`.** better-auth nativo no lo exige en signup pre-auth (anyone can sign up). Como wrappeás, heredás. Middleware compartido es item de Pro.
- **Sin rate limiting.** Mismo argumento.

**Cookie forwarding pattern**:

```typescript
// onboarding.service.ts
const signUpResponse = await this.auth.api.signUpEmail({
  body: { email, password, name },
  headers: toWebHeaders(req.headers),
  asResponse: true, // ← clave: devuelve Response, no plain data
});

if (!signUpResponse.ok) {
  const errorBody = await signUpResponse.json();
  throw new HttpException(errorBody, signUpResponse.status); // shape verbatim
}

// Set-Cookie multi-valor: Node 22 expone getSetCookie() que devuelve string[]
const setCookies = signUpResponse.headers.getSetCookie();
// ... en el controller: res.setHeader('set-cookie', setCookies)
```

Implementación de referencia: PR [#25](https://github.com/Dev-Joseluis/Nexus/pull/25) en Nexus.

### 5. Test infrastructure aislada

Sin esto, los E2E tests pollute la dev DB y el feedback loop se rompe.

**Setup:**

- `docker-compose.test.yml` con postgres+redis en puertos distintos del dev stack (`5443` vs `5442`).
- `.env.test` con `APP_DATABASE_URL` apuntando al stack test.
- `pnpm test:db:up` / `test:db:migrate` / `test:db:down` como aliases.
- `pnpm test:integration` que orquesta up → migrate → run vitest → cleanup vía trap.
- En cada test E2E, **guard explícito** en `beforeAll`:

```typescript
if (process.env["NEXUS_TEST_DB"] !== "isolated") {
  throw new Error("E2E requiere NEXUS_TEST_DB=isolated. Use pnpm test:integration.");
}
```

Sin el guard, una corrida accidental de `pnpm test` ensucia dev DB con users de prueba, slugs colisionan en runs futuros, debugging se vuelve infierno.

**Test crítico que valida la 3ra capa de defensa** (RLS via NOBYPASSRLS role):

```typescript
it("Con setTenantContext(A), SELECT * FROM memberships solo devuelve rows del tenant A", async () => {
  const appClient = createAppClient(); // nexus_app, NOBYPASSRLS
  const rows = await appClient.db.transaction(async (tx) => {
    await setTenantContext(tx, tenantAId);
    return tx.select().from(schema.memberships); // SIN WHERE — única defensa: RLS
  });
  expect(rows.every(r => r.tenantId === tenantAId)).toBe(true);
});
```

Si alguien rompe la policy en una migración futura, este test truena en CI. Es el **goal de oro** del Sprint 1 — el test que define "el sistema es realmente multi-tenant, no solo aspiracionalmente".

Implementación de referencia: PRs [#22](https://github.com/Dev-Joseluis/Nexus/pull/22) (test stack aislado) y [#25](https://github.com/Dev-Joseluis/Nexus/pull/25) (cross-tenant tests).

### 6. CI con quality gates

Workflow GitHub Actions con 4 jobs en paralelo: lint+typecheck, build, security audit (`pnpm audit` + gitleaks), unit+integration tests. Cada job tiene su block `env:` con secrets necesarios (BETTER_AUTH_SECRET para CI puede ser fixed string; el real va en GitHub Secrets).

**Punto fácil de romper:** cuando agregás una env var nueva al schema de Zod (ej: `BETTER_AUTH_SECRET`), tenés que sumarla al `env:` del job de tests. Si no, el CI rompe con "Required" pero los tests locales pasan. Aprendido en PR #24.

Implementación de referencia: `.github/workflows/ci.yml` en Nexus + PR [#23](https://github.com/Dev-Joseluis/Nexus/pull/23) (Husky + ESLint flat).

## Orden de implementación recomendado

Si vas a clonar el pattern para un proyecto nuevo, este es el orden que minimiza retrabajo:

1. **Monorepo scaffold** — pnpm workspaces, Turbo, tsconfig base, lint base.
2. **Schema + RLS desde la primera tabla** — `users`, `tenants`, `memberships` + policies + role NOBYPASSRLS. Tests del role.
3. **Bootstrap NestJS + Config + Db modules + /health endpoint** — minimum viable boot.
4. **Test infrastructure aislada** — `docker-compose.test.yml` + scripts antes del primer feature test.
5. **Quality gates** — Husky + lint-staged + ESLint flat + CI con todos los jobs.
6. **Auth integration** (better-auth) + AuthGuard + /me endpoint.
7. **Onboarding endpoint** + /me/memberships + cross-tenant E2E (el goal de oro).

Cada uno de estos pasos es un PR de ~12 archivos, ~1-2 días de trabajo. El total: 7 PRs, 2 semanas. Tag al cierre. Próximo sprint arranca con baseline.

## Decisiones que dejamos para "después" (y por qué)

Items que ya están en `backlog-pro.md` de Nexus al cierre de Sprint 1 — todo justificable diferir hasta tener PMF:

- **Rate limiting en `/api/auth/*` y `/api/onboarding/*`** — atacante puede enumerar emails / brute-force. Para piloto controlado (~10-100 users conocidos), aceptable.
- **Email verification en signup** — `autoSignIn: true` permite registrar cualquier email y entrar. Usable para piloto cerrado, bloqueante pre-Pro.
- **CSRF/Origin check estricto** — herencia del comportamiento de better-auth.
- **ADR de queries cross-tenant del propio user** — formalizar el caso `/me/memberships`. Defer hasta tener 2-3 patrones similares (`/me/notifications`, `/me/audit-log`) para generalizar.
- **Reconciliation del orphan window de onboarding** — manual via SQL en piloto, cron job pre-Pro.
- **Wrapper custom de adapter para atomicidad real** — el approach (c) que descartamos. Reabrir si el cron job no alcanza.

Pattern: cada item tiene **trigger explícito** ("cuando X, entonces hacerlo"), no "algún día". Los backlogs sin triggers se atrofian.

## Cuándo NO usar este baseline literal

- **Stack incompatible** — si tu proyecto es Python/Django o Ruby/Rails, las decisiones cambian (RLS pattern aplica, framework integration no). El handbook tiene `multitenant-db/rls-with-nobypassrls-role.md` que es framework-agnostic.
- **Single-tenant SaaS** — un cliente único o "el cliente es la organización entera". Sacate `tenant_id`, RLS no aplica, mucho de este overhead desaparece.
- **App con concept de "espacios" pero sin aislamiento legal/comercial** (ej: Notion-style workspaces dentro de la misma cuenta). El pattern aplica pero relajado — no necesitás role NOBYPASSRLS si el riesgo de leak entre workspaces de un mismo user es aceptable.
- **Setup donde el customer-facing producto se separa físicamente del admin tooling** — el admin tool puede usar un superuser sin riesgo, el customer-facing API usa NOBYPASSRLS. Mismo pattern, dos deploys.

## Referencias

- **Nexus repo (implementación de referencia):** https://github.com/Dev-Joseluis/Nexus
- **Tag de baseline:** [`v1.0-sprint1-baseline`](https://github.com/Dev-Joseluis/Nexus/releases/tag/v1.0-sprint1-baseline)
- **PRs del Sprint 1:**
  - [#10](https://github.com/Dev-Joseluis/Nexus/pull/10) — monorepo scaffold
  - [#12](https://github.com/Dev-Joseluis/Nexus/pull/12), [#14](https://github.com/Dev-Joseluis/Nexus/pull/14) — DB role + factories
  - [#18](https://github.com/Dev-Joseluis/Nexus/pull/18) — bootstrap NestJS
  - [#22](https://github.com/Dev-Joseluis/Nexus/pull/22) — docker-compose.test.yml
  - [#23](https://github.com/Dev-Joseluis/Nexus/pull/23) — ESLint flat + Husky
  - [#24](https://github.com/Dev-Joseluis/Nexus/pull/24) — better-auth integration
  - [#25](https://github.com/Dev-Joseluis/Nexus/pull/25) — onboarding + cross-tenant RLS
- **Docs relacionados del handbook:**
  - `multitenant-db/rls-with-nobypassrls-role.md` — el pattern RLS framework-agnostic.
  - `workflow/cowork-claude-code-separation.md` — workflow operativo Cowork ↔ Claude Code.
- **ADRs de Nexus relevantes** (en `docs/adr/`):
  - ADR-002 — Postgres + extensiones como storage engine único.
  - ADR-008 — Tenant context propagation via transaction-per-request.
  - ADR-009 — Exemption de RLS sobre tabla `tenants` durante piloto.
