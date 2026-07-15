# 02 - Modelo logico

## Estado del documento

Actualizado con la especificacion objetivo y el modelo implementado al 2026-07-15.

## Convenciones logicas

- Cada tabla de negocio pertenece a un usuario mediante `user_id`.
- El periodo mensual se guarda como `period` en formato `YYYY-MM`.
- El dinero se representa como decimal.
- El codigo implementado usa identificadores `Int`; los UUID quedan como una propuesta historica no aplicada.
- Las tablas criticas usan borrado logico con `deleted_at` o estado.

## Entidades y relaciones principales

| Entidad origen | Relacion | Entidad destino | Cardinalidad |
| --- | --- | --- | --- |
| Usuario | tiene | Cuentas | 1 a muchos |
| Usuario | tiene | Categorias | 1 a muchos |
| Usuario | inicializa | Periodos de categorias | 1 a muchos |
| Categoria | contiene | Subcategorias | 1 a muchos |
| Usuario | registra | Movimientos | 1 a muchos |
| Cuenta | participa en | Movimientos | 1 a muchos |
| Pago recurrente | genera | Ocurrencias recurrentes | 1 a muchos |
| Ocurrencia recurrente | crea | Movimiento | 1 a 0..1 |
| Pago variable | crea | Movimiento | 1 a 0..1 |
| Suscripcion | genera | Renovaciones | 1 a muchos |
| Renovacion | crea | Movimiento | 1 a 0..1 |
| Prestamo | tiene | Cobros | 1 a muchos |
| Cobro de prestamo | crea | Movimiento | 1 a 0..1 |
| Deuda | tiene | Pagos | 1 a muchos |
| Pago de deuda | crea | Movimiento | 1 a 0..1 |
| Meta | tiene | Aportes | 1 a muchos |
| Aporte | crea | Movimiento | 1 a 0..1 |
| Plan mensual | pertenece a | Usuario | muchos a 1 |
| Plan mensual | puede pertenecer a | Cuenta | muchos a 0..1 |

## Nota importante

Las relaciones con `source_type` y `source_id` permiten saber de que modulo nacio un movimiento.

Cada categoria implementada incluye `period`. `CategoryPeriod(user_id, period)` es unico y evita volver a copiar categorias eliminadas cuando se consulta nuevamente un mes.
