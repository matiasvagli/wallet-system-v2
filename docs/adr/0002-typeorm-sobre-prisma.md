# ADR 0002: TypeORM sobre Prisma

- **Estado:** Propuesto
- **Fecha:** 2026-08-22

## Contexto

Cada servicio necesita persistencia en Postgres. La arquitectura sigue un diseño
por capas donde el dominio define un **puerto** (interfaz de repositorio) y la
capa de infraestructura provee el **adaptador**. El ORM no debe filtrarse al
dominio.

Además, al manejar dinero, las migraciones tienen que poder revertirse de forma
controlada.

## Opciones consideradas

### A) Prisma
- ✅ Mejor experiencia de desarrollo; el cliente tipado es excelente
- ✅ Más demandado en ofertas laborales recientes
- ❌ No genera migraciones de rollback: `prisma migrate` solo va hacia adelante
- ❌ Su cliente generado no encaja naturalmente con el patrón Repository; termina
  envuelto igual, con lo que se pierde parte de su ventaja

### B) TypeORM
- ✅ Migraciones con `up()` y `down()` nativas y reversibles
- ✅ Integra con el sistema de inyección de dependencias de NestJS
- ✅ El `Repository` de TypeORM se adapta directo al puerto del dominio
- ❌ DX inferior; el tipado es menos estricto que el de Prisma
- ❌ Su API cambió bastante entre versiones mayores

## Decisión

**TypeORM.**

El criterio que desempató fue la **reversibilidad de migraciones**. En un sistema
que mueve dinero, una migración que no se puede revertir de forma controlada es un
riesgo operativo concreto, y ese peso supera la mejor ergonomía de Prisma.

El ORM queda confinado a `infrastructure/persistence/`. El dominio solo conoce la
interfaz del repositorio, así que reemplazarlo más adelante afecta un directorio,
no el sistema.

## Consecuencias

**Positivas:** cada migración tiene rollback explícito. El dominio es testeable
con un repositorio en memoria, sin levantar Postgres.

**Negativas:** hay que escribir a mano las entidades de TypeORM además de las
entidades de dominio, con el mapeo entre ambas. Es duplicación deliberada: es el
precio de que el dominio no dependa del framework.

**Cuándo revisar esta decisión:** si Prisma incorpora migraciones reversibles, la
comparación cambia y vale reconsiderarla.
