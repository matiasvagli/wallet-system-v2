# ADR 0004: Outbox transaccional e idempotencia por clave única

- **Estado:** Propuesto
- **Fecha:** 2026-08-22

## Contexto

Dos problemas distintos que suelen confundirse:

**1. Escritura dual.** Al completar una transferencia hay que guardar en Postgres
y publicar un evento a RabbitMQ. No son una operación atómica: si la base
commitea y RabbitMQ está caído, el evento se pierde para siempre y los servicios
quedan desincronizados sin que nadie se entere.

**2. Entrega duplicada.** RabbitMQ garantiza *at-least-once*: un mensaje puede
llegar dos veces (por reintento, por un ack perdido, por un consumidor que
reinicia). Si un débito se procesa dos veces, el usuario pierde plata.

## Opciones consideradas

### Para la escritura dual

**A) Publicar directo tras el commit** — simple, y pierde eventos ante cualquier
fallo de red. Descartada.

**B) Outbox transaccional** — el evento se inserta en una tabla `outbox` dentro de
la *misma transacción* que el cambio de negocio. Un proceso aparte lee las filas
pendientes y las publica.
- ✅ Atomicidad real: o se guardan las dos cosas, o ninguna
- ❌ Introduce latencia y un proceso más

### Para la idempotencia

**C) Buscar por texto en la descripción de la transacción** — el enfoque que usaba
la versión anterior de este sistema: `WHERE description LIKE '%<clave>%'`.
Descartada por un motivo concreto: la clave `debit-tx-1` coincide con la
descripción generada por `debit-tx-12`. El segundo débito se salta silenciosamente
y devuelve éxito. Es una pérdida de dinero causada por una comparación de strings.

**D) Columna dedicada con restricción UNIQUE** — la clave de idempotencia es una
columna propia con índice único. El reintento choca contra la restricción de la
base de datos.

## Decisión

**Outbox transaccional** (opción B) para publicar eventos, e **idempotencia por
columna `idempotency_key` con `UNIQUE`** (opción D) en toda operación que mueva
saldo.

El criterio para la idempotencia: la garantía tiene que vivir en **la base de
datos**, no en el código de aplicación. Un `SELECT` previo seguido de un `INSERT`
tiene una ventana de carrera entre ambos; una restricción `UNIQUE` no la tiene.
El flujo correcto es intentar el `INSERT` y tratar la violación de unicidad como
"ya procesado", no consultar antes de escribir.

Cada paso de la saga lleva su propia clave derivada de forma determinística del
identificador de la transacción:

```
debit-<transaction_id>
credit-<transaction_id>
compensate-<transaction_id>
```

Determinística importa: un reintento del mismo paso genera la misma clave y la
base lo rechaza. Si fueran aleatorias, cada reintento sería un movimiento nuevo.

## Consecuencias

**Positivas:** ningún evento se pierde ante una caída de RabbitMQ. Un mensaje
duplicado es inofensivo por construcción, no por suerte de timing.

**Negativas:** la tabla `outbox` crece y necesita una limpieza periódica de filas
ya publicadas. El publicador agrega latencia entre el commit y la publicación del
evento: el sistema es **eventualmente consistente**, y eso se asume de forma
explícita.

**Garantía que NO se obtiene:** el outbox asegura entrega *at-least-once*, nunca
*exactly-once*. Los consumidores tienen que ser idempotentes de todas formas — el
outbox no los exime, y creer lo contrario es el error clásico con este patrón.
