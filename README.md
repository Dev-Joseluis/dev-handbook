# dev-handbook

Lecciones destiladas de proyectos reales. Captura el **por qué** detrás de las decisiones técnicas, no solo el **qué**. Pensado para:

- Yo mismo dentro de 6 meses, cuando empiece un proyecto nuevo y dude "¿por qué decidimos X en el último proyecto?".
- Cualquier colaborador que se sume a un proyecto en progreso y necesite entender convenciones sin arqueología de git.
- Sesiones nuevas de Claude (Cowork / Claude Code) que lean esto para alinearse con el patrón establecido.

## Cómo usar este handbook

**Como autor** (cuando aprendas algo que valga la pena):

- Escribí el doc mientras el aprendizaje está fresco — idealmente la misma semana del evento.
- Cada doc debe ser **auto-contenido**: alguien que cae directo ahí tiene que entenderlo sin leer otros primero (aunque puede referenciarlos).
- Si el doc nace de un error concreto, incluí postmortem: qué pasó, por qué pasó, qué hicimos, qué haríamos distinto.

**Como lector** (cuando arranques un proyecto nuevo):

- Empezá por `workflow/` para el framework operativo.
- Revisá `antipatterns/` antes de tomar decisiones arquitectónicas — filtra errores ya cometidos.
- Usá `multitenant-db/`, `ci-cd/`, `subagents/` como referencias específicas cuando llegues a ese problema.

## Estructura

```
workflow/          Cómo opera el equipo (Cowork ↔ Claude Code, git flow,
                   decisiones cross-cutting)
multitenant-db/    Patrones de base de datos multi-tenant (RLS, roles,
                   migrations, queries por tenant)
antipatterns/      Qué NO hacer y por qué — con evidencia de los eventos
                   concretos que lo demostraron
ci-cd/             GitHub Actions, Turbo, tooling de integración continua
subagents/         Diseño de subagents de Claude Code, red lines, patrones
                   de prompts
postmortems/       Incidentes reales resueltos — qué aprendimos, qué cambió
                   como resultado
```

## Convenciones de escritura

- **Español.** Es el idioma de trabajo. Si alguna vez el handbook se abre a contributors internacionales, traducimos.
- **Headers en prosa, no imperativos.** "El problema que esta decisión resuelve" > "Problema".
- **Ejemplos concretos con paths reales.** "En `packages/db/src/client.ts`" > "En el cliente de DB".
- **Red lines explícitas.** Si algo NUNCA se hace, decilo directo — con mayúsculas cuando haga falta.
- **Referencias a commits y ADRs.** Cuando cites una decisión del pasado, incluí el commit SHA o ADR number para anclaje.
- **Longitud moderada.** Target: 800-2000 palabras por doc. Si crece más, probablemente necesitás subdividirlo.

## Changelog

Usá commits descriptivos. No hay versioning formal — este es un doc vivo.

## Referencias

- Nexus (el proyecto donde nacieron la mayoría de estas lecciones): https://github.com/Dev-Joseluis/Nexus
