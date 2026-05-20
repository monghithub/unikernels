# CLAUDE.md — Unikernels: De cero a experto

Contexto persistente para Claude Code en este proyecto de estudio.

## Qué es este repo

Curso autodidacta de 80 lecciones (16 semanas, L–V, 30 min/día) sobre unikernels.
El índice maestro y hoja de ruta está en `README.md`.
Cada lección tiene su propio archivo Markdown en la carpeta de fase correspondiente.

## Estructura de carpetas

```
README.md                          ← índice maestro con calendario completo
CLAUDE.md                          ← este archivo
.claude/commands/leccion.md        ← skill /leccion N

fase-1-fundamentos/                días 01–15  (semanas 1–3)
fase-2-concepto-unikernel/         días 16–25  (semanas 4–5)
fase-3-practica-nanos/             días 26–40  (semanas 6–8)
fase-4-practica-unikraft/          días 41–55  (semanas 9–11)
fase-5-seguridad-produccion/       días 56–65  (semanas 12–13)
fase-6-nivel-experto/              días 66–80  (semanas 14–16)
```

## Skill disponible

### `/leccion <N>`

Genera el contenido completo de la lección del día N.

- Lee README.md para obtener título, tipo y semana
- Genera ~600-900 palabras estructuradas en bloques de 10/15/5 min
- Sobreescribe el placeholder en la carpeta de fase correcta
- Cierra el issue de GitHub correspondiente (#N)

**Ejemplo:** `/leccion 7` genera la lección "Rings de CPU: privilegios hardware".

## Seguimiento de progreso

El progreso se rastrea a través de los issues de GitHub:
- **Issue abierto** = lección pendiente
- **Issue cerrado** = lección completada
- Repo: `monghithub/unikernels`
- Issues: https://github.com/monghithub/unikernels/issues

Labels disponibles: `fase-1` … `fase-6`, `lectura`, `practica`, `repaso`, `proyecto`

Para ver cuántas lecciones quedan:
```bash
gh issue list --repo monghithub/unikernels --state open | wc -l
```

## Convenciones del proyecto

- Los archivos de lección se llaman `leccion-NN-slug.md` (NN con cero inicial)
- No borrar los encabezados YAML de los placeholders, sobreescribir todo el contenido
- Al generar una lección, siempre incluir la navegación `← anterior · siguiente →` al final
- Los commits de lección siguen el patrón: `leccion(NN): [título corto]`

## Contexto pedagógico

- El alumno estudia en sesiones de exactamente 30 minutos, L–V
- Nivel de partida: sabe programar (Go/Python/Node/Rust) y usa Linux, no conoce internos de OS ni virtualización
- Las lecciones de práctica deben ser ejecutables en una máquina Linux con KVM disponible
- Para las fases 1–2 (fundamentos teóricos) prioriza la construcción de modelo mental sobre la exhaustividad
- Para las fases 3–4 (práctica) cada comando debe tener su salida esperada documentada

## Herramientas del entorno

- OS: Linux con KVM
- `ops` CLI para Nanos (instalar en día 26)
- `kraft` CLI para Unikraft (instalar en día 41)
- QEMU como hypervisor local
- `gh` CLI para gestión de issues
- Git con remote `origin` → `git@github.com:monghithub/unikernels.git`
