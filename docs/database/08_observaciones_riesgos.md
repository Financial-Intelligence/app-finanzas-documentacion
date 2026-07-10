# 08 - Observaciones y riesgos del modelo de datos

## Estado del documento

Actualizado con la especificacion actual.

## Observaciones

1. La especificacion usa tablas en ingles.
2. Los documentos anteriores usaban nombres en espanol en algunos diagramas.
3. La documentacion debe alinearse a los nombres reales del backend: `accounts`, `movements`, `categories`, etc.
4. El modulo de cuentas ya tiene avances reales y debe considerarse fuente confirmada.
5. La especificacion incluye presupuestos y planes mensuales, que antes estaban como pendientes.

## Riesgos

### Diferencia entre documentacion y backend real

Cuando existan migraciones reales, se debe comparar contra estos documentos.

### Borrado fisico

Si se elimina fisicamente una cuenta, categoria o movimiento, se puede romper historial.

### Saldos inconsistentes

Cuando movimientos se implementen, editar o cancelar movimientos confirmados debe recalcular saldos.

### Duplicidad de origen

Un pago recurrente o suscripcion no debe crear multiples movimientos para el mismo periodo.

### Mezcla de tarjetas de credito con saldo real

Las tarjetas de credito no deben sumar al saldo total.

## Recomendaciones

1. Crear migraciones alineadas a esta especificacion.
2. Mantener `movements` como fuente central.
3. Agregar indices por `user_id`, `period`, `status` y `source_type`.
4. Usar transacciones cuando una operacion toque varias tablas.
5. Actualizar esta documentacion cada vez que cambie el backend.

