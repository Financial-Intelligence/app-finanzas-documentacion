# 03 - Reglas de negocio

## Estado del documento

Actualizado con la especificacion de base de datos/API y el avance real del modulo de cuentas.

## Reglas confirmadas generales

1. `movements` es la fuente central de verdad financiera.
2. Solo los movimientos confirmados afectan el saldo real.
3. Los movimientos pendientes sirven para proyeccion.
4. Los movimientos cancelados o eliminados no deben contarse como activos.
5. Todo movimiento pertenece a un usuario.
6. Toda consulta debe filtrar por el usuario autenticado.
7. El dinero debe manejarse con decimal, no con numeros aproximados.
8. El periodo mensual usa formato `YYYY-MM`.
9. Las operaciones que afectan varias tablas deben hacerse juntas para evitar datos inconsistentes.
10. El selector anual puede mostrar los doce meses sin crear registros ni ejecutar doce consultas.
11. Cambiar el año visible no prepara periodos; seleccionar un mes si consulta ese periodo.
12. El mes inicial del frontend se calcula con la zona horaria `America/Lima`.

## Reglas de cuentas

1. Las cuentas pertenecen a un usuario.
2. Las cuentas se crean activas por defecto.
3. Las cuentas pueden inactivarse sin borrarlas.
4. Una cuenta no principal solo puede eliminarse fisicamente si nunca tuvo movimientos.
5. La primera cuenta del usuario se marca como principal automaticamente.
6. Solo una cuenta del usuario puede ser principal.
7. Si una cuenta principal se inactiva, deja de ser principal.
8. Las tarjetas de credito no suman al saldo total.
9. Las tarjetas de credito suman al credito disponible.
10. El saldo actual de una cuenta normal nace desde el saldo inicial.
11. Los movimientos confirmados modifican saldos dentro de una transaccion de base de datos.

## Reglas de movimientos

Estado implementado al 2026-07-15:

- Los pendientes no cambian saldo.
- Los confirmados cambian saldo.
- Ingresos y egresos nacen pendientes.
- El monto esperado se conserva y el saldo usa el monto real confirmado.
- Una transferencia inmediata puede nacer confirmada; una programada nace pendiente.
- Los cancelados no cuentan y conservan historial.
- Gastos y transferencias requieren saldo suficiente.
- Las transferencias requieren cuentas distintas y de la misma moneda.
- Las cuentas inactivas y `CREDIT_CARD` no aceptan nuevos movimientos.
- Editar, cancelar o eliminar logicamente un confirmado revierte primero su efecto anterior.
- Solo los movimientos manuales se editan o eliminan desde la API general.

1. `income` suma a la cuenta.
2. `expense` resta a la cuenta.
3. `transfer` resta en la cuenta origen y suma en la cuenta destino.
4. Una transferencia no puede tener la misma cuenta como origen y destino.
5. El monto siempre debe ser positivo; el tipo define si suma o resta.
6. Cada movimiento debe poder indicar su origen con `source_type` y `source_id`.
7. Si se edita o cancela un movimiento confirmado, se debe corregir el efecto anterior en saldo.

## Reglas de categorias

1. Las categorias se dividen en ingreso y egreso.
2. Una categoria puede tener subcategorias.
3. El total mensual por categoria usa solo movimientos confirmados.
4. Cada categoria pertenece a un periodo `YYYY-MM`.
5. El primer acceso a un mes copia las categorias activas del ultimo mes anterior.
6. Editar o eliminar una categoria de agosto no modifica julio.
7. Una categoria o subcategoria usada por movimientos visibles no puede eliminarse hasta reclasificar o eliminar esos movimientos.
4. Una categoria usada historicamente no debe borrarse fisicamente.

## Reglas de pagos recurrentes

1. Un pago recurrente activo genera ocurrencias por periodo.
2. Una ocurrencia se registra una sola vez por periodo segun su frecuencia.
3. Al registrar el pago se crea un movimiento.
4. El monto previsto se conserva.
5. El monto real puede ser distinto y sirve para comparar esperado contra real.
6. Estados operativos: activo, pausado, finalizado.

## Reglas de pagos variables

1. Pertenecen a un periodo especifico.
2. No se copian automaticamente al mes siguiente.
3. Al confirmarse crean un movimiento.
4. Solo deben confirmarse una vez.

## Reglas de suscripciones

1. Son independientes de pagos recurrentes.
2. Al registrar pago crean egreso.
3. Al pagarse se actualiza el ultimo periodo pagado.
4. Se reprograma la siguiente renovacion segun frecuencia.

## Reglas de prestamos por cobrar

1. Representan dinero que el usuario presto.
2. Al crear pueden generar egreso si se descuenta ahora.
3. Cada cobro genera ingreso.
4. El saldo por cobrar baja con cada cobro.
5. Cuando el saldo pendiente llega a cero, el prestamo queda cobrado.

## Reglas de deudas por pagar

1. Representan dinero recibido que debe devolverse.
2. Al crear pueden generar ingreso si el dinero se recibio ahora.
3. Cada pago genera egreso.
4. El saldo pendiente baja con cada pago.
5. Cuando el saldo llega a cero, la deuda queda pagada.

## Reglas de metas

1. Una meta tiene monto objetivo y avance actual.
2. Cada aporte genera movimiento.
3. El aporte puede ser transferencia entre cuentas.
4. Al llegar al monto objetivo, la meta queda completada.
5. Si vence sin completarse, puede marcarse como vencida.
