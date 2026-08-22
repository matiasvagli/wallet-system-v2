# ADR 0003: Saga orquestada, no coreografiada

- **Estado:** Propuesto
- **Fecha:** 2026-08-22

## Contexto

Una transferencia entre dos usuarios toca dos servicios y no puede resolverse con
una transacción ACID: hay que debitar en el origen y acreditar en el destino,
sabiendo que cualquiera de los dos pasos puede fallar.

El patrón Saga resuelve esto con transacciones locales encadenadas más acciones de
compensación. Tiene dos variantes.

## Opciones consideradas

### A) Saga coreografiada
Cada servicio escucha eventos y reacciona; no hay coordinador central.
- ✅ Bajo acoplamiento; agregar un participante no toca a los demás
- ❌ **Nadie conoce el estado global de la transferencia.** Responder "¿en qué
  estado está esta transacción?" obliga a reconstruirlo desde varios servicios
- ❌ El flujo no está escrito en ningún lado: se deduce leyendo todos los handlers
- ❌ Las compensaciones en cascada son difíciles de razonar y de testear

### B) Saga orquestada
Un servicio coordina los pasos y decide cuándo compensar.
- ✅ El flujo está en un archivo, legible de arriba a abajo
- ✅ El estado de la saga es una fila en una tabla: consultable y auditable
- ✅ La lógica de compensación es testeable sin levantar la infraestructura
- ❌ El orquestador es un punto central de acoplamiento y de falla

## Decisión

**Saga orquestada**, con `transactions-service` como orquestador.

Dos criterios desempataron:

1. **Auditabilidad.** En un sistema de dinero, "¿dónde está esta transferencia?"
   tiene que responderse con una consulta, no con una investigación entre logs de
   varios servicios.
2. **Es la parte que se quiere mostrar.** Un flujo de saga explícito y legible
   comunica el patrón; uno coreografiado lo esconde entre handlers.

El estado se modela con dos campos separados: `status` (el resultado de negocio:
`PENDING` / `COMPLETED` / `FAILED`) y `saga_step` (el avance técnico: `DEBITING` /
`CREDITING` / `COMPENSATING` / ...). Mezclarlos en un solo campo hace imposible
distinguir "falló" de "está compensando".

## Consecuencias

**Positivas:** el estado de cualquier transferencia se consulta con un `SELECT`.
La lógica de compensación se testea con dobles de prueba, sin Docker.

**Negativas:** `transactions-service` se vuelve crítico — si se cae, ninguna
transferencia avanza. Se mitiga con reintentos idempotentes y persistiendo el paso
de la saga antes de ejecutarlo, de modo que un orquestador que reinicia pueda
retomar donde quedó.

**Un caso que hay que resolver explícitamente:** si el crédito falla *y* la
compensación también falla, el dinero queda debitado sin acreditar. Ese estado no
se puede resolver de forma automática: se marca como
`requiere_revision_manual` y se emite un evento a una cola de revisión. Una saga
que no contempla el fallo de su propia compensación está incompleta.
