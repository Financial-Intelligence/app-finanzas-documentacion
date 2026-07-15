# 08 - Avance backend modulo cuentas

## Estado

Implementado en backend como primera version funcional y considerado completado para el alcance base de cuentas.

## Endpoints definidos

```text
GET    /api/accounts
GET    /api/accounts/summary
POST   /api/accounts
GET    /api/accounts/:id
GET    /api/accounts/:id/movements
PATCH  /api/accounts/:id
DELETE /api/accounts/:id
PATCH  /api/accounts/:id/status
```

## Decisiones confirmadas

1. Las cuentas pertenecen a un usuario.
2. Las cuentas se crean activas por defecto.
3. Las cuentas pueden inactivarse mediante el cambio de estado.
4. Tambien existe eliminacion fisica para cuentas que no son principales y no tienen historial.
5. Las cuentas de credito no suman al saldo total.
6. Las cuentas de credito suman al credito disponible.
7. El saldo actual de una cuenta normal nace desde el saldo inicial.
8. Los movimientos confirmados ya modifican saldos reales.
9. La primera cuenta del usuario se marca como principal automaticamente.
10. `isPrimary` permite marcar una cuenta principal.
11. Solo una cuenta del usuario puede quedar como principal.
12. Si una cuenta principal se inactiva, deja de ser principal.

## Tipos de cuenta

```text
BANK_ACCOUNT
SAVINGS
CASH
DIGITAL_WALLET
CREDIT_CARD
INVESTMENT
OTHER
```

## Relacion con la interfaz

La pantalla principal de cuentas usa:

1. `GET /api/accounts/summary` para tarjetas superiores.
2. `GET /api/accounts?status=active` para listar cuentas activas.
3. `GET /api/accounts?status=inactive` para filtro de inactivas.
4. `POST /api/accounts` para crear nueva cuenta.
5. `PATCH /api/accounts/:id` para editar.
6. `PATCH /api/accounts/:id/status` para activar o inactivar.
7. `DELETE /api/accounts/:id` para eliminar una cuenta que no sea principal ni tenga historial.
8. `GET /api/accounts/:id/movements` para consultar el historial paginado.

## Regla de resumen

```text
saldo total = cuentas activas que NO son CREDIT_CARD
credito disponible = cuentas activas que SI son CREDIT_CARD
cuenta principal = cuenta activa con `isPrimary` = true
```

## Cambio sobre pagos recurrentes

La accion futura sera `registrar pago`.

No se usara como concepto principal `confirmar recurrente`.

Cuando se registre un pago:

1. El formulario abrira con el monto previsto.
2. El usuario podra guardar un monto real distinto.
3. El monto previsto se conservara.
4. El monto real servira para resultado real.
5. La diferencia servira para comparar esperado contra real.


## Estado final del modulo cuentas base

El modulo de cuentas queda completo para el alcance actual del backend.

Incluye:

1. CRUD base, con eliminacion fisica restringida para cuentas no principales.
2. Listado por usuario autenticado.
3. Filtro por activas, inactivas o todas.
4. Resumen de saldo total y credito disponible.
5. Cuenta principal por usuario.
6. Tarjetas de credito separadas del saldo total.
7. Documentacion Swagger y README del backend actualizados.
8. Validacion de propiedad: cada operacion solo accede a cuentas del usuario autenticado.

Queda pendiente para otro modulo:

1. Graficos reales por mes.
2. Resultado esperado y real mensual.
3. Pagos recurrentes, variables, deudas, prestamos y metas.
