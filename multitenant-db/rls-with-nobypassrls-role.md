# Multi-tenant isolation: Postgres RLS con NOBYPASSRLS role

## Por qué este doc

Si estás arrancando un sistema multi-tenant donde distintos tenants deben NO poder ver datos de otros tenants (SaaS tradicional, marketplace, sistema de gestión empresarial con múltiples clientes), necesitás una estrategia de aislamiento de datos que sobreviva errores de implementación. Este doc describe el pattern que adoptamos en Nexus, con las decisiones que llevaron ahí y las trampas que aparecen en el camino.

## Resumen del pattern — defensa en profundidad

Tres capas de defensa, cada una independiente y redundante:

1. **Application layer:** validación en el boundary de API. El token de sesión contiene `user_id`; el middleware resuelve el `tenant_id` desde memberships y lo propaga al resto del request.
2. **Database policy layer:** cada tabla con columna `tenant_id` tiene una RLS policy que filtra por `current_setting('app.current_tenant')`.
3. **Database role layer:** la app conecta a Postgres con un role `NOBYPASSRLS` — esto fuerza que las RLS policies se apliquen incluso si el código de la app intenta evadirlas accidentalmente.

Cada capa protege contra fallos de las otras. Sin la capa 3, la capa 2 es decorativa (el superuser bypassea RLS). Sin la capa 2, la capa 1 depende 100% de la disciplina del developer (un `WHERE` olvidado basta para un leak). Sin la capa 1, los errores de la capa 2 aparecen como "sin datos" en la UI en vez de "sesión inválida".

## Capa 1 — Application layer

En cada request autenticado, un middleware resuelve el tenant del user:

```typescript
// Ejemplo conceptual (NestJS)
@Injectable()
export class TenantGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.user; // seteado por AuthGuard previo

    const membership = await this.membershipService.getByUserId(user.id);
    if (!membership) throw new ForbiddenException("No tenant");

    request.tenantId = membership.tenantId; // accesible al handler
    return true;
  }
}
```

**Regla inviolable:** el `tenant_id` NUNCA viene del body, query, o path params del cliente. Solo del claim de sesión del user autenticado. Si el cliente lo envía, el middleware lo ignora. El momento en que el cliente dicta su propio tenant, toda la arquitectura se cae — alguien va a encontrar el ID de otro tenant (URL compartida, error en response, scraping de public API) y usarlo.

## Capa 2 — Database RLS policies

Para cada tabla con `tenant_id`:

```sql
-- Enable RLS on the table
ALTER TABLE memberships ENABLE ROW LEVEL SECURITY;

-- FORCE — aplica incluso al owner de la tabla
ALTER TABLE memberships FORCE ROW LEVEL SECURITY;

-- Policy: rows visibles solo si tenant_id matchea el contexto
CREATE POLICY tenant_isolation ON memberships
  USING      (tenant_id = current_setting('app.current_tenant')::uuid)
  WITH CHECK (tenant_id = current_setting('app.current_tenant')::uuid);
```

**Detalles que importan:**

- `ENABLE ROW LEVEL SECURITY` es necesario pero no suficiente. Por default, el **owner** de la tabla (típicamente el superuser que corrió la migration) está exento. Sin `FORCE`, si tu role de app llega a ser owner (setup simple común), las policies no se aplican.
- `USING` filtra `SELECT`. `WITH CHECK` filtra `INSERT`/`UPDATE` — previene insertar rows con `tenant_id` ajeno incluso con permisos.
- `current_setting('app.current_tenant')` lee un GUC (Grand Unified Configuration — variable de session de Postgres). La app debe setear este valor antes de cada query.
- `::uuid` fuerza el cast. Si `app.current_tenant` no está seteado, `current_setting()` devuelve empty string `""`, y `''::uuid` tira error. **Esto es deliberado, no un bug.** Es fail-fast: una query sin contexto explota en vez de devolver silenciosamente 0 rows (ver sección "Fail-closed semantics" abajo).

## Capa 3 — NOBYPASSRLS role

Postgres tiene el atributo de role `BYPASSRLS` que ignora completamente todas las RLS policies. El superuser tiene este atributo por default. Si la app conecta como superuser (common antipattern), las capas 1 y 2 pueden ser perfectas pero inútiles.

Solución: crear un role específico para la app con `NOBYPASSRLS` explícito.

```sql
-- Migration idempotente (custom SQL, corrida post-migrations de ORM)
DO $$
BEGIN
    IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname = 'nexus_app') THEN
        EXECUTE format(
          'CREATE ROLE nexus_app WITH LOGIN PASSWORD %L NOBYPASSRLS',
          current_setting('app.nexus_app_password')
        );
    ELSE
        -- Defensive: garantizar NOBYPASSRLS y actualizar password si existía
        ALTER ROLE nexus_app NOBYPASSRLS;
        EXECUTE format(
          'ALTER ROLE nexus_app PASSWORD %L',
          current_setting('app.nexus_app_password')
        );
    END IF;
END
$$;

-- Grants: CRUD sobre tablas + sequences, nada de DDL
GRANT USAGE ON SCHEMA public TO nexus_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO nexus_app;
GRANT SELECT, USAGE, UPDATE ON ALL SEQUENCES IN SCHEMA public TO nexus_app;

-- Default privileges: tablas futuras (creadas por migrations) heredan grants
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO nexus_app;
```

**Dos connection strings separadas:**

- `DATABASE_URL` — conecta como superuser. Uso exclusivo: correr migrations, scripts administrativos, tests que requieren setup cross-tenant. Bypassa RLS.
- `APP_DATABASE_URL` — conecta como `nexus_app`. Uso del runtime del producto (todos los endpoints que sirven al cliente). Respeta RLS.

Regla: **nunca usar `DATABASE_URL` en el runtime del producto**. Si aparece en un controller, guard, o service de apps/api, es un bug crítico.

## Propagación del contexto — transaction-per-request con SET LOCAL

Cada request autenticado debe correr en un contexto Postgres donde `app.current_tenant` esté seteado al `tenant_id` del user. Tres estrategias realistas:

- **A. Transaction-per-request:** middleware abre transaction, ejecuta `SET LOCAL app.current_tenant`, corre el handler dentro, commitea.
- **B. Connection-per-request con SET SESSION:** setea session var, corre queries, connection se devuelve al pool con `DISCARD ALL`.
- **C. AsyncLocalStorage + ORM wrapper:** contexto mágico propagado por Node; wrapper de ORM emite `SET LOCAL` automáticamente.

**Elegimos A.** Justificación completa en el ADR-008 de Nexus. Resumen:

```typescript
// Middleware (pseudocode)
async function tenantContextMiddleware(req, res, next) {
  await db.transaction(async (tx) => {
    // set_config con tercer argumento = true = SET LOCAL (scoped a transaction)
    await tx.execute(
      sql`SELECT set_config('app.current_tenant', ${req.tenantId}, true)`,
    );

    req.tx = tx; // handler usa req.tx para todas sus queries
    await next();
  });
}
```

**Por qué SET LOCAL y no SET SESSION:**

`SET LOCAL` está scoped a la transaction actual. Al commit o rollback, el GUC vuelve a su valor anterior. Esto garantiza que el contexto no leakea a queries futuras — incluso si la connection vuelve al pool con estado residual, la próxima transaction empieza desde cero.

`SET SESSION` persistiría en la connection hasta que alguien lo resetee explícitamente. Requiere que el pool haga `DISCARD ALL` entre requests. Si el pool falla (bug de librería, edge case, connection returning sin reset), el request siguiente heredaría el contexto del anterior — leakage cross-tenant silencioso, la falla más peligrosa posible en multi-tenant.

## Helpers de aplicación

Exponé 3 helpers sobre la primitiva de SET LOCAL para que los handlers no escriban SQL raw:

```typescript
// rls.ts

/** Setea el tenant context para la transaction. Uso: middleware. */
export async function setTenantContext(
  tx: PgTransaction,
  tenantId: string,
): Promise<void> {
  await tx.execute(
    sql`SELECT set_config('app.current_tenant', ${tenantId}, true)`,
  );
}

/** Limpia el contexto. Uso: antes de operaciones sistema. */
export async function clearTenantContext(tx: PgTransaction): Promise<void> {
  await tx.execute(sql`SELECT set_config('app.current_tenant', '', true)`);
}

/**
 * Fail-closed: ejecuta fn con el contexto limpio. Si fn hace query a una
 * tabla tenant-scoped, la RLS crashea (invalid uuid cast). Uso: admin ops.
 */
export async function asSystem<T>(
  tx: PgTransaction,
  fn: (tx: PgTransaction) => Promise<T>,
): Promise<T> {
  await clearTenantContext(tx);
  return fn(tx);
}
```

`asSystem` es el complemento de `setTenantContext`. Cuando una operación necesita genuinamente cruzar tenants (cron jobs, admin dashboard, analytics), usa `asSystem` explícitamente. El fail-closed semantics garantiza que accidentes quedan visibles.

## Fail-closed semantics — por qué queremos que explote

Lo más importante del pattern: **si el contexto está roto, la query debe fallar ruidosamente, no silenciosamente**.

Evidencia concreta con el cast `::uuid`:

```sql
-- Sin app.current_tenant seteado, current_setting devuelve ""
SELECT * FROM memberships;
-- → ERROR: invalid input syntax for type uuid: ""
```

Comparalo con el alternativo silent:

```sql
-- Policy "amigable" que maneja el empty string con NULLIF
CREATE POLICY tenant_isolation ON memberships
  USING (tenant_id::text = NULLIF(current_setting('app.current_tenant'), ''))
  WITH CHECK (...);

-- Resultado: SELECT sin contexto devuelve 0 rows limpios.
```

La versión silent parece "más amigable" pero es peligrosa: un bug en el middleware (contexto no seteado por algún path que no pasa por el guard) produce "user ve lista vacía" en UI. El user puede interpretar eso como "mi cuenta está vacía, voy a crear un item nuevo" — y ese item termina insertado con contexto ausente, probablemente con `tenant_id` dictado desde otro lado (request body? default fallback?). Cross-tenant data pollution silencioso.

La versión crashing produce `500 Internal Server Error` que los monitors y logs capturan inmediatamente. El equipo de on-call despierta, el bug se fixea en el boundary correcto (el middleware), no hay leak. **Dolor agudo y localizado > dolor crónico y difuso.**

## Testing strategy

Los integration tests validan las tres capas:

**Test 1 — El role tiene NOBYPASSRLS:**

```typescript
it("nexus_app has rolbypassrls = false", async () => {
  const result = await adminClient`
    SELECT rolbypassrls FROM pg_roles WHERE rolname = 'nexus_app'
  `;
  expect(result[0].rolbypassrls).toBe(false);
});
```

**Test 2 — RLS filtra cross-tenant queries correctamente:**

```typescript
it("RLS aisla queries por tenant", async () => {
  // Setup: 2 tenants con memberships distintas (insertadas via adminClient)
  // Test: con app.current_tenant = tenantA, SELECT trae solo rows de A
  await appClient.begin(async (tx) => {
    await setTenantContext(tx, tenantA.id);
    const rows = await tx`SELECT * FROM memberships`;
    expect(rows.every((r) => r.tenantId === tenantA.id)).toBe(true);
  });
});
```

**Test 3 — Query crashea sin contexto (contrato de fail-fast):**

```typescript
it("query falla si app.current_tenant no está seteado", async () => {
  await expect(
    appClient`SELECT * FROM memberships`,
  ).rejects.toThrow(/invalid input syntax for type uuid/);
});
```

El tercer test es el más valioso. Documenta que el fail-fast es **comportamiento esperado por diseño**, no un bug que alguien debería "arreglar" en el futuro con un `NULLIF`. Si alguien relaja la policy para suavizar el crash, este test falla ruidosamente y obliga a un ADR que justifique romper la red line.

## Trampas comunes (antipatterns)

**Usar el superuser para el runtime del producto.** La trampa #1. Con BYPASSRLS activo, las capas 2 y 3 son decorativas. Motivó issue #9 de Nexus (materializar el role + refactor de client.ts). Mitigación: conexiones separadas, nunca `DATABASE_URL` en runtime.

**Olvidar `FORCE ROW LEVEL SECURITY`.** Sin `FORCE`, el owner de la tabla está exento. Si el role de la app llega a ser owner (setup simple), las policies no se aplican. Mitigación: `FORCE` en cada ALTER TABLE al habilitar RLS, como convención.

**Cargar el `tenant_id` del cliente.** Si el cliente puede dictar su propio tenant via request body/query, toda la arquitectura se cae. Mitigación: `tenant_id` viene del claim de sesión, NUNCA del input del cliente.

**`SET SESSION` en lugar de `SET LOCAL`.** Si el pool no hace `DISCARD ALL` correctamente, el contexto leakea entre requests. Mitigación: `SET LOCAL` scoped a transaction.

**Migration del role no versionada.** Si el role se creó a mano en desarrollo pero no vive en una SQL migration, cada deploy limpio crea una DB sin el role. La app falla al conectar, o alguien lo "fixea" conectando como superuser y la RLS queda bypassed. Mitigación: migration idempotente (`custom/002-create-app-role.sql` en Nexus convention).

**Queries con `tx.execute(sql\`...\`)` raw que olvidan el contexto.** Escape hatches son útiles pero pueden bypasear helpers. Mitigación: ESLint rule (tech-debt, post-MVP) que warning sobre uso directo de `createAppClient()` fuera del middleware de request.

## Referencias

- **Nexus repo** (implementación de referencia):
  - `packages/db/src/migrations/custom/001-rls-policies.sql` — policies sobre tablas tenant-scoped.
  - `packages/db/src/migrations/custom/002-create-app-role.sql` — role NOBYPASSRLS + grants + default privileges.
  - `packages/db/src/rls.ts` — helpers `setTenantContext`, `clearTenantContext`, `asSystem`.
  - `packages/db/src/client.ts` — factories `createAppClient()` y `createAdminClient()`.
  - `packages/db/test/app-role.test.ts` + `packages/db/test/client.test.ts` — integration tests validando las 3 capas.
  - `docs/adr/008-tenant-context-transaction-per-request.md` — análisis completo de SET LOCAL vs SET SESSION vs AsyncLocalStorage.
- **Commits relevantes en Nexus:**
  - PR #12 — materialización del role `nexus_app` con NOBYPASSRLS.
  - PR #14 — refactor de client.ts a factories + ADR-008.
- **Postgres docs:**
  - https://www.postgresql.org/docs/current/ddl-rowsecurity.html — RLS oficial.
  - https://www.postgresql.org/docs/current/sql-set.html — SET LOCAL vs SET SESSION.
