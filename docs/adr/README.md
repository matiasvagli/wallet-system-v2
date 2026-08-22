# Architecture Decision Records

Registro de las decisiones técnicas del proyecto: qué se decidió, cuándo, y sobre
todo **por qué**.

Un ADR no se edita cuando la decisión cambia — se escribe uno nuevo que reemplaza
al anterior. El historial de cómo se pensó el sistema es tan valioso como el
estado actual.

| # | Decisión | Estado |
|---|---|---|
| [0001](./0001-microservicios-desde-el-inicio.md) | Microservicios desde el inicio | Propuesto |
| [0002](./0002-typeorm-sobre-prisma.md) | TypeORM sobre Prisma | Propuesto |
| [0003](./0003-saga-orquestada.md) | Saga orquestada, no coreografiada | Propuesto |
| [0004](./0004-outbox-e-idempotencia.md) | Outbox transaccional e idempotencia | Propuesto |

## Estados

| Estado | Significado |
|---|---|
| **Propuesto** | La decisión está tomada y razonada, pero todavía no implementada. Puede cambiar al chocar con la realidad del código. |
| **Aceptado** | Implementada y funcionando. Se marca así en el PR que la implementa. |
| **Reemplazado** | Ya no rige. Apunta al ADR que la sustituye — no se borra, el historial es parte del valor. |

Plantilla: [`template.md`](./template.md)
