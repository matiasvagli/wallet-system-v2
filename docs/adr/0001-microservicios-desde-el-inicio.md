# ADR 0001: Microservicios desde el inicio

- **Estado:** Propuesto
- **Fecha:** 2026-08-22

## Contexto

El sistema modela una billetera digital con transferencias entre usuarios. El
volumen esperado es cero: no hay usuarios reales ni requisitos de escala.

El objetivo del proyecto es **demostrar el manejo de transacciones distribuidas**
(saga, compensaciones, idempotencia, mensajería asíncrona). Ese objetivo, y no la
escala, es la restricción que manda.

Existe además un antecedente propio: una versión anterior de este sistema quedó
con dos implementaciones distintas del mismo flujo de transferencia conviviendo
sin que ninguna estuviera terminada. Esa experiencia pesa en esta decisión.

## Opciones consideradas

### A) Monolito modular
- ✅ Mucho más rápido de construir y de operar
- ✅ Una transacción ACID resuelve la transferencia sin saga
- ❌ **No permite demostrar el objetivo del proyecto**: sin fronteras de proceso
  reales, la saga es artificial

### B) Microservicios desde el día 1
- ✅ Las transacciones distribuidas son un problema real, no simulado
- ✅ Fuerza a resolver de verdad idempotencia, compensación y consistencia eventual
- ❌ Más piezas móviles, más tiempo de desarrollo, debugging más difícil

## Decisión

**Microservicios desde el inicio**, con cuatro procesos: `api-gateway`,
`auth-service`, `wallet-service` y `transactions-service`.

El criterio que desempató: en un sistema con usuarios reales y volumen bajo, esta
arquitectura sería sobre-ingeniería y la opción A sería la correcta. Acá el
sistema distribuido *es* el entregable — la complejidad no es accidental, es el
tema.

Para que la decisión no se degrade se agregan dos reglas duras:

1. **Una base de datos por servicio.** Ningún servicio lee tablas de otro. Sin
   esto, son microservicios solo en el diagrama.
2. **Un solo camino de ejecución por caso de uso.** Cada flujo se resuelve de una
   sola manera. Un flujo con dos implementaciones activas sobre el mismo dato es
   riesgo de doble ejecución — en un sistema de dinero, doble débito.

## Consecuencias

**Positivas:** la saga, el outbox y la idempotencia son necesidades reales del
diseño y no adornos. Las fronteras entre servicios son explícitas.

**Negativas:** levantar el sistema requiere Docker Compose con siete contenedores.
Un bug de integración cruza procesos y es más difícil de diagnosticar que un stack
trace. El costo de desarrollo es varias veces el de un monolito equivalente.

**Cuándo revisar esta decisión:** si el proyecto pasa de demo técnica a producto
con usuarios, el primer paso sería consolidar `auth` y `wallet` en un solo
servicio y dejar la saga únicamente donde cruza una frontera que exista por una
razón de negocio.
