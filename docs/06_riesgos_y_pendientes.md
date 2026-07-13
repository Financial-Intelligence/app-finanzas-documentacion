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

Actualmente el modulo Cuentas permite borrar fisicamente una cuenta que no sea principal.
Cuando Movimientos exista, se debe impedir el borrado si la cuenta tiene historial.

Para entidades con historial, la accion normal debe ser inactivar, cancelar o usar borrado logico.

### Calculos distintos entre modulos

Dashboard, cuentas y reportes deben leer la misma fuente de datos.

No deben calcular cada uno con reglas distintas.

### Tarjetas de credito

Ya aparecen dentro de cuentas, pero no deben mezclarse con saldo total.

Queda pendiente su logica financiera completa.

## Pendientes principales

1. Implementar modulo Movimientos.
2. Conectar movimientos confirmados con saldos reales.
3. Implementar Categorias.
4. Implementar pagos recurrentes y ocurrencias.
5. Implementar pagos variables.
6. Implementar suscripciones.
7. Implementar prestamos y cobros.
8. Implementar deudas y pagos.
9. Implementar metas y aportes.
10. Implementar presupuestos.
11. Implementar dashboard y reportes.
12. Completar modelo fisico con migraciones reales cuando existan.
13. Actualizar ERD desde la base real cuando el backend avance.
