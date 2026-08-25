# Implementar y corregir el módulo visual de Préstamos

Este archivo contiene únicamente el trabajo pendiente del frontend para el
módulo de **Préstamos por cobrar**. Úsalo como instrucción completa para una IA
o desarrollador. No crear rutas nuevas ni modificar el backend.

## Objetivo

Conectar la vista de Préstamos al backend real y reflejar esta diferencia:

- **Eliminar**: corrige un registro equivocado. Borra definitivamente el
  préstamo, sus cobros y los movimientos que generó, restaurando los saldos.
- **Perdonar préstamo**: el préstamo fue real, pero se decide no cobrarlo.
  Se conserva como historial con estado `FORGIVEN` y no cambia ningún saldo.
- **Cobrado**: el pendiente llega a cero y permanece como historial con estado
  `COLLECTED`.

Todas las rutas requieren JWT. Reutiliza el cliente HTTP o interceptor actual
para enviar `Authorization: Bearer <token>`; no repetir la lógica del token en
cada componente.

## Listado, pestañas y calendario

Al abrir Préstamos o cambiar el mes global, llama a:

```http
GET /api/loans?status=active&period=YYYY-MM
```

Los préstamos son globales: cambiar de mes no crea ni copia préstamos. El
parámetro `period` solo cambia `collectedThisPeriod`, el monto cobrado durante
ese mes.

Usar estas pestañas y consultas:

| Pestaña visual | Query `status` | Contador |
| --- | --- | --- |
| Activos | `active` | `counts.active` |
| Cobrados | `collected` | `counts.collected` |
| Perdonados | `forgiven` | `counts.forgiven` |
| Todos | `all` | `counts.all` |

No recalcular valores que ya devuelve la API. Usar:

- `summary.totalOutstanding`: tarjeta **Por cobrar**;
- `summary.totalCollected` y `summary.collectedPercentage`: **Ya cobrado**;
- `summary.totalLent` y `summary.collectedThisPeriod`: **Total prestado**;
- `summary.nextDueLoan`: **Próximo a vencer**;
- `collectedAmount`, `collectedPercentage`, `collectionsCount`, `isOverdue`,
  `sourceAccount` y `returnAccount`: tarjeta individual.

Cuando `summary` tenga varias monedas, sus campos superiores pueden ser `null`.
No sumar monedas distintas: mostrar por separado `summary.byCurrency`.

## Crear préstamo

Usar:

```http
POST /api/loans
```

El formulario contiene: deudor, concepto opcional, monto, fecha del préstamo,
cuenta de origen, cuenta donde entrarán los cobros, fecha límite opcional,
descripción opcional y el interruptor `Descontar de mi cuenta ahora`.

### Regla obligatoria del interruptor

Debe iniciar **desactivado** y enviar `createExpense: false`.

Texto visual exacto debajo del interruptor:

> Márcalo solo si ya entregaste el dinero. Se creará un gasto y bajará el saldo de la cuenta. Si aún no lo entregas, déjalo desactivado.

Comportamiento:

- `createExpense: false`: crea solo el préstamo; no modifica cuentas ni crea
  movimiento. Es el valor predeterminado del backend.
- `createExpense: true`: crea un gasto confirmado y descuenta el monto de la
  cuenta de origen porque el dinero ya fue entregado.

Ejemplo de body:

```json
{
  "debtor": "Rivaldo",
  "concept": "Préstamo personal",
  "originalAmount": 300,
  "sourceAccountId": 15,
  "returnAccountId": 10,
  "loanDate": "2026-08-25",
  "dueDate": "2026-09-10",
  "description": null,
  "createExpense": false
}
```

No mostrar cuentas inactivas ni tarjetas de crédito. Si se omite
`returnAccountId`, el backend usa la cuenta de origen. Las dos cuentas deben
usar la misma moneda.

## Editar préstamo

Autorrellenar y enviar solo cambios con:

```http
PATCH /api/loans/:id
```

Campos: `debtor`, `concept`, `originalAmount`, `sourceAccountId`,
`returnAccountId`, `loanDate`, `dueDate` y `description`.

Si `collectionsCount > 0` o `expenseMovementId !== null`, bloquear visualmente
`originalAmount`, `sourceAccountId` y `loanDate`. Los demás siguen editables.
El backend también rechaza esos cambios con `409`.

## Cobrar y revisar historial

El botón de una tarjeta y el botón general **Registrar cobro** deben abrir el
mismo formulario. Para cargar los cobros de un préstamo usar:

```http
GET /api/loans/:id
```

La respuesta incluye `loan.collections`, ordenados del más reciente al más
antiguo. Mostrar un historial dentro del detalle o formulario con monto, fecha,
cuenta y la acción **Eliminar cobro**.

Registrar un cobro:

```http
POST /api/loans/:id/collections
```

```json
{
  "amount": 100,
  "toAccountId": 10,
  "date": "2026-08-25"
}
```

`Cobrar todo` debe abrir el formulario con `amount=outstanding`; nunca enviar
la petición sin confirmación del usuario.

Para un cobro equivocado, pedir confirmación y usar:

```http
DELETE /api/loans/:id/collections/:collectionId
```

Este endpoint elimina definitivamente el cobro y su ingreso, descuenta el
dinero de la cuenta receptora y restaura el pendiente. Si responde `409`, la
cuenta ya no tiene saldo suficiente: mostrar `message` y no cerrar la vista.

## Eliminar un préstamo equivocado

Mostrar la acción **Eliminar registro equivocado** con confirmación explícita.
No llamarla solamente “Eliminar”, porque es una acción definitiva.

Mensaje de confirmación:

> Se eliminará este préstamo definitivamente. Se revertirán sus cobros y el gasto inicial, si existe; las cuentas volverán al saldo anterior. Esta acción no se puede deshacer.

Ruta:

```http
DELETE /api/loans/:id
```

El backend procesa todo junto:

1. revierte los cobros y sus ingresos;
2. revierte el gasto inicial, si `createExpense` fue verdadero;
3. borra préstamo, cobros y movimientos definitivamente.

Al éxito, cerrar el detalle/modal, mostrar éxito y retirar la tarjeta. Si llega
`409`, no se eliminó nada: mostrar el `message` del backend. Esto ocurre cuando
una cuenta receptora ya no conserva saldo suficiente para retirar un cobro.

## Perdonar un préstamo real

Mostrar **Perdonar préstamo** solo cuando `status === "ACTIVE"`. Esta acción es
para dinero que sí se prestó, pero el usuario decide no cobrar. No equivale a
eliminar un error.

Pedir confirmación:

> Este préstamo fue real y dejará de estar pendiente de cobro. No se devolverá dinero ni se modificarán saldos. Quedará en el historial como perdonado.

Ruta sin body:

```http
POST /api/loans/:id/forgive
```

Después del éxito, mover la tarjeta a **Perdonados** y **Todos**. Mostrar la
etiqueta `Perdonado`, el monto pendiente que no se cobrará y no ofrecer
Registrar cobro ni Cobrar todo.

## Movimientos y actualización de pantalla

Los movimientos creados por préstamos usan `sourceType=LOAN`. Deben aparecer en
Movimientos, pero no ofrecer Editar, Cancelar o Eliminar desde allí: las
correcciones se hacen exclusivamente desde Préstamos.

Después de crear, editar, cobrar, eliminar un cobro, perdonar o eliminar un
préstamo, invalidar y volver a consultar:

- listado y detalle de préstamos;
- cuentas y resumen de cuentas;
- movimientos de los períodos involucrados;
- historial de las cuentas involucradas.

Usar fechas `YYYY-MM-DD`, montos numéricos y mostrar el campo `message` que
envía el backend para errores `400`, `401`, `404` y `409`. Ante `401`, aplicar
el cierre de sesión global ya existente. No cerrar formularios ni modales cuando
la respuesta sea un error.

## Pruebas obligatorias

- El interruptor inicia desactivado y envía `createExpense=false`.
- Crear sin descuento no cambia saldo ni crea movimiento.
- Crear con descuento disminuye la cuenta y crea gasto `LOAN`.
- Cobro parcial, cobro total y paso a `COLLECTED`.
- Eliminación individual de un cobro y restauración del pendiente.
- Eliminación completa de préstamo con y sin gasto inicial.
- Error `409` al intentar eliminar cuando no hay saldo suficiente para revertir
  un cobro.
- Perdón de préstamo: estado `FORGIVEN`, sin cambios de saldo y visible en la
  pestaña Perdonados.
- Bloqueo visual de monto, cuenta origen y fecha cuando existe historial.
- Movimientos `LOAN` visibles, sin acciones manuales.
- Invalidación correcta de préstamos, cuentas y movimientos.

No se considera terminado hasta probar todos los casos con el backend real, sin
mocks ni endpoints distintos de los descritos aquí.

---

# Actualización obligatoria: cuotas e intereses de Deudas y Préstamos

## Regla que debe explicar la interfaz

La tasa mensual ya no es solo informativa. En Deudas y Préstamos el backend
genera una cuota cada mes, el día del vencimiento, sin mover dinero de cuentas.
El usuario registra un pago o cobro únicamente cuando ocurrió en la vida real.

Cada cuota generada usa interés simple sobre el monto original aún pendiente:

```text
interés del mes = monto original pendiente × tasa mensual / 100
```

Los intereses anteriores no generan nuevos intereses. Si transcurrieron varios
meses, al abrir la lista o el detalle el backend crea las cuotas pendientes de
esos meses. No calcular ni crear cuotas en el frontend.

## Formularios de nueva Deuda y nuevo Préstamo

En ambos formularios agregar o conservar estos campos:

- **Cuota mensual estimada** (`installment`): monto esperado para ese mes.
- **Tasa mensual (%)** (`monthlyInterestRate`): interés aplicado cada mes.
- **Primer vencimiento**: `nextDueDate` para Deudas y `dueDate` para Préstamos.

Si la cuota o la tasa es mayor a cero, el primer vencimiento es obligatorio.
Si ambos son cero, el vencimiento sigue siendo solo una fecha de referencia y
no se generan cuotas mensuales.

Texto de ayuda debajo de ambos campos:

> La cuota se registra como pendiente cada mes; no descuenta ninguna cuenta automáticamente. Al pagar o cobrar, el monto cubre primero los intereses pendientes y después reduce el monto original.

Si `installment` es menor que el interés mensual mostrado, advertir:

> Esta cuota no alcanza para reducir el monto original; primero debe cubrir el interés mensual.

## Datos que ahora devuelve el backend

En una Deuda:

```text
balance                 monto original pendiente
accruedInterest         intereses pendientes de pago
totalOutstanding        total real pendiente: balance + accruedInterest
installment             cuota mensual configurada
monthlyInterestRate     tasa mensual configurada
installments[]          historial de cuotas generadas
```

En un Préstamo, los equivalentes son `outstanding`, `accruedInterest`,
`totalOutstanding`, `installment`, `monthlyInterestRate` e `installments[]`.

Cada elemento de `installments[]` trae:

```text
dueDate, interestAmount, principalAmount, totalAmount,
paidInterestAmount, paidPrincipalAmount, paidAmount,
remainingAmount, status, isOverdue
```

Mostrar en tarjetas el **total real pendiente** usando `totalOutstanding`; no
mostrar solo `balance` u `outstanding` cuando haya intereses pendientes.

## Registro de pago o cobro

Al registrar pago de Deuda o cobro de Préstamo, el backend recibe el monto total
normal. La respuesta ahora incluye:

```text
payment.interestAmount / collection.interestAmount
payment.principalAmount / collection.principalAmount
```

En el comprobante e historial mostrar la división claramente, por ejemplo:

```text
Pago S/ 150.00
Interés: S/ 20.00
Reducción del monto original: S/ 130.00
```

El máximo permitido es `totalOutstanding`, no solo el monto original pendiente.
Al revertir un pago/cobro equivocado, el backend restaura tanto interés como
monto original y actualiza las cuotas; refrescar detalle, lista, cuentas y
movimientos.

## Historial visual de cuotas

En el detalle de cada Deuda y Préstamo, agregar una sección **Cuotas e
intereses** ordenada por vencimiento. Cada fila muestra fecha, interés, parte
del monto original, total de la cuota, abonado, pendiente y estado:

- `PENDING`: pendiente.
- `PARTIAL`: pagada parcialmente.
- `PAID`: cubierta por completo.
- Si `isOverdue` es verdadero y no está `PAID`, mostrar **Vencida**.

Los resúmenes y las tarjetas se deben volver a consultar al abrir la pantalla,
porque esa consulta puede hacer aparecer cuotas de meses que transcurrieron.

## Pruebas obligatorias adicionales

- Deuda y préstamo con S/ 100, tasa 10%, cuota S/ 50 y primer vencimiento dos
  meses atrás: deben existir dos cuotas y S/ 20 de interés pendiente.
- Pago/cobro de S/ 30: S/ 20 deben ir a interés y S/ 10 al monto original.
- Revertir dicho pago/cobro restaura S/ 20 de interés y S/ 10 de monto original.
- Tasa 0 y cuota mayor a 0 genera cuotas sin interés.
- Cuota menor que el interés muestra la advertencia visual.
- Ninguna cuota descuenta o ingresa dinero por sí sola.

---

# Implementar y ajustar el módulo visual de Deudas

## Contexto funcional obligatorio

Una **deuda** es dinero que el usuario debe. Es global: no se duplica al cambiar
de mes. El selector de mes solo afecta `paidThisPeriod`; no debe ocultar ni
crear deudas nuevas.

Estados del backend:

- `ACTIVE`: sigue pendiente.
- `PAID`: se pagó por completo.
- `FORGIVEN`: el acreedor condonó una deuda real. Queda como historial y no
  modifica saldos.

Usar solo estas rutas:

```http
GET    /api/debts?status=active|paid|forgiven|all&period=YYYY-MM
GET    /api/debts/:id
POST   /api/debts
PATCH  /api/debts/:id
POST   /api/debts/:id/payments
DELETE /api/debts/:id/payments/:paymentId
POST   /api/debts/:id/forgive
DELETE /api/debts/:id
```

No calcular saldos en el cliente: mostrar valores del backend.

## Pantalla principal

Mantener las tarjetas del prototipo: **Total que debes**, **Ya pagado**, **Cuota
mensual** y **Próximo vencimiento**. Si hay varias monedas, usar
`summary.byCurrency`; nunca sumarlas en una sola cifra.

Pestañas y query:

- **Activas**: `status=active`.
- **Pagadas**: `status=paid`.
- **Condonadas**: `status=forgiven`.
- **Todas**: `status=all`.

Los contadores son `counts.active`, `counts.paid`, `counts.forgiven` y
`counts.all`. En una tarjeta `FORGIVEN`, mostrar **Condonada** y no ofrecer
acciones de pago.

La tarjeta activa muestra acreedor, tipo, cuenta de pago, saldo pendiente,
monto original, progreso, vencimiento, cantidad de pagos y acciones de pago,
edición, corrección y condonación.

## Nueva deuda

Campos: acreedor, tipo, monto total, cuota estimada, tasa mensual, cuenta de
pago, próximo vencimiento y descripción. Agregar el interruptor inicialmente
**desactivado**:

**¿Recibiste este dinero ahora?**

> Márcalo solo si el dinero entró hoy a una de tus cuentas. Se registrará un ingreso y aumentará ese saldo. Si la deuda ya existía, déjalo desactivado.

- Desactivado: enviar `createIncome: false`; no pedir cuenta/fecha de recepción
  y no mover saldo.
- Activado: enviar `createIncome: true`, mostrar y exigir `receiveAccountId` y
  `receivedOn`. Explicar que se creará un ingreso confirmado.

Después de crear, refrescar deudas, cuentas, movimientos y resúmenes.

## Pagos e historial

**Registrar pago** usa `POST /api/debts/:id/payments` y envía `amount`,
`accountId` y `date`. El pago crea un egreso confirmado, disminuye la cuenta y
el saldo pendiente. No permitir monto mayor al pendiente y mostrar el mensaje
del backend si no hay saldo.

En el historial, cada pago debe tener **Eliminar pago equivocado**. Confirmar:

> Se eliminará este pago y se devolverá el dinero a la cuenta. El saldo pendiente de la deuda se restaurará. Esta acción no se puede deshacer.

Usar:

```http
DELETE /api/debts/:id/payments/:paymentId
```

Tras el éxito, el pago y su movimiento ya no existen: refrescar deuda, cuentas,
movimientos y resumen.

## Corregir una deuda creada por error

La acción se llama **Eliminar registro equivocado**, no solo “Eliminar”.
Confirmación:

> Se eliminará esta deuda definitivamente. Se revertirán sus pagos y el ingreso inicial, si existe; las cuentas volverán al saldo anterior. Esta acción no se puede deshacer.

```http
DELETE /api/debts/:id
```

El backend revierte pagos e ingreso inicial y borra deuda, pagos y movimientos.
Si llega `409`, no se eliminó nada: mostrar `message`. Puede ocurrir si la
cuenta receptora ya no conserva saldo para revertir el ingreso inicial.

## Deuda condonada por el acreedor

Mostrar **Marcar como condonada** solo para `status === "ACTIVE"`. Es para una
deuda real que el acreedor perdonó, no para un error.

> El acreedor condonó esta deuda. Dejará de estar pendiente y no se modificará ningún saldo. Quedará en el historial como condonada.

```http
POST /api/debts/:id/forgive
```

Después del éxito, moverla a **Condonadas** y **Todas**, quitar acciones de pago
y mostrar la etiqueta de estado.

## Movimientos y actualización

Los movimientos de deudas usan `sourceType=DEBT`. Deben verse en Movimientos,
pero sin Editar, Cancelar ni Eliminar allí: las correcciones se hacen desde
Deudas.

Después de crear, editar, pagar, eliminar pago, condonar o eliminar deuda,
invalidar y consultar nuevamente lista/detalle de deudas, cuentas, resumen,
movimientos de períodos afectados e historial de cuentas. Usar fechas
`YYYY-MM-DD`, montos numéricos y mostrar `message` ante `400`, `401`, `404` y
`409`, sin cerrar el modal ante error.

## Pruebas obligatorias

- El interruptor inicia apagado: no hay movimiento ni cambio de saldo.
- Activarlo crea ingreso `DEBT` y aumenta la cuenta receptora.
- Pago parcial, pago total y paso a `PAID`.
- Eliminar un pago restaura saldo y elimina su movimiento.
- Eliminar deuda con/sin ingreso revierte todo y elimina deuda, pagos y
  movimientos.
- Error `409` si falta saldo para revertir el ingreso inicial.
- Condonar cambia a `FORGIVEN`, no cambia saldos y se muestra en Condonadas.
- Movimientos `DEBT` visibles sin acciones manuales.

No se considera terminado hasta probar todos los casos con el backend real, sin
mocks ni endpoints distintos de los descritos aquí.
