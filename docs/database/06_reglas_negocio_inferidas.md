# 06 - Reglas de negocio inferidas

## Estado del documento

Actualizado desde la especificacion y el avance real de cuentas.

## Reglas confirmadas por documentacion actual

| Regla | Fuente |
| --- | --- |
| `movements` es la fuente central de verdad. | Especificacion base de datos/API. |
| Solo movimientos confirmados afectan saldos reales. | Especificacion base de datos/API. |
| Ingresos y egresos nacen pendientes y conservan monto esperado y real por separado. | Flujo confirmado 2026-07-15. |
| Las categorias se copian al inicializar un mes y luego ese mes es independiente. | Modulo Categorias mensual. |
| Pendientes participan en proyeccion. | Especificacion base de datos/API. |
| Las cuentas pueden inactivarse mediante `isActive`. | Código actual del módulo Cuentas. |
| Una cuenta no principal puede eliminarse fisicamente solo si no tiene historial. | Codigo actual de Cuentas y Movimientos. |
| Primera cuenta se marca principal automaticamente. | Avance backend cuentas. |
| Solo puede existir una cuenta principal por usuario. | Avance backend cuentas. |
| Tarjetas de credito no suman al saldo total. | Avance backend cuentas. |
| Tarjetas de credito suman al credito disponible. | Avance backend cuentas. |
| Recurrentes se registraran como pago conservando monto previsto y monto real. | Avance backend cuentas. |

## Reglas inferidas para validar al implementar

| Regla | Motivo |
| --- | --- |
| Si una cuenta esta inactiva, no deberia recibir nuevos movimientos manuales. | Evita usar cuentas desactivadas. |
| Si se cambia el estado de una cuenta principal a inactiva, debe quedar sin principal. | Ya confirmado parcialmente. |
| Si un movimiento confirmado se edita, debe recalcular saldos. | Necesario para consistencia. |
| Si una ocurrencia ya fue registrada, no debe registrarse otra vez en el mismo periodo. | Evita duplicados. |
