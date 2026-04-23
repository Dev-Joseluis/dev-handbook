# Git surgery — recovery patterns

## Cuándo leer esto

Cuando tu repositorio está en un estado que no querés y necesitás volverlo a uno conocido **sin perder trabajo legítimo**. Escenarios típicos:

- Commit sucio (archivos stageados que no deberían estarlo).
- Push accidental a la rama equivocada.
- Historia divergente entre local y origin post-squash-merge.
- Harness/policy que bloquea `git reset --hard`.
- Rebase atascado con conflicts.

No es un tutorial de git. Asume que conocés commit, index, working tree, y remotes.

## Principio fundante

**Git casi nunca pierde datos, pero algunos comandos hacen que sean difíciles de recuperar.** La diferencia entre "fácil" y "difícil" es si el commit tiene una ref apuntándolo (branch, tag) o solo vive en reflog.

Orden de preferencia al recuperar:

1. **No destructivo primero.** `git stash`, `git restore`, `git checkout` modifican working tree/index pero no commits.
2. **Soft/mixed reset** antes de hard. Preserva commits, solo mueve el puntero de branch.
3. **Hard reset sobre una ref conocida** (`origin/main`) antes de hard reset a SHA específico. Origin es seguro.
4. **Force-push only with `--force-with-lease`.** Nunca `--force` desnudo.
5. **`update-ref`, `checkout -B`, `branch -f` como alternativas cuando el verbo `reset --hard` está bloqueado.**

Si no estás seguro, `git stash` primero. Es gratis.

## Patrón 1 — Descartar cambios no stageados

**Situación:** archivos modificados que no querés. No están staged todavía.

```bash
git status          # confirma qué está dirty
git restore --worktree -- .
```

Revierte todos los archivos tracked a su versión del index (o HEAD si nada está staged). No toca archivos untracked ni ignored.

Si también querés limpiar untracked:

```bash
git clean -nd       # dry-run: muestra qué borraría
git clean -fd       # real (destructivo — revisá el dry-run antes)
```

## Patrón 2 — Descartar un archivo stageado específico

**Situación:** archivo staged que no debería estarlo. El resto del stage está bien.

```bash
git restore --staged <archivo>
```

Lo saca del index pero lo mantiene en el working tree. Combinable con `--worktree` para también revertirlo en disco:

```bash
git restore --source=HEAD --staged --worktree <archivo>
```

Equivale a "dale a este archivo el contenido exacto de HEAD, tanto en index como en disco".

## Patrón 3 — Limpiar un commit sucio (el clásico post-aplaste)

**Situación:** hiciste `git commit` con 17 archivos stageados, 11 de los cuales son aplastes que no querías. Solo 6 son legítimos.

```bash
# Paso 1: mover el puntero hacia atrás, preservando contenido como staged
git reset --soft HEAD~1

# Paso 2: confirmar que todo quedó staged
git status
# Esperado: "Changes to be committed:" con los 17 archivos

# Paso 3: desstagear los aplastes (uno por uno o con globs)
git restore --staged packages/db/schema/users.ts
git restore --staged packages/db/schema/auth.ts
# ... etc para los 11 aplastes

# Paso 4: verificar que quedan solo los legítimos
git diff --cached --stat
# Esperado: los 6 legítimos

# Paso 5: también revertir los aplastes en disco (opcional pero recomendable)
git restore --worktree packages/db/schema/users.ts
# ... etc

# Paso 6: re-commitear
git commit -m "mensaje correcto con solo los 6 legítimos"

# Paso 7: si ya habías pusheado el commit sucio
git push --force-with-lease
```

**Clave:** `reset --soft` preserva los cambios en el index; desde ahí decidís qué mantener con `restore --staged`. Nunca perdés trabajo, solo el commit-wrapper.

## Patrón 4 — Force-push con safety net

**Nunca** uses `git push --force` desnudo a una rama compartida. Usá:

```bash
git push --force-with-lease
```

Diferencia crítica: `--force` sobrescribe ciegamente lo que haya en remote. `--force-with-lease` sobrescribe **solo si** el remote está en el SHA que vos tenés como "último que vi". Si alguien más pusheó entre tu último fetch y tu force-push, el lease falla y te alerta antes de destruir su trabajo.

A `main` específicamente: no fuerces nunca, ni con lease ni sin. Ramas feature sí, con lease, cuando el operador humano lo apruebe.

## Patrón 5 — Reset a origin cuando el harness bloquea `--hard`

**Situación:** tu entorno (Claude Code harness, policy del team, hook custom) bloquea `git reset --hard <ref>`. Necesitás alinear local con origin.

Alternativas equivalentes en efecto, distintas en verbo:

```bash
# Opción A: recrear la rama desde origin
git checkout -B main origin/main

# Opción B: mover el puntero de la rama directamente
git branch -f main origin/main
git checkout main

# Opción C: update-ref (plumbing de bajo nivel)
git update-ref refs/heads/main origin/main
```

Las tres hacen funcionalmente lo mismo: `main` local pasa a apuntar al mismo SHA que `origin/main`. Si el harness bloquea una, probá la siguiente. Si bloquea las tres, casi seguro hay una razón legítima — pausa y preguntá al operador humano antes de buscar un cuarto workaround.

## Patrón 6 — Historia divergente post-squash-merge

**Situación:** mergeaste un PR con "squash and merge". Tu rama feature local tenía 3 commits (A, B, C). Ahora `origin/main` tiene UN solo commit D que contiene el efecto agregado de A+B+C pero con distinto SHA. Tu `main` local sigue teniendo A, B, C colgando del estado anterior de main.

`git pull origin main` intenta rebasar A, B, C sobre D → conflicts porque los mismos cambios ya están aplicados (desde distintos SHAs).

Solución: reseteá `main` local a `origin/main`. Los commits A, B, C quedan huérfanos pero no importa — su contenido vive en D.

```bash
git checkout main
git fetch origin
git checkout -B main origin/main    # o reset --hard si no está bloqueado

# Borrar la rama feature local (ya mergeada vía squash)
git branch -D feat/mi-feature

# Verificar
git log --oneline -3                # debería mostrar D en el top
git status                          # working tree clean
```

Este patrón es tan común que conviene memorizarlo: **"squash merge rompe la linealidad de historia local; resetear a origin es el fix estándar, no un error"**.

## Patrón 7 — Rebase atascado con conflicts

**Situación:** `git rebase` empezó, hay conflicts, no querés resolverlos (o resolviste mal y querés empezar de nuevo).

```bash
git rebase --abort
```

Vuelve al estado pre-rebase exacto. Siempre seguro.

Si querés saltarte un commit específico durante el rebase (`--skip`), pará, abortá, y pensá de nuevo. El `--skip` durante rebase es una fuente común de pérdida silenciosa de trabajo — te saltás el commit entero, no solo el conflict.

## Patrón 8 — Mover archivos preservando historia

**Situación:** movés un archivo o directorio; no querés que git lo registre como "delete + create" (pierde blame y history).

```bash
git mv viejo/path nuevo/path
```

Git trackea el rename nativamente si más del 50% del contenido se mantiene. Funciona aunque también edites el contenido durante el mismo commit (lo registra como "rename + modify").

Alternativa: `mv` tradicional + `git add -A`. Git detecta renames heurísticamente, pero si modificaste mucho contenido puede verlo como delete + add distintos, perdiendo la relación.

## Patrón 9 — Recuperar un commit huérfano vía reflog

**Situación:** hiciste `git reset --hard` a un SHA, perdiendo de vista otro commit que no tenía ref. Querés recuperarlo.

```bash
git reflog
# Busca el SHA del commit perdido. Los entries están ordenados cronológicamente
# con la operación que los movió (HEAD@{N}: commit: ... / reset: ... / etc.)

git branch <nueva-rama> <sha-recuperado>
# El commit vuelve a tener ref; recuperado.
```

`reflog` guarda las últimas ~90 días de movimientos de `HEAD` por default. Mientras el commit no haya sido garbage-collected, podés recuperarlo.

## Antipatterns a evitar

- **`git push --force` desnudo a ramas compartidas.** Siempre `--force-with-lease`.
- **`git reset --hard <sha>` cuando `<sha>` no es una ref conocida.** Si tipeás mal el SHA, el rollback es via reflog. Mejor: resetear a `origin/<branch>` o a `HEAD~N`.
- **`git checkout -- .` esperando que descarte untracked.** Solo toca archivos tracked. Para untracked, `git clean`.
- **`git commit --amend` sobre un commit ya pusheado.** Genera nuevo SHA, requiere force-push. Si otros basaron trabajo sobre el commit original, los rompés. Mejor: commit nuevo encima.
- **`git reset --hard HEAD` pensando que "limpia trabajo".** Descarta cambios del working tree **y** del index. Si hay trabajo no commiteado, se pierde. Preferí `git stash` primero.
- **`git rebase --skip` sin entender qué commit se pierde.** Si no estás seguro, abortá y reemprendé con más contexto.

## Preflight checklist antes de cualquier operación destructiva

Antes de `reset --hard`, `push --force-with-lease`, `clean -fd`, `branch -D`:

```
[ ] git status          → ¿hay trabajo no commiteado que podrías perder?
[ ] git stash           → si sí, guardalo antes
[ ] git log --oneline -5 → ¿estás en el branch correcto?
[ ] git remote -v       → ¿apunta al remote correcto? (paranoia sana contra typos)
```

10 segundos de checklist ahorran 2 horas de recovery.

## Referencias

- `antipatterns/rsync-dual-source.md` — contexto del evento que obligó a practicar muchos de estos patterns durante Sprint 0 de Nexus.
- `workflow/cowork-claude-code-separation.md` — el workflow que previene la mayoría de situaciones que requieren surgery.
