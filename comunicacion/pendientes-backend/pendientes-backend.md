# Pendientes del backend solicitados por frontend

Fecha de corte: 2026-07-15

## Ya resuelto

1. Modulo Movimientos implementado.
2. `movementsCount` agregado a `GET /api/accounts/:id`.
3. Historial disponible en `GET /api/accounts/:id/movements`.
4. Eliminacion de cuenta bloqueada cuando existe historial.
5. Los movimientos confirmados actualizan saldos; cancelarlos o eliminarlos logicamente revierte el efecto.
6. Categorias y subcategorias mensuales implementadas con copia automatica y proteccion por movimientos.
7. Ingresos y egresos nacen pendientes y conservan monto esperado y monto real.

## Bug bloqueante: POST /:id/skip responde 500 en los tres modulos

Afecta a `recurring-payments`, `variable-payments` y `subscriptions`. Reproducido en vivo: crear cualquier tarjeta, ejecutar `POST /api/recurring-payments/:id/skip` con `{"occurrenceDate": "<fecha vigente>"}` devuelve 500:

```text
DriverAdapterError: new row for relation "RecurringConfirmation" violates
check constraint "RecurringConfirmation_amounts_positive"
```

Causa: las tablas `RecurringConfirmation`, `VariableConfirmation` y `SubscriptionPayment` tienen, desde su migracion de creacion, un CHECK:

```sql
CONSTRAINT "RecurringConfirmation_amounts_positive" CHECK (
    "expectedAmount" > 0 AND "actualAmount" > 0
)
```

(mismo patron en `VariableConfirmation_amounts_positive` y `SubscriptionPayment_amounts_positive`, en `prisma/migrations/20260825120000_create_recurring_payments`, `20260825160000_create_variable_payments` y `20260825170000_create_subscriptions`).

La migracion que agrego omitir/deshacer ocurrencia (`20260826150000_add_skipped_occurrences_and_pause_history`) inserta `actualAmount: 0` a proposito cuando `resolutionStatus: "SKIPPED"` (es el comportamiento documentado: una omision no crea movimiento ni monto real), pero nunca relajo estos tres constraints. El insert siempre viola la regla y la transaccion revierte con 500.

`POST /:id/skip` queda inutilizable en los tres modulos hasta que se corrija. `DELETE /:id/skip/:occurrenceDate` (deshacer) no se vio afectado porque nunca llega a insertar.

Arreglo sugerido: una migracion nueva que reemplace los tres constraints para permitir `actualAmount = 0` solo cuando `resolutionStatus = 'SKIPPED'`, por ejemplo:

```sql
ALTER TABLE "RecurringConfirmation" DROP CONSTRAINT "RecurringConfirmation_amounts_positive";
ALTER TABLE "RecurringConfirmation" ADD CONSTRAINT "RecurringConfirmation_amounts_positive" CHECK (
    "expectedAmount" > 0 AND (
        "actualAmount" > 0 OR ("actualAmount" = 0 AND "resolutionStatus" = 'SKIPPED')
    )
);
-- Repetir para VariableConfirmation_amounts_positive y SubscriptionPayment_amounts_positive.
```

El frontend ya tiene el boton "Saltar..." y el flujo de deshacer construidos, con tests unitarios en verde, esperando este fix para poder verificarse en vivo.

## Pendiente principal

### GET /api/accounts/:id/summary

Continua pendiente. El modulo Movimientos ya permite calcular ingresos, gastos, transferencias y resultado real, pero todavia no existen los datos necesarios para:

- saldo heredado planificado;
- ingresos fijos del mes;
- resultado esperado;
- serie historica esperado contra real.

Estos valores dependen de `monthly_plans` y pagos recurrentes. El endpoint no se implementa con ceros o datos inventados.

Contrato esperado:

```text
GET /api/accounts/:id/summary?period=YYYY-MM
```

El objeto se devolvera directamente, sin envoltorio `{ summary }`, cuando sus dependencias existan.

## Decisiones aun pendientes fuera de Movimientos

1. Definir si inactivar la cuenta principal debe bloquearse o reasignar otra automaticamente.
2. Definir el uso final o retiro de `expectedMonthlyAmount`.
3. Mantener `monthlyPlan` solamente en `/accounts/:id/summary` para evitar duplicar logica.
