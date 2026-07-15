# Lumen Finanzas — Especificación de Base de Datos y API

> **Estado implementado al 2026-07-15:** el backend actual usa IDs `Int`, no UUID. Ya existen `users`, `accounts`, `categories`, `subcategories` y `movements`. Las demás entidades de este documento siguen siendo diseño objetivo.

> Documento base para el desarrollo fullstack. Derivado del análisis completo del piloto frontend
> (Next.js 14 + React 18) ubicado en `lumen-next/`: `lib/data.js`, `lib/store.js`,
> `components/modals.jsx` y todas las pantallas en `components/screens/`.
>
> **Estado del piloto:** frontend funcional con datos mock y store reactivo en memoria
> (persistencia en `localStorage`). Este documento define cómo reemplazar ese mock por un backend real.

**Índice**

1. [Visión general del sistema](#1-visión-general-del-sistema)
2. [Módulos detectados en el piloto](#2-módulos-detectados-en-el-piloto)
3. [Modelo de base de datos propuesto](#3-modelo-de-base-de-datos-propuesto)
4. [Relaciones entre tablas](#4-relaciones-entre-tablas)
5. [Enums del sistema](#5-enums-del-sistema)
6. [Reglas de negocio importantes](#6-reglas-de-negocio-importantes)
7. [Endpoints completos por módulo](#7-endpoints-completos-por-módulo)
8. [DTOs sugeridos](#8-dtos-sugeridos)
9. [Consideraciones para frontend](#9-consideraciones-para-frontend)
10. [Recomendación de fases de desarrollo](#10-recomendación-de-fases-de-desarrollo)
11. [Supuestos técnicos](#11-supuestos-técnicos)

---

## 1. Visión general del sistema

**Lumen Finanzas** es una aplicación de **planificación financiera personal mensual**, no un simple
registrador de gastos. Su diferenciador es comparar, mes a mes, el **resultado esperado** (planificado)
contra el **resultado real** (ejecutado), para medir **cumplimiento financiero** más que riqueza acumulada.

Problema que resuelve: la persona empieza cada mes con un **monto inicial** (saldo heredado del mes anterior
+ ingresos recurrentes como el sueldo), planifica cuánto gastará, ahorrará y cuánto debería sobrarle, y luego
la app le muestra si va cumpliendo, superando o incumpliendo esa meta.

Lógica principal del sistema:

- **Cuentas** guardan el dinero real (bancos, efectivo, billeteras, tarjetas) y participan en la planificación mensual.
- **Movimientos** son la **fuente central de verdad financiera**: todo ingreso, egreso, transferencia, pago
  de deuda, cobro de préstamo, aporte a meta, pago de suscripción o confirmación de recurrente/variable
  termina como una fila en `movements`.
- **Categorías/Subcategorías** clasifican los movimientos.
- **Pagos recurrentes** y **Suscripciones** se copian/renuevan automáticamente entre meses.
- **Pagos variables** existen solo en el mes en que se crean.
- **Préstamos por cobrar** (activos temporales) y **Deudas por pagar** (pasivos) afectan el saldo al crearse y al liquidarse.
- **Metas** acumulan aportes progresivos hacia un objetivo con fecha límite, midiendo si el avance va a ritmo.
- **Dashboard / Reportes** consolidan todo: esperado vs real, próximos vencimientos, progreso de metas, distribución por categorías.

Concepto transversal clave: el **período mensual** (`YYYY-MM`). El piloto trabaja fijo en `2026-05`
(`MONTH` en `lib/store.js`), pero el backend debe soportar navegación entre períodos. Los cálculos mensuales
consideran solo movimientos **confirmados** del período; los **pendientes** participan en la **proyección**
pero no alteran el saldo real.

---

## 2. Módulos detectados en el piloto

### 2.1 Cuentas
- **Propósito:** administrar cuentas financieras y su rol en la planificación mensual.
- **Pantallas:** `Accounts.jsx` (lista + tarjeta "añadir") y `AccountDetail` (detalle con panel de planificación
  esperado-vs-real, monto inicial, resultado esperado, badge de cumplimiento, 4 tarjetas de totales, historial filtrable).
- **Acciones:** listar, crear, ver detalle, editar, desactivar/eliminar, registrar ingreso/egreso/transferencia desde el detalle.
- **Datos del backend:** cuentas con saldo actual, tipo, banco, moneda, estado, última actualización; snapshot de
  planificación mensual (`heredado`, `ingresoRecurrente`, `montoInicial`, `esperado`, serie `months[{expected, real}]`);
  totales de movimientos por período.
- **Relaciones:** `users` (dueño), `movements` (historial), `monthly_plans` (planificación), origen/destino de transferencias.

### 2.2 Categorías
- **Propósito:** catálogo central para clasificar ingresos y egresos, con subcategorías.
- **Pantallas:** `Categories.jsx` (pestañas Egresos/Ingresos, tarjetas con icono, color, nº subcategorías y gastado en el mes).
- **Acciones implementadas:** listar por mes y tipo, crear, editar y eliminar con sus subcategorías. Importar/exportar queda para una fase futura.
- **Datos del backend:** categorías con `type`, `name`, `color`, `icon`; subcategorías; total mensual **solo de movimientos confirmados**.
- **Relaciones:** `subcategories`, `movements`, `recurring_payments`, `variable_payments`, `subscriptions`, `budgets`.

### 2.3 Pagos Recurrentes / Fijos
- **Propósito:** ingresos/egresos que se repiten y proyectan cada mes automáticamente.
- **Pantallas:** `Recurring.jsx` (5 KPIs: egresos proyectados/confirmados, ingresos proyectados/confirmados, próximo pago; subtabs Activos/Pausados/Finalizados; tarjetas con Confirmar/Pausar/Eliminar; Importar/Exportar).
- **Acciones:** crear, editar, confirmar ocurrencia (una sola vez por período), pausar, reactivar, finalizar, eliminar.
- **Datos del backend:** definición recurrente (nombre, tipo, monto, cuenta, categoría, frecuencia, próxima fecha,
  estado operativo) + ocurrencias por período con estado de cumplimiento.
- **Relaciones:** `accounts`, `categories`, `movements` (al confirmar genera un movimiento con `source_type=recurring`).

### 2.4 Pagos Variables / Randoms
- **Propósito:** gastos/ingresos planificados **solo para un mes específico**; no se copian al siguiente.
- **Pantallas:** `Variables.jsx` (5 KPIs análogos a recurrentes; subtabs Todos/Pendientes/Confirmados; tarjetas con Confirmar/Eliminar; Importar/Exportar).
- **Acciones:** listar por período, crear, editar, confirmar (una sola vez), cancelar, eliminar, importar, exportar.
- **Datos del backend:** movimiento planificado con período de pertenencia (`period` = `YYYY-MM`), tipo, monto, cuenta, categoría, subcategoría opcional, fecha, estado.
- **Relaciones:** `accounts`, `categories`, `movements` (al confirmar genera movimiento con `source_type=variable`).

### 2.5 Suscripciones
- **Propósito:** servicios/membresías con renovación periódica.
- **Pantallas:** `Subscriptions.jsx` (KPIs Total mensual, Pagadas este mes, Costo anual, Próxima renovación; subtabs Todas/Pendientes/Confirmadas/Pausadas; tarjetas con Confirmar pago/Pausar).
- **Acciones:** crear, editar, confirmar pago (genera egreso + reprograma próxima renovación + marca período pagado), pausar, reactivar, finalizar, eliminar.
- **Datos del backend:** suscripción (nombre, monto, cuenta, categoría, frecuencia, próxima renovación, estado operativo, último período pagado).
- **Relaciones:** `accounts`, `categories`, `movements` (confirmar genera movimiento con `source_type=subscription`).

### 2.6 Préstamos por Cobrar
- **Propósito:** dinero que el usuario prestó y espera recuperar (activo temporal).
- **Pantallas:** `Loans.jsx` (KPIs Por cobrar, Ya cobrado, Total prestado, Próximo a vencer; subtabs Activos/Cobrados/Todos; tarjetas con barra de progreso, Registrar cobro/Cobrar todo).
- **Acciones:** crear (opcionalmente descuenta de la cuenta al crear), editar, registrar cobro (parcial o total), confirmar devolución, eliminar.
- **Datos del backend:** préstamo (deudor, concepto, monto original, saldo por cobrar, cuenta origen, cuenta retorno, fecha préstamo, fecha estimada devolución, estado) + historial de cobros.
- **Relaciones:** `accounts`, `movements` (crear = egreso opcional; cobro = ingreso con `source_type=loan`), `loan_collections`.

### 2.7 Deudas por Pagar
- **Propósito:** dinero que el usuario recibió prestado y debe devolver (pasivo).
- **Pantallas:** `Debts.jsx` (KPIs Total que debes, Ya pagado, Cuota mensual, Próximo vencimiento; subtabs Activas/Pagadas/Todas; tarjetas con progreso, Registrar pago/Pagar cuota).
- **Acciones:** crear, editar, registrar pago (parcial/cuota/total), confirmar pago, eliminar.
- **Datos del backend:** deuda (acreedor, tipo, monto original, saldo pendiente, cuenta de pago, tasa mensual opcional, cuota estimada, fecha recepción, fecha de pago, estado) + historial de pagos.
- **Relaciones:** `accounts`, `movements` (crear = ingreso opcional; pago = egreso con `source_type=debt`), `debt_payments`.

### 2.8 Metas
- **Propósito:** objetivos financieros con aportes progresivos hacia un monto y fecha límite.
- **Pantallas:** `Goals.jsx` (KPIs Total ahorrado, Metas activas, Aporte del mes; tarjetas con progreso, aporte mensual sugerido, Registrar aporte/Pausar) y `GoalDetail` (info general, KPIs Ahorrado/Falta/Aporte sugerido/Ritmo, gráfico esperado-vs-real, historial de aportes).
- **Acciones:** crear, editar, pausar/reactivar, completar (automático al alcanzar objetivo), registrar aporte, ver historial, ver detalle.
- **Datos del backend:** meta (nombre, tipo, monto objetivo, monto actual, fecha límite, estado, salud/ritmo, cuenta destino opcional) + historial de aportes; cálculo de aporte mensual requerido y esperado-a-la-fecha.
- **Relaciones:** `accounts`, `movements` (aporte = transferencia con `source_type=goal`), `goal_contributions`.

### 2.9 Movimientos
- **Propósito:** libro diario global; **fuente central de verdad** de toda actividad financiera.
- **Pantallas:** `Movements.jsx` (accesos rápidos Ingreso/Gasto/Transferir; KPIs Total ingresos/egresos/transferencias; filtros: búsqueda global, tipo, cuenta, categoría, estado; tabla con Confirmar / Ir al origen / Eliminar).
- **Acciones:** crear ingreso/egreso/transferencia, confirmar, cancelar, eliminar, filtrar, navegar al módulo de origen (préstamo/deuda/meta/suscripción/variable/recurrente).
- **Datos del backend:** movimientos del período con cuenta, categoría, tipo, estado, monto, descripción y vínculo a su origen (`source_type` + `source_id`).
- **Relaciones:** recibe de **todos** los módulos; alimenta cuentas, categorías, dashboard y reportes.

### 2.10 Dashboard / Reportes
- **Propósito:** consolidación y visualización.
- **Pantallas:** `Dashboard.jsx` (saludo, KPIs, gráficos ingreso/egreso, distribución por categoría, próximos vencimientos, progreso de metas), `Reports.jsx`, `Budgets.jsx` (presupuestos por categoría), `Calendar.jsx` (vencimientos en calendario).
- **Acciones:** solo lectura (agregaciones); presupuestos permiten CRUD.
- **Datos del backend:** agregados mensuales (ingresos/egresos confirmados, saldo total, ahorro, esperado vs real), series temporales, distribución por categoría, próximos pagos/renovaciones/vencimientos, progreso de metas.
- **Relaciones:** lee de todas las tablas; `budgets` referencia `categories`.

---

## 3. Modelo de base de datos propuesto

Convenciones globales:

- **PK:** `id UUID` (default `gen_random_uuid()`).
- **Tenant:** `user_id UUID` en todas las tablas de negocio (multi-usuario).
- **Timestamps:** `created_at TIMESTAMPTZ NOT NULL DEFAULT now()`, `updated_at TIMESTAMPTZ NOT NULL DEFAULT now()`.
- **Soft delete:** `deleted_at TIMESTAMPTZ NULL` en entidades con historial (categorías, subcategorías, movimientos, recurrentes, suscripciones, préstamos, deudas, metas). La implementación actual de cuentas no usa `deleted_at`.
- **Dinero:** `DECIMAL(14,2)` (nunca float).
- **Período:** `period CHAR(7)` con formato `YYYY-MM`.
- **Índice implícito** en toda FK; se listan los índices adicionales relevantes.

---

### Tabla: users

Descripción: usuario autenticado, dueño de todos los datos financieros.

Campos:

| Campo          | Tipo         | Requerido | Descripción                          |
| -------------- | ------------ | --------- | ------------------------------------ |
| id             | UUID         | Sí        | Identificador único                  |
| name           | VARCHAR(120) | Sí        | Nombre para mostrar                  |
| email          | VARCHAR(180) | Sí        | Correo único                         |
| password_hash  | TEXT         | Sí        | Hash de contraseña (bcrypt/argon2)   |
| base_currency  | currency     | Sí        | Moneda base (default `PEN`)          |
| savings_target | DECIMAL(14,2)| No        | Meta de sobrante mensual configurada |
| avatar         | VARCHAR(8)   | No        | Iniciales/emoji de avatar            |
| created_at     | TIMESTAMPTZ  | Sí        | Fecha de creación                    |
| updated_at     | TIMESTAMPTZ  | Sí        | Última actualización                 |

- **Relaciones:** 1—N con accounts, categories, movements, recurring_payments, variable_payments, subscriptions, loans, debts, goals, budgets, monthly_plans.
- **Índices:** `UNIQUE(email)`.
- **Reglas:** `email` normalizado a minúsculas; nunca exponer `password_hash`.

---

### Tabla: accounts

Descripción: cuentas financieras del usuario (banco, ahorros, billetera, efectivo, tarjeta de crédito, inversión).

Campos:

| Campo           | Tipo           | Requerido | Descripción                                        |
| --------------- | -------------- | --------- | -------------------------------------------------- |
| id              | INT            | Sí        | Identificador único                                |
| user_id         | INT            | Sí        | FK → users                                         |
| name            | VARCHAR(120)   | Sí        | Nombre (ej. "BCP Sueldo")                          |
| type            | account_type   | Sí        | Tipo de cuenta                                     |
| institution     | VARCHAR(120)   | No        | Banco/entidad (ej. "BCP", "Yape")                 |
| currency        | currency       | Sí        | Moneda de la cuenta (default `PEN`)                |
| initial_balance | DECIMAL(14,2)  | Sí        | Saldo inicial al crear                             |
| current_balance | DECIMAL(14,2)  | Sí        | Saldo actual; inicia con el saldo inicial en cuentas normales |
| expected_monthly_amount | DECIMAL(14,2) | Sí      | Preparado para planificación; aún no se modifica desde este módulo |
| credit_limit    | DECIMAL(14,2)  | No        | Límite (solo `CREDIT_CARD`)                        |
| available_credit| DECIMAL(14,2)  | No        | Crédito disponible (solo `CREDIT_CARD`)            |
| is_active       | BOOLEAN        | Sí        | Cuenta activa (default `true`)                      |
| is_primary      | BOOLEAN        | Sí        | Cuenta principal del usuario (default `false`)      |
| created_at      | TIMESTAMPTZ    | Sí        | —                                                  |
| updated_at      | TIMESTAMPTZ    | Sí        | —                                                  |

- **Relaciones actuales:** N—1 users. La relación con movements se agregará con ese módulo.
- **Índices:** `(user_id)`, `(user_id, is_active)`, `(user_id, is_primary)`, `(user_id, type)` y nombre único por usuario.
- **Reglas actuales:** al crear una cuenta normal, `current_balance` inicia con `initial_balance`.
  Las tarjetas guardan saldo inicial y actual en cero, requieren `credit_limit` y, si no se indica
  `available_credit`, este inicia con el mismo valor del límite. La primera cuenta del usuario se marca
  como principal y solo una cuenta por usuario puede ser principal.
- **Eliminación actual:** una cuenta no principal puede eliminarse físicamente. Cuando exista el módulo
  Movimientos se deberá bloquear la eliminación de cuentas con historial.
- **Enums actuales:** `BANK_ACCOUNT`, `SAVINGS`, `CASH`, `DIGITAL_WALLET`, `CREDIT_CARD`, `INVESTMENT`, `OTHER`.

---

### Tabla: categories

Descripción: catálogo de categorías de ingreso/egreso.

Campos:

| Campo      | Tipo          | Requerido | Descripción                        |
| ---------- | ------------- | --------- | ---------------------------------- |
| id         | INT           | Sí        | Identificador único actual         |
| user_id    | INT           | Sí        | FK → users                         |
| period     | CHAR(7)       | Sí        | Mes independiente `YYYY-MM`        |
| type       | category_type | Sí        | `income` o `expense`               |
| name       | VARCHAR(80)   | Sí        | Nombre                             |
| color      | VARCHAR(9)    | No        | Color hex                          |
| icon       | VARCHAR(40)   | No        | Clave de icono de UI               |
| is_active  | BOOLEAN       | Sí        | Activa (default `true`)            |
| sort_order | INT           | No        | Orden de despliegue                |
| created_at | TIMESTAMPTZ   | Sí        | —                                  |
| updated_at | TIMESTAMPTZ   | Sí        | —                                  |

- **Relaciones:** 1—N subcategories; referenciada por movements, recurring_payments, variable_payments, subscriptions, budgets.
- **Índices:** `(user_id, period)`, `(user_id, type, is_active)`, único `(user_id, period, type, name)`.
- **Reglas actuales:** el primer acceso a un mes copia el último periodo anterior. El borrado afecta solo ese mes y se bloquea si existen movimientos visibles que usan la categoría.
- **Enums:** `category_type`.

### Tabla: category_periods

Marca que las categorías de un usuario ya fueron inicializadas para un mes. Usa `id INT`, `user_id INT`, `period CHAR(7)` e `initialized_at TIMESTAMPTZ`, con único `(user_id, period)`. Evita volver a copiar una categoría eliminada del mismo mes.

---

### Tabla: subcategories

Descripción: subdivisión de una categoría (ej. "Servicios → Luz, Agua, Internet").

Campos:

| Campo       | Tipo         | Requerido | Descripción            |
| ----------- | ------------ | --------- | ---------------------- |
| id          | UUID         | Sí        | Identificador único    |
| category_id | UUID         | Sí        | FK → categories        |
| name        | VARCHAR(80)  | Sí        | Nombre de subcategoría |
| is_active   | BOOLEAN      | Sí        | Activa (default `true`)|
| sort_order  | INT          | No        | Orden                  |
| created_at  | TIMESTAMPTZ  | Sí        | —                      |
| updated_at  | TIMESTAMPTZ  | Sí        | —                      |

- **Relaciones:** N—1 categories; referenciada por movements (`subcategory_id`).
- **Índices:** `(category_id, is_active)`, `UNIQUE(category_id, name)`.

---

### Tabla: movements  ⭐ fuente central de verdad

Descripción: todo movimiento monetario del sistema. Cualquier acción de cualquier módulo que mueva dinero crea una fila aquí.

Campos:

| Campo             | Tipo            | Requerido | Descripción                                                     |
| ----------------- | --------------- | --------- | --------------------------------------------------------------- |
| id                | INT             | Sí        | Identificador único                                             |
| user_id           | INT             | Sí        | FK → users                                                      |
| type              | movement_type   | Sí        | `income`, `expense`, `transfer`                                 |
| status            | movement_status | Sí        | `pending`, `confirmed`, `cancelled` (default actual `pending`)  |
| expected_amount   | DECIMAL(14,2)   | Sí        | Monto previsto positivo                                         |
| actual_amount     | DECIMAL(14,2)   | No        | Monto real; nulo mientras está pendiente                        |
| account_id        | UUID            | Sí        | FK → accounts (origen; en income = cuenta destino real)         |
| to_account_id     | UUID            | No        | FK → accounts (destino; requerido si `type=transfer`)           |
| category_id       | UUID            | No        | FK → categories                                                 |
| subcategory_id    | UUID            | No        | FK → subcategories                                              |
| period            | CHAR(7)         | Sí        | `YYYY-MM` del movimiento (derivado de `occurred_on`)            |
| occurred_on       | DATE            | Sí        | Fecha del movimiento                                            |
| occurred_at       | TIMESTAMPTZ     | No        | Fecha/hora exacta si aplica                                     |
| description       | VARCHAR(240)    | No        | Descripción libre                                              |
| currency          | currency        | Sí        | Moneda (hereda de la cuenta)                                    |
| source_type       | source_type     | Sí        | Origen: `manual`, `recurring`, `variable`, `subscription`, `loan`, `debt`, `goal`, `transfer` |
| source_id         | UUID            | No        | ID del registro de origen (recurring/variable/subscription/loan/debt/goal) |
| deleted_at        | TIMESTAMPTZ     | No        | Soft delete                                                    |
| created_at        | TIMESTAMPTZ     | Sí        | —                                                              |
| updated_at        | TIMESTAMPTZ     | Sí        | —                                                              |

- **Relaciones:** N—1 users, accounts (origen), accounts (destino), categories, subcategories; polimórfica a origen vía (`source_type`, `source_id`).
- **Índices:** `(user_id, period)`, `(account_id, period)`, `(to_account_id)`, `(category_id)`, `(user_id, type, status)`, `(source_type, source_id)`.
- **Reglas:**
  - Solo `status='confirmed'` afecta `current_balance` (ver §6).
  - `type='transfer'` requiere `to_account_id ≠ account_id`.
  - Ingresos y egresos nacen pendientes. Al confirmar se guarda `actual_amount` sin cambiar `expected_amount`.
  - El saldo usa `actual_amount`; ambos montos son positivos y el signo visual lo determina `type`.
  - Nunca borrado físico si se requiere auditoría histórica: preferir `deleted_at` o `status='cancelled'`.
- **Enums:** `movement_type`, `movement_status`, `source_type`, `currency`.
- **Ejemplo:** un pago de deuda genera `{ type:'expense', source_type:'debt', source_id:<debt_id>, category_id:<'Deudas'> }`.

---

### Tabla: recurring_payments

Descripción: definición de un ingreso/egreso recurrente que se proyecta cada período.

Campos:

| Campo             | Tipo              | Requerido | Descripción                                     |
| ----------------- | ----------------- | --------- | ----------------------------------------------- |
| id                | UUID              | Sí        | Identificador único                             |
| user_id           | UUID              | Sí        | FK → users                                       |
| type              | movement_type     | Sí        | `income` o `expense` (nunca `transfer`)          |
| name              | VARCHAR(120)      | Sí        | Nombre (ej. "Alquiler", "Sueldo")               |
| amount            | DECIMAL(14,2)     | Sí        | Monto estimado                                   |
| account_id        | UUID              | Sí        | FK → accounts                                    |
| category_id       | UUID              | No        | FK → categories                                  |
| frequency         | payment_frequency | Sí        | Frecuencia                                       |
| next_date         | DATE              | Sí        | Próxima fecha programada                         |
| status            | recurring_status  | Sí        | `active`, `paused`, `finalized` (default `active`)|
| last_confirmed    | CHAR(7)           | No        | Último período confirmado (`YYYY-MM`)            |
| deleted_at        | TIMESTAMPTZ       | No        | Soft delete                                      |
| created_at        | TIMESTAMPTZ       | Sí        | —                                                |
| updated_at        | TIMESTAMPTZ       | Sí        | —                                                |

- **Relaciones:** N—1 accounts, categories; 1—N recurring_occurrences; genera movements (`source_type=recurring`).
- **Índices:** `(user_id, status)`, `(user_id, next_date)`.
- **Reglas:** al iniciar un período, los `active` generan (o proyectan) una ocurrencia. Confirmar solo una vez por período (`last_confirmed`). `finalized`/`paused` no proyectan.
- **Enums:** `payment_frequency`, `recurring_status`, `movement_type`.

---

### Tabla: recurring_occurrences

Descripción: instancia por período de un pago recurrente (permite confirmar una sola vez por período y consultar "ocurrencias del mes").

Campos:

| Campo         | Tipo               | Requerido | Descripción                              |
| ------------- | ------------------ | --------- | ---------------------------------------- |
| id            | UUID               | Sí        | Identificador único                      |
| recurring_id  | UUID               | Sí        | FK → recurring_payments                  |
| user_id       | UUID               | Sí        | FK → users                               |
| period        | CHAR(7)            | Sí        | `YYYY-MM`                                |
| due_date      | DATE               | Sí        | Fecha programada en ese período          |
| amount        | DECIMAL(14,2)      | Sí        | Monto proyectado                         |
| status        | fulfillment_status | Sí        | `pending` o `confirmed`                  |
| movement_id   | UUID               | No        | FK → movements (creado al confirmar)     |
| confirmed_at  | TIMESTAMPTZ        | No        | —                                        |
| created_at    | TIMESTAMPTZ        | Sí        | —                                        |
| updated_at    | TIMESTAMPTZ        | Sí        | —                                        |

- **Índices:** `UNIQUE(recurring_id, period)`, `(user_id, period, status)`.
- **Reglas:** confirmar cambia a `confirmed`, crea el `movement` y reprograma `recurring_payments.next_date`.

---

### Tabla: variable_payments

Descripción: gasto/ingreso planificado que pertenece **solo** a un período (no se copia al siguiente).

Campos:

| Campo           | Tipo               | Requerido | Descripción                          |
| --------------- | ------------------ | --------- | ------------------------------------ |
| id              | UUID               | Sí        | Identificador único                  |
| user_id         | UUID               | Sí        | FK → users                           |
| type            | movement_type      | Sí        | `income` o `expense`                 |
| name            | VARCHAR(120)       | Sí        | Nombre (ej. "Comprar zapatillas")   |
| amount          | DECIMAL(14,2)      | Sí        | Monto                                |
| account_id      | UUID               | Sí        | FK → accounts                        |
| category_id     | UUID               | No        | FK → categories                      |
| subcategory_id  | UUID               | No        | FK → subcategories                   |
| period          | CHAR(7)            | Sí        | Período de pertenencia (`YYYY-MM`)   |
| planned_date    | DATE               | Sí        | Fecha prevista                       |
| status          | fulfillment_status | Sí        | `pending` o `confirmed`              |
| description     | VARCHAR(240)       | No        | Descripción                         |
| movement_id     | UUID               | No        | FK → movements (al confirmar)        |
| deleted_at      | TIMESTAMPTZ        | No        | Soft delete                          |
| created_at      | TIMESTAMPTZ        | Sí        | —                                    |
| updated_at      | TIMESTAMPTZ        | Sí        | —                                    |

- **Índices:** `(user_id, period, status)`.
- **Reglas:** confirmar (una vez) crea `movement` (`source_type=variable`) y marca `confirmed`. Al cambiar de mes no se replica.
- **Enums:** `movement_type`, `fulfillment_status`.

---

### Tabla: subscriptions

Descripción: servicios/membresías con renovación periódica.

Campos:

| Campo         | Tipo                | Requerido | Descripción                                  |
| ------------- | ------------------- | --------- | -------------------------------------------- |
| id            | UUID                | Sí        | Identificador único                          |
| user_id       | UUID                | Sí        | FK → users                                   |
| name          | VARCHAR(120)        | Sí        | Nombre (ej. "Streaming Plus")               |
| amount        | DECIMAL(14,2)       | Sí        | Monto de cobro                               |
| account_id    | UUID                | Sí        | FK → accounts                                |
| category_id   | UUID                | No        | FK → categories (default categoría Suscripciones) |
| frequency     | payment_frequency   | Sí        | Frecuencia de renovación                     |
| next_renewal  | DATE                | Sí        | Próxima renovación                           |
| status        | subscription_status | Sí        | `active`, `paused`, `finalized`              |
| last_paid     | CHAR(7)             | No        | Último período pagado (`YYYY-MM`)            |
| color         | VARCHAR(9)          | No        | Color de UI                                  |
| initial       | VARCHAR(4)          | No        | Inicial mostrada                             |
| deleted_at    | TIMESTAMPTZ         | No        | Soft delete                                  |
| created_at    | TIMESTAMPTZ         | Sí        | —                                            |
| updated_at    | TIMESTAMPTZ         | Sí        | —                                            |

- **Relaciones:** N—1 accounts, categories; 1—N subscription_renewals; genera movements (`source_type=subscription`).
- **Índices:** `(user_id, status)`, `(user_id, next_renewal)`.
- **Reglas:** confirmar pago genera egreso, setea `last_paid=<período>` y avanza `next_renewal` según `frequency`. Una confirmación por período.
- **Enums:** `subscription_status`, `payment_frequency`.

---

### Tabla: subscription_renewals

Descripción: registro de cada renovación/pago confirmado de una suscripción (histórico y control por período).

Campos:

| Campo           | Tipo               | Requerido | Descripción                    |
| --------------- | ------------------ | --------- | ------------------------------ |
| id              | UUID               | Sí        | Identificador único            |
| subscription_id | UUID               | Sí        | FK → subscriptions             |
| user_id         | UUID               | Sí        | FK → users                     |
| period          | CHAR(7)            | Sí        | `YYYY-MM`                      |
| amount          | DECIMAL(14,2)      | Sí        | Monto cobrado                  |
| status          | fulfillment_status | Sí        | `pending` / `confirmed`        |
| movement_id     | UUID               | No        | FK → movements                 |
| paid_at         | TIMESTAMPTZ        | No        | —                              |
| created_at      | TIMESTAMPTZ        | Sí        | —                              |

- **Índices:** `UNIQUE(subscription_id, period)`.

---

### Tabla: loans  (préstamos por cobrar)

Descripción: dinero prestado a terceros que se espera recuperar.

Campos:

| Campo             | Tipo          | Requerido | Descripción                                 |
| ----------------- | ------------- | --------- | ------------------------------------------- |
| id                | UUID          | Sí        | Identificador único                         |
| user_id           | UUID          | Sí        | FK → users                                  |
| debtor            | VARCHAR(120)  | Sí        | Persona/entidad que debe                    |
| concept           | VARCHAR(160)  | No        | Concepto (ej. "Préstamo personal")         |
| original_amount   | DECIMAL(14,2) | Sí        | Monto prestado                              |
| outstanding       | DECIMAL(14,2) | Sí        | Pendiente por cobrar                        |
| source_account_id | UUID          | Sí        | FK → accounts (de dónde salió el dinero)    |
| return_account_id | UUID          | No        | FK → accounts (dónde entra al cobrarse)     |
| loan_date         | DATE          | Sí        | Fecha de entrega                            |
| due_date          | DATE          | No        | Fecha estimada de devolución                |
| status            | loan_status   | Sí        | `active`, `collected`                       |
| description       | VARCHAR(240)  | No        | Notas                                       |
| color             | VARCHAR(9)    | No        | Color de UI                                 |
| deleted_at        | TIMESTAMPTZ   | No        | Soft delete                                 |
| created_at        | TIMESTAMPTZ   | Sí        | —                                           |
| updated_at        | TIMESTAMPTZ   | Sí        | —                                           |

- **Relaciones:** N—1 accounts (origen y retorno); 1—N loan_collections; genera movements.
- **Índices:** `(user_id, status)`, `(user_id, due_date)`.
- **Reglas:** crear con "descontar ahora" genera egreso desde `source_account_id`. Cada cobro genera ingreso a `return_account_id` (o al origen) con `source_type=loan`. Al llegar `outstanding=0` → `status='collected'`.
- **Enums:** `loan_status`.

---

### Tabla: loan_collections

Descripción: historial de cobros de un préstamo.

| Campo         | Tipo          | Requerido | Descripción            |
| ------------- | ------------- | --------- | ---------------------- |
| id            | UUID          | Sí        | Identificador único    |
| loan_id       | UUID          | Sí        | FK → loans             |
| user_id       | UUID          | Sí        | FK → users             |
| amount        | DECIMAL(14,2) | Sí        | Monto cobrado          |
| collected_on  | DATE          | Sí        | Fecha de cobro         |
| account_id    | UUID          | Sí        | FK → accounts (destino)|
| movement_id   | UUID          | No        | FK → movements         |
| created_at    | TIMESTAMPTZ   | Sí        | —                      |

- **Índices:** `(loan_id)`.

---

### Tabla: debts  (deudas por pagar)

Descripción: dinero recibido en préstamo que se debe devolver.

Campos:

| Campo           | Tipo          | Requerido | Descripción                               |
| --------------- | ------------- | --------- | ----------------------------------------- |
| id              | UUID          | Sí        | Identificador único                       |
| user_id         | UUID          | Sí        | FK → users                                |
| creditor        | VARCHAR(120)  | Sí        | Acreedor (persona/banco/tienda)           |
| type            | VARCHAR(60)   | No        | Tipo (tarjeta, préstamo bancario, personal…)|
| original_amount | DECIMAL(14,2) | Sí        | Monto total de la deuda                    |
| balance         | DECIMAL(14,2) | Sí        | Saldo pendiente                            |
| account_id      | UUID          | Sí        | FK → accounts (cuenta de pago)             |
| receive_account_id | UUID       | No        | FK → accounts (cuenta donde ingresó al recibir)|
| rate_monthly    | DECIMAL(6,3)  | No        | Tasa mensual % (opcional)                  |
| installment     | DECIMAL(14,2) | No        | Cuota estimada                             |
| received_on     | DATE          | No        | Fecha de recepción                         |
| due_date        | DATE          | No        | Próximo vencimiento / fecha de pago        |
| status          | debt_status   | Sí        | `active`, `paid`                           |
| description     | VARCHAR(240)  | No        | Notas                                      |
| color           | VARCHAR(9)    | No        | Color UI                                   |
| deleted_at      | TIMESTAMPTZ   | No        | Soft delete                                |
| created_at      | TIMESTAMPTZ   | Sí        | —                                          |
| updated_at      | TIMESTAMPTZ   | Sí        | —                                          |

- **Relaciones:** N—1 accounts; 1—N debt_payments; genera movements (`source_type=debt`).
- **Índices:** `(user_id, status)`, `(user_id, due_date)`.
- **Reglas:** crear con "recibir ahora" genera ingreso a `receive_account_id`. Cada pago genera egreso desde `account_id`. `balance=0` → `status='paid'`.
- **Enums:** `debt_status`.

---

### Tabla: debt_payments

Descripción: historial de pagos de una deuda.

| Campo       | Tipo          | Requerido | Descripción          |
| ----------- | ------------- | --------- | -------------------- |
| id          | UUID          | Sí        | Identificador único  |
| debt_id     | UUID          | Sí        | FK → debts           |
| user_id     | UUID          | Sí        | FK → users           |
| amount      | DECIMAL(14,2) | Sí        | Monto pagado         |
| paid_on     | DATE          | Sí        | Fecha de pago        |
| account_id  | UUID          | Sí        | FK → accounts (origen)|
| movement_id | UUID          | No        | FK → movements       |
| created_at  | TIMESTAMPTZ   | Sí        | —                    |

- **Índices:** `(debt_id)`.

---

### Tabla: goals

Descripción: objetivo financiero con progreso hacia un monto y fecha límite.

Campos:

| Campo            | Tipo          | Requerido | Descripción                                    |
| ---------------- | ------------- | --------- | ---------------------------------------------- |
| id               | UUID          | Sí        | Identificador único                            |
| user_id          | UUID          | Sí        | FK → users                                     |
| name             | VARCHAR(120)  | Sí        | Nombre                                         |
| type             | goal_type     | Sí        | Tipo de meta                                   |
| target_amount    | DECIMAL(14,2) | Sí        | Monto objetivo                                 |
| current_amount   | DECIMAL(14,2) | Sí        | Acumulado (default 0)                          |
| deadline         | DATE          | No        | Fecha límite                                   |
| account_id       | UUID          | No        | FK → accounts (dónde se guarda el dinero)      |
| status           | goal_status   | Sí        | `active`, `completed`, `paused`, `overdue`     |
| health           | goal_health   | No        | Ritmo: `on_track`, `behind`, `completed`       |
| description      | VARCHAR(240)  | No        | Notas                                          |
| deleted_at       | TIMESTAMPTZ   | No        | Soft delete                                    |
| created_at       | TIMESTAMPTZ   | Sí        | —                                              |
| updated_at       | TIMESTAMPTZ   | Sí        | —                                              |

- **Relaciones:** N—1 accounts; 1—N goal_contributions; genera movements (`source_type=goal`, `type=transfer`).
- **Índices:** `(user_id, status)`, `(user_id, deadline)`.
- **Reglas:** cada aporte suma a `current_amount`. Al alcanzar `target_amount` → `status='completed'`. `deadline` pasada sin completar → `overdue`. Aporte mensual requerido = `(target-current)/meses_restantes` (calculado en servicio, no persistido).
- **Enums:** `goal_type`, `goal_status`, `goal_health`.

---

### Tabla: goal_contributions

Descripción: historial de aportes a una meta.

| Campo             | Tipo          | Requerido | Descripción                     |
| ----------------- | ------------- | --------- | ------------------------------- |
| id                | UUID          | Sí        | Identificador único             |
| goal_id           | UUID          | Sí        | FK → goals                      |
| user_id           | UUID          | Sí        | FK → users                      |
| amount            | DECIMAL(14,2) | Sí        | Monto aportado                  |
| contributed_on    | DATE          | Sí        | Fecha del aporte                |
| from_account_id   | UUID          | Sí        | FK → accounts (origen)          |
| to_account_id     | UUID          | No        | FK → accounts (destino/ahorro)  |
| movement_id       | UUID          | No        | FK → movements                  |
| description       | VARCHAR(240)  | No        | Notas                           |
| created_at        | TIMESTAMPTZ   | Sí        | —                               |

- **Índices:** `(goal_id)`.

---

### Tabla: budgets

Descripción: presupuesto/límite mensual (general o por categoría).

Campos:

| Campo       | Tipo          | Requerido | Descripción                                 |
| ----------- | ------------- | --------- | ------------------------------------------- |
| id          | UUID          | Sí        | Identificador único                         |
| user_id     | UUID          | Sí        | FK → users                                  |
| scope       | budget_scope  | Sí        | `general` o `category`                      |
| category_id | UUID          | No        | FK → categories (requerido si `scope=category`)|
| name        | VARCHAR(80)   | Sí        | Nombre                                      |
| amount      | DECIMAL(14,2) | Sí        | Límite mensual                              |
| period      | CHAR(7)       | No        | Período (null = recurrente cada mes)        |
| created_at  | TIMESTAMPTZ   | Sí        | —                                           |
| updated_at  | TIMESTAMPTZ   | Sí        | —                                           |

- **Reglas:** `used` (gastado) se calcula en runtime desde movimientos confirmados del período, no se persiste.
- **Enums:** `budget_scope`.

---

### Tabla: monthly_plans

Descripción: snapshot de planificación por cuenta y período (soporta el gráfico esperado-vs-real del detalle de cuenta y del dashboard).

Campos:

| Campo              | Tipo          | Requerido | Descripción                                       |
| ------------------ | ------------- | --------- | ------------------------------------------------- |
| id                 | UUID          | Sí        | Identificador único                               |
| user_id            | UUID          | Sí        | FK → users                                        |
| account_id         | UUID          | No        | FK → accounts (null = plan global del usuario)    |
| period             | CHAR(7)       | Sí        | `YYYY-MM`                                          |
| carried_over       | DECIMAL(14,2) | Sí        | Saldo heredado del mes anterior                   |
| recurring_income   | DECIMAL(14,2) | Sí        | Ingresos recurrentes esperados                    |
| initial_amount     | DECIMAL(14,2) | Sí        | `carried_over + recurring_income`                 |
| planned_expenses   | DECIMAL(14,2) | Sí        | Gastos previstos                                  |
| planned_savings    | DECIMAL(14,2) | Sí        | Ahorros previstos                                 |
| planned_debts      | DECIMAL(14,2) | Sí        | Deudas previstas                                  |
| expected_result    | DECIMAL(14,2) | Sí        | `initial - planned_expenses - savings - debts`    |
| real_result        | DECIMAL(14,2) | No        | Resultado real al cierre (calculado)              |
| created_at         | TIMESTAMPTZ   | Sí        | —                                                 |
| updated_at         | TIMESTAMPTZ   | Sí        | —                                                 |

- **Índices:** `UNIQUE(user_id, account_id, period)`, `(user_id, period)`.
- **Reglas:** `expected_result` fijo una vez iniciado el período; `real_result` se recalcula con movimientos confirmados. Cumplimiento = `real_result - expected_result`.

---

### Tabla: refresh_tokens

Descripción: tokens de refresco para JWT (o sesiones).

| Campo       | Tipo         | Requerido | Descripción                    |
| ----------- | ------------ | --------- | ------------------------------ |
| id          | UUID         | Sí        | Identificador único            |
| user_id     | UUID         | Sí        | FK → users                     |
| token_hash  | TEXT         | Sí        | Hash del refresh token         |
| expires_at  | TIMESTAMPTZ  | Sí        | Expiración                     |
| revoked_at  | TIMESTAMPTZ  | No        | Revocación (logout)            |
| created_at  | TIMESTAMPTZ  | Sí        | —                              |

- **Índices:** `(user_id)`, `UNIQUE(token_hash)`.

---

## 4. Relaciones entre tablas

Relaciones principales:

- Un **usuario** tiene muchas cuentas, categorías, movimientos, recurrentes, variables, suscripciones, préstamos, deudas, metas, presupuestos y planes.
- Una **cuenta** tiene muchos movimientos (como origen y como destino) y muchos planes mensuales.
- Un **movimiento** pertenece opcionalmente a una categoría y subcategoría, y referencia su origen vía (`source_type`, `source_id`).
- Una **categoría** tiene muchas subcategorías y es referenciada por movimientos, recurrentes, variables, suscripciones y presupuestos.
- Un **pago recurrente** genera muchas ocurrencias (una por período); cada ocurrencia confirmada crea un movimiento.
- Un **pago variable** pertenece a un solo período; al confirmarse crea un movimiento.
- Una **suscripción** genera muchas renovaciones; cada pago confirmado crea un movimiento.
- Un **préstamo por cobrar** tiene muchos cobros; crear puede generar un egreso, cada cobro genera un ingreso.
- Una **deuda por pagar** tiene muchos pagos; recibir genera un ingreso, cada pago genera un egreso.
- Una **meta** tiene muchos aportes; cada aporte genera una transferencia.
- Un **plan mensual** pertenece a un usuario (y opcionalmente a una cuenta) por período.

Diagrama textual:

```
users
├── accounts
│   ├── movements (account_id / to_account_id)
│   └── monthly_plans
├── categories
│   ├── subcategories
│   │   └── movements (subcategory_id)
│   └── movements (category_id)
├── movements  ⭐ (fuente central; source_type + source_id → origen)
├── recurring_payments
│   └── recurring_occurrences ──▶ movements
├── variable_payments ──▶ movements
├── subscriptions
│   └── subscription_renewals ──▶ movements
├── loans
│   └── loan_collections ──▶ movements
├── debts
│   └── debt_payments ──▶ movements
├── goals
│   └── goal_contributions ──▶ movements
├── budgets ──▶ categories
└── refresh_tokens
```

---

## 5. Enums del sistema

| Enum                  | Valores                                                                                  | Significado |
| --------------------- | ---------------------------------------------------------------------------------------- | ----------- |
| `account_type`        | `bank`, `savings`, `digital_wallet`, `cash`, `credit_card`, `investment`, `other`        | Tipo de cuenta financiera |
| `account_status`      | `active`, `inactive`, `archived`                                                         | Estado de la cuenta |
| `currency`            | `PEN`, `USD`                                                                             | Moneda (base `PEN`, preparado multi-moneda) |
| `movement_type`       | `income`, `expense`, `transfer`                                                          | Naturaleza del movimiento |
| `movement_status`     | `pending`, `confirmed`, `cancelled`                                                      | `pending` proyecta; `confirmed` afecta saldo; `cancelled` anulado |
| `category_type`       | `income`, `expense`                                                                      | Clasificación de la categoría |
| `payment_frequency`   | `daily`, `weekly`, `biweekly`, `monthly`, `quarterly`, `annual`                          | Periodicidad de recurrentes/suscripciones |
| `recurring_status`    | `active`, `paused`, `finalized`                                                          | Estado operativo del recurrente |
| `fulfillment_status`  | `pending`, `confirmed`                                                                   | Cumplimiento de una ocurrencia/variable/renovación |
| `subscription_status` | `active`, `paused`, `finalized`                                                          | Estado operativo de la suscripción |
| `loan_status`         | `active`, `collected`                                                                    | Préstamo por cobrar activo o ya recuperado |
| `debt_status`         | `active`, `paid`                                                                         | Deuda pendiente o liquidada |
| `goal_type`           | `saving`, `emergency_fund`, `debt_payoff`, `specific_purchase`, `investment`, `savings_replenishment`, `custom` | Tipo de objetivo |
| `goal_status`         | `active`, `completed`, `paused`, `overdue`                                               | Estado de la meta |
| `goal_health`         | `on_track`, `behind`, `completed`                                                        | Ritmo respecto al plan |
| `source_type`         | `manual`, `recurring`, `variable`, `subscription`, `loan`, `debt`, `goal`, `transfer`   | Módulo de origen del movimiento |
| `budget_scope`        | `general`, `category`                                                                    | Alcance del presupuesto |

> Nota de mapeo con el piloto: el mock usa `active/paused/cancelled` para suscripciones, `paid` para deudas,
> `collected` para préstamos, `on-track/behind/completed` para salud de metas. El backend normaliza a los valores de arriba.

---

## 6. Reglas de negocio importantes

1. **Movimientos confirmados afectan el saldo real.** Solo `movements.status='confirmed'` modifica `accounts.current_balance`.
2. **Movimientos pendientes participan en la planificación pero no en el saldo real.** Se usan para proyectar el resultado esperado, no para el saldo.
3. **Signo por tipo.** `income` suma a la cuenta; `expense` resta; `transfer` resta en `account_id` y suma en `to_account_id`.
4. **Transferencias** deben crear salida en origen y entrada en destino de forma atómica; `to_account_id ≠ account_id`.
5. **Categorías mensuales.** Se copian al inicializar un mes y luego quedan independientes. El borrado solo afecta ese periodo y se bloquea mientras existan movimientos visibles que las usen.
6. **Total mensual por categoría** = suma de movimientos `confirmed` del período; ignora `pending` y `cancelled`.
7. **Pagos recurrentes activos generan ocurrencias** por período; confirmar una ocurrencia crea el movimiento, marca `last_confirmed` y avanza `next_date`. **Solo se confirma una vez por período** según la frecuencia.
8. **Pagos variables pertenecen solo a su período**; al cambiar de mes no se replican. Confirmar (una vez) crea el movimiento.
9. **Suscripciones activas generan renovaciones**; confirmar el pago crea un egreso, setea `last_paid=<período>` y reprograma `next_renewal` según `frequency`.
10. **Préstamo por cobrar:** al crear (con opción "descontar ahora") sale dinero de la cuenta origen; al confirmar cada cobro entra dinero a la cuenta de retorno; `outstanding=0 → status='collected'`.
11. **Deuda por pagar:** al recibir entra dinero a la cuenta; al pagar sale dinero de la cuenta de pago; `balance=0 → status='paid'`.
12. **Metas reciben aportes** (como transferencia); cada aporte suma a `current_amount`; alcanzar `target_amount → completed`; `deadline` vencida sin completar → `overdue`.
13. **`movements` es la fuente central de verdad.** Todo pago/cobro/aporte/confirmación de cualquier módulo debe reflejarse como fila en `movements` con `source_type` + `source_id`, de modo que el módulo Movimientos y los reportes lean de una sola tabla.
14. **Idempotencia de confirmaciones.** Confirmar una ocurrencia/variable/renovación ya confirmada no debe duplicar movimientos (validar por `last_*` / `UNIQUE(period)` / estado).
15. **Planificación mensual.** `initial_amount = carried_over + recurring_income`; `expected_result = initial_amount - planned_expenses - planned_savings - planned_debts`; cumplimiento = `real_result - expected_result` (positivo = sobre la meta).
16. **Multi-tenant estricto.** Toda consulta filtra por `user_id`; nunca exponer datos de otro usuario.
17. **Consistencia transaccional.** Operaciones que tocan más de una tabla (transferencia, confirmación con movimiento, aporte a meta) se ejecutan en una transacción.
18. **Recalcular saldo al eliminar/editar** un movimiento confirmado: revertir su efecto anterior y aplicar el nuevo.
19. **Esperado contra real.** Ingresos y egresos nacen pendientes con `expected_amount`; al confirmar se guarda `actual_amount` y el saldo usa este último.

---

## 7. Endpoints completos por módulo

Convenciones:

- Base URL: `/api`.
- Autenticación: `Authorization: Bearer <access_token>` en todos salvo auth públicos.
- Respuestas de colección: `{ "data": [...], "meta": { "total", "page", "pageSize" } }`.
- Errores: `{ "error": { "code", "message", "details?" } }` con HTTP status apropiado (400, 401, 403, 404, 409, 422, 500).
- Todos los recursos están **scoped al usuario autenticado**.

### Auth

#### POST /api/auth/register
Registro. Body: `{ name, email, password }`. Response `201`: `{ user, accessToken, refreshToken }`. Errores: `409` email existente, `422` validación.

#### POST /api/auth/login
Login. Body: `{ email, password }`. Response: `{ user, accessToken, refreshToken }`. Errores: `401` credenciales inválidas.

#### POST /api/auth/refresh
Refresca el access token. Body: `{ refreshToken }`. Response: `{ accessToken, refreshToken }`. Errores: `401` token inválido/expirado/revocado.

#### GET /api/auth/me
Perfil del usuario autenticado. Response: `{ user }`.

#### POST /api/auth/logout
Revoca el refresh token. Body: `{ refreshToken }`. Response `204`.

---

### Cuentas

#### GET /api/accounts
Lista cuentas del usuario. Query: `type`, `status`, `search`. Response:
```json
{
  "data": [
    { "id": "uuid", "name": "BCP Sueldo", "type": "bank", "bankName": "BCP", "currency": "PEN",
      "currentBalance": 7635.65, "creditLimit": null, "status": "active", "color": "#3b82f6",
      "emoji": "BC", "tag": "Principal", "lastMovementAt": "2026-05-09T09:42:00Z" }
  ]
}
```

#### POST /api/accounts
Crea cuenta. Body: `CreateAccountDto`. Response `201` con la cuenta. Regla: `currentBalance` inicia en `initialBalance`.

#### GET /api/accounts/:id
Detalle de cuenta. Actualmente devuelve `{ account }` e incluye `movementsCount`. El plan mensual se obtendrá desde `/summary` cuando exista ese módulo.

#### PATCH /api/accounts/:id
Edita cuenta. Body: `UpdateAccountDto` (parcial). Response: cuenta actualizada.

#### DELETE /api/accounts/:id
Elimina físicamente una cuenta que no sea principal y no tenga historial. Response `200`: `{ "message": "Cuenta eliminada correctamente" }`. Si incumple una regla responde `409`.

#### GET /api/accounts/:id/summary
Resumen mensual de la cuenta. Query: `period`. Response:
```json
{ "period": "2026-05", "income": 4150, "expense": 1245.5, "transfersOut": 700, "transfersIn": 200,
  "plan": { "carriedOver": 2600, "recurringIncome": 3500, "initialAmount": 6100, "expectedResult": 900,
            "realResult": 950, "fulfillment": 50, "months": [ { "label": "May", "expected": 900, "real": 950 } ] } }
```

#### GET /api/accounts/:id/movements
Movimientos de la cuenta. Query: `type`, `status`, `period`, `page`, `pageSize`. Response: colección de movimientos.

---

### Categorías

#### GET /api/categories
Lista. Query: `period=YYYY-MM`, `type=INCOME|EXPENSE`. Inicializa el mes copiando el último anterior cuando corresponde. Response: `categories[]`, `counts` y `monthTotal` calculado solo con confirmados.

#### POST /api/categories
Crea en el periodo indicado. Body: `{ period?, type, name, color?, icon?, subcategories[] }`. Response `201`.

#### GET /api/categories/:id
Obtiene la categoría con sus subcategorías para autorrellenar el formulario de edición.

#### PATCH /api/categories/:id
Edita nombre, color, icono y la colección de subcategorías desde la categoría padre. Solo afecta el mes de esa categoría.

#### DELETE /api/categories/:id
Elimina la categoría solo de su mes. Response `200`. Si la categoría o sus subcategorías tienen movimientos visibles responde `409` y exige reclasificar o eliminar esos movimientos. Importar/exportar queda pendiente y no tiene endpoints actuales.

---

### Movimientos

#### GET /api/movements
Lista con filtros. Query: `period`, `type`, `accountId`, `categoryId`, `subcategoryId`, `status`, `search`, `sourceType`, `page`, `pageSize`. Response: colección + `meta.total`. `totals` contiene montos y cantidades confirmadas del mes completo, independientes de los filtros de tabla.

#### POST /api/movements/income
Crea ingreso manual siempre `PENDING` con `expectedAmount`. Response `201`.

#### POST /api/movements/expense
Crea egreso manual siempre `PENDING` con `expectedAmount`. Response `201`.

#### POST /api/movements/transfer
Crea transferencia. Body: `CreateTransferDto`. Regla: transacción atómica origen/destino; `toAccountId ≠ accountId`. Response `201`.

#### POST /api/movements/:id/confirm
Confirma un movimiento `PENDING`. Body obligatorio: `{ actualAmount, accountId?, toAccountId?, categoryId?, subcategoryId?, date?, occurredAt?, description? }`. Permite corregir todo el formulario, conserva `expectedAmount` y actualiza el saldo con `actualAmount`.

#### POST /api/movements/:id/cancel
Cancela (`status='cancelled'`); revierte efecto en saldo si estaba confirmado. Response: movimiento.

#### GET /api/movements/:id
Detalle. Actualmente devuelve el movimiento, sus cuentas y clasificación. La resolución del módulo de origen se agregará cuando esos módulos existan.

#### PATCH /api/movements/:id
Edita un movimiento manual no cancelado. Si estaba confirmado, revierte el saldo anterior y aplica el nuevo dentro de la misma transacción.

#### DELETE /api/movements/:id
Elimina de forma lógica y revierte saldo. Response actual `200` con mensaje. Solo admite movimientos manuales; los movimientos automáticos futuros se gestionarán desde su módulo de origen.

---

### Pagos Recurrentes

#### GET /api/recurring
Lista. Query: `status`, `type`. Response: definiciones + `projected`/`confirmed` del período.

#### POST /api/recurring
Crea. Body: `CreateRecurringDto`. Response `201`.

#### PATCH /api/recurring/:id
Edita. Body parcial. Response: definición.

#### POST /api/recurring/:id/pause
Pausa (`status='paused'`). Response: definición.

#### POST /api/recurring/:id/resume
Reactiva (`status='active'`). Response: definición.

#### POST /api/recurring/:id/finalize
Finaliza (`status='finalized'`, deja de proyectar). Response: definición.

#### DELETE /api/recurring/:id
Elimina/desactiva. Response `204`.

#### POST /api/recurring/:id/confirm
Confirma la ocurrencia del período. Body: `{ period? }`. Regla: una vez por período; crea movimiento y avanza `next_date`. Error `409` si ya confirmada. Response: `{ occurrence, movement }`.

#### GET /api/recurring/occurrences
Ocurrencias del mes. Query: `period`, `status`. Response: colección de `recurring_occurrences`.

---

### Pagos Variables

#### GET /api/variables
Lista por mes. Query: `period` (requerido), `status`, `type`. Response: colección + KPIs `{ expenseProjected, expenseConfirmed, incomeProjected, incomeConfirmed, next }`.

#### POST /api/variables
Crea. Body: `CreateVariableDto`. Regla: `period` fijo al crear. Response `201`.

#### PATCH /api/variables/:id
Edita. Body parcial. Response: registro.

#### POST /api/variables/:id/confirm
Confirma (una vez); crea movimiento (`source_type=variable`). Error `409` si confirmado. Response: `{ variable, movement }`.

#### POST /api/variables/:id/cancel
Cancela. Response: registro.

#### DELETE /api/variables/:id
Elimina (soft). Response `204`.

#### POST /api/variables/import
Importa planificación previa. Query: `period` destino. Body: `{ items: [...] }`. Response: `{ created }`.

#### GET /api/variables/export
Exporta. Query: `period`, `format`. Response: archivo/estructura.

---

### Suscripciones

#### GET /api/subscriptions
Lista. Query: `status` (`all`/`pending`/`confirmed`/`paused`). Response: colección + KPIs `{ monthlyTotal, paidTotal, annualCost, nextRenewal }`.

#### POST /api/subscriptions
Crea. Body: `CreateSubscriptionDto`. Response `201`.

#### PATCH /api/subscriptions/:id
Edita. Body parcial. Response: suscripción.

#### POST /api/subscriptions/:id/pause
Pausa. Response: suscripción.

#### POST /api/subscriptions/:id/resume
Reactiva. Response: suscripción.

#### POST /api/subscriptions/:id/finalize
Finaliza. Response: suscripción.

#### POST /api/subscriptions/:id/confirm
Confirma el pago del período; crea egreso, setea `last_paid`, avanza `next_renewal`. Error `409` si ya pagado el período. Response: `{ subscription, movement }`.

#### GET /api/subscriptions/upcoming
Próximas renovaciones. Query: `days` (default 30). Response: colección ordenada por `next_renewal`.

---

### Préstamos por Cobrar

#### GET /api/loans
Lista. Query: `status` (`active`/`collected`/`all`). Response: colección + KPIs `{ outstanding, collected, totalLent, next }`.

#### POST /api/loans
Crea. Body: `CreateLoanDto`. Regla: `createExpense=true` genera egreso desde `sourceAccountId`. Response `201`.

#### PATCH /api/loans/:id
Edita. Body parcial. Response: préstamo.

#### POST /api/loans/:id/collect
Registra cobro (parcial/total). Body: `{ amount, toAccountId?, date? }`. Regla: genera ingreso; `outstanding=0 → collected`. Response: `{ loan, movement }`.

#### GET /api/loans/:id
Detalle con historial de cobros (`loan_collections`).

#### DELETE /api/loans/:id
Elimina/desactiva. Response `204`.

---

### Deudas por Pagar

#### GET /api/debts
Lista. Query: `status` (`active`/`paid`/`all`). Response: colección + KPIs `{ totalOwed, paidOff, monthlyInstallment, next }`.

#### POST /api/debts
Crea. Body: `CreateDebtDto`. Regla: `createIncome=true` genera ingreso a `receiveAccountId`. Response `201`.

#### PATCH /api/debts/:id
Edita. Body parcial. Response: deuda.

#### POST /api/debts/:id/pay
Registra pago (parcial/cuota/total). Body: `{ amount, fromAccountId?, date? }`. Regla: genera egreso; `balance=0 → paid`. Response: `{ debt, movement }`.

#### GET /api/debts/:id
Detalle con historial de pagos (`debt_payments`).

#### DELETE /api/debts/:id
Elimina/desactiva. Response `204`.

---

### Metas

#### GET /api/goals
Lista. Query: `status`. Response: colección + KPIs `{ totalSaved, activeCount, monthContribution, completedCount }` y por meta `monthlyNeeded`.

#### POST /api/goals
Crea. Body: `CreateGoalDto`. Response `201`.

#### PATCH /api/goals/:id
Edita. Body parcial. Response: meta.

#### POST /api/goals/:id/pause
Pausa/reactiva (toggle o `resume` separado). Response: meta.

#### POST /api/goals/:id/complete
Marca completada manualmente. Response: meta.

#### POST /api/goals/:id/contributions
Registra aporte. Body: `CreateContributionDto`. Regla: transferencia (`source_type=goal`), suma a `current_amount`, auto-completa si llega al objetivo. Response: `{ goal, contribution, movement }`.

#### GET /api/goals/:id/contributions
Historial de aportes.

#### GET /api/goals/:id
Detalle: meta + `contributions[]` + cálculos (`monthlyNeeded`, `expectedToDate`, `pace`, serie `months[{expected, real}]`).

---

### Dashboard / Reportes

#### GET /api/dashboard/summary
Resumen general mensual. Query: `period`. Response:
```json
{ "period": "2026-05", "totalBalance": 10736.22, "income": 4150, "expense": 1245.5,
  "balanceMonth": 2904.5, "savings": 3250, "expectedResult": 900, "realResult": 950, "fulfillment": 50 }
```

#### GET /api/dashboard/income-expense
Serie ingresos vs egresos. Query: `months` (default 6). Response: `{ labels, income[], expense[] }`.

#### GET /api/dashboard/expected-vs-real
Esperado vs real por mes. Query: `accountId?`, `months`. Response: `{ months: [{ label, expected, real }] }`.

#### GET /api/dashboard/upcoming-payments
Próximos pagos (recurrentes + variables pendientes). Query: `days`. Response: colección ordenada por fecha.

#### GET /api/dashboard/upcoming-subscriptions
Próximas renovaciones. Response: colección.

#### GET /api/dashboard/upcoming-debts
Próximos vencimientos de deudas. Response: colección.

#### GET /api/dashboard/upcoming-loans
Préstamos próximos a vencer. Response: colección.

#### GET /api/dashboard/goals-progress
Progreso de metas. Response: colección con `pct`, `monthlyNeeded`, `pace`.

#### GET /api/dashboard/category-distribution
Distribución de egresos por categoría del período. Query: `period`, `type`. Response: `[{ categoryId, name, color, amount }]`.

#### (Presupuestos) GET/POST/PATCH/DELETE /api/budgets
CRUD de presupuestos; `GET` incluye `used` calculado del período.

---

## 8. DTOs sugeridos

```ts
// ===== Enums (TS) =====
type AccountType = 'bank' | 'savings' | 'digital_wallet' | 'cash' | 'credit_card' | 'investment' | 'other';
type Currency = 'PEN' | 'USD';
type MovementType = 'income' | 'expense' | 'transfer';
type MovementStatus = 'pending' | 'confirmed' | 'cancelled';
type CategoryType = 'income' | 'expense';
type PaymentFrequency = 'daily' | 'weekly' | 'biweekly' | 'monthly' | 'quarterly' | 'annual';
type GoalType = 'saving' | 'emergency_fund' | 'debt_payoff' | 'specific_purchase' | 'investment' | 'savings_replenishment' | 'custom';

// ===== Auth =====
interface RegisterDto { name: string; email: string; password: string; }
interface LoginDto { email: string; password: string; }

// ===== Cuentas =====
interface CreateAccountDto {
  name: string;
  type: AccountType;
  bankName?: string;
  currency: Currency;         // default 'PEN'
  initialBalance: number;
  creditLimit?: number;       // solo credit_card
  color?: string;
  tag?: string;
}
interface UpdateAccountDto extends Partial<CreateAccountDto> { status?: 'active' | 'inactive' | 'archived'; }

// ===== Categorías =====
interface CreateCategoryDto {
  period?: string;            // YYYY-MM; default America/Lima
  type: CategoryType;
  name: string;
  color?: string;
  icon?: string;
  subcategories?: string[];
}

// ===== Movimientos =====
interface CreateIncomeDto {
  amount: number;             // > 0
  accountId: number;
  categoryId?: number;
  date: string;               // YYYY-MM-DD
  description?: string;
}
interface CreateExpenseDto extends CreateIncomeDto { subcategoryId?: number; }
interface CreateTransferDto {
  amount: number;
  accountId: number;          // origen
  toAccountId: number;        // destino (≠ origen)
  date: string;
  status?: MovementStatus;
  description?: string;
}
interface ConfirmMovementDto {
  actualAmount: number;
  accountId?: number;
  toAccountId?: number;
  categoryId?: number;
  subcategoryId?: number;
  date?: string;
  description?: string;
}

// ===== Recurrentes / Variables =====
interface CreateRecurringDto {
  type: 'income' | 'expense';
  name: string;
  amount: number;
  accountId: string;
  categoryId?: string;
  frequency: PaymentFrequency;
  nextDate: string;
}
interface CreateVariableDto {
  type: 'income' | 'expense';
  name: string;
  amount: number;
  accountId: string;
  categoryId?: string;
  subcategoryId?: string;
  period: string;             // YYYY-MM
  plannedDate: string;
  status?: 'pending' | 'confirmed';
  description?: string;
}

// ===== Suscripciones =====
interface CreateSubscriptionDto {
  name: string;
  amount: number;
  accountId: string;
  categoryId?: string;
  frequency: PaymentFrequency;
  nextRenewal: string;
}

// ===== Préstamos / Deudas =====
interface CreateLoanDto {
  debtor: string;
  concept?: string;
  originalAmount: number;
  sourceAccountId: string;
  returnAccountId?: string;
  loanDate: string;
  dueDate?: string;
  description?: string;
  createExpense?: boolean;    // descontar de la cuenta al crear
}
interface CreateDebtDto {
  creditor: string;
  type?: string;
  originalAmount: number;
  accountId: string;          // cuenta de pago
  receiveAccountId?: string;
  rateMonthly?: number;
  installment?: number;
  receivedOn?: string;
  dueDate?: string;
  description?: string;
  createIncome?: boolean;     // ingresar a la cuenta al recibir
}

// ===== Metas =====
interface CreateGoalDto {
  name: string;
  type: GoalType;
  targetAmount: number;
  deadline?: string;
  accountId?: string;
  description?: string;
}
interface CreateContributionDto {
  amount: number;
  fromAccountId: string;
  toAccountId?: string;
  date: string;
  description?: string;
}
```

---

## 9. Consideraciones para frontend

Mapeo pantalla → endpoints (para reemplazar el store mock `LumenStore`/`LumenData`):

- **Dashboard:** `GET /dashboard/summary`, `/dashboard/income-expense`, `/dashboard/category-distribution`,
  `/dashboard/upcoming-payments`, `/dashboard/goals-progress`. Cargar por período seleccionado.
- **Cuentas (lista):** `GET /accounts` (KPIs de saldo total y crédito disponible derivados en cliente o vía summary).
- **Detalle de cuenta:** `GET /accounts/:id` + `GET /accounts/:id/summary?period=` (panel esperado-vs-real, monto inicial,
  resultado esperado, 4 tarjetas) + `GET /accounts/:id/movements` (historial filtrable por tipo).
- **Movimientos:** `GET /movements` con todos los filtros conectados al backend (`period`, `type`, `accountId`,
  `categoryId`, `status`, `search`). Acciones: `POST /movements/income|expense|transfer`, `/:id/confirm`, `/:id/cancel`, `DELETE`.
  El botón "Ir al origen" usa `source_type`+`source_id` del movimiento.
- **Categorías:** `GET /categories?period=&type=` con copia mensual, `monthTotal` confirmado, creación, edición y borrado protegido. Import/export queda futuro.
- **Pagos recurrentes:** `GET /recurring` (5 KPIs: proyectado/confirmado por tipo + próximo) y `GET /recurring/occurrences?period=`.
  Acciones confirmar/pausar/reactivar/finalizar/eliminar.
- **Pagos variables:** `GET /variables?period=` (siempre con período). Confirmar una sola vez; import/export por período.
- **Suscripciones:** `GET /subscriptions?status=` (tabs) + KPIs; `POST /:id/confirm` para pagar y reprogramar.
- **Préstamos / Deudas:** `GET /loans|/debts?status=` + KPIs; `POST /:id/collect` y `/:id/pay` para registrar cobro/pago.
- **Metas:** `GET /goals` (con `monthlyNeeded`); detalle `GET /goals/:id` (aportes + esperado-vs-real);
  `POST /goals/:id/contributions` para aportar.

Datos que necesita cada formulario (drawers en `modals.jsx`):

- **Ingreso/Egreso:** lista de cuentas, categorías del periodo, fecha y monto esperado; nacen pendientes. La confirmación pide monto real y permite corregir los datos. **Transferencia:** inmediata confirmada o programada pendiente.
- **Cuenta:** tipos de cuenta (enum), monedas.
- **Recurrente/Variable:** cuentas, categorías/subcategorías, frecuencias (enum).
- **Suscripción:** cuentas, categorías, frecuencias.
- **Préstamo/Deuda:** cuentas (origen/retorno/pago), tipos.
- **Meta / Aporte:** tipos de meta (enum), cuentas origen/destino, avance actual (para sugerencias de monto).

Recomendaciones de consumo:

- Selector de mes global → propagar `period` a todos los `GET` mensuales (el piloto lo tiene fijo; el backend ya lo soporta).
- Usar optimistic UI + refetch tras confirmar/pagar/aportar (esas operaciones mutan saldo y KPIs).
- Cachear catálogos (cuentas, categorías) e invalidarlos al crear/editar.
- Todos los montos como `number` con 2 decimales; formatear en cliente (`fmt`/`fmtNum` ya existen en `lib/data.js`).

---

## 10. Recomendación de fases de desarrollo

**Fase 1 — Núcleo.** Auth (JWT + refresh), usuarios, cuentas, categorías/subcategorías, movimientos
(income/expense/transfer, confirmar/cancelar, filtros). Saldo real derivado de confirmados. *Habilita: Cuentas, Categorías, Movimientos, detalle de cuenta básico.*

**Fase 2 — Planificación.** `monthly_plans` (esperado vs real), pagos recurrentes + ocurrencias, pagos variables.
KPIs proyectado/confirmado. *Habilita: panel esperado-vs-real, Recurrentes, Variables.*

**Fase 3 — Compromisos.** Suscripciones (renovaciones), préstamos por cobrar (cobros), deudas por pagar (pagos),
con su efecto en saldo vía movimientos. *Habilita: Suscripciones, Préstamos, Deudas.*

**Fase 4 — Metas y consolidación.** Metas + aportes (con `monthlyNeeded`, esperado-vs-real), dashboard y reportes,
presupuestos. *Habilita: Metas, Dashboard, Reportes, Presupuestos, Calendario.*

**Fase 5 — Extras.** Importación/exportación (categorías y variables), estadísticas avanzadas, multi-moneda real
(tipo de cambio), notificaciones de vencimientos, tarjetas de crédito operativas.

---

## 11. Supuestos técnicos

- **Backend:** Node.js con **NestJS** (o Express) — estructura modular por dominio (un módulo por sección de §2).
- **Base de datos:** **PostgreSQL**.
- **ORM:** **Prisma** (enums nativos, migraciones, relaciones tipadas).
- **IDs:** **UUID** como PK en todas las tablas (`@default(uuid())`).
- **Auth:** **JWT** (access token corto + refresh token con rotación y revocación en `refresh_tokens`).
- **Soft delete:** `deleted_at` / `is_active` en entidades con historial. La implementación actual de cuentas usa `is_active` para inactivar, pero también permite eliminar físicamente cuentas no principales.
- **Timestamps:** `created_at` y `updated_at` en todas las tablas.
- **Dinero:** `Decimal(14,2)` (Prisma `Decimal`); nunca punto flotante.
- **Moneda:** base **PEN**, esquema preparado para multi-moneda (campo `currency` por cuenta/movimiento; tipo de cambio en fase 5).
- **`movements` es la fuente principal de verdad financiera:** toda mutación monetaria pasa por ahí; los saldos y reportes se derivan de sus filas confirmadas.
- **Transacciones:** operaciones multi-tabla (transferencias, confirmaciones que generan movimiento, aportes) dentro de `prisma.$transaction`.
- **Validación:** DTOs con `class-validator` (NestJS) / Zod; montos `> 0`, fechas `YYYY-MM-DD`, períodos `YYYY-MM`.
- **Multi-tenant:** guard/middleware que inyecta `user_id` y filtra toda consulta.
- **Zona horaria:** almacenar en UTC (`TIMESTAMPTZ`); `occurred_on` como `DATE` para lógica de período.
- **Paginación:** `page`/`pageSize` en listados grandes (movimientos).

---

> **Fin del documento.** Cubre las 20 tablas, enums, reglas de negocio, ~70 endpoints y DTOs necesarios
> para reemplazar el mock del piloto (`lib/data.js` + `lib/store.js`) por un backend real sin ambigüedad.
