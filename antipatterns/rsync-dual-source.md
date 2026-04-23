# Antipattern: rsync dual source

## Resumen en 2 líneas

- **El patrón:** dos working copies del mismo repo con sync unidireccional (rsync, watcher, cron) como puente.
- **Por qué es tóxico:** la herramienta de sync decide qué sobrescribir basada en metadata (timestamps, tamaño, hashes parciales), no en contenido semántico. Si una de las copies tiene contenido viejo pero metadata nueva, pisa las correcciones de la otra silenciosamente.

## El setup que parecía razonable

Durante Sprint 0 de Nexus, la arquitectura operativa inicial era:

- **Cowork** editaba archivos en su workspace de Windows: `C:\Users\<user>\Documents\Claude\Projects\<project>\nexus\`
- **Claude Code** editaba el clon local de WSL: `~/code/Nexus/` (filesystem ext4 nativo)
- Un script `sync-to-github.sh` en WSL corría `rsync -a --delete` con exclusiones sensatas (`.git/`, `node_modules/`, `.env`, lockfiles, dirs de build)
- Antes de cada commit: el humano corría el script, inspeccionaba `git status` en WSL, y commiteaba

En papel: razonable. rsync es maduro, el script era ~100 líneas bien documentadas, las exclusiones cubrían los sospechosos obvios, y el workflow era manual (on-demand, no watcher automático).

En la práctica: dos eventos serios de pérdida de trabajo en los primeros 7 PRs del proyecto, documentados abajo.

## Evento 1 — aplaste silencioso del fix de CVE en PR #1

**Contexto:** desarrollo de PR #1 (schema inicial + RLS + primer subagent).

**Secuencia:**

1. Cowork armó `packages/db/package.json` con `drizzle-orm@^0.33.0` — versión que estaba en su snapshot de referencia.
2. Claude Code, al correr `pnpm audit`, detectó advisory GHSA en versiones <0.45 de drizzle-orm. Actualizó a `^0.45.2`.
3. En el mismo PR, Claude Code ajustó el schema de `users` por cambios de better-auth (eliminó columna `passwordHash`, cambió `emailVerified` de timestamp a boolean).
4. Fin de sesión. Al día siguiente, sesión nueva. `sync-to-github.sh` corre como primer paso del flow.
5. rsync trae la copia de Cowork (drizzle 0.33, `passwordHash` presente, `emailVerified` timestamp) sobre la de WSL (drizzle 0.45, schema corregido).
6. CI rojo. `pnpm audit --audit-level=high` tira warning por CVE. Inicialización de better-auth falla por schema mismatch.

**Tiempo invertido en diagnóstico antes de entender qué pasó:** ~40 minutos. El equipo (humano + Cowork + Claude Code) asumió primero que había un bug nuevo en drizzle o en better-auth, hasta que Claude Code notó que las versiones instaladas no coincidían con los fixes commiteados.

## Evento 2 — 17 archivos revertidos simultáneamente en PR #7

**Contexto:** desarrollo de PR #7 (roster de subagents + fixes acumulados de CI).

**Secuencia:**

1. En sesiones previas, Claude Code había fixeado múltiples problemas de CI: cambió `NODE_VERSION` de 20 a 22, migró de `gitleaks-action@v2` (que requería license organizacional) a gitleaks CLI, corrigió sintaxis de `uniqueIndex` en 3 schemas por requisito de Drizzle 0.31+, corrigió el path de dotenv en `migrate.ts`, actualizó versiones en `packages/db/package.json`, y aplicó Prettier a 9 archivos markdown.
2. Cowork, en paralelo, diseñaba los 7 prompts de subagents.
3. Al correr `sync-to-github.sh`, `git status` mostró **17 archivos modificados** simultáneamente, cada uno reintroduciendo un bug ya resuelto:

   - `.github/workflows/ci.yml` volvía a `NODE_VERSION: "20"` y a `gitleaks-action@v2` con license error.
   - `packages/db/package.json` revertía drizzle a `^0.33.0` / drizzle-kit a `^0.24.0`.
   - `packages/db/src/schema/auth.ts`, `memberships.ts`, `tenants.ts` revertían `uniqueIndex` de sintaxis array (0.31+) a sintaxis object (pre-0.31).
   - `packages/db/src/schema/users.ts` re-introducía `passwordHash` column y cambiaba `emailVerified` de boolean a timestamp.
   - `packages/db/drizzle.config.ts` y `migrate.ts` revertían el fix de dotenv path (importante para ESM hoisting).
   - 9 archivos markdown perdían el formato de Prettier aplicado.

4. Recovery requirió: `git reset --soft origin/main`, `git restore --staged` archivo por archivo para descartar los 17 aplastes, re-commitear solo los 6 archivos deliberados, `git push --force-with-lease`.

**Tiempo invertido en este evento:** ~2 horas entre diagnóstico, recovery, y re-validación de CI. En dos iteraciones posteriores de CI fallando por residuales, antes de que todo quedara verde.

Este evento fue el que disparó la decisión de rediseñar el workflow por completo. Se formalizó en PR #8 (sección §6 de CLAUDE.md + deprecación del script).

## Análisis forense — por qué rsync no puede ganar este juego

rsync no tiene bug alguno. Hace exactamente lo que promete hacer: sincronizar un directorio source a un destino, tratando al source como autoridad. El problema es usarlo en un caso de uso **para el que no fue diseñado**: reconciliar dos copies que ambas creen ser autoritativas.

La heurística por default de rsync para decidir "qué sobrescribir":

- Si tamaños son distintos → sync.
- Si tamaños son iguales pero mtimes son distintos → sync el que tiene mtime más reciente.
- Con `-a`, preserva permisos, owner, group, timestamps (archive mode).

La trampa: **mtimes son fácilmente actualizados por operaciones que NO modifican contenido.** Causas documentadas que afectan este caso:

1. **IDE file watchers.** VS Code, IntelliJ, etc. pueden abrir archivos y guardarlos sin cambios (mismo contenido → mtime nuevo).
2. **Herramientas de backup desktop.** OneDrive, Google Drive, Dropbox re-escriben archivos periódicamente — mismo contenido → mtime nuevo.
3. **File sync de extensiones.** Settings Sync de VS Code, extensions que tocan archivos del workspace.
4. **Operaciones de lectura en filesystems raros.** Algunos filesystems (especialmente cross-platform mounts como NTFS vía WSL) tienen semántica imperfecta de `atime` vs `mtime`.
5. **El propio cliente Cowork.** Cuando Cowork lee archivos del workspace para contexto de conversación, en algunos setups el read-lock puede propagar como un write-touch sobre la metadata.

Resultado empírico del setup Nexus: la copia vieja de Cowork terminaba consistentemente con mtimes más recientes que la copia correcta de WSL. rsync "sincronizaba" según su contrato — pero el contrato no era lo que queríamos.

## Lecciones generalizables para proyectos nuevos

### 1. Evitá sync automático entre dos copies del mismo source truth

Si dos entornos necesitan acceso al mismo código, los caminos aceptables son (en orden de preferencia):

- **Ideal**: un solo filesystem, ambos entornos apuntan ahí (UNC path si el OS lo permite, shared volume, network mount).
- **Aceptable**: un entorno tiene el repo; el otro tiene acceso de lectura vía API (git remote, GitHub raw URLs, API de storage).
- **Inaceptable**: dos working copies locales con sync automático.

### 2. Si tenés que hacer sync, que sea explícito y auditable

Scripts on-demand (corridos manualmente) son menos peligrosos que watchers automáticos. Pero incluso los on-demand fallan cuando el operador no revisa cuidadosamente cada `git status` resultante. La atención humana es finita; después de 3 syncs "limpios" seguidos, la tentación de confiar sin revisar crece. El 4to sync es el que te traiciona.

### 3. Timestamps no son evidencia de contenido

Para decidir "cuál versión es la buena", comparar hashes de contenido (como hace `unison`) es correcto. Comparar mtimes (como hace rsync por default) es una heurística frágil.

Si realmente necesitás un sync robusto, `unison` con hashing habilitado es mejor que rsync. Pero introduce su propia complejidad (conflict resolution interactiva, estado persistente entre runs), y **sigue siendo trabajar alrededor del problema en vez de eliminarlo**.

### 4. Fallar silencioso es peor que fallar ruidoso

El peor aspecto del setup con rsync no fue el aplaste en sí — los bugs aparecen en cualquier proyecto. Fue que los aplastes eran **silenciosos**. El script imprimía "sync OK", `git status` mostraba archivos modificados, pero "modificados" es ambiguo: ¿son cambios deliberados del operador? ¿o son reversiones inyectadas por la máquina?

En ausencia de otra señal, el operador confía en su memoria. "Yo no toqué `ci.yml`... pero si aparece modified, será que sí que lo toqué". Con el tiempo esa confianza se erosiona hasta que un día el operador revisa cuidadosamente y descubre la traición silenciosa. El costo emocional de ese momento es desproporcionado al costo técnico de los archivos revertidos.

### 5. El rediseño del flujo es más barato que el recovery repetido

Una vez que identificaste un patrón como tóxico, no lo iteres — reemplazalo. Cada recovery individual se siente "manejable" (2 horas, una vez). Pero el costo real se mide acumulando: 2 horas × cada PR del año + la erosión de confianza en el flujo + los fixes laterales que se filtran (porque se asume que el rsync puede estar mintiendo, y eso contamina cómo pensás sobre otros problemas).

El rediseño total (eliminar sync, establecer workflow explícito) costó ~4 horas de diseño + ~2 horas de implementación. Pay-off: cero eventos adicionales en los PRs siguientes (#8, #10, #12, #14 y más).

## Checklist de prevención para proyectos nuevos

Al arrancar un proyecto nuevo con múltiples entornos de trabajo, antes de escribir una línea de código:

- ¿Cuántas copies del código van a existir? Si hay más de una, ¿por qué?
- Si hay más de una, ¿cuál es autoritativa? ¿Lo podés documentar en el README en una sola frase sin cláusulas "excepto cuando..."?
- ¿Cómo fluye la información entre copies? Si la respuesta es "rsync", "watcher", o "sync script", **pará y reconsiderá**.
- ¿Qué pasa si ambos entornos editan el mismo archivo en ventanas solapadas? ¿El sistema detecta el conflicto o el último en sync gana silenciosamente?
- ¿Hay un patrón de "unified view" posible? (UNC path, shared volume, accessed via `git remote` solo)

Si cualquiera de estas preguntas revela un patrón similar al rsync-dual-source, rediseñá antes de empezar a desarrollar. Corregir después cuesta mucho más.

## Referencias

- `workflow/cowork-claude-code-separation.md` — el patrón que reemplaza este antipattern en proyectos con Cowork + Claude Code.
- `workflow/git-surgery.md` — cómo recuperar cuando el daño ya está hecho (reset + force-push-with-lease, `git restore --staged`, detección de aplastes vía `git diff`).
- Commits relevantes en Nexus:
  - PR #1 — primer evidencia del aplaste (fix de CVE drizzle revertido).
  - PR #7 — el evento de 17 aplastes simultáneos que disparó el rediseño.
  - PR #8 — deprecación formal de `sync-to-github.sh` con guard `exit 1` por default.
