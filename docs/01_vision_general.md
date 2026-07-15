# 01 - Vision general del sistema financiero

## Estado del documento

Actualizado con la especificacion `FINANCE_APP_DATABASE_AND_API_SPEC.md` y el avance real del backend del modulo de cuentas.

Este documento sigue siendo vivo: debe cambiar cuando el backend confirme nuevas reglas o cuando una decision cambie.

## Nombre del sistema

El sistema se identifica en la especificacion como **Lumen Finanzas**.

## Proposito del sistema

Lumen Finanzas es una aplicacion de planificacion financiera personal mensual.

No busca solamente registrar gastos. Su objetivo principal es ayudar al usuario a:

- Planificar cada mes.
- Registrar lo que realmente ocurre.
- Comparar lo esperado contra lo real.
- Medir cumplimiento financiero.
- Mantener historial ordenado por periodos mensuales.

## Idea central

Cada mes el usuario parte de un monto inicial, formado por dinero heredado del mes anterior e ingresos esperados.

Luego planifica:

- Ingresos fijos, como el sueldo.
- Ingresos casuales u ocasionales.
- Gastos.
- Ahorros.
- Deudas.
- Prestamos.
- Suscripciones.
- Pagos recurrentes.
- Pagos variables.

Con eso se calcula un resultado esperado. Al avanzar el mes, los movimientos confirmados permiten calcular lo real.

## Concepto clave: periodo mensual

El sistema trabaja por periodo mensual en formato `YYYY-MM`.

Ejemplo:

- `2026-05`
- `2026-06`
- `2026-07`

El periodo permite separar datos por mes y tambien permite planificar meses futuros.

La navegacion mensual se presentara en el frontend como un calendario anual: año navegable y doce botones de mes organizados en una cuadricula de 4 por 3. Mostrar el calendario no crea datos. Un periodo se consulta o prepara solamente cuando el usuario selecciona un mes.

## Movimiento como fuente central

La tabla `movements` es la fuente central de verdad financiera.

Esto significa que todo lo que mueva dinero debe terminar reflejado como movimiento:

- Ingreso.
- Egreso.
- Transferencia.
- Pago recurrente registrado.
- Pago variable confirmado.
- Pago de suscripcion.
- Cobro de prestamo.
- Pago de deuda.
- Aporte a meta.

El modulo futuro de ingresos separara los ingresos fijos, que pueden repetirse cada periodo, de los ingresos casuales, que corresponden a entradas ocasionales. Ambos generaran movimientos pendientes y conservaran por separado el monto esperado y el monto realmente recibido.

## Pendiente y confirmado

Un movimiento pendiente sirve para planificar.

Un movimiento confirmado afecta el saldo real.

Ingresos y egresos se crean pendientes con un monto esperado. Al confirmarlos se registra el monto real, que puede ser diferente, sin modificar la definicion fija o recurrente que los origino. Las transferencias inmediatas pueden confirmarse al crearse; las programadas permanecen pendientes.

Un movimiento cancelado o eliminado no debe contarse como activo, pero debe conservarse para historial cuando corresponda.

## Usuario y acceso

El sistema es personal para cada usuario.

Puede existir mas de un usuario registrado, pero cada usuario solo ve y administra su propia informacion.

No se esta planteando administracion de otros usuarios, roles avanzados ni panel de administracion.

El login ya existe como parte del backend.

## Modulo cuentas: avance real

El modulo de cuentas ya tiene una primera version funcional en backend.

Endpoints definidos:

```text
GET    /api/accounts
GET    /api/accounts/summary
POST   /api/accounts
GET    /api/accounts/:id
PATCH  /api/accounts/:id
PATCH  /api/accounts/:id/status
```

Reglas confirmadas:

- Las cuentas pertenecen al usuario autenticado.
- Las cuentas se crean activas por defecto.
- Las cuentas pueden inactivarse mediante el cambio de estado.
- Una cuenta no principal solo puede eliminarse fisicamente si no tiene historial de movimientos.
- La primera cuenta del usuario se marca como principal automaticamente.
- Solo puede existir una cuenta principal activa por usuario.
- Si una cuenta principal se inactiva, deja de ser principal.
- Las tarjetas de credito no suman al saldo total.
- Las tarjetas de credito suman al credito disponible.
- Los movimientos reales ya estan implementados y los confirmados modifican saldos.

## Tipos de cuenta confirmados

```text
BANK_ACCOUNT
SAVINGS
CASH
DIGITAL_WALLET
CREDIT_CARD
INVESTMENT
OTHER
```

## Resultado esperado y resultado real

El resultado esperado se calcula con la planificacion del periodo.

El resultado real se calcula con movimientos confirmados.

La comparacion entre ambos permite medir cumplimiento financiero.

## Reglas transversales importantes

- El ahorro no es gasto; se registra como transferencia.
- Las suscripciones son un modulo separado de pagos recurrentes.
- Los pagos variables pertenecen solo al periodo donde se crean.
- Los pagos recurrentes y suscripciones generan ocurrencias por periodo.
- Los ingresos fijos se planifican para repetirse; los ingresos casuales no se repiten automaticamente.
- Las categorias se copian al preparar un mes nuevo y luego cada mes queda independiente. Solo pueden eliminarse de un mes si ningun movimiento visible de ese periodo las utiliza.
- Las operaciones que afectan varias tablas deben hacerse de forma consistente.

## Documentos relacionados

- [Modulos](02_modulos.md)
- [Reglas de negocio](03_reglas_negocio.md)
- [Arquitectura base del sistema](07_arquitectura_base_sistema.md)
- [Avance backend modulo cuentas](08_avance_cuentas_backend.md)
- [Modelo fisico](database/03_modelo_fisico.md)
