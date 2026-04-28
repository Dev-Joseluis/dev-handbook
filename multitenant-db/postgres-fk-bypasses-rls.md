# Antipattern: trusting Postgres FK constraints to enforce multi-tenant isolation

## Resumen en 2 líneas

- **El antipattern**: confiar en `FK + RLS` para bloquear referencias cross-tenant en INSERTs/UPDATEs.
- **Por qué falla**: las FK constraints de Postgres se validan a nivel system (con privilegios elevados), no respetan las policies de RLS del calling role.

## El descubrimiento

Durante Sprint 2 de Nexus, el cross-tenant E2E test del PR #37 (stock_movements) reveló un bug crítico: usuarios autenticados en tenant A podían crear `stock_movements` referenciando `item_id` o `supplier_id` que pertenecían al tenant B. La INSERT retornaba 201 SUCCESS y contaminaba el tenant target con datos cruzados.

El razonamiento original era plausible:

- `nexus_app` role tiene `NOBYPASSRLS`.
- Tabla `items` tiene RLS habilitada con `FORCE ROW LEVEL SECURITY`.
- Policy filtra `WHERE tenant_id = current_setting('app.current_tenant')::uuid`.
- Si `nexus_app` intenta hacer FK reference a `item_id` que no es de su tenant, debería fallar con FK violation 23503.

Pero ese razonamiento es incorrecto. Postgres ejecuta la validación de FK constraints **a nivel sistema**, con privilegios que ignoran RLS. Cuando el INSERT incluye `item_id`, Postgres pregunta "¿existe una fila con ese ID en `items`?" — la respuesta es sí, sin importar la RLS policy del role que está ejecutando.

> "Reference checks in foreign key constraints are not affected by RLS, since they're considered a part of database integrity rather than a security check."
> — Documentación oficial de Postgres

## Por qué es peligroso

El bug es **silent**. La INSERT retorna 201 con un row del tenant correcto pero referenciando una entidad cruzada. Tres consecuencias:

1. **Contaminación cross-tenant invisible.** El tenant target acumula referencias a IDs que sus usuarios no pueden ver via UI (RLS filtra los SELECTs), pero los IDs siguen ahí en la DB.
2. **Reports cross-tenant rotos.** Aggregations o joins que tocan los IDs ajenos pueden retornar resultados inconsistentes.
3. **Audit trail comprometido.** ¿Quién creó la fila? Audit logs muestran al user de tenant A creando referencias del tenant B — pattern impossible si la regla se respetara.

## Mitigación: SELECT pre-INSERT explícito

Antes de cada INSERT/UPDATE que crea FK references cross-entity, hacer SELECT explícito sobre la entidad referenciada **bajo el RLS context del request**. Si el SELECT retorna 0 rows, abortar con 404 NOT_FOUND_IN_TENANT.

```typescript
// apps/api/src/stock-movements/stock-movements.service.ts
async create(dto: CreateStockMovementDto, tx: AppTransaction) {
  // SELECT pre-INSERT — RLS filtra los SELECT, garantizando
  // que solo accedemos a items del tenant actual.
  const item = await tx
    .select({ id: schema.items.id })
    .from(schema.items)
    .where(eq(schema.items.id, dto.itemId))
    .limit(1);

  if (item.length === 0) {
    throw new NotFoundException({
      code: "ITEM_NOT_FOUND_IN_TENANT",
      message: `Item ${dto.itemId} no existe en el tenant actual`,
    });
  }

  // Si type === 'in', mismo check para supplierId
  if (dto.type === "in" && dto.supplierId) {
    const supplier = await tx
      .select({ id: schema.suppliers.id })
      .from(schema.suppliers)
      .where(eq(schema.suppliers.id, dto.supplierId))
      .limit(1);

    if (supplier.length === 0) {
      throw new NotFoundException({
        code: "SUPPLIER_NOT_FOUND_IN_TENANT",
        message: `Supplier ${dto.supplierId} no existe en el tenant actual`,
      });
    }
  }

  // Ahora sí, INSERT con confianza
  return tx.insert(schema.stock_movements).values({
    itemId: dto.itemId,
    supplierId: dto.supplierId,
    // ...
  });
}
```

## Defense in depth — `isPgForeignKeyViolation` como fallback

El SELECT pre-INSERT es la primary defense. Pero también vale capturar el errcode `23503` como fallback (cubre race conditions o casos donde el item se borra entre el SELECT y el INSERT):

```typescript
function isPgForeignKeyViolation(err: unknown): {
  isViolation: boolean;
  constraint?: string;
} {
  // Drizzle wrappa errores — chequear ambos paths
  const code = (err as any)?.code ?? (err as any)?.cause?.code;
  const constraint =
    (err as any)?.constraint_name ?? (err as any)?.cause?.constraint_name;
  return {
    isViolation: code === "23503",
    constraint,
  };
}
```

## Cuándo aplica

Este pattern aplica a **toda entidad tenant-scoped que referencia otra entidad tenant-scoped via FK**.

Casos típicos:

- `stock_movements.item_id` → `items.id`
- `stock_movements.supplier_id` → `suppliers.id`
- `purchase_orders.supplier_id` → `suppliers.id`
- `stock_transfers.from_location_id` → `locations.id`

NO aplica a:

- FKs hacia entidades globales (`users`, `tenants`) — esas no tienen RLS o son cross-tenant by design.
- FKs hacia la propia tabla (self-references) — el RLS de la tabla ya filtra ambos extremos.

## Costo

1-2 round-trips extra por INSERT/UPDATE con FKs cross-entity. En piloto con baja carga, marginal. En Pro con alta concurrencia, considerar:

- **Caching del lookup en request scope** (memoizar lookups dentro de la misma transaction).
- **Validation batch** para INSERTs múltiples (un SELECT con `WHERE id IN (...)` en lugar de N selects individuales).
- **SECURITY DEFINER function** que combina lookup + INSERT atómico bypaseando RLS internamente con privilegios escopados.

Ninguna de estas es necesaria en piloto.

## Referencias

- Bug original cazado: [Nexus PR #37](https://github.com/Dev-Joseluis/Nexus/pull/37) — cross-tenant E2E test.
- Documentación oficial Postgres: https://www.postgresql.org/docs/current/sql-createpolicy.html
- Service de Nexus que implementa el pattern: `apps/api/src/stock-movements/stock-movements.service.ts`.
- Test que verifica el pattern: `apps/api/test/cross-tenant-inventory.flow.test.ts`.

## Lección general

Nunca confíes en una capa de seguridad sin probarla end-to-end. Sprint 2 de Nexus tenía 3 capas (NOBYPASSRLS role + FORCE RLS + RLS policies) y aún así el bug pasó porque ninguna de las 3 cubría la validación de FK. **El cross-tenant E2E test fue la única defensa que lo capturó** antes de producción.
