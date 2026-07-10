# 05 - Relaciones y cardinalidades

## Estado del documento

Actualizado con la especificacion actual.

## Relaciones principales

| Origen | Destino | Cardinalidad | Explicacion |
| --- | --- | --- | --- |
| `users` | `accounts` | 1 a muchos | Un usuario tiene muchas cuentas. |
| `users` | `categories` | 1 a muchos | Un usuario define sus categorias. |
| `categories` | `subcategories` | 1 a muchos | Una categoria puede tener varias subcategorias. |
| `users` | `movements` | 1 a muchos | Un usuario registra muchos movimientos. |
| `accounts` | `movements` | 1 a muchos | Una cuenta puede aparecer como origen o destino. |
| `recurring_payments` | `recurring_occurrences` | 1 a muchos | Un recurrente genera ocurrencias. |
| `recurring_occurrences` | `movements` | 1 a 0..1 | Una ocurrencia confirmada crea movimiento. |
| `variable_payments` | `movements` | 1 a 0..1 | Un variable confirmado crea movimiento. |
| `subscriptions` | `subscription_renewals` | 1 a muchos | Una suscripcion genera renovaciones. |
| `subscription_renewals` | `movements` | 1 a 0..1 | Una renovacion pagada crea movimiento. |
| `loans` | `loan_collections` | 1 a muchos | Un prestamo puede tener varios cobros. |
| `loan_collections` | `movements` | 1 a 0..1 | Un cobro crea ingreso. |
| `debts` | `debt_payments` | 1 a muchos | Una deuda puede tener varios pagos. |
| `debt_payments` | `movements` | 1 a 0..1 | Un pago crea egreso. |
| `goals` | `goal_contributions` | 1 a muchos | Una meta puede tener muchos aportes. |
| `goal_contributions` | `movements` | 1 a 0..1 | Un aporte crea movimiento. |
| `users` | `budgets` | 1 a muchos | Un usuario puede tener presupuestos. |
| `users` | `monthly_plans` | 1 a muchos | Un usuario tiene planes mensuales. |
| `accounts` | `monthly_plans` | 1 a muchos opcional | Un plan puede ser global o por cuenta. |

## Relacion polimorfica de movimientos

`movements.source_type` y `movements.source_id` indican de donde nacio el movimiento.

Ejemplos:

- `source_type=recurring`
- `source_type=variable`
- `source_type=subscription`
- `source_type=loan`
- `source_type=debt`
- `source_type=goal`

