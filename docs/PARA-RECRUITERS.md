# Para quien esté evaluando este proyecto

Esta página está escrita para alguien que tiene quince minutos y quiere saber qué
sabe hacer el autor. No repite el README: explica **el criterio detrás del
código**.

> **Estado actual:** en construcción. El README tiene el roadmap con lo que ya
> funciona. Prefiero decirlo acá antes que dejar que lo descubras solo.

---

## 1. Qué es

Una billetera digital donde los usuarios transfieren dinero entre sí. La
funcionalidad es deliberadamente chica; el problema interesante es otro:

**El débito y el crédito ocurren en servicios distintos, y no comparten una
transacción de base de datos.** Si el crédito falla después de que el débito
salió bien, el dinero desaparece. Todo el diseño gira alrededor de que eso no
pase — y de qué hacer cuando igual pasa.

## 2. Las decisiones que importan

Cada una está en un [ADR](./adr/) con las opciones descartadas. El resumen:

**Elegí microservicios sabiendo que un monolito sería mejor ingeniería.**
Con cero usuarios, un monolito modular con una transacción ACID resolvería esto
en una fracción del tiempo. Lo digo explícitamente en el
[ADR-0001](./adr/0001-microservicios-desde-el-inicio.md). Los elegí porque el
objetivo del proyecto es demostrar transacciones distribuidas, y sin fronteras de
proceso reales la saga sería una simulación. Saber cuándo una arquitectura es
sobre-ingeniería me parece más importante que saber implementarla.

**Elegí TypeORM sobre Prisma aun siendo Prisma más demandado.**
El criterio fue que Prisma no genera migraciones de rollback. En un sistema que
mueve dinero, una migración que no se puede revertir de forma controlada es un
riesgo operativo. La ergonomía cede ante eso.
([ADR-0002](./adr/0002-typeorm-sobre-prisma.md))

**Elegí saga orquestada por auditabilidad.**
La versión coreografiada tiene menos acoplamiento, pero nadie conoce el estado
global de la transferencia. En un sistema de dinero, "¿dónde está esta
transferencia?" tiene que responderse con una consulta, no con una investigación
entre logs de cuatro servicios.
([ADR-0003](./adr/0003-saga-orquestada.md))

**La idempotencia vive en la base de datos, no en el código.**
Una restricción `UNIQUE` sobre la clave de idempotencia, no un `SELECT` previo
seguido de un `INSERT`. La diferencia es que el `SELECT` tiene una ventana de
carrera entre la lectura y la escritura, y la restricción no.
([ADR-0004](./adr/0004-outbox-e-idempotencia.md))

## 3. Cómo se trabajó

Un solo autor, pero con proceso de equipo — porque el proceso también es parte de
lo que quiero mostrar:

- **Todo entra por Pull Request.** `main` protegida, sin push directo
- **Un issue por unidad de trabajo**, cerrado por su PR
- **Conventional Commits**, con el *por qué* en el cuerpo del mensaje
- **CI obligatorio**: lint, typecheck y tests antes de poder mergear
- **ADRs** para toda decisión costosa de revertir

El historial de commits y los PRs cerrados son parte del entregable. Si querés ver
cómo razono un cambio, la descripción de cualquier PR sirve más que el diff.

**Sobre la asistencia de IA.** Trabajé con un agente en parte de la
implementación. Los commits donde participó lo declaran con `Co-Authored-By`, así
que está en el historial y no hace falta que me creas. Lo que no delegué: la
arquitectura, los ADRs y qué entra a `main`. Si un PR está mergeado es porque lo
leí, lo entendí y lo banco.

## 4. Lo que este proyecto arregla de una versión anterior

Este sistema tiene una v1 que escribí en 2024 y retomé brevemente con asistencia
de IA. Al volver a abrirla encontré tres cosas que vale la pena nombrar, porque
explican decisiones de esta versión:

**Dos implementaciones del mismo flujo conviviendo.** La v1 tenía la transferencia
resuelta de dos maneras — una saga por HTTP y un consumidor de eventos que hacía
lo mismo — sin que ninguna estuviera terminada. Ninguna de las dos estaba mal por
sí sola; el problema era tenerlas juntas. Si ambas se hubieran activado, el
resultado era doble débito. De ahí la regla dura de esta versión: **un solo camino
de ejecución por caso de uso.**

**Idempotencia por comparación de strings.** La v1 verificaba si una operación ya
se había procesado buscando la clave dentro de un campo de texto con `LIKE`. La
clave `debit-tx-1` coincide con la descripción generada por `debit-tx-12`: el
segundo débito se saltaba en silencio y devolvía éxito. Una pérdida de dinero
causada por una comparación de strings. De ahí que acá la idempotencia sea una
columna con `UNIQUE`.

**Endpoints de saldo sin autenticación.** El servicio de billetera exponía
`credit` y `debit` con permisos abiertos y publicaba su puerto al host. Cualquiera
con acceso a la red podía acreditarse dinero. De ahí que acá **solo el gateway
publique puerto**.

La lección de fondo no es técnica: es que **el código sin decisiones documentadas
se vuelve ilegible para su propio autor** en menos de un año. Los ADRs de este
repo existen por eso.

## 5. Lo que NO hice, a propósito

Tan importante como lo que está:

**Kubernetes.** El objetivo es demostrar transacciones distribuidas, no
orquestación de contenedores. Docker Compose comunica la arquitectura sin agregar
una capa que no aporta al tema.

**CQRS y Event Sourcing.** Encajarían en un sistema de billeteras y son tentadores
de mostrar. No están porque este sistema no tiene la asimetría de lectura/escritura
que los justifica. Agregarlos sería demostrar que conozco el patrón, no que sé
cuándo aplicarlo.

**Cobertura de tests del 100%.** Los tests están donde hay lógica que puede
fallar: las compensaciones de la saga, los casos borde de idempotencia, la
aritmética de saldos. Testear getters infla el número y no protege nada.

**Un frontend.** El proyecto es sobre el backend distribuido. Un frontend
agregaría superficie sin agregar señal.

**Alta disponibilidad y réplicas.** Con cero usuarios, diseñar para escala sería
resolver un problema que no tengo.

## 6. Probarlo en dos minutos

```bash
git clone https://github.com/matiasvagli/wallet-system-v2
cd wallet-system-v2
cp .env.example .env
docker compose up -d
```

## 7. Por dónde empezar a leer

Si tenés poco tiempo y querés evaluar criterio técnico, en este orden:

1. [ADR-0001](./adr/0001-microservicios-desde-el-inicio.md) — por qué esta
   arquitectura, y por qué en otro contexto estaría mal
2. [ADR-0004](./adr/0004-outbox-e-idempotencia.md) — el detalle más fino del
   diseño
3. `services/transactions-service/src/domain/` — la lógica de la saga, sin
   dependencias de framework
4. Cualquier Pull Request cerrado — cómo se justifica un cambio

---

Matias Vagliviello · [github.com/matiasvagli](https://github.com/matiasvagli) ·
vaglimatias@gmail.com
