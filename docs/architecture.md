# Arquitectura

## Estructura del repositorio

Monorepo con pnpm workspaces. Un repositorio, cuatro servicios desplegables de
forma independiente.

```
wallet-system-v2/
├── packages/
│   └── contracts/          # eventos y tipos compartidos entre servicios
└── services/
    ├── api-gateway/
    ├── auth-service/
    ├── wallet-service/
    └── transactions-service/
```

**Por qué monorepo y no un repo por servicio:** el contrato de eventos entre
servicios cambia seguido en esta etapa. Con repos separados, cada cambio de
contrato obliga a publicar un paquete y coordinar versiones. En un monorepo, el
compilador marca al instante qué servicios rompe un cambio de contrato.

El costo es que se pierde independencia de despliegue real. Con cuatro servicios
y un autor, la compensación es clara.

## Capas dentro de cada servicio

Todos los servicios siguen la misma estructura, ordenada por una regla:
**`domain/` no importa nada de NestJS ni del ORM.**

```
src/
├── domain/              # entidades, value objects, puertos (interfaces)
├── application/         # casos de uso — orquestan el dominio
├── infrastructure/      # adaptadores: TypeORM, RabbitMQ, Redis
└── interfaces/          # controllers HTTP, handlers de mensajes, DTOs
```

Las dependencias apuntan siempre hacia adentro:

```
interfaces ──► application ──► domain ◄── infrastructure
```

`domain/` no depende de nada. `infrastructure/` implementa las interfaces que
`domain/` define — por eso la flecha va hacia el dominio y no al revés.

**Qué habilita en la práctica:** la lógica de la saga se testea con un repositorio
en memoria, sin Docker, sin Postgres y sin levantar el servidor. Si esa lógica
necesitara infraestructura para testearse, estaría mal desacoplada.

## Comunicación entre servicios

Dos mecanismos, con criterios distintos:

**HTTP síncrono** — cuando quien llama necesita la respuesta para decidir el paso
siguiente. Es el caso del orquestador de la saga: no puede acreditar sin saber si
el débito salió bien.

**Eventos por RabbitMQ** — cuando el emisor no necesita respuesta y el receptor
puede procesarlo más tarde: notificaciones, proyecciones, auditoría.

El criterio: si el emisor necesita la respuesta para continuar, es HTTP. Si no, es
un evento. Usar eventos donde hace falta una respuesta obliga a correlacionar
mensajes a mano y complica el flujo sin beneficio.

## Datos

Una base de Postgres por servicio. Ningún servicio accede a las tablas de otro: lo
que necesite, lo pide por su API o lo recibe por evento.

Los saldos usan `NUMERIC` de Postgres, nunca punto flotante. `0.1 + 0.2` en coma
flotante no da `0.3`, y en un sistema de dinero ese error se acumula.

## Seguridad

- **Solo `api-gateway` publica puerto.** El resto de los servicios es alcanzable
  únicamente desde la red interna de Docker
- El gateway valida el JWT y propaga la identidad del usuario hacia adentro
- Los servicios internos no confían en el `user_id` que venga en el cuerpo de una
  petición: usan el que propaga el gateway
- Sin secretos en el repositorio; todo por variables de entorno
- Los montos y las claves de idempotencia se validan en el borde, antes de tocar
  la capa de aplicación
