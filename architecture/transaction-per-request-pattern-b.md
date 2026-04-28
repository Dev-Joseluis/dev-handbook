# Pattern: transaction-per-request en NestJS interceptors (Pattern B)

## Resumen en 2 líneas

- **Pattern B** (recomendado): la transaction wrappea el handler vía Promise lifecycle. Si el handler tira, la Promise rechaza, drizzle/db rollbackea automáticamente.
- **Pattern A** (antipattern sutil): try/catch interno que captura excepciones del handler sin re-throw → la transaction commitea con datos parciales aunque el cliente reciba 500.

## El antipattern (Pattern A) — bug latente

```typescript
// MAL — Pattern A. Funciona en happy path, falla silencioso en errores.
async intercept(context: ExecutionContext, next: CallHandler) {
  return new Observable(observer => {
    this.db.transaction(async (tx) => {
      try {
        await setTenantContext(tx, tenantId);
        const result = await firstValueFrom(next.handle());
        observer.next(result);
        observer.complete();
        // ↓ tx.commit() ocurre acá implícitamente cuando
        //   la async function retorna
      } catch (err) {
        observer.error(err);
        // ↑ El catch propaga el error al observer (cliente recibe 500),
        //   PERO la async function retorna `undefined` después.
        //   drizzle ve "función resolvió OK" → COMMIT.
        //   Datos parciales del handler quedan persistidos.
      }
    });
  });
}
```

El bug no es obvio leyendo el código. Aparece solo cuando:

1. El handler hace múltiples INSERTs.
2. Falla en algún punto intermedio.
3. El catch atrapa el error, lo propaga al observer.
4. La async function de drizzle resuelve normalmente.
5. drizzle commit. **Los INSERTs hechos antes del fallo quedan committed.**
6. El cliente recibe HTTP 500 pensando que toda la operación fue rolleada.

## El pattern correcto (Pattern B)

```typescript
// BIEN — Pattern B. La transaction lifecycle dictada por el flujo natural.
async intercept(context: ExecutionContext, next: CallHandler) {
  return new Observable(observer => {
    this.db.transaction(async (tx) => {
      await setTenantContext(tx, tenantId);
      // expose tx via request.tx o AsyncLocalStorage
      request.tx = tx;

      // El handler corre. Si throwea, propaga naturalmente.
      // Si retorna OK, el resultado se propaga.
      return await lastValueFrom(next.handle());
    })
      .then(result => {
        observer.next(result);
        observer.complete();
      })
      .catch(err => {
        // drizzle YA hizo rollback antes de llegar acá.
        // El observer solo refleja el error al cliente.
        observer.error(err);
      });
  });
}
```

Cambios clave:

- **Sin try/catch interno.** El throw del handler propaga naturalmente al async fn de drizzle.
- **drizzle.transaction maneja commit/rollback** según resolve/reject de la async function.
- **El Observable solo refleja el resultado final** vía `.then`/`.catch` afuera de la transaction.

## Test de regresión obligatorio

Pattern A es seductor — cualquier refactor futuro puede re-introducirlo. Protección: test de unit que **verifica el rollback contract directamente**:

```typescript
it("[CRITICAL] propagates handler throw through transaction callback", async () => {
  const dbMock = createMockDb(); // mock que registra commit/rollback calls
  const interceptor = new TenantContextInterceptor(dbMock /* ... */);

  const handler = {
    handle: () => throwError(() => new Error("handler boom")),
  };

  await expect(
    firstValueFrom(interceptor.intercept(mockContext, handler)),
  ).rejects.toThrow("handler boom");

  // Lo que importa: rollback fue llamado, NO commit
  expect(dbMock.transactions[0].rolledBack).toBe(true);
  expect(dbMock.transactions[0].committed).toBe(false);
});
```

Este test pasa con Pattern B. **Falla con Pattern A** porque el catch interno se traga la excepción → drizzle ve "OK" → commit. El test es la red de seguridad permanente del pattern.

## Por qué `lastValueFrom` y no `firstValueFrom`

Para handlers REST con única emisión, son equivalentes. Pero `lastValueFrom` es el conservative default:

- `firstValueFrom`: toma la primera emisión y completa. Si en el futuro algún handler emite múltiples values (streaming, server-sent events), corta data.
- `lastValueFrom`: espera completion y toma el último value. Funciona idéntico para single-emit, robusto para multi-emit.

## Cuándo aplica

Cualquier interceptor (o middleware equivalente en otros frameworks) que envuelve el handler en una transaction database. Patterns equivalentes:

- NestJS Interceptor + drizzle transaction (este caso).
- Express middleware + Prisma `$transaction`.
- Hono context + Kysely transaction.
- Fastify hooks + TypeORM transaction.

Todos sufren del mismo bug si usan Pattern A. La forma del fix es la misma: dejar que el throw del handler propague al async fn de la DB library.

## Trade-off

Pattern B requiere que el handler exponga su result vía Observable o Promise — el Interceptor convierte. Si el handler es síncrono, la conversión es trivial. Si es asíncrono con efectos de side (logs, metrics), esos también deben respetar el flow.

Algunos frameworks (Express con vanilla middleware) no tienen este pattern naturalmente — requieren wrappers manuales. NestJS lo facilita via Interceptors + RxJS.

## Referencias

- Implementación de Nexus: `apps/api/src/tenant/tenant-context.interceptor.ts`
- Test de regresión: `apps/api/src/tenant/tenant-context.interceptor.test.ts`
- ADR-010 de Nexus: `docs/adr/010-tenant-context-interceptor.md`
- PR donde se adoptó: [Nexus PR #33](https://github.com/Dev-Joseluis/Nexus/pull/33)

## Lección general

En async error handling, el silent commit es el bug peor — los datos quedan en estado inconsistente sin que nadie sepa. Patterns que delegan lifecycle a la library (drizzle, Prisma, etc.) son más robustos que patterns que mezclan try/catch propios con lifecycle ajeno. Cuando el throw es el contract, **respetá el throw**.
