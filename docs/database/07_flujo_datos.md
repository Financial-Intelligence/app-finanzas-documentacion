# 07 - Flujo de datos

## Estado del documento

Actualizado con los flujos actuales y previstos.

## Flujo actual de cuentas

```text
Usuario autenticado
  -> endpoint de cuentas
  -> validacion de datos
  -> tabla accounts
  -> respuesta al frontend
```

Resumen:

- `GET /api/accounts` lee cuentas del usuario.
- `GET /api/accounts/summary` calcula saldo total y credito disponible.
- `POST /api/accounts` crea cuenta.
- `PATCH /api/accounts/:id` edita cuenta.
- `PATCH /api/accounts/:id/status` activa o inactiva.

## Flujo futuro de movimientos

```text
Modulo origen
  -> crea o registra accion financiera
  -> tabla movements
  -> actualiza saldo si esta confirmado
  -> dashboard/reportes leen movements
```

## Flujo futuro de recurrentes

```text
recurring_payments
  -> recurring_occurrences
  -> registrar pago
  -> movements
  -> accounts
```

## Flujo futuro de variables

```text
variable_payments
  -> confirmar
  -> movements
  -> accounts
```

## Flujo futuro de suscripciones

```text
subscriptions
  -> subscription_renewals
  -> registrar pago
  -> movements
  -> accounts
```

## Flujo futuro de prestamos

```text
loans
  -> loan_collections
  -> movements
  -> accounts
```

## Flujo futuro de deudas

```text
debts
  -> debt_payments
  -> movements
  -> accounts
```

## Flujo futuro de metas

```text
goals
  -> goal_contributions
  -> movements
  -> accounts
```

