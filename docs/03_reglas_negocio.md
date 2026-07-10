# 03 - Reglas de negocio

## Estado del documento

Documento pendiente de completar conforme se definan reglas confirmadas del sistema.

## Objetivo

Centralizar las reglas que deben cumplirse en el sistema financiero.

## Reglas confirmadas

### Cuentas

1. Las cuentas representan lugares donde existe o se mueve dinero.
2. Una cuenta puede estar activa o inactiva.
3. Una cuenta inactiva no se borra fisicamente.
4. Las cuentas inactivas se conservan para no romper historial futuro.
5. Las cuentas se crean activas por defecto.
6. El saldo actual de una cuenta normal nace igual al saldo inicial.
7. Las cuentas tipo tarjeta de credito no suman al saldo total.
8. Las cuentas tipo tarjeta de credito suman al credito disponible.
9. Cada usuario puede tener una cuenta principal.
10. La primera cuenta creada por un usuario se marca como principal automaticamente.
11. Si una cuenta se marca como principal, las demas cuentas del usuario dejan de ser principales.
12. Una cuenta inactiva no debe quedar como principal.

### Movimientos y pagos planificados

1. La accion para pagos planificados sera `registrar pago`.
2. El formulario de registrar pago cargara el monto previsto por defecto.
3. Si el usuario paga un monto diferente, se guarda un monto real diferente.
4. El monto previsto no se reemplaza ni se pierde.
5. La diferencia entre previsto y real sirve para comparar planificacion contra realidad.
6. Un movimiento pendiente ayuda al calculo esperado.
7. Un movimiento confirmado ayuda al calculo real.
8. Un movimiento eliminado debe quedar visible en historial, pero no debe contarse.

## Reglas inferidas

1. Las cuentas de credito requieren campos propios como limite y credito disponible.
2. El saldo total debe calcularse desde cuentas activas no crediticias.
3. El credito disponible debe calcularse desde cuentas activas de credito.
4. La cuenta principal debe venir desde el backend con `isPrimary`, no inventarse en frontend.
5. Los formularios de ingreso, gasto y transferencia deben reutilizarse despues para registrar pagos de otros modulos.

## Reglas pendientes de confirmar

1. Reglas avanzadas de tarjetas de credito, como consumos, pagos, deuda usada y fechas de corte.
2. Reglas exactas de cierre mensual.
3. Reglas para modificar meses pasados.
4. Reglas de pagos parciales en prestamos y deudas.
5. Reglas de importacion desde Excel.
