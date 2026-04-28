# Methodology: Cross-tenant E2E como gate del sprint multi-tenant

## Resumen en 2 líneas

- En sprints de un sistema multi-tenant, el cross-tenant E2E test es **el gate** del sprint, no un test más.
- Sin él, todos los demás tests pueden pasar verde mientras la propiedad central (aislamiento entre tenants) está rota.

## El problema que resuelve

Sistemas multi-tenant tienen una propiedad de seguridad fundamental: **un tenant no puede ver ni modificar datos de otro tenant**. Esa propiedad típicamente se construye en capas:

1. Application layer (guards, middleware) que setean tenant context.
2. Database layer (RLS policies, role attributes como NOBYPASSRLS).
3. Schema constraints (FKs, CHECK constraints).

Cualquiera de las 3 capas puede tener un bug que rompa el aislamiento. Tests unitarios cubren cada capa por separado. Tests de integración cubren combinaciones puntuales. **Pero ninguno valida que las 3 juntas mantienen la propiedad end-to-end.**

El cross-tenant E2E test es ese validador. Ejercita el sistema completo con 2 tenants en paralelo y verifica que ninguna operación de tenant A afecta a tenant B.

## Estructura del test

Setup: 2 users autenticados, cada uno en su propio tenant, con datos sembrados.

```typescript
describe("Cross-tenant isolation E2E", () => {
  let agentA: TestAgent; // user A logged in, tenant A active
  let agentB: TestAgent; // user B logged in, tenant B active
  let dataA: { suppliers: Supplier[]; items: Item[]; movements: Movement[] };
  let dataB: { suppliers: Supplier[]; items: Item[]; movements: Movement[] };

  beforeAll(async () => {
    // Crear 2 users via signup completo (signup → tenant → membership).
    // Sembrar datos asimétricos en cada tenant para detectar leaks por cardinalidad.
    // Tenant A: 2 suppliers + 3 items + 5 movements
    // Tenant B: 1 supplier + 1 item + 2 movements
  });

  // Cardinalidad correcta — RLS filtra automáticamente
  it("User A solo ve sus 2 suppliers", () => {});
  it("User A solo ve sus 3 items", () => {});
  it("User A solo ve sus 5 stock_movements", () => {});
  it("User B solo ve sus 1 supplier", () => {});

  // Cross-creation bloqueada — FK + RLS verification
  it("User A no puede crear movement con itemId del tenant B", () => {});
  it("User A no puede crear movement con supplierId del tenant B", () => {});

  // Cross-read bloqueado — RLS hace 404 invisible
  it("User A GET /api/items/ → 404", () => {});

  // Computaciones respetan tenant context
  it("User A balance es correcto con cardinalidad real", () => {});
  it("User A no puede leer balance de item del tenant B", () => {});

  // Operations especiales del entity (immutability, etc.)
  it("PATCH a movement de tenant propio → 405", () => {});
});
```

## Por qué cardinalidad asimétrica

Si tenant A y tenant B tienen el mismo número de items (digamos 3 each), un test que verifica `length === 3` no distingue entre "RLS filtra correctamente" y "RLS no filtra y la query devuelve los 6 items pero alguna otra cosa los corta a 3".

Cardinalidad asimétrica (A=3, B=1) hace que cada test verifique la cardinalidad esperada del tenant específico. Si RLS falla, los counts son visiblemente incorrectos.

## Por qué setup completo via signup

Tentación: insertar datos directos vía DB con `createAdminClient` para velocidad. Antipattern.

Razón: el signup completo (POST /api/onboarding/sign-up) ejercita la stack:

- AuthGuard asigna user.
- TenantContextInterceptor o equivalente resuelve tenant.
- OnboardingService crea tenant + membership.
- Cookie de session.

Si algún paso del flow tiene bug, el setup falla y el test no llega a las assertions. Eso es información — el flow E2E está roto, no solo los aislamientos. Insertar via createAdminClient esconde bugs en el flow de signup.

## Cuándo escribirlo

**El último test del último PR del sprint multi-tenant.** Razones:

- Antes del último PR, faltan endpoints/entidades. El test parcial es deuda — captura aislamiento solo de lo que existe, no del sistema completo.
- El último PR es donde aparecen las business operations más complejas (en Nexus: stock_movements con FK references). Ahí se materializan las superficies de ataque más sutiles.
- Es el gate explícito: "si este test no pasa verde, el sprint NO está cerrado".

En Nexus PR #37 (último de Sprint 2), el cross-tenant E2E descubrió un bug crítico (Postgres FK bypassea RLS — ver `multitenant-db/postgres-fk-bypasses-rls.md`). Sin el test, ese bug habría llegado a producción. Con el test, fue cazado antes del primer commit a staging.

## Anti-patterns que el test catchea

- **FK bypass de RLS**: el más sutil — INSERT con FK referencing a otro tenant succeed silently.
- **Manual SET LOCAL incorrecto**: developer setea tenant context manualmente y se confunde de tenant_id.
- **AuthGuard que no setea user**: cualquier endpoint queda público sin que se note hasta el cross-tenant.
- **Interceptor que skipea cuando no debería**: opt-out activado por error en endpoint tenant-scoped.
- **Cookies cross-domain mal configuradas**: en setups con múltiples subdomains, cookies del tenant A pueden filtrarse al tenant B.
- **Cache compartido sin partitioning**: Redis/memcached con keys que no incluyen tenant_id.

Cada uno de estos bugs es invisible a tests unitarios. El cross-tenant E2E es el ÚNICO que los catchea.

## Costo

Setup pesado (~50-80 líneas de seed code). Run lento (cada test hace HTTP requests reales contra DB real). Maintenance: cada vez que aparece nueva entidad tenant-scoped, agregar al setup + 1-2 tests al describe.

Para piloto, el costo es absorbible. Para production-grade con N entidades, considerar generadores de fixtures + parallelization de tests.

## Cuándo NO escribirlo

- Sistema single-tenant: cero valor, eliminar.
- MVP early-stage donde no hay tenant logic todavía: defer hasta que aparezca.
- Sprints que solo tocan business logic dentro de un tenant existente (CRUD pure de un dominio): tests existentes cubren. El cross-tenant E2E se actualiza solo cuando el setup cambia.

## Referencias

- Implementación de Nexus: `apps/api/test/cross-tenant-inventory.flow.test.ts`
- PR donde se incorporó como gate: [Nexus PR #37](https://github.com/Dev-Joseluis/Nexus/pull/37)
- Bug que el test cazó: ver `multitenant-db/postgres-fk-bypasses-rls.md`
- Cross-tenant E2E inicial (más simple, Sprint 1): `apps/api/test/cross-tenant.flow.test.ts`

## Lección general

Tests son seguros para regresiones, pero el primer test de una propiedad nueva debe escribirse antes del bug que lo dispara. Para sistemas multi-tenant, **el aislamiento es la propiedad central** — merece su test de oro escrito como gate, no agregado retroactivamente cuando aparezca el primer leak.
