# Nuevo módulo: Dashboard

Crear la pantalla **Dashboard** usando una sola consulta:

```text
GET /api/dashboard?period=YYYY-MM
Authorization: Bearer <token>
```

El `period` debe ser el mes seleccionado de forma global en la aplicación. Si
no se envía, el backend usa el mes actual. No recalcular montos en el frontend:
la respuesta ya entrega valores reales, proyecciones, comparaciones y datos
para los gráficos.

## Encabezado

- Saludar con `user.name`.
- Mostrar el mes de `period`.
- El botón **Registrar gasto** abre el flujo de gasto manual ya existente.
- Las acciones rápidas abren los flujos ya existentes de gasto, ingreso,
  transferencia y aporte a meta; no crean movimientos desde el dashboard por
  su cuenta.

## Tarjetas superiores

Usar `summary`:

```text
summary.totalBalance
summary.activeAccounts
summary.income.actualAmount
summary.income.projectedAmount
summary.income.comparisonPercentage
summary.expense.actualAmount
summary.expense.projectedAmount
summary.expense.comparisonPercentage
summary.net.actualAmount
summary.net.projectedAmount
summary.net.savingsRatePercentage
summary.currency
```

- **Saldo total**: `totalBalance` y cantidad de cuentas activas.
- **Ingresos del mes**, **Egresos del mes** y **Balance del mes** muestran el
  monto real confirmado. Si la proyección es distinta, mostrarla como “Al cierre
  del mes: S/ …”, sin sumarla al monto real.
- Para comparaciones `null`, mostrar “Sin referencia anterior”.

## Ingresos vs Egresos

Usar `charts.incomeExpenseTrend`. Hay seis puntos mensuales; cada punto tiene:

```text
label
incomeActualAmount
expenseActualAmount
incomePlannedAmount
expensePlannedAmount
netActualAmount
netProjectedAmount
```

Mostrar líneas verdes y rojas con los montos reales. Las cantidades planeadas
deben verse de forma diferenciada (línea punteada, etiqueta o tooltip), nunca
como dinero confirmado. Antes de dibujar, convertir los valores vacíos,
`null`, `undefined` o no numéricos a `0`; nunca usar coordenadas SVG con `NaN`
ni dividir entre cero. Si el gráfico no tiene datos, mostrar un estado vacío.

## Gastos por categoría

Usar `expensesByCategory.categories` para la dona y la lista. Cada elemento
incluye `category`, `actualExpenseAmount`, `plannedExpenseAmount` y
`projectedExpenseAmount`.

Si `emptyState.hasExpenseCategories` es `false`, mostrar “Aún no hay gastos
confirmados este mes” en vez de una dona vacía o con valores inventados.

## Presupuesto y metas

- Usar `budget` para la tarjeta **Presupuesto del mes**. Puede ser `null` o no
  tener límite general. En ese caso, mostrar “Sin presupuesto general
  configurado” y un enlace a Presupuestos.
- La barra usa el monto real `budget.general.actualExpenseAmount`; la proyección
  se muestra aparte con `projectedExpenseAmount`.
- Mostrar hasta tres categorías con límite desde `budget.categories`.
- Usar `goals.activeCount` y `goals.items` para **Metas destacadas**. Cada meta
  ya incluye avance, porcentaje, estado y aporte sugerido. Si no hay metas,
  mostrar un enlace para crear una.

## Parte inferior

- **Últimos movimientos**: usar `recentMovements`; son como máximo seis
  movimientos confirmados del mes seleccionado.
- **Cuentas principales**: usar `accounts`; mostrar nombre, tipo, saldo actual,
  moneda y distinguir la cuenta principal cuando `isPrimary` sea verdadero.
- **Próximos pagos**: usar `upcomingPayments`. Cada ítem ya trae fecha, monto,
  dirección, estado, tipo y `source` para navegar al módulo correspondiente.
- Si no hay datos, usar los indicadores de `emptyState` y mostrar mensajes
  claros en lugar de tarjetas o gráficos vacíos.

# Nuevo módulo: Notificaciones dentro de la aplicación

El ícono de campana debe consultar:

```text
GET /api/notifications?status=all&limit=20
Authorization: Bearer <token>
```

Hacer la consulta al cargar la aplicación, cada vez que se abra la campana y
después de registrar, omitir, pausar, reanudar o editar un pago. También puede
actualizarse cada 60 segundos mientras la aplicación esté abierta.

La respuesta tiene:

```text
unreadCount
notifications[]
```

Mostrar `unreadCount` como una insignia sobre la campana solo cuando sea mayor
que cero. Cada notificación incluye:

```text
id
type
title
message
amount
currency
dueDate
sourceModule
sourceId
sourceAction
isRead
createdAt
```

## Panel de la campana

- Mostrar las no leídas primero.
- Si no hay notificaciones, mostrar: **“Estás al día. No tienes notificaciones
  nuevas.”**
- Si `amount` y `currency` existen, formatear el importe según la moneda:
  `PEN` como `S/`, `USD` como `US$`; no convertir monedas ni sumar importes de
  monedas diferentes.
- Diferenciar visualmente pago próximo, pago vencido, presupuesto al 50/80/100
  %, meta atrasada, meta vencida y meta completada.
- Al hacer clic en una notificación, navegar según `sourceModule` y `sourceId`;
  luego marcarla como leída.

Para marcar una como leída:

```text
PATCH /api/notifications/:id/read
```

Para marcar todas como leídas:

```text
PATCH /api/notifications/read-all
```

No usar notificaciones del navegador ni correo por ahora. Todo se muestra
solamente dentro de la aplicación.

# Transferencias entre monedas

Las cuentas ya traen su moneda en `currency` (por ejemplo `PEN` o `USD`). Al
crear o editar una transferencia, comparar la moneda de la cuenta de origen
con la de destino.

- Si ambas monedas son iguales, conservar el formulario actual: solo se pide el
  monto enviado. No mostrar conversión.
- Si las monedas son distintas, mostrar claramente:
  - **Monto enviado** y la moneda de origen.
  - **Monto que recibirá la cuenta destino** y la moneda de destino.
  - **Tipo de cambio** opcional, con la ayuda: “cuánto recibe la moneda destino
    por cada unidad enviada”.
- El usuario debe ingresar el monto recibido **o** el tipo de cambio. Si llena
  uno, calcular y mostrar el otro como vista previa. Si llena ambos y no
  coinciden, mostrar un error antes de enviar.
- No consultar ni inventar cotizaciones automáticas por ahora. El usuario
  indica el dato real de su operación.

Crear una transferencia:

```text
POST /api/movements/transfer
Authorization: Bearer <token>
```

Ejemplo de S/ 380 que se convierten a US$ 100:

```json
{
  "amount": 380,
  "destinationAmount": 100,
  "accountId": 1,
  "toAccountId": 2,
  "date": "2026-08-27",
  "description": "Cambio de soles a dólares"
}
```

También se puede enviar `exchangeRate` en vez de `destinationAmount`. Para el
ejemplo anterior sería `0.26315789`, porque son dólares recibidos por cada sol
enviado.

Para una transferencia pendiente, al confirmar usar el formulario de monto
real existente. Si las monedas son distintas, incluir igualmente
`destinationAmount` o `exchangeRate` en:

```text
POST /api/movements/:id/confirm
```

La respuesta de cualquier transferencia incluye:

```text
currency                    moneda enviada
destinationCurrency         moneda recibida
expectedAmount              monto previsto enviado
actualAmount                monto real enviado
destinationExpectedAmount   monto previsto recibido
destinationActualAmount     monto real recibido
exchangeRate                tipo de cambio aplicado
```

En el historial o detalle de una transferencia entre monedas, mostrar ambos
importes, por ejemplo: **“Enviaste S/ 380.00 · Recibiste US$ 100.00”**, y el
tipo de cambio. Al cancelar o eliminar, el backend revierte ambos saldos.

No sumar ni comparar montos de monedas diferentes en una sola tarjeta. Mostrar
cada saldo con su moneda; una conversión global a moneda base no existe todavía.

# Configuración: preferencias de notificaciones

En **Configuración > Notificaciones** mostrar únicamente tres interruptores
reales. No mostrar por ahora opciones de correo ni noticias del producto,
porque esas funciones no existen todavía.

Al abrir esta sección consultar:

```text
GET /api/notifications/preferences
Authorization: Bearer <token>
```

La respuesta es:

```json
{
  "preferences": {
    "budgetAlertsEnabled": true,
    "paymentRemindersEnabled": true,
    "goalAlertsEnabled": true
  }
}
```

Interruptores y textos:

- **Alertas de presupuesto**: “Te avisamos al 50 %, 80 % y cuando superes tu
  presupuesto.” Usa `budgetAlertsEnabled`.
- **Pagos y cobros próximos**: “Te recordamos pagos o cobros próximos y
  vencidos.” Usa `paymentRemindersEnabled`.
- **Avance de metas**: “Te avisamos si una meta se atrasa, vence o se completa.”
  Usa `goalAlertsEnabled`.

Todos vienen activos por defecto. Al cambiar uno, enviar solo ese campo:

```text
PATCH /api/notifications/preferences
Authorization: Bearer <token>
```

Ejemplo:

```json
{
  "paymentRemindersEnabled": false
}
```

Esperar la respuesta antes de confirmar visualmente el interruptor; si falla,
volver al valor anterior y mostrar el error. Después de guardar, volver a
consultar `GET /api/notifications` para actualizar la insignia de la campana.

Cuando un grupo está desactivado, sus avisos no aparecen ni cuentan en la
campana. No se borran del historial: al reactivarlo, pueden mostrarse de nuevo
si aún siguen vigentes.
