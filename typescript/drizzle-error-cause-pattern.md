# Pattern: Drizzle wrappa errores con `.cause` — chequear ambos paths al mapear errcodes

## Resumen en 2 líneas

- Cuando un service NestJS necesita distinguir errcodes específicos de Postgres (unique violation, FK violation, etc.) para mapear a HTTP status codes, los errores de Drizzle pueden estar wrappados con la causa real en `err.cause`.
- Helpers de detección deben chequear **ambos paths**: `err.code` (caso plano) y `err.cause.code` (caso wrappado).

## El descubrimiento

Durante Nexus PR #36 (items CRUD), el primer caller de `isPgUniqueViolation` solo chequeaba `err.code` directo. Los 2 tests de SKU duplicate fallaron con HTTP 500 porque el error real estaba en `err.cause.code = '23505'`.

Drizzle (en versiones recientes) wrappa los errores de la library subyacente (postgres-js, pg, etc.) con un Error genérico que tiene message tipo `"Failed query"` y la causa nested.

```typescript
// Error que tira postgres-js cuando hay unique violation:
{
  code: '23505',
  detail: 'Key (tenant_id, sku)=(...) already exists.',
  constraint_name: 'items_tenant_id_sku_unique',
  // ...
}

// Cuando Drizzle lo wrappa, queda:
{
  message: 'Failed query: ...',
  cause: {
    code: '23505',
    detail: '...',
    constraint_name: '...',
    // ...
  }
}
```

## El helper canónico

```typescript
function isPgUniqueViolation(err: unknown): boolean {
  // Chequear ambos paths — Drizzle puede o no wrappear según el call site
  const code = (err as any)?.code ?? (err as any)?.cause?.code;
  return code === "23505";
}

function isPgForeignKeyViolation(err: unknown): {
  isViolation: boolean;
  constraint?: string;
} {
  const code = (err as any)?.code ?? (err as any)?.cause?.code;
  const constraint =
    (err as any)?.constraint_name ?? (err as any)?.cause?.constraint_name;
  return {
    isViolation: code === "23503",
    constraint,
  };
}
```

## Por qué Drizzle wrappa

Drizzle abstrae diferentes drivers (postgres-js, pg, neon, postgres-cloudflare). El wrapper unifica la API de errores para que código consumer no dependa del driver específico. La causa real (con shape del driver) queda accesible vía `.cause` para casos donde el consumer necesita errcodes nativos.

## Cuándo aplica

Cualquier service que mapea Postgres errcodes a HTTP status codes:

- Unique violation (`23505`) → 409 CONFLICT
- FK violation (`23503`) → 404 NOT_FOUND o 422 UNPROCESSABLE_ENTITY
- CHECK constraint violation (`23514`) → 422 UNPROCESSABLE_ENTITY
- NOT NULL violation (`23502`) → 400 BAD_REQUEST (probablemente Zod ya lo cazó antes)
- Exclusion constraint violation (`23P01`) → 409 CONFLICT

Pattern recurrente: try/catch alrededor de la operación Drizzle, dentro del catch checkear errcodes específicos.

## Cuándo extraer a un helper compartido

Rule of three: **al tercer caller con error mapping, extraer a `apps/api/src/common/db-errors.ts`.**

Antes de eso, helpers locales en cada service. Razón: extraer prematuramente con 1-2 callers genera abstracciones que pueden no calzar con el 3er caller cuando aparezca.

## Caveat sobre `as any`

El `(err as any)?.code` es type-cast inseguro. Alternativa estricta: definir un type guard exhaustivo:

```typescript
interface PgErrorLike {
  code?: string;
  detail?: string;
  constraint_name?: string;
  cause?: PgErrorLike;
}

function asPgError(err: unknown): PgErrorLike | null {
  if (typeof err !== "object" || err === null) return null;
  return err as PgErrorLike;
}

function isPgUniqueViolation(err: unknown): boolean {
  const e = asPgError(err);
  return e?.code === "23505" || e?.cause?.code === "23505";
}
```

Esto es ~5 líneas más por helper pero sin `any`. Para piloto, el `as any` es aceptable. Para hardening pre-Pro, considerar el type guard.

## Referencias

- Bug original cazado: [Nexus PR #36](https://github.com/Dev-Joseluis/Nexus/pull/36) — items CRUD, primer caller real del helper.
- Helpers de Nexus: `apps/api/src/items/items.service.ts` (isPgUniqueViolation), `apps/api/src/stock-movements/stock-movements.service.ts` (isPgForeignKeyViolation).
- Postgres error codes oficiales: https://www.postgresql.org/docs/current/errcodes-appendix.html

## Lección general

Library wrappers cambian la shape de errores entre versiones. Cuando un service depende de una errcode específico, **escribir el detection helper de manera defensiva** chequeando paths plurales — el caso wrappado de hoy puede ser el caso plano de mañana, o viceversa.
