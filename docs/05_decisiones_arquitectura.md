# 05 - Decisiones de arquitectura

## Estado del documento

Actualizado con la especificacion de base de datos/API y el avance del modulo de cuentas.

## Decisiones actuales

1. La documentacion vive en un repositorio separado del frontend y backend.
2. El sistema usa login y datos separados por usuario autenticado.
3. No se plantea administracion de otros usuarios ni roles avanzados.
4. `movements` sera la fuente central de verdad financiera.
5. Las tablas principales usaran UUID.
6. El dinero se guardara como `DECIMAL(14,2)`.
7. El periodo mensual se representara como `YYYY-MM`.
8. Las tablas criticas usaran borrado logico, no borrado fisico.
9. Las cuentas ya tienen avance backend funcional.
10. Las cuentas no se eliminan; se inactivan.
11. Tarjetas de credito se separan del saldo total.
12. Pagos recurrentes usaran el concepto futuro de registrar pago.
13. Dashboard y reportes leeran datos ya calculados o registrados, no seran fuente principal.

## Decisiones sobre modelo de datos

- `users` guarda el usuario autenticado.
- `refresh_tokens` guarda sesiones o renovacion de acceso.
- `accounts` guarda cuentas financieras.
- `categories` y `subcategories` clasifican movimientos.
- `movements` centraliza ingresos, egresos y transferencias.
- `recurring_payments` define recurrentes.
- `recurring_occurrences` guarda ocurrencias por periodo.
- `variable_payments` guarda planificaciones de un periodo.
- `subscriptions` y `subscription_renewals` separan suscripciones de recurrentes.
- `loans` y `loan_collections` permiten cobros parciales o totales.
- `debts` y `debt_payments` permiten pagos parciales o totales.
- `goals` y `goal_contributions` controlan metas.
- `budgets` queda incluido para presupuestos.
- `monthly_plans` guarda snapshots mensuales de planificacion.

## Decisiones pendientes

- Definir implementacion final de `monthly_plans`.
- Definir si los movimientos cancelados y eliminados compartiran estado o campo separado.
- Definir alcance exacto de reportes avanzados.
- Definir automatizacion de documentacion desde base de datos real.
