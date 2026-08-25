# Prompt para frontend: implementar únicamente Pagos Variables

Implementa el módulo visual de **Pagos Variables** consumiendo el backend real.
No uses mocks, no inventes rutas y no calcules fechas, contadores, diferencias
ni estados en el frontend.

## Regla principal del mes

Cada tarjeta pertenece solamente al mes que el usuario tiene abierto.

- Si está viendo agosto, crearla con `period: "2026-08"`.
- Si cambia a septiembre, crearla con `period: "2026-09"`.
- Una tarjeta de agosto no debe aparecer automáticamente en septiembre.
- Al volver a agosto, debe verse la tarjeta y su historial de agosto.

El frontend siempre debe enviar en `period` el mes abierto en el selector
global. La fecha inicial también debe pertenecer a ese mes.

## Rutas disponibles

Todas requieren JWT:

```http
GET    /api/variable-payments?period=YYYY-MM&view=all|pending|confirmed&type=EXPENSE|INCOME
GET    /api/variable-payments/:id
POST   /api/variable-payments
PATCH  /api/variable-payments/:id
POST   /api/variable-payments/:id/register
DELETE /api/variable-payments/:id
```

Frecuencias y etiquetas:

```text
DAILY    Diario
WEEKLY   Semanal
MONTHLY  Mensual
```

La fecha inicial es la primera ocurrencia. El backend calcula las siguientes
solo hasta el último día de ese mes:

- `DAILY`: cada día;
- `WEEKLY`: cada 7 días;
- `MONTHLY`: una sola ocurrencia.

## Pantalla principal

Consultar nuevamente al cambiar el mes abierto. Mostrar los resúmenes del
backend:

- egresos proyectados: `summary.expensesExpected`;
- egresos registrados: `summary.expensesActual`;
- ingresos proyectados: `summary.incomeExpected`;
- ingresos registrados: `summary.incomeActual`;
- diferencia: `summary.difference`;
- próximo movimiento: `summary.nextVariablePayment` y
  `summary.nextOccurrenceDate`.

Pestañas:

- **Todos**: `view=all`;
- **Pendientes**: `view=pending`;
- **Confirmados**: `view=confirmed`.

Usar `counts.all`, `counts.pending` y `counts.confirmed`.

Cada tarjeta muestra:

- nombre y descripción;
- tipo, cuenta, categoría y subcategoría cuando exista;
- frecuencia y monto esperado por ocurrencia;
- `progress`, por ejemplo `2 / 4`;
- `nextOccurrenceDate`;
- `periodExpectedAmount`;
- `periodActualAmount`;
- `periodDifference`;
- estado usando `viewStatus`: `PENDING` o `CONFIRMED`.

Acciones:

- para `EXPENSE`: **Registrar gasto**;
- para `INCOME`: **Registrar ingreso**;
- **Editar**;
- **Eliminar**.

No usar el texto “Confirmar” como acción principal.

## Formulario de creación

Campos:

- tipo: gasto o ingreso;
- nombre;
- descripción opcional;
- monto esperado por ocurrencia;
- frecuencia: diaria, semanal o mensual;
- fecha inicial dentro del mes abierto;
- cuenta obligatoria;
- categoría obligatoria;
- subcategoría opcional solo para gastos.

No mostrar un selector de estado. Toda tarjeta nueva comienza pendiente y se
confirma únicamente cuando se registra el gasto o ingreso real. No mostrar
tarjetas de crédito como cuentas disponibles. Filtrar categorías por tipo y
por el mes abierto.

Ejemplo:

```json
{
  "period": "2026-08",
  "name": "Comprar zapatillas",
  "description": "Compra planificada para agosto",
  "type": "EXPENSE",
  "expectedAmount": 260,
  "frequency": "MONTHLY",
  "startDate": "2026-08-20",
  "accountId": 15,
  "categoryId": 19,
  "subcategoryId": 8
}
```

Para un ingreso no enviar `subcategoryId`, o enviarlo como `null`.

## Editar

Usar `PATCH /api/variable-payments/:id` y enviar únicamente los campos que
cambiaron. No se puede cambiar el mes de la tarjeta. Si se cambia la fecha
inicial, debe continuar dentro de ese mismo mes.

Los cambios afectan las ocurrencias pendientes. Las confirmaciones y
movimientos ya realizados conservan sus montos, cuenta, categoría y fechas
originales. Si cambia la frecuencia o la fecha inicial dentro del mes actual,
el nuevo calendario comienza desde hoy; un mes ya terminado no permite cambiar
su calendario. Después de editar, volver a consultar lista y detalle.

## Registrar gasto o ingreso

El botón abre un segundo formulario con:

- nombre de la tarjeta;
- ocurrencia a registrar: `nextOccurrenceDate`;
- monto esperado;
- monto real editable, inicialmente igual al esperado;
- fecha real, inicialmente hoy;
- vista previa de la diferencia.

Enviar:

```json
{
  "occurrenceDate": "2026-08-20",
  "actualAmount": 240,
  "registeredDate": "2026-08-20"
}
```

`registeredDate` es opcional, pero debe pertenecer al mismo mes de la tarjeta.
La respuesta incluye `confirmation` con monto esperado, monto real,
diferencia, fechas, cuenta y movimiento.

La diferencia ya viene calculada por el backend:

- gasto: esperado menos real;
- ingreso: real menos esperado;
- positivo: resultado favorable;
- negativo: exceso o faltante.

No modificar saldos localmente. Después de registrar, refrescar Pagos
Variables, Cuentas, resumen de Cuentas y Movimientos. Ante un registro repetido
el backend responde `409`; mostrar su `message` y refrescar la tarjeta.

## Eliminar

Antes de eliminar, mostrar:

> Si esta tarjeta todavía no tiene registros, se eliminará completamente. Si ya tiene gastos o ingresos registrados, la tarjeta dejará de aparecer, pero sus movimientos se conservarán como historial.

La respuesta trae:

- `deletionType: "DELETED"`: no tenía confirmaciones y se borró completamente;
- `deletionType: "HISTORICAL"`: tenía confirmaciones y conservó sus movimientos.

Nunca eliminar manualmente esos movimientos desde el frontend.

## Integración con Cuentas

Consultar usando el mes abierto:

```http
GET /api/accounts?period=YYYY-MM
GET /api/accounts/summary?period=YYYY-MM
```

Cada cuenta incluye:

- `variableDifference`: diferencia producida por Pagos Variables en ese mes;
- `planningDifference`: diferencia total de planificación de la cuenta.

El resumen contiene los mismos campos dentro de `summary`. Mostrar
`variableDifference` separado del saldo real con el título **Diferencia de
pagos variables**:

- positivo: **A favor**;
- negativo: **Exceso / faltante**;
- cero: **Sin diferencia**.

No sumar la diferencia a `currentBalance`; cada movimiento ya aplicó su monto
real y volver a sumarla duplicaría el resultado.

## Actualización y errores

Después de crear, editar, registrar o eliminar, invalidar y consultar otra vez
los datos afectados. Usar fechas `YYYY-MM-DD`, montos numéricos y mostrar
`message` ante respuestas `400`, `401`, `404` o `409`. No cerrar el formulario
si hubo un error.

## Pruebas obligatorias

- Crear una tarjeta en agosto y comprobar que no aparece en septiembre.
- Volver a agosto y comprobar que conserva su contador e historial.
- Mensual: `0 / 1`, registrar y pasar a Confirmados.
- Semanal: calcular solo fechas del mes abierto.
- Diario iniciado a mitad del mes: contar únicamente los días restantes.
- Gasto e ingreso con monto menor, igual y mayor al esperado.
- Bloquear el registro duplicado de una ocurrencia.
- Editar sin modificar confirmaciones anteriores.
- Eliminar sin confirmaciones: desaparición completa.
- Eliminar con confirmaciones: tarjeta oculta y movimientos conservados.
- Mostrar `variableDifference` sin modificar dos veces el saldo.
- Cambiar el selector global de mes y refrescar todos los datos del módulo.

No se considera terminado hasta probar todos estos casos contra el backend
real, sin mocks ni rutas distintas de las indicadas.

---

# Prompt adicional para frontend: implementar Suscripciones

Conserva todo lo ya implementado de Pagos Variables y añade el módulo visual de
**Suscripciones** consumiendo el backend real. Las suscripciones representan
únicamente gastos. No agregues la opción de ingreso, no uses mocks y no
calcules fechas, estados, contadores ni diferencias en el frontend.

## Rutas disponibles

Todas requieren JWT:

```http
GET    /api/subscriptions?period=YYYY-MM&view=all|pending|confirmed|paused
GET    /api/subscriptions/:id?period=YYYY-MM
POST   /api/subscriptions
PATCH  /api/subscriptions/:id
POST   /api/subscriptions/:id/register
POST   /api/subscriptions/:id/pause
POST   /api/subscriptions/:id/resume
DELETE /api/subscriptions/:id
```

Frecuencias y etiquetas:

```text
DAILY          Diaria
WEEKLY         Semanal
MONTHLY        Mensual
ANNUAL         Anual
CUSTOM_MONTHS  Cada cierta cantidad de meses
```

Cuando se elija `CUSTOM_MONTHS`, mostrar el campo obligatorio **Se repite cada
N meses** y enviar `intervalMonths` entre 2 y 120. Para las demás frecuencias
no enviarlo o enviarlo como `null`.

## Comportamiento por mes

Las suscripciones son globales y no se duplican al cambiar de mes. El selector
global solo determina el mes cuyos pagos, estados y contadores se muestran.

El backend devuelve `viewStatus`:

- `PENDING`: tiene renovaciones pendientes en el mes abierto;
- `CONFIRMED`: todas las renovaciones de ese mes están pagadas;
- `PAUSED`: la suscripción está pausada;
- `NO_DUE`: no tiene ningún cobro durante ese mes.

`NO_DUE` aparece únicamente en **Todas**. No debe mostrarse como pendiente ni
confirmada; mostrar su próxima renovación.

## Pantalla principal

Usar directamente el resumen del backend:

- total del mes: `summary.totalExpectedThisPeriod`;
- pagado este mes: `summary.paidThisPeriod`;
- costo proyectado de los próximos 12 meses:
  `summary.projectedCostNext12Months`;
- diferencia del mes: `summary.difference`;
- cantidad activa: `summary.activeSubscriptions`;
- próxima renovación: `summary.nextSubscription` y
  `summary.nextRenewalDate`.

El costo de 12 meses ya considera la frecuencia diaria, semanal, mensual,
anual o personalizada. No multiplicar montos en el frontend.

Pestañas:

- **Todas**: `view=all`;
- **Pendientes**: `view=pending`;
- **Confirmadas**: `view=confirmed`;
- **Pausadas**: `view=paused`.

Usar `counts.all`, `counts.pending`, `counts.confirmed`, `counts.paused` y
`counts.noDue`.

Cada tarjeta muestra:

- nombre y descripción;
- cuenta, categoría y subcategoría cuando exista;
- frecuencia y cantidad de meses cuando sea personalizada;
- monto esperado por renovación;
- `progress`, por ejemplo `0 / 1`;
- `nextRenewalDate`;
- `periodExpectedAmount`;
- `periodActualAmount`;
- `periodDifference`;
- `projectedCostNext12Months`;
- estado usando `viewStatus`.

Acciones:

- **Registrar pago**;
- **Editar**;
- **Pausar** o **Reanudar**;
- **Eliminar**.

No utilizar el texto **Confirmar pago**.

## Nueva suscripción

Campos:

- nombre obligatorio;
- descripción opcional;
- monto esperado obligatorio;
- próxima fecha o primer cobro obligatorio;
- cuenta de pago obligatoria;
- categoría de gasto obligatoria;
- subcategoría opcional;
- frecuencia obligatoria;
- cantidad de meses obligatoria solo para `CUSTOM_MONTHS`.

No mostrar tarjetas de crédito como cuentas disponibles. Filtrar categorías de
gasto según el mes del primer cobro.

Ejemplo mensual:

```json
{
  "name": "Netflix",
  "description": "Plan familiar",
  "expectedAmount": 50,
  "frequency": "MONTHLY",
  "intervalMonths": null,
  "startDate": "2026-08-15",
  "accountId": 15,
  "categoryId": 19,
  "subcategoryId": 8
}
```

Ejemplo cada tres meses:

```json
{
  "name": "Servicio trimestral",
  "expectedAmount": 90,
  "frequency": "CUSTOM_MONTHS",
  "intervalMonths": 3,
  "startDate": "2026-08-15",
  "accountId": 15,
  "categoryId": 19
}
```

## Registrar pago

El botón **Registrar pago** abre un segundo formulario. Mostrar:

- nombre de la suscripción;
- renovación a registrar: `nextRenewalDate`;
- monto esperado;
- monto real editable, inicialmente igual al esperado;
- fecha real del pago, inicialmente hoy;
- vista previa de la diferencia.

Enviar:

```json
{
  "occurrenceDate": "2026-08-15",
  "actualAmount": 45,
  "paidDate": "2026-08-15"
}
```

`paidDate` es opcional. La respuesta incluye `payment` con esperado, real,
diferencia, fechas, cuenta y movimiento. El backend crea un gasto confirmado y
descuenta el monto real de la cuenta.

La diferencia es `monto esperado - monto real`:

- positiva: pagó menos de lo esperado;
- negativa: pagó más de lo esperado;
- cero: pagó exactamente lo esperado.

No cambiar saldos en el frontend. Ante `409` por renovación duplicada, mostrar
`message` y refrescar la tarjeta. Después del pago, refrescar Suscripciones,
Cuentas, resumen de Cuentas y Movimientos.

## Editar

Usar `PATCH /api/subscriptions/:id` y enviar solo campos modificados. Los
cambios afectan renovaciones futuras. Los pagos y movimientos anteriores
conservan sus datos originales.

Si cambia frecuencia, cantidad de meses o fecha inicial, el nuevo calendario
comienza desde hoy o desde una fecha futura elegida. No crear pagos atrasados.

## Pausar y reanudar

`pause` y `resume` no llevan body. Una suscripción pausada no muestra
**Registrar pago** y su `nextRenewalDate` será `null`.

Al reanudar, el calendario continúa desde la fecha real de reanudación. No
generar pagos correspondientes al tiempo en que estuvo pausada.

## Eliminar

Antes de eliminar mostrar:

> Si la suscripción todavía no tiene pagos, se eliminará completamente. Si ya tiene pagos registrados, dejará de aparecer, pero sus movimientos se conservarán como historial.

La respuesta devuelve:

- `deletionType: "DELETED"`: no tenía pagos y se borró completamente;
- `deletionType: "HISTORICAL"`: tenía pagos y conservó el historial.

Nunca eliminar sus movimientos manualmente desde el frontend.

## Integración con Cuentas

Consultar según el mes abierto:

```http
GET /api/accounts?period=YYYY-MM
GET /api/accounts/summary?period=YYYY-MM
```

Cada cuenta y el resumen incluyen:

- `subscriptionDifference`: diferencia de Suscripciones del mes;
- `planningDifference`: diferencia total de planificación.

Mostrar `subscriptionDifference` separado del saldo real con el título
**Diferencia de suscripciones**. No sumarlo a `currentBalance`, porque el gasto
real ya modificó la cuenta.

## Actualización y errores

Al cambiar el mes global, volver a consultar Suscripciones y Cuentas. Después
de crear, editar, registrar, pausar, reanudar o eliminar, invalidar todos los
datos afectados. Mostrar `message` ante `400`, `401`, `404` y `409`, y no cerrar
el formulario si ocurrió un error.

## Pruebas obligatorias

- Mensual: pendiente, registrar pago, confirmada y pendiente nuevamente al
  siguiente mes.
- Diaria y semanal con contadores exactos del mes.
- Anual: un mes sin cobro usa `NO_DUE` y muestra la próxima renovación.
- Personalizada cada 3 meses: solo genera renovaciones en los meses correctos.
- Costo proyectado de 12 meses para cada frecuencia.
- Pago menor, igual y mayor al monto esperado.
- Bloqueo de renovación duplicada.
- Edición sin alterar pagos históricos.
- Pausa y reanudación sin pagos atrasados.
- Eliminación completa sin pagos y conservación histórica con pagos.
- `subscriptionDifference` visible sin modificar dos veces el saldo.

No se considera terminado hasta probar todos los casos contra el backend real,
sin mocks ni rutas distintas de las indicadas.
