# Cómo se trabaja en este repo

Este proyecto sigue un flujo de trabajo de equipo aunque hoy tenga un solo autor.
La razón es simple: el proceso es parte de lo que el proyecto demuestra.

## Setup inicial

```bash
pnpm install
git config core.hooksPath .githooks   # activa el hook que protege main
```

El hook `pre-push` aborta cualquier intento de pushear directo a `main`. Es un
recordatorio, no una jaula: se puede saltear con `--no-verify`, pero si estás
escribiendo eso ya sabés que estás rompiendo tu propia regla.

## Flujo de trabajo

```
main ──────●────────────●────────────●──
            \          / \          /
             ●───●────●   ●───●────●
        feat/wallet-domain    feat/saga-orchestrator
```

1. **Issue** — toda unidad de trabajo arranca con un issue que describe el *qué* y el *por qué*
2. **Rama** — se crea desde `main` actualizado
3. **Commits** — pequeños, atómicos, en formato Conventional Commits
4. **Pull Request** — describe la decisión tomada, no solo el cambio
5. **CI** — lint, typecheck y tests tienen que pasar en verde
6. **Merge** — squash a `main`, cerrando el issue

`main` está protegida: no se pushea directo.

## Convención de ramas

| Prefijo | Uso | Ejemplo |
|---|---|---|
| `feat/` | Funcionalidad nueva | `feat/wallet-debit-usecase` |
| `fix/` | Corrección de bug | `fix/saga-retry-state` |
| `refactor/` | Cambio interno sin cambio de comportamiento | `refactor/extract-money-vo` |
| `docs/` | Solo documentación | `docs/adr-outbox-pattern` |
| `chore/` | Tooling, CI, dependencias | `chore/setup-eslint` |
| `test/` | Solo tests | `test/saga-compensation` |

## Conventional Commits

```
<tipo>(<alcance opcional>): <descripción en imperativo>

[cuerpo opcional: el POR QUÉ, no el qué]

[footer opcional: Closes #12]
```

Tipos: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`.

**Bien:**
```
feat(wallet): agregar lock pesimista en débito

Dos requests concurrentes sobre la misma wallet pisaban el saldo
(lost update). Se usa SELECT FOR UPDATE dentro de la transacción.

Closes #14
```

**Mal:** `cambios`, `fix`, `update wallet service`

El cuerpo del commit explica **por qué**; el diff ya muestra el qué.

## Definition of Done

Un PR se mergea cuando:

- [ ] La lógica de dominio tiene tests (los casos borde, no los getters)
- [ ] CI en verde: lint + typecheck + tests
- [ ] No hay secretos hardcodeados
- [ ] Endpoints nuevos declaran explícitamente su autenticación
- [ ] Si cambia una decisión de arquitectura, hay un ADR que la respalda
- [ ] Si cambia la arquitectura, el README está actualizado

## Architecture Decision Records

Las decisiones técnicas importantes se documentan en [`docs/adr/`](./docs/adr/).

Se escribe un ADR cuando la decisión es **costosa de revertir** o cuando en seis
meses nadie va a recordar por qué se eligió así. No para cada decisión menor.

Usar [`docs/adr/template.md`](./docs/adr/template.md) como base.
