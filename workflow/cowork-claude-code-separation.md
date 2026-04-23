# Workflow Cowork ↔ Claude Code

## Resumen en 3 líneas

- **Cowork** (cliente desktop, Windows): arquitecto y redactor. NO escribe al filesystem del repo.
- **Claude Code** (CLI en WSL): constructor. Única fuente de verdad del código que llega a `main`.
- **Humano**: puente deliberado. Copia código desde Cowork hacia Claude Code; pega output desde Claude Code hacia Cowork.

## El problema que esta división resuelve

El primer intento de trabajar con los dos entornos usaba una arquitectura intuitiva pero rota: Cowork editaba archivos en su workspace de Windows (`C:\Users\...\nexus\`), y un script `sync-to-github.sh` en WSL corría `rsync` para traer esos cambios al clon local (`~/code/Nexus/`) antes de commitear. El rsync era unidireccional (Cowork → WSL) y con `--delete`.

El problema de esa arquitectura es estructural, no de implementación: **dos working copies compitiendo por ser source of truth**. Claude Code editaba archivos en WSL para corregir bugs (ejemplo: CVE en drizzle, configs de CI, schemas de auth). Esas correcciones vivían solo en WSL hasta el próximo rsync. El rsync, al ver que Cowork tenía versiones "más recientes según timestamps" (pero desactualizadas en contenido), pisaba las correcciones de WSL con las versiones viejas de Cowork.

Resultado documentado: dos PRs consecutivos (PR #1 y PR #7 del repo Nexus) requirieron `git reset --soft origin/main` + `git push --force-with-lease` para limpiar reversiones silenciosas. En PR #7 específicamente, el rsync revirtió **17 archivos** en un solo sync — entre ellos la versión de drizzle parcheada contra un CVE, el schema de `users` con la columna correcta de `passwordHash`, y el `ci.yml` con Node 22. Cada reversión fue silenciosa: no hubo error, no hubo warning, solo el CI rojo diciendo que problemas ya resueltos habían vuelto.

Postmortem completo en `antipatterns/rsync-dual-source.md`.

La conclusión no fue "mejor rsync" — fue **eliminar la fuente del conflicto**. Si hay un solo source of truth, no hay conflicto posible.

## Los tres roles

### Cowork — cliente desktop, Windows

**Rol:** arquitecto y redactor. **No escribe al filesystem del repo.**

**Responsabilidades:**

- Diseño de features, ADRs, razonamiento sobre trade-offs.
- Revisión de PRs con lectura cuidadosa de diffs.
- Escritura de código en **bloques de chat** formateados para copiar manualmente.
- Lectura de código actual: siempre desde WSL (pegado por el humano) o desde `https://raw.githubusercontent.com/<org>/<repo>/<branch>/<path>`. **Nunca** desde el workspace local — está desactualizado y no se sincroniza con Claude Code.
- Discusión con el humano sobre prioridades, alternativas, trade-offs de arquitectura.

**Lo que NO hace:**

- NO edita archivos en el filesystem con intención de que lleguen al repo.
- NO ejecuta `git`, `pnpm`, `docker` o comandos que modifiquen el estado del proyecto.
- NO asume que su workspace local refleja el estado real del repo.

### Claude Code — CLI en WSL (Ubuntu)

**Rol:** constructor. **Única fuente de verdad del código.**

**Responsabilidades:**

- Ejecuta todos los comandos de build: `git`, `pnpm`, `docker`, migrations, tests, dev server.
- Edita archivos en el repo con las tools `Edit` y `Write`.
- Crea commits y los pushea directo a feature branches.
- Abre PRs vía `gh pr create`.
- Monitorea CI con `gh pr checks <N> --watch`.
- Valida que todo pase verde antes de pedir squash-merge al humano.
- Reporta fallas de CI con los logs del job rojo para que el humano (o Cowork) diagnostique.

**Lo que NO hace:**

- NO lee del workspace de Cowork (ubicación típica `/mnt/c/Users/.../nexus/`). Ignora esa copia completamente.
- NO hace merge a `main` directo — pasa por PR + branch protection.
- NO usa `git push --force` a `main`. A ramas feature sí, con `--force-with-lease` y solo cuando el humano lo autoriza.

### Humano — el puente deliberado

**Rol:** copiar información entre entornos con atención consciente.

**Responsabilidades:**

- Copia bloques de código desde Cowork hacia Claude Code cuando Cowork propone una implementación.
- Pega output desde Claude Code hacia Cowork cuando aparece un error, log, diff, o decisión que necesita review.
- Toma decisiones estratégicas en Cowork (qué priorizar, qué postergar, qué escalar a ADR).
- Aprueba squash-merges desde la UI de GitHub.
- Interviene cuando algo se vuelve ambiguo o se desvía del patrón — por ejemplo, si un LLM pide hacer algo que viola una red line, el humano lo corrige antes de que se ejecute.

**El humano NO es un canal mudo.** La tentación de automatizar el puente (hacer que Cowork escriba directo al filesystem, o que Claude Code lea desde el workspace de Cowork) siempre existe. Resistirla es lo que hace que esta arquitectura funcione. La fricción manual es deliberada: cada copy-paste es una oportunidad de review. Un bug que entra por copy-paste manual entra porque **vos lo miraste y no lo viste**, no porque un sistema silenciosamente lo inyectó.

## Protocolo copy-paste en la práctica

### Cuando Cowork propone código

Cowork escribe el código en un bloque con tres backticks en el chat. Vos seleccionás, copiás, y pegás a Claude Code con un mensaje del tipo:

> Pegale esto textualmente: [código]

Claude Code interpreta el código, aplica los `Edit` o `Write` necesarios, y reporta resultado. Si el código es largo, Claude Code usa `cat > archivo.ext << 'EOF'` para escribir en un solo shot (el HEREDOC con `'EOF'` previene interpolación de shell).

### Cuando Claude Code reporta output

Claude Code imprime logs, errores, diff stats, resultados de tests. Vos copiás los fragmentos relevantes (típicamente los últimos 30-50 líneas de un log de falla, o el diff stat completo de un commit) y pegás en Cowork para que razone sobre ellos.

No tenés que pegar todo — usá tu criterio. Si el log tiene 500 líneas de exitoso + 10 de error al final, pegás solo las 10 del error más 5 de contexto. Cowork pide más si necesita.

### Cuando el flujo se rompe

Señales de que algo se desvió del patrón (y cómo corregir):

- **Cowork te dice "voy a crear el archivo".** ERROR — Cowork no crea archivos del repo. Pedile que te dé el contenido como bloque de código para que vos lo copies a Claude Code.
- **Claude Code te pregunta "¿querés que lea el workspace de Cowork?".** ERROR — NO. Pedile que lea desde WSL (`~/code/Nexus/`) o desde la URL raw de GitHub.
- **Aparece un `rsync` o un script similar en una propuesta.** ALERT — la razón de ser del handbook es que esto no vuelva. Rechazalo y escalá a discusión.
- **Un diff inexplicable aparece en `git status`.** INVESTIGÁ antes de stagear. Probablemente hay un file watcher, un extension de IDE, o un sync residual escribiendo por su cuenta. Ver `workflow/git-surgery.md` para recovery patterns.

## Red lines (inviolables)

- **Cowork NUNCA escribe al workspace local** con intención de sincronizar al repo. Si necesita artifacts persistentes (docs, diagramas), los emite como artifacts del chat o como bloques de código para copiar.
- **Claude Code NUNCA lee de `/mnt/c/...`** ni de ningún otro mount del workspace de Cowork. Solo del filesystem WSL nativo.
- **No resucitar scripts de sync automático** (rsync, watchers, bridges) sin un ADR firmado que justifique romper la separación y describa los mitigations.
- **El humano NO automatiza su rol de puente.** Si la fricción del copy-paste duele, el remedio no es "hacer que una máquina lo haga por mí" — es rediseñar de cuál lado debería vivir la decisión.

## Alternativas consideradas y rechazadas

### Unificar folders vía UNC path (`\\wsl.localhost\Ubuntu\home\...`)

**Idea:** que Cowork apunte directo al filesystem de WSL vía Windows Share. Un solo working copy, sin sync.

**Por qué se descartó:** el file picker de Cowork (Electron app) no acepta paths UNC. Intento documentado en sesión de 2026-04-23. Probado con `\\wsl.localhost\...` directo y con workaround de `net use N: \\wsl.localhost\...` — el primero no lo reconoce el picker, el segundo tira error de sistema 64 porque WSL no expone SMB estándar.

**Alternativa adicional** (`mklink /D C:\Nexus \\wsl.localhost\...`): requiere CMD como admin + algunos apps no siguen symlinks a UNC correctamente. Probable que funcione pero demanda setup manual cada vez que se crea un proyecto nuevo y agrega una capa de indirección que oscurece problemas.

**Queda como posible mejora futura** si Cowork agrega soporte nativo para paths UNC.

### Rsync bidireccional con detección de conflictos

**Idea:** en lugar de one-way rsync, detectar cuál lado modificó primero y preservar.

**Por qué se descartó:** sync bidireccional con conflict resolution es un problema muy difícil (ver Dropbox, OneDrive — years of engineering y siguen causando merge conflicts). No es razonable hacerlo en un bash script, y aunque funcionara, solo enmascara el hecho de que la arquitectura fundamental (dos fuentes de verdad) es la que genera el conflicto.

### Mover todo el proyecto a `/mnt/c/...` (filesystem Windows, accesible desde WSL)

**Idea:** el proyecto vive en el filesystem de Windows; Cowork lo ve nativo; WSL lo accede vía `/mnt/c/`.

**Por qué se descartó:** la performance de WSL2 sobre `/mnt/c/` es significativamente peor que sobre el filesystem Linux nativo. Para proyectos Node.js con `pnpm install` (miles de archivos pequeños), la diferencia es de 30s a 3-5 minutos. Documentado en la guía oficial de WSL2 — "para proyectos Node, usar filesystem Linux nativo".

## Preguntas frecuentes

**¿Puedo hacer un cambio rápido directo desde VS Code conectado a WSL sin pasar por Cowork?**

Sí, si el cambio es trivial (typo, comentario, refactor mecánico de 1 línea). Claude Code es la fuente de verdad — VS Code conectado a WSL es Claude Code con una UI distinta. Pero si el cambio tiene implicaciones de diseño (toca arquitectura, toca red lines, toca múltiples archivos relacionados), pasá por Cowork primero. La discusión es donde agregás valor; el typing es mecánico.

**¿Y si me olvido y edito algo desde el workspace de Cowork?**

No es catastrófico si lo cachás rápido. Tres pasos:

1. Copiá el cambio que hiciste (mentalmente o a un bloc).
2. Revertí en Cowork workspace a la versión de GitHub (o simplemente ignoralo — no se va a sincronizar).
3. Hacé el mismo cambio en Claude Code.

La carpeta Cowork es decorativa — lo que no está en WSL + Git no existe.

**¿Hay tooling que me ayude a no desviarme del patrón?**

- El script deprecated `sync-to-github.sh` ahora aborta con `exit 1` por default. Si alguien intenta correrlo por error, falla ruidosamente. Fuerzarlo requiere `FORCE_RUN_DEPRECATED=1` explícitamente.
- La sección §6 de `CLAUDE.md` del proyecto Nexus documenta el workflow para cualquier sesión nueva de Claude Code.
- Un hook pre-commit en git podría verificar que `git config user.email` matchea el humano (no un bot) — no implementado todavía, candidato a tech-debt.

## Referencias

- Postmortem del incidente que motivó esta decisión: `antipatterns/rsync-dual-source.md`.
- Recovery patterns cuando el estado se rompe: `workflow/git-surgery.md`.
- Nexus `CLAUDE.md` §6 — versión resumida del contrato, leída por cada sesión nueva de Claude Code.
- Commits relevantes en Nexus:
  - Deprecación de `sync-to-github.sh`: PR #8.
  - Sección §6 agregada a CLAUDE.md: PR #8 (mismo).
