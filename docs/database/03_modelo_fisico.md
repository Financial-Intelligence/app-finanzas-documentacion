# 03 - Modelo fisico

## Estado del documento

Actualizado desde `FINANCE_APP_DATABASE_AND_API_SPEC.md`.

Este modelo representa la especificacion objetivo. Debe compararse con las migraciones reales cuando existan.

## Convenciones

- Clave primaria: `id UUID`.
- Usuario propietario: `user_id UUID`.
- Dinero: `DECIMAL(14,2)`.
- Fechas de auditoria: `created_at`, `updated_at`.
- Borrado logico: `deleted_at` en tablas criticas.
- Periodo mensual: `period CHAR(7)` con formato `YYYY-MM`.

## Tablas

| Tabla | Responsabilidad |
| --- | --- |
| `users` | Usuario autenticado. |
| `refresh_tokens` | Sesiones o renovacion de acceso. |
| `accounts` | Cuentas financieras. |
| `categories` | Categorias de ingreso o egreso. |
| `subcategories` | Subcategorias. |
| `movements` | Fuente central de verdad financiera. |
| `recurring_payments` | Definicion de pagos recurrentes. |
| `recurring_occurrences` | Ocurrencias por periodo. |
| `variable_payments` | Pagos variables de un periodo. |
| `subscriptions` | Suscripciones. |
| `subscription_renewals` | Renovaciones por periodo. |
| `loans` | Prestamos por cobrar. |
| `loan_collections` | Cobros de prestamos. |
| `debts` | Deudas por pagar. |
| `debt_payments` | Pagos de deudas. |
| `goals` | Metas financieras. |
| `goal_contributions` | Aportes a metas. |
| `budgets` | Presupuestos. |
| `monthly_plans` | Plan mensual por usuario o cuenta. |

## Indices principales esperados

- `users.email` unico.
- `accounts(user_id, status)`.
- `accounts(user_id, type)`.
- `categories(user_id, type, is_active)`.
- `subcategories(category_id, is_active)`.
- `movements(user_id, period)`.
- `movements(account_id, period)`.
- `movements(user_id, type, status)`.
- `movements(source_type, source_id)`.
- `recurring_occurrences(recurring_id, period)` unico.
- `subscription_renewals(subscription_id, period)` unico.
- `monthly_plans(user_id, account_id, period)` unico.

## Enums principales

| Enum | Valores |
| --- | --- |
| `account_type` | `bank`, `savings`, `digital_wallet`, `cash`, `credit_card`, `investment`, `other` |
| `account_status` | `active`, `inactive`, `archived` |
| `currency` | `PEN`, `USD` |
| `movement_type` | `income`, `expense`, `transfer` |
| `movement_status` | `pending`, `confirmed`, `cancelled` |
| `category_type` | `income`, `expense` |
| `payment_frequency` | `daily`, `weekly`, `biweekly`, `monthly`, `quarterly`, `annual` |
| `source_type` | `manual`, `recurring`, `variable`, `subscription`, `loan`, `debt`, `goal`, `transfer` |

