# Prompt para frontend: módulo de Metas

Implementa únicamente el módulo de **Metas** y su integración nueva con
**Cuentas**. Los demás módulos ya están terminados y no deben reconstruirse.

El backend es la fuente de verdad para montos, estados, ritmo, aporte sugerido,
porcentaje, dinero reservado y gráfico. No calcules esos valores nuevamente en
el frontend. Después de cualquier operación, consulta otra vez la meta y las
cuentas afectadas.

Todas las peticiones requieren `Authorization: Bearer <token>`.

## Concepto principal

Una meta no crea ni representa una cuenta nueva. El dinero siempre permanece en
una cuenta real y la meta reserva una parte de ese saldo.

- Varias metas pueden usar la misma cuenta destino.
- No permitas seleccionar tarjetas de crédito como origen o destino.
- Si origen y destino son la misma cuenta, el aporte reserva saldo sin crear un
  movimiento.
- Si son diferentes, el aporte crea una transferencia real hacia la cuenta
  destino y reserva allí el dinero.
- El dinero reservado no puede usarse en gastos o transferencias normales.

## Listado

```http
GET /api/goals?period=YYYY-MM&view=all
```

Filtros admitidos:

```text
view: all | active | paused | overdue | completed | cancelled
type: SAVINGS | EMERGENCY_FUND | SPECIFIC_PURCHASE | DEBT_PAYMENT | OTHER
```

La respuesta contiene:

- `goals`: tarjetas calculadas por el backend.
- `counts`: cantidades por estado.
- `summary.totalReserved`: dinero actualmente reservado.
- `summary.totalTarget`: suma de los objetivos.
- `summary.activeGoals`: activas, pausadas y vencidas.
- `summary.completedThisYear`.
- `summary.periodContributionAmount` y `periodContributionCount` para el mes
  seleccionado.

Cada tarjeta debe mostrar:

- nombre, tipo, descripción, fecha límite y cuenta destino;
- `savedAmount`, `targetAmount`, `remainingAmount` y `progressPercentage`;
- `suggestedContribution`, `remainingPeriods` y `nextSuggestedDate`;
- `paceAmount`, `paceStatus`, `viewStatus` y `reservedAmount`.

Estados entregados por el backend:

```text
viewStatus: ACTIVE | PAUSED | OVERDUE | COMPLETED | CANCELLED
paceStatus: AHEAD | ON_TRACK | BEHIND | PAUSED | OVERDUE | COMPLETED
```

No determines los estados comparando fechas en el frontend. Usa exactamente los
valores recibidos.

## Crear una meta

```http
POST /api/goals
Content-Type: application/json
```

```json
{
  "name": "Ahorrar S/ 8,400 este año",
  "description": "Fondo anual",
  "type": "SAVINGS",
  "targetAmount": 8400,
  "startDate": "2026-01-01",
  "deadline": "2026-12-31",
  "frequency": "MONTHLY",
  "destinationAccountId": 4
}
```

Campos del formulario:

- Nombre obligatorio.
- Descripción opcional.
- Tipo de meta obligatorio.
- Monto objetivo obligatorio.
- Fecha de inicio obligatoria.
- Fecha límite obligatoria.
- Frecuencia: `DAILY`, `WEEKLY` o `MONTHLY`.
- Cuenta destino obligatoria.

La cuenta destino es donde se guardará físicamente el dinero. No crees cuentas
automáticamente. Si el usuario no tiene una cuenta apropiada, muestra un enlace
para ir a Cuentas y crearla manualmente.

## Editar

```http
PATCH /api/goals/:id
```

Envía solamente los campos modificados.

- Si ya existen aportes, bloquea la fecha inicial y la cuenta destino.
- El objetivo no puede ser menor que el monto ya ahorrado.
- Cambiar objetivo, plazo o frecuencia modifica el plan futuro y conserva el
  historial anterior.
- Después de editar, usa la respuesta del backend para actualizar aporte
  sugerido, ritmo y gráfico.

## Registrar aporte

Abre un formulario lateral con:

- meta seleccionada y avance actual;
- monto sugerido como referencia;
- monto del aporte editable y obligatorio;
- cuenta de origen obligatoria;
- fecha del aporte, por defecto hoy.

```http
POST /api/goals/:id/contributions
Content-Type: application/json
```

```json
{
  "amount": 700,
  "sourceAccountId": 1,
  "contributedOn": "2026-08-26"
}
```

Reglas:

- No aceptar fechas futuras ni anteriores al inicio de la meta.
- El aporte no puede superar `remainingAmount`.
- Mostrar el saldo disponible sin reservar de la cuenta, no solo el saldo total.
- Si origen y destino coinciden, informar: “Se reservará dinero de esta cuenta;
  no se creará una transferencia”.
- Si son diferentes, informar: “Se transferirá el dinero a la cuenta destino y
  quedará reservado para esta meta”.
- El usuario puede aportar más o menos que `suggestedContribution`.
- Al alcanzar el objetivo, la meta pasa automáticamente a `COMPLETED`.

Los aportes atrasados no se convierten en deudas. El backend recalcula el monto
sugerido usando el dinero faltante y los periodos restantes.

## Eliminar un aporte

```http
DELETE /api/goals/:id/contributions/:contributionId
```

Pide confirmación antes de eliminarlo.

- Si fue una reserva, se libera esa parte del saldo.
- Si creó una transferencia, el backend la revierte.
- Si la cuenta destino no tiene saldo disponible para revertirla, el backend
  responde `409`. Muestra el mensaje y conserva el aporte en pantalla.
- Si una meta completada deja de alcanzar el objetivo, vuelve a activa.

## Pausar y reanudar

```http
POST /api/goals/:id/pause
POST /api/goals/:id/resume
```

Pausar no cambia la fecha límite. Al reanudar, el backend conserva el plazo
original y recalcula el monto sugerido, que podría aumentar.

## Eliminar o cancelar

```http
DELETE /api/goals/:id
POST /api/goals/:id/cancel
```

- Sin aportes: `DELETE` elimina completamente la meta.
- Con aportes: `DELETE` la cancela y conserva el historial.
- Cancelar libera el dinero reservado, pero no lo devuelve automáticamente a la
  cuenta de origen. El saldo permanece en la cuenta destino.
- La respuesta incluye `cancellationType`: `DELETED` o `HISTORICAL`.

## Usar fondos

En una meta `COMPLETED`, muestra la acción **Usar fondos**:

```http
POST /api/goals/:id/use-funds
```

Esta acción libera la reserva para que el saldo pueda gastarse o transferirse.
No crea un gasto ni mueve dinero. El usuario registra después el gasto o la
transferencia real desde el módulo correspondiente.

## Detalle de una meta

```http
GET /api/goals/:id
```

Construye la vista con:

- barra de progreso y porcentaje;
- tarjetas Ahorrado, Falta, Aporte sugerido y Ritmo;
- gráfico esperado contra real usando `chart`;
- historial usando `contributions`;
- botón Registrar aporte;
- acciones Editar, Pausar/Reanudar, Cancelar y, cuando corresponda, Usar fondos.

Para el gráfico:

- línea esperada: `expectedAmount`, gris y discontinua;
- línea real: `actualAmount`, línea principal;
- `actualAmount: null` representa un mes futuro; no inventes un punto real;
- conserva exactamente el orden recibido en `chart`.

Aunque la frecuencia sea diaria o semanal, el gráfico se presenta agrupado por
meses.

## Integración con Cuentas

Estas consultas incluyen datos nuevos:

```http
GET /api/accounts
GET /api/accounts/summary
GET /api/accounts/:id
```

En cada cuenta muestra por separado:

- `currentBalance`: saldo total real.
- `goalReservedAmount`: dinero reservado para metas.
- `availableUnreservedBalance`: saldo que todavía puede utilizarse.

En el resumen general utiliza:

- `totalGoalReserved`.
- `totalAvailableUnreserved`.

No vuelvas a restar la reserva: `availableUnreservedBalance` ya viene calculado.
Una cuenta con dinero reservado no puede inactivarse. Muestra los mensajes `409`
del backend tal como llegan.

## Actualización obligatoria

Después de crear, editar, aportar, eliminar un aporte, pausar, reanudar, cancelar,
eliminar o usar fondos:

1. vuelve a consultar la meta o el listado;
2. vuelve a consultar las cuentas afectadas;
3. usa los nuevos valores del backend;
4. no ajustes manualmente saldos, ritmo ni progreso en el estado local.

## Pruebas obligatorias del frontend

- Crear una meta con cada tipo y frecuencia.
- Aportar usando la misma cuenta destino y comprobar que solo se reserva saldo.
- Aportar desde otra cuenta y comprobar la transferencia.
- Mostrar saldo total, reservado y disponible.
- Impedir seleccionar una tarjeta de crédito.
- Mostrar todos los estados y ritmos recibidos.
- Editar conservando el historial anterior.
- Verificar que el gráfico esperado/real se actualice.
- Eliminar un aporte de reserva y uno de transferencia.
- Mostrar correctamente errores `400`, `404` y `409`.
- Pausar y reanudar sin cambiar la fecha límite.
- Completar una meta y usar sus fondos.
- Eliminar una meta vacía y cancelar otra con historial.

---

# Mejoras nuevas: pagos recurrentes, variables y suscripciones

Implementa únicamente las mejoras descritas aquí. No reconstruyas estos módulos
ni modifiques el trabajo ya terminado. El backend es la fuente de verdad para el
calendario, los contadores, los estados, los importes y las pausas.

## Precio unitario y total del periodo

En las tarjetas de los tres módulos muestra por separado:

- `expectedAmount`: precio esperado de una sola ocurrencia. Ejemplo: S/ 40 por
  semana.
- `periodExpectedAmount`: total esperado del periodo seleccionado. Ejemplo:
  S/ 200 para cinco ocurrencias de S/ 40.
- `registeredAmount`: total realmente registrado.
- `expectedConfirmations`: cantidad de ocurrencias del periodo.
- `confirmedCount`: ocurrencias registradas con movimiento.
- `skippedCount`: ocurrencias omitidas porque no hubo pago o ingreso.
- `resolvedCount`: suma de registradas y omitidas.

El contador y la barra deben usar:

```text
resolvedCount / expectedConfirmations
```

No multipliques importes ni reconstruyas fechas en el frontend.

## Saltar una ocurrencia

Agrega en cada tarjeta una acción secundaria para indicar que una ocurrencia no
sucedió. El texto debe adaptarse al tipo y la frecuencia, por ejemplo:

- **Saltar gasto de este día**.
- **Saltar ingreso de esta semana**.
- **Saltar pago de este mes**.
- **Saltar renovación de este año**.

Si existen varias ocurrencias vencidas, abre un selector y permite omitirlas de
manera individual. No permitas escoger una fecha futura.

Peticiones:

```http
POST /api/recurring-payments/:id/skip
POST /api/variable-payments/:id/skip
POST /api/subscriptions/:id/skip
Content-Type: application/json
```

```json
{
  "occurrenceDate": "2026-08-26"
}
```

Una omisión:

- no crea movimiento;
- no modifica el saldo de la cuenta;
- devuelve `resolutionStatus: "SKIPPED"`;
- guarda `actualAmount: 0`;
- cuenta como resuelta;
- conserva la diferencia entre lo esperado y cero.

Para gastos y suscripciones, omitir S/ 40 produce una diferencia positiva de
S/ 40. Para ingresos, produce una diferencia negativa de S/ 40.

Después de omitir, vuelve a consultar el listado. No ajustes contadores ni
saldos manualmente.

## Deshacer una omisión

En el historial ofrece **Deshacer omisión** solamente cuando
`resolutionStatus` sea `SKIPPED`:

```http
DELETE /api/recurring-payments/:id/skip/:occurrenceDate
DELETE /api/variable-payments/:id/skip/:occurrenceDate
DELETE /api/subscriptions/:id/skip/:occurrenceDate
```

Al deshacerla, la ocurrencia vuelve a pendiente o vencida según el estado que
entregue el backend. Tampoco se mueve dinero.

## Pausar y reanudar

Esta mejora aplica a pagos recurrentes y suscripciones.

- Pausar inicia un intervalo sin actividad desde la fecha actual.
- Las ocurrencias anteriores a la pausa siguen pendientes o vencidas y pueden
  registrarse u omitirse.
- Las fechas comprendidas dentro de la pausa no deben aparecer como deuda ni
  generar ocurrencias.
- Reanudar conserva la fecha inicial y el ritmo original.
- No reinicies el calendario desde la fecha de reanudación.
- No cambies localmente el precio unitario, el total ni el contador.
- Usa `pausePeriods` para mostrar el historial de pausas cuando sea necesario.

Ejemplo: una recurrencia semanal iniciada el 1 de agosto, pausada del 10 al 22 y
reanudada el 23 conserva las fechas 1, 8 y 29. No se transforma en una
recurrencia semanal iniciada el 23.

Después de pausar o reanudar, vuelve a consultar el listado y representa los
valores recibidos.

## Errores que debe mostrar el frontend

Muestra el mensaje enviado por el backend cuando:

- se intenta registrar u omitir dos veces la misma ocurrencia;
- se intenta omitir una fecha futura;
- la fecha no pertenece al calendario configurado;
- la fecha estuvo dentro de una pausa;
- no existe la omisión que se intenta deshacer.

## Pruebas obligatorias del frontend

- Mostrar precio unitario y total del periodo sin confundirlos.
- Registrar una ocurrencia y comprobar el aumento de `confirmedCount`.
- Omitir una ocurrencia con importe cero y comprobar `skippedCount`.
- Confirmar que una omisión no cambia el saldo de la cuenta.
- Deshacer una omisión y comprobar que vuelve a pendiente o vencida.
- Bloquear fechas futuras.
- Elegir individualmente entre varios vencimientos.
- Pausar y reanudar conservando el calendario original.
- Verificar que las fechas dentro de la pausa no se contabilicen.
- Verificar que los vencimientos anteriores a la pausa se conserven.

---

# Prompt para frontend: módulo de Presupuestos

Implementa únicamente el módulo de **Presupuestos** y no reconstruyas los
módulos existentes. El presupuesto no crea, edita, elimina ni confirma
movimientos: únicamente los analiza.

Todas las peticiones requieren `Authorization: Bearer <token>`.

## Principio principal

```text
Movimientos confirmados = gasto real
Pagos recurrentes, variables y suscripciones pendientes = proyección
Presupuesto = control y comparación, nunca genera dinero ni movimientos
```

Solo los movimientos con gasto confirmado afectan `actualExpenseAmount`. Las
transferencias, ingresos, pagos pendientes y pagos omitidos no aumentan el gasto
real.

## Pantalla principal

Usa el mismo mes seleccionado globalmente. Consulta:

```http
GET /api/budgets?period=YYYY-MM
```

La respuesta incluye `general`, `categories` y `summary`. No recalcules
importes, porcentajes, alertas ni proyecciones en el frontend.

Encabezado igual al diseño:

- Título **Presupuestos**.
- Texto: “Controla tus gastos por categoría y recibe alertas a tiempo.”
- Botón principal: **Configurar presupuesto general** si no hay límite, o
  **Editar presupuesto general** si ya existe.
- Tarjeta de presupuesto general: gasto real, límite, disponible, porcentaje,
  gasto proyectado, disponible estimado y estado.
- Tarjetas de resumen: `summary.totalCategoryLimit`,
  `summary.actualExpenseAmount` y `summary.alerts.total`.
- Sección **Por categoría** con las tarjetas entregadas en `categories`.
- Bloque inferior de alertas que explique los umbrales 50%, 80% y 100%.

Las categorías ya existen. El backend devuelve exclusivamente categorías activas
de gasto del mes seleccionado; no solicites crear categorías desde Presupuestos.

## Datos de cada categoría

Cada elemento de `categories` contiene:

- `category`: nombre, color e ícono.
- `limitAmount`: límite del mes o `null` cuando no fue definido.
- `actualExpenseAmount`: gasto confirmado real.
- `pendingProjectedAmount`: pagos planificados que todavía no se registraron.
- `projectedExpenseAmount`: gasto real más lo pendiente planificado.
- `availableAmount`: límite menos gasto real.
- `projectedAvailableAmount`: límite menos gasto proyectado.
- `percentUsed` y `projectedPercentUsed`.
- `alertLevel`: `NO_LIMIT`, `CALM`, `ATTENTION`, `NEAR_LIMIT` o `EXCEEDED`.
- `configurationSource`: `CURRENT`, `INHERITED` o `NONE`.
- `configurationSourcePeriod`: mes desde el que se heredó el límite, si aplica.

Cuando `limitAmount` sea `null`, mostrar **Sin límite definido** y el botón
**Definir límite**. Si existe, mostrar **Editar límite** y **Quitar límite**.

Colores de estado:

```text
CALM: verde, menor de 50% real usado
ATTENTION: amarillo, desde 50%
NEAR_LIMIT: naranja, desde 80%
EXCEEDED: rojo, desde 100%
NO_LIMIT: neutro
```

La alerta se basa en `percentUsed` real, no en la proyección. La proyección debe
mostrarse como información secundaria, por ejemplo: “Estimado al cierre:
S/ 375 de S/ 600”.

## Configurar el presupuesto general

Al presionar el botón principal, abre el formulario de la imagen: un único campo
de monto y una explicación de las alertas. El periodo se toma del selector global
actual; no permitas elegir un mes distinto dentro del formulario.

```http
PUT /api/budgets/general
Content-Type: application/json
```

```json
{
  "period": "2026-09",
  "limitAmount": 3000
}
```

Para quitarlo, pedir confirmación y usar:

```http
DELETE /api/budgets/general?period=2026-09
```

## Definir o editar el límite de una categoría

Desde cada categoría abre un formulario con nombre de categoría, límite mensual
y las alertas informativas. No envíes el monto ya gastado: el backend lo calcula.

```http
PUT /api/budgets/categories/:categoryId
Content-Type: application/json
```

```json
{
  "period": "2026-09",
  "limitAmount": 600
}
```

Para quitar solo el límite de ese mes:

```http
DELETE /api/budgets/categories/:categoryId?period=2026-09
```

Se permite reducir el límite incluso debajo de lo ya gastado. Antes de guardar,
muestra una advertencia: “Ya gastaste S/ X. Este presupuesto quedará superado”.
No bloquees la operación.

## Cambio de mes e historial

Al cambiar a otro mes, consulta nuevamente `GET /api/budgets?period=...`.

- El gasto real del nuevo mes empieza en S/ 0 porque todavía no sucedió.
- Los límites se heredan del último mes configurado cuando no existe una
  configuración explícita para el nuevo mes.
- Los pagos pendientes se calculan como proyección del nuevo mes.
- Editar o quitar un límite en septiembre afecta solo septiembre.
- Los movimientos y límites de agosto permanecen como historial.

Muestra “Heredado de agosto de 2026” si `configurationSource` es `INHERITED`.
No crees copias locales de límites ni inventes datos para meses futuros.

## Proyección

La proyección llega ya calculada. Ejemplo:

```text
Comida
Gasto real: S/ 175
Pagos pendientes previstos: S/ 200
Estimado al cierre: S/ 375
```

No sumes manualmente el total esperado completo de un pago recurrente: el backend
solo entrega en `pendingProjectedAmount` las ocurrencias que aún siguen pendientes.
Cuando se registra un pago, deja de ser proyección y pasa a gasto real mediante
Movimientos.

## Actualización obligatoria

Después de definir, editar o quitar un límite, y después de crear, editar,
confirmar o eliminar un gasto, vuelve a consultar Presupuestos. Nunca ajustes
manualmente importes, barras, porcentajes o alertas en el estado local.

## Pruebas obligatorias del frontend

- Mostrar únicamente categorías activas de gasto.
- Mostrar una categoría sin límite y definirlo.
- Editar un límite y reducirlo por debajo de lo gastado.
- Quitar el límite solo del mes seleccionado.
- Configurar y quitar el presupuesto general.
- Cambiar de mes y comprobar límites heredados, gasto real S/ 0 y proyección.
- Registrar, editar y eliminar un gasto confirmado y comprobar el recálculo.
- Confirmar que transferencias e ingresos no aumentan el gasto usado.
- Mostrar correctamente los cinco estados de alerta.
- Mostrar proyección sin contarla dos veces como gasto real.

---

# Prompt para frontend: módulo de Calendario financiero

Implementa únicamente el módulo de **Calendario financiero**. No calcules
ocurrencias, vencimientos, montos ni estados en el frontend: el backend entrega
todos los eventos ya preparados.

Todas las peticiones requieren `Authorization: Bearer <token>`.

## Consulta principal

Para la vista mensual usa:

```http
GET /api/calendar?period=YYYY-MM
```

Para vista semanal o diaria usa un rango completo:

```http
GET /api/calendar?startDate=2026-08-24&endDate=2026-08-30
```

El rango puede tener como máximo 63 días. Al cambiar de mes, semana o día,
vuelve a consultar la ruta. Nunca combines cálculos locales con los datos de
otros módulos.

## Diseño de la pantalla

Construye la pantalla como el diseño de referencia:

- título **Calendario financiero**;
- subtítulo: “Pagos pendientes, recurrentes, suscripciones y vencimientos.”;
- controles **Mensual**, **Semanal** y **Diaria**;
- navegación anterior y siguiente;
- grilla principal de calendario;
- panel lateral **Próximos pagos**;
- panel lateral **Leyenda**.

La vista mensual debe dibujar los días con `events` agrupados por `date`.
La vista semanal y diaria usan la misma respuesta, cambiando únicamente la
forma de mostrarla. El frontend calcula solo la cuadrícula visual y los nombres
de los días; no genera eventos financieros.

## Datos entregados por el backend

La respuesta contiene:

```text
period
range.startDate / range.endDate
today
events
upcoming
legend
counts
```

Cada elemento de `events` incluye:

```text
id                 identificador estable para la interfaz
date               YYYY-MM-DD
title
description
amount
direction          INCOME | EXPENSE | TRANSFER
kind               MOVEMENT | RECURRING | VARIABLE | SUBSCRIPTION | DEBT | LOAN | GOAL | TRANSFER
status             PENDING | OVERDUE | CONFIRMED
legendKey
account
category
source.module
source.id
source.action
```

`upcoming` contiene hasta seis eventos futuros ordenados. Se muestra en el
panel lateral aunque algunos correspondan al siguiente mes.

## Leyenda y colores

Usa exactamente `legend` para pintar la leyenda. Cada elemento contiene `key`,
`label` y `color`.

```text
INCOME       verde: ingreso programado
EXPENSE      rojo: egreso pendiente
TRANSFER     azul: transferencia
SUBSCRIPTION morado: suscripción
GOAL         amarillo: meta o aporte
RECURRING    naranja: pago recurrente
VARIABLE     celeste: pago variable
CONFIRMED    gris: movimiento ya registrado
OVERDUE      rojo intenso: vencido
```

El color viene definido por `legendKey`. No uses verde o rojo solamente por el
signo del monto, porque una suscripción o un aporte debe conservar su identidad.

## Estados importantes

- `PENDING`: aún no ocurrió o no se registró.
- `OVERDUE`: fecha anterior a hoy que sigue pendiente.
- `CONFIRMED`: ya existe el movimiento real registrado.

Los pagos omitidos no aparecen como pendientes. Un pago registrado aparece una
sola vez: no dupliques el movimiento y el pago recurrente en el mismo día.

## Interacción con eventos

Al hacer clic en un evento, usa `source.module`, `source.id` y `source.action`:

- `REGISTER`: abrir el formulario de registrar pago, gasto o ingreso del módulo
  correspondiente, con la fecha de la ocurrencia.
- `PAY`: abrir pago de deuda.
- `COLLECT`: abrir cobro de préstamo.
- `CONFIRM`: abrir confirmación del movimiento pendiente.
- `VIEW`: abrir el detalle o historial correspondiente.

No confirmes ni registres dinero directamente desde la grilla sin abrir el
formulario correcto. Después de cualquier acción, recarga Calendario y el
módulo afectado.

## Monto y visualización

Muestra el monto según `direction`:

```text
INCOME:     +S/ monto
EXPENSE:    -S/ monto
TRANSFER:   S/ monto con ícono de transferencia
```

En la celda mensual muestra título, color y monto si hay espacio. Si hay más
eventos de los que caben, usa “+ N más” y abre un panel del día al hacer clic.
El panel del día debe listar título, monto, cuenta, categoría, estado y acción.

Resalta `today`. Los días fuera del mes se ven atenuados; no les asignes eventos
del mes por error.

## Panel de próximos pagos

Usa directamente `upcoming`:

- nombre;
- fecha;
- monto con signo;
- color de `legendKey`;
- estado vencido cuando corresponda.

No calcules este panel recorriendo pagos recurrentes, variables o suscripciones
por separado.

## Pruebas obligatorias del frontend

- Mostrar mensual, semanal y diaria con la misma fuente de datos.
- Agrupar correctamente los eventos por fecha.
- Mostrar pagos recurrentes, variables, suscripciones, deudas, préstamos,
  transferencias, aportes y movimientos.
- No duplicar un pago ya registrado.
- No mostrar un pago omitido como pendiente.
- Mostrar estados pendiente, vencido y confirmado.
- Verificar los colores y leyenda entregados por el backend.
- Navegar de mes y actualizar los datos.
- Abrir la acción correcta según `source.action`.
- Mostrar próximos pagos, incluso cuando pertenezcan al mes siguiente.

---

# Prompt para frontend: módulo de Reportes

Implementa únicamente el módulo visual de **Reportes**. El backend calcula los
montos reales, la proyección, los períodos, comparativas y gráficos. El
frontend no debe sumar movimientos ni volver a calcular pagos pendientes.

Todas las peticiones requieren `Authorization: Bearer <token>`.

## Consulta

Usa una sola ruta:

```http
GET /api/reports
```

Vistas disponibles:

```http
GET /api/reports?view=weekly&date=YYYY-MM-DD
GET /api/reports?view=monthly&period=YYYY-MM
GET /api/reports?view=annual&year=YYYY
GET /api/reports?view=custom&startDate=YYYY-MM-DD&endDate=YYYY-MM-DD
```

La vista semanal siempre va de **lunes a domingo**. La fecha solo indica qué
semana se desea mirar. La vista personalizada permite como máximo 366 días.

Filtros opcionales para cualquiera de las vistas:

```text
accountId=<id de cuenta>
categoryId=<id de categoría>
type=ALL | INCOME | EXPENSE | TRANSFER
```

Al cambiar pestaña, fechas o filtros, vuelve a consultar esta ruta. Para llenar
los selectores usa las rutas ya existentes de Cuentas y Categorías.

## Regla central: real y proyectado

La pantalla debe diferenciar siempre dos conceptos:

- **Real / registrado:** solo movimientos confirmados.
- **Proyectado al cierre:** lo real más los pagos aún pendientes del período.

La proyección incluye pagos recurrentes, pagos variables, suscripciones,
cuotas de deudas y cobros de préstamos pendientes. No incluye transferencias
como ingreso o gasto, porque el dinero solo cambia de cuenta.

Usar los datos recibidos:

```text
actual.incomeAmount
actual.expenseAmount
actual.netAmount
actual.transferAmount
actual.movementCount
actual.savingsRatePercentage

projected.incomeAmount
projected.expenseAmount
projected.netAmount
projected.pendingIncomeAmount
projected.pendingExpenseAmount
projected.pendingCount
```

En las tarjetas se recomienda mostrar el monto grande real y, debajo,
“Proyectado al cierre: S/ ...” cuando haya pagos pendientes. Así el usuario no
confunde lo que ya ocurrió con lo que podría ocurrir antes de cerrar el mes.

## Diseño general

Seguir el estilo de las referencias:

- título **Reportes**;
- subtítulo: “Vistas analíticas semanales, mensuales y anuales.”;
- pestañas: **Semanal**, **Mensual**, **Anual** y **Personalizado**;
- botón **Filtros** que abre/cierra cuenta, categoría y tipo;
- botón **Exportar** visible pero deshabilitado, con texto o tooltip
  “Próximamente”. No existe una ruta de exportación todavía.

No mostrar transferencias como ingreso, egreso, saldo neto ni ahorro. Solo se
consideran en la tarjeta/lista de cuentas con más movimiento cuando el filtro
de tipo sea `ALL` o `TRANSFER`.

## Vista semanal

Con `view=weekly` mostrar cuatro tarjetas:

- Ingresos de la semana: `actual.incomeAmount` y variación
  `comparison.incomeChangePercentage`.
- Egresos de la semana: `actual.expenseAmount` y variación
  `comparison.expenseChangePercentage`.
- Balance semanal: `actual.netAmount`, más su proyección si existe.
- Movimiento más grande: `highlights.largestMovement`.

Mostrar el gráfico de barras **Gastos por día** usando:

```text
charts.trend (granularity DAY)
date, label,
expenseActualAmount, expensePlannedAmount
```

La comparativa con la semana anterior usa `categoryComparison`:

```text
category
actualExpenseAmount
plannedExpenseAmount
projectedExpenseAmount
previousActualExpenseAmount
```

Cada fila debe enseñar gasto real, proyección y el monto real de la semana
anterior. Si no hubo gasto, mostrar S/ 0.00 sin inventar valores.

## Vista mensual

Construir las cuatro tarjetas de la referencia:

- Ingresos del mes: real, variación versus `comparison` y proyección al cierre.
- Egresos del mes: real, variación y proyección al cierre.
- Saldo neto: `actual.netAmount` y `projected.netAmount`.
- Ahorro del mes: `highlights.goalContributionAmount`, con la etiqueta
  “Aportes a metas”.

El gráfico **Evolución del saldo** usa `charts.trend`. Para mensual el backend
entrega seis puntos mensuales, con:

```text
incomeActualAmount
expenseActualAmount
incomePlannedAmount
expensePlannedAmount
netActualAmount
netProjectedAmount
```

Usar líneas o áreas distintas para ingresos y egresos reales; para el mes
seleccionado se puede añadir una línea punteada o etiqueta para la proyección.

Para **Categoría con mayor gasto** usar `highlights.topExpenseCategory`. El
porcentaje de la dona es `expenseSharePercentage`. Si no hay gastos, mostrar un
estado vacío en lugar de una dona con datos falsos.

Para **Cuentas con más movimiento** usar `accounts`, mostrando nombre,
`movementCount` y `volumeAmount`. Para **Presupuesto usado** usar `budget`:

```text
budget.limitAmount
budget.actualExpenseAmount
budget.projectedExpenseAmount
budget.percentUsed
budget.projectedPercentUsed
budget.categories
```

`budget` solo llega en la vista mensual sin filtros. Si se selecciona cuenta,
categoría o tipo, llega como `null`: ocultar este panel, porque el presupuesto
general no debe mezclarse con una vista parcial. Si no hay límite general,
mostrar “Sin presupuesto general configurado”, sin calcular porcentajes locales.

## Vista anual

Mostrar:

- ingresos del año: `actual.incomeAmount` y proyección recibida;
- egresos del año: `actual.expenseAmount` y proyección recibida;
- ahorro acumulado: `actual.netAmount`;
- cumplimiento de metas: `goals.completionPercentage` y
  `goals.completedGoals` de `goals.totalGoals`.

El gráfico **Ingresos vs Egresos** usa `charts.trend` con
`trendGranularity: MONTH`. Hay un punto por mes. Usar barras verdes para
`incomeActualAmount` y rojas para `expenseActualAmount`; los montos planificados
se muestran claramente como proyección, no como dinero ya recibido o gastado.

En las tarjetas inferiores usa el mismo arreglo para identificar el mes con
mayor ingreso y gasto. La tasa de ahorro llega calculada en
`actual.savingsRatePercentage`. Si llega como `null`, mostrar “Sin ingresos
registrados”; no dividir entre cero ni volver a calcularla.

## Vista personalizada

Antes de consultar, mostrar un formulario con:

- Desde (`startDate`);
- Hasta (`endDate`);
- Cuenta;
- Categoría;
- Tipo: Todos, Ingresos, Gastos o Transferencias;
- botón **Generar reporte**.

Tras presionar el botón, usar la respuesta normal y mostrar las tarjetas,
gráfico y listas aplicando los filtros. Para rangos de hasta 62 días,
`charts.trend` llega por día; para rangos mayores llega por mes. Revisar
`charts.trendGranularity` para elegir el gráfico correcto.

## Campos complementarios

La respuesta también incluye:

```text
range.startDate / range.endDate
filters
comparison.range
comparison.actual
comparison.incomeChangePercentage
comparison.expenseChangePercentage
comparison.netChangePercentage
categories
categoryComparison
goals
```

`categories` lista los gastos reales, pendientes y proyectados por categoría.
Usarla para tablas, dona y estados vacíos. No filtrar ni sumar por nombre en el
frontend: usar los datos y los identificadores entregados.

Cuando un porcentaje de comparación sea `null`, significa que el período
anterior fue cero y no existe una comparación porcentual válida. Mostrar “Sin
referencia anterior”, no `0%`.

## Pruebas obligatorias del frontend

- Semana de lunes a domingo, incluso cuando cruza de mes.
- Cambiar entre semanal, mensual, anual y personalizado.
- Mostrar real y proyectado sin sumarlos visualmente como si fueran un único
  movimiento confirmado.
- Confirmar que una transferencia no altere ingresos, egresos ni saldo neto.
- Probar filtros por cuenta, categoría y tipo.
- Mostrar gráfico diario y mensual según `trendGranularity`.
- Mostrar estados vacíos para categorías, movimiento mayor, cuentas o
  presupuesto inexistentes.
- No habilitar Exportar ni llamar una ruta de exportación hasta que el backend
  incorpore esa función.
