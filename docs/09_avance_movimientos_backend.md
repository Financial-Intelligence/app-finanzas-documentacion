# 09 - Avance backend del modulo Movimientos

Fecha de corte: 2026-07-15

## Estado real

La primera version funcional del modulo Movimientos esta implementada. Es la fuente central del historial de ingresos, gastos y transferencias.

## API disponible

```text
GET    /api/movements
POST   /api/movements/income
POST   /api/movements/expense
POST   /api/movements/transfer
GET    /api/movements/:id
PATCH  /api/movements/:id
POST   /api/movements/:id/confirm
POST   /api/movements/:id/cancel
DELETE /api/movements/:id
GET    /api/accounts/:id/movements
```

El listado permite filtros y paginacion, y devuelve totales confirmados.

## Modelo implementado

- `Category`, `Subcategory` y `Movement` ya existen y estan relacionados.
- Los IDs son `Int`, coherentes con el backend existente.
- `expectedAmount` conserva el monto previsto y `actualAmount` guarda el monto real confirmado.
- Estados: `PENDING`, `CONFIRMED`, `CANCELLED`.
- La eliminacion de movimientos es logica mediante `deletedAt`.

## Reglas implementadas

1. Solo un confirmado cambia el saldo real.
2. Ingresos y egresos nacen pendientes.
3. Al confirmar se puede corregir monto real, fecha, cuenta, categoria, subcategoria y descripcion.
4. El saldo usa el monto real sin modificar el esperado ni una futura definicion recurrente.
5. Una transferencia inmediata puede nacer confirmada; una programada queda pendiente.
6. Ingreso suma; gasto resta; transferencia mueve saldo entre cuentas.
7. Gasto y transferencia requieren saldo suficiente.
8. Una transferencia no cruza monedas ni usa la misma cuenta dos veces.
9. Editar, confirmar, cancelar y eliminar aplican el saldo de forma atomica.

## Integracion con Cuentas

- El detalle devuelve `movementsCount`.
- Una cuenta con historial no puede eliminarse fisicamente.
- El historial considera la cuenta como origen o destino.

## Verificacion

- Prisma validado y cliente generado.
- JavaScript comprobado sintacticamente.
- 16 pruebas automatizadas aprobadas.

## Pendientes relacionados

- `GET /api/accounts/:id/summary`, que necesita planes mensuales y pagos recurrentes.
- Integraciones con modulos futuros mediante `sourceType` y `sourceId`.
