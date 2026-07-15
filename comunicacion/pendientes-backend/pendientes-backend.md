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
