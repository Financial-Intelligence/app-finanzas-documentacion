# 04 - Flujos principales

## Estado del documento

Actualizado con la especificacion de API y modelo de datos.

## Flujo base: crear cuenta

1. El usuario autenticado envia datos de la cuenta.
2. El backend valida nombre, tipo, saldo inicial y moneda.
3. La cuenta se crea activa.
4. Si es la primera cuenta del usuario, se marca como principal.
5. El saldo actual nace desde el saldo inicial.

Endpoint actual:

```text
POST /api/accounts
```

## Flujo base: listar cuentas

1. El usuario pide sus cuentas.
2. El backend filtra por usuario autenticado.
3. Puede filtrar por estado.
4. Devuelve cuentas activas, inactivas o todas segun consulta.

Endpoint actual:

```text
GET /api/accounts
```

## Flujo base: resumen de cuentas

1. El usuario pide resumen.
2. El backend suma cuentas activas que no son tarjeta de credito.
3. El backend suma credito disponible de cuentas tipo tarjeta.
4. Devuelve saldo total, credito disponible y cuenta principal.

Endpoint actual:

```text
GET /api/accounts/summary
```

## Flujo futuro: registrar movimiento

1. El usuario o modulo origen crea un movimiento.
2. Ingresos y egresos quedan pendientes con un monto esperado.
3. Al confirmar se registran monto real y correcciones del formulario.
4. Solo el monto real confirmado afecta saldo.
5. Una transferencia inmediata puede confirmarse al crearla; una programada queda pendiente.

## Flujo mensual de categorias

1. El usuario abre el selector anual de meses.
2. El frontend muestra doce meses en cuadricula 4 por 3 sin consultar los doce periodos.
3. Cambiar el año visible no crea datos.
4. El usuario selecciona un mes y el frontend construye el periodo `YYYY-MM`.
5. El frontend consulta categorias y movimientos de ese periodo.
6. Si el periodo nunca fue inicializado, se copian categorias y subcategorias del ultimo mes anterior.
7. Desde ese momento el mes queda independiente.
8. Las tarjetas suman solo montos reales confirmados del periodo.
9. El borrado se bloquea si existen movimientos visibles que usan la categoria o subcategoria.
10. Se guarda `source_type` y `source_id` si viene de otro modulo.

Endpoints previstos:

```text
POST /api/movements/income
POST /api/movements/expense
POST /api/movements/transfer
```

## Flujo futuro: registrar pago recurrente

1. El recurrente genera una ocurrencia del periodo.
2. El formulario abre con monto previsto.
3. El usuario puede registrar monto real distinto.
4. Se conserva monto previsto.
5. Se crea movimiento confirmado.
6. Se actualiza la ocurrencia.
7. Se calcula diferencia entre previsto y real.

## Flujo futuro: registrar pago variable

1. El pago variable pertenece a un periodo.
2. El usuario lo registra o confirma.
3. Se crea movimiento con `source_type=variable`.
4. No se copia al siguiente periodo.

## Flujo futuro: registrar suscripcion

1. La suscripcion tiene proxima renovacion.
2. Al registrar pago se crea egreso.
3. Se actualiza ultimo periodo pagado.
4. Se reprograma la siguiente renovacion.

## Flujo futuro: registrar prestamo

1. El usuario registra dinero prestado.
2. Si se descuenta ahora, se crea egreso.
3. El prestamo mantiene saldo por cobrar.
4. Cada cobro genera ingreso.
5. Cuando el saldo pendiente llega a cero, queda cobrado.

## Flujo futuro: registrar deuda

1. El usuario registra deuda recibida.
2. Si se recibio ahora, se crea ingreso.
3. La deuda mantiene saldo pendiente.
4. Cada pago genera egreso.
5. Cuando el saldo llega a cero, queda pagada.

## Flujo futuro: aporte a meta

1. El usuario registra aporte.
2. Se crea movimiento, normalmente como transferencia.
3. Se incrementa avance de la meta.
4. Si llega al monto objetivo, la meta queda completada.
