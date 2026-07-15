# 06 - Riesgos y pendientes

## Estado del documento

Actualizado con los avances actuales.

## Riesgos actuales

### Duplicar movimientos

Un recurrente, variable, suscripcion, prestamo, deuda o meta no debe generar dos movimientos por la misma accion.

Control esperado:

- Usar `source_type`.
- Usar `source_id`.
- Validar estado antes de registrar.
- Usar restricciones unicas por periodo cuando aplique.

### Mezclar saldo real con proyeccion

Pendiente no debe modificar saldo real.

Confirmado si debe afectar saldo.

### Duplicar suscripciones como recurrentes

Las suscripciones son modulo independiente.

No deben registrarse tambien como pago recurrente normal.

### Borrado fisico con historial

El modulo Cuentas solo permite borrar fisicamente una cuenta no principal sin historial. La relacion de base de datos tambien protege el historial.

Para entidades con historial, la accion normal debe ser inactivar, cancelar o usar borrado logico.

### Calculos distintos entre modulos

Dashboard, cuentas y reportes deben leer la misma fuente de datos.

No deben calcular cada uno con reglas distintas.

### Tarjetas de credito

Ya aparecen dentro de cuentas, pero no deben mezclarse con saldo total.

Queda pendiente su logica financiera completa.

## Pendientes principales

1. Implementar planes mensuales y el resumen de detalle de cuenta.
2. Implementar pagos recurrentes y ocurrencias usando monto esperado y real.
3. Implementar el modulo de ingresos fijos e ingresos casuales.
4. Implementar pagos variables.
5. Implementar suscripciones.
6. Implementar prestamos y cobros.
7. Implementar deudas y pagos.
8. Implementar metas y aportes.
9. Implementar presupuestos.
10. Implementar dashboard y reportes.
11. Ampliar pruebas de integracion con una base de datos exclusiva para pruebas.
