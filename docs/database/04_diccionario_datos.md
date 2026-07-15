# 04 - Diccionario de datos

## Estado del documento

Actualizado con los campos implementados de Movimientos al 2026-07-15 y los modulos objetivo restantes.

## Campos comunes

| Campo | Significado |
| --- | --- |
| `id` | Identificador unico del registro. |
| `user_id` | Usuario propietario del dato. |
| `created_at` | Fecha de creacion. |
| `updated_at` | Fecha de ultima actualizacion. |
| `deleted_at` | Marca de borrado logico. |
| `period` | Mes de trabajo en formato `YYYY-MM`. |

## `users`

| Campo | Descripcion |
| --- | --- |
| `name` | Nombre visible del usuario. |
| `email` | Correo unico para login. |
| `password_hash` | Contrasena protegida; nunca debe exponerse. |
| `base_currency` | Moneda principal del usuario. |
| `savings_target` | Meta de sobrante mensual. |

## `accounts`

| Campo | Descripcion |
| --- | --- |
| `name` | Nombre de la cuenta. |
| `type` | Tipo de cuenta. |
| `bank_name` | Banco o entidad. |
| `currency` | Moneda de la cuenta. |
| `initial_balance` | Saldo inicial. |
| `current_balance` | Saldo actual. |
| `credit_limit` | Limite de credito, si aplica. |
| `status` | Estado de la cuenta. |
| `last_movement_at` | Ultima actualizacion por movimiento. |

## `categories`

| Campo | Descripcion |
| --- | --- |
| `type` | Ingreso o egreso. |
| `period` | Mes independiente al que pertenece la categoria. |
| `name` | Nombre de categoria. |
| `color` | Color usado en interfaz. |
| `icon` | Icono usado en interfaz. |
| `is_active` | Indica si la categoria esta activa. |

## `movements`

| Campo | Descripcion |
| --- | --- |
| `type` | Ingreso, egreso o transferencia. |
| `status` | Pendiente, confirmado o cancelado. |
| `expected_amount` | Monto previsto antes de confirmar. |
| `actual_amount` | Monto real; es nulo mientras esta pendiente. |
| `account_id` | Cuenta principal del movimiento. |
| `to_account_id` | Cuenta destino cuando es transferencia. |
| `category_id` | Categoria asociada. |
| `subcategory_id` | Subcategoria asociada. |
| `occurred_on` | Fecha del movimiento. |
| `occurred_at` | Fecha y hora exacta opcional. |
| `description` | Descripcion del movimiento. |
| `currency` | Moneda heredada de la cuenta origen. |
| `source_type` | Modulo que origino el movimiento. |
| `source_id` | Registro origen dentro del modulo. |
| `deleted_at` | Oculta logicamente un movimiento eliminado. |

## `recurring_payments`

| Campo | Descripcion |
| --- | --- |
| `type` | Ingreso o egreso recurrente. |
| `name` | Nombre del recurrente. |
| `amount` | Monto estimado. |
| `frequency` | Frecuencia. |
| `next_date` | Proxima fecha programada. |
| `status` | Activo, pausado o finalizado. |
| `last_confirmed` | Ultimo periodo registrado. |

## `variable_payments`

| Campo | Descripcion |
| --- | --- |
| `name` | Nombre del pago variable. |
| `amount` | Monto planificado. |
| `period` | Periodo al que pertenece. |
| `planned_date` | Fecha prevista. |
| `status` | Pendiente o confirmado. |
| `movement_id` | Movimiento creado al confirmar. |

## `subscriptions`

| Campo | Descripcion |
| --- | --- |
| `name` | Nombre de la suscripcion. |
| `amount` | Monto de cobro. |
| `frequency` | Frecuencia de renovacion. |
| `next_renewal` | Proxima renovacion. |
| `status` | Activa, pausada o finalizada. |
| `last_paid` | Ultimo periodo pagado. |

## `loans`

| Campo | Descripcion |
| --- | --- |
| `debtor` | Persona o entidad que debe. |
| `original_amount` | Monto prestado. |
| `outstanding` | Saldo por cobrar. |
| `source_account_id` | Cuenta de donde salio el dinero. |
| `return_account_id` | Cuenta donde entra el cobro. |
| `status` | Activo o cobrado. |

## `debts`

| Campo | Descripcion |
| --- | --- |
| `creditor` | Persona o entidad a quien se debe. |
| `original_amount` | Monto original. |
| `balance` | Saldo pendiente. |
| `account_id` | Cuenta desde donde se paga. |
| `receive_account_id` | Cuenta donde ingreso el dinero recibido. |
| `status` | Activa o pagada. |

## `goals`

| Campo | Descripcion |
| --- | --- |
| `name` | Nombre de la meta. |
| `type` | Tipo de meta. |
| `target_amount` | Monto objetivo. |
| `current_amount` | Monto acumulado. |
| `deadline` | Fecha limite. |
| `status` | Activa, completada, pausada o vencida. |
| `health` | Ritmo de avance. |

## `monthly_plans`

| Campo | Descripcion |
| --- | --- |
| `carried_over` | Saldo heredado del mes anterior. |
| `recurring_income` | Ingreso recurrente esperado. |
| `initial_amount` | Monto inicial. |
| `planned_expenses` | Gastos previstos. |
| `planned_savings` | Ahorros previstos. |
| `planned_debts` | Deudas previstas. |
| `expected_result` | Resultado esperado. |
| `real_result` | Resultado real. |
