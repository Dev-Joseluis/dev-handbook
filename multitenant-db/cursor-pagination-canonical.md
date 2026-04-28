# Pattern: Cursor pagination canónica — simple y compuesta

## Resumen en 2 líneas

- Cursor pagination escala mejor que offset y es **estable bajo inserts/deletes concurrentes**, pero requiere encoding consistente del cursor.
- Un helper canónico bien diseñado soporta cursor simple (`{ id }`) y compuesto (`{ occurredAt, id }`) con la misma API — el caller solo cambia el shape del anchor.

## Por qué cursor sobre offset

Offset pagination (`?page=N&limit=20`):

- ✅ Simple, frontend muestra page numbers.
- ❌ Inserts/deletes durante navegación → skip o duplicados.
- ❌ Lento sobre datasets grandes (`OFFSET 100000` = scan de 100k rows).

Cursor pagination (`?cursor=<base64>&limit=20`):

- ✅ Inmune a inserts/deletes — cursor apunta a row específica, no posición.
- ✅ Performante con índice apropiado (`ORDER BY ... WHERE col > cursor` = index scan).
- ✅ Migración a streaming pagination es trivial.
- ❌ No hay "página 47" — solo siguiente/anterior.
- ❌ Más código que offset (~20-30 líneas extra una vez, después se reusa).

Para SaaS multi-tenant donde tenants pueden tener listados de miles a millones de rows, cursor es el default correcto.

## Helper canónico — interface mínima

```typescript
// apps/api/src/common/pagination.ts

export interface CursorPaginationParams {
  cursor?: string; // base64-encoded JSON anchor
  limit?: number; // default 20, max 100
}

export interface CursorPaginatedResult {
  data: T[];
  nextCursor: string | null;
}

export function encodeCursor(anchor: unknown): string {
  return Buffer.from(JSON.stringify(anchor)).toString("base64");
}

export function decodeCursor(cursor: string): T {
  return JSON.parse(Buffer.from(cursor, "base64").toString("utf8")) as T;
}

export async function paginate(
  rows: T[],
  limit: number,
  toAnchor: (row: T) => unknown,
): Promise<CursorPaginatedResult> {
  const hasMore = rows.length > limit;
  const data = hasMore ? rows.slice(0, limit) : rows;
  const nextCursor =
    hasMore && data.length > 0
      ? encodeCursor(toAnchor(data[data.length - 1]))
      : null;
  return { data, nextCursor };
}
```

Diseño clave: `paginate` recibe los rows ya queryados (con `LIMIT + 1`), el limit, y un callback `toAnchor` que dice qué shape tiene el cursor. El caller controla la query y el shape del anchor.

## Cursor simple — pattern de uso

```typescript
// suppliers.service.ts
async list(params: CursorPaginationParams, tx: AppTransaction) {
  const limit = Math.min(params.limit ?? 20, 100);
  const anchor = params.cursor
    ? decodeCursor(params.cursor)
    : null;

  const rows = await tx
    .select()
    .from(schema.suppliers)
    .where(
      and(
        isNull(schema.suppliers.deletedAt),
        anchor ? lt(schema.suppliers.id, anchor.id) : undefined,
      ),
    )
    .orderBy(desc(schema.suppliers.id))
    .limit(limit + 1);

  return paginate(rows, limit, (row) => ({ id: row.id }));
}
```

Ordering por `id DESC` da consistencia (UUIDs ordenan deterministically). Para suppliers/items no hay otra columna semántica para ordenar — el id es suficiente.

## Cursor compuesto — pattern de uso

Para entidades con ordering temporal (stock_movements ordenadas por `occurred_at`), el cursor necesita ambas columnas para estabilidad bajo timestamps duplicados:

```typescript
// stock-movements.service.ts
async list(params: CursorPaginationParams, tx: AppTransaction) {
  const limit = Math.min(params.limit ?? 20, 100);
  const anchor = params.cursor
    ? decodeCursor(params.cursor)
    : null;

  const rows = await tx
    .select()
    .from(schema.stock_movements)
    .where(
      anchor
        ? or(
            lt(schema.stock_movements.occurredAt, new Date(anchor.occurredAt)),
            and(
              eq(schema.stock_movements.occurredAt, new Date(anchor.occurredAt)),
              lt(schema.stock_movements.id, anchor.id),
            ),
          )
        : undefined,
    )
    .orderBy(
      desc(schema.stock_movements.occurredAt),
      desc(schema.stock_movements.id),
    )
    .limit(limit + 1);

  return paginate(rows, limit, (row) => ({
    occurredAt: row.occurredAt.toISOString(),
    id: row.id,
  }));
}
```

La WHERE clause compuesta dice: "rows después del anchor" en orden lexicográfico de `(occurredAt, id)`. Drizzle helpers (`or`, `and`, `lt`, `eq`) en lugar de raw SQL `(occurred_at, id) < (?, ?)` dan type safety.

## Indices que necesita

Para que el query plan sea óptimo, los índices deben matchear el ORDER BY:

```sql
-- Para cursor simple
CREATE INDEX idx_suppliers_id_desc ON suppliers (id DESC);

-- Para cursor compuesto
CREATE INDEX idx_stock_movements_occurred_id_desc
  ON stock_movements (occurred_at DESC, id DESC);

-- Multi-tenant: anteponer tenant_id (RLS filtra por ahí primero)
CREATE INDEX idx_stock_movements_tenant_occurred_id_desc
  ON stock_movements (tenant_id, occurred_at DESC, id DESC);
```

Verificar con `EXPLAIN ANALYZE` que el query usa el índice (Index Scan, no Seq Scan + Sort).

## Cuándo NO usar cursor pagination

- Endpoints que devuelven < 20 rows típicamente (no hay paginación necesaria).
- UIs que necesitan "saltar a página N" (admin tables, reports). Para eso, offset es lo correcto a pesar de las desventajas — UX manda.
- Datasets que cambian raramente (configuraciones, lookups). Offset alcanza.

## Referencias

- Helper de Nexus: `apps/api/src/common/pagination.ts`
- Caller con cursor simple: `apps/api/src/suppliers/suppliers.service.ts` (PR #35)
- Caller con cursor compuesto: `apps/api/src/stock-movements/stock-movements.service.ts` (PR #37)

## Lección general

Cuando un pattern recurre 3+ veces con variaciones, **un helper bien diseñado abstrae lo común y deja que el caller especifique lo variable**. El helper de Nexus soporta simple + compuesto sin modificación porque el `toAnchor` callback es genérico — eso es el core del diseño.
