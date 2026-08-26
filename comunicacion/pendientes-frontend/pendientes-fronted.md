# Prompt para frontend: nuevas mejoras de planificación y Cuentas

Implementa únicamente los cambios descritos aquí. Pagos Recurrentes, Pagos
Variables y Suscripciones ya existen; no reconstruyas esos módulos ni elimines
sus comportamientos actuales.

El backend es la fuente de verdad para fechas, estados, contadores y montos.
Después de crear, editar, registrar, pausar o reanudar, vuelve a consultar el
listado correspondiente.

## Días personalizados en los tres módulos

Pagos Recurrentes, Pagos Variables y Suscripciones admiten ahora:

```text
frequency: CUSTOM_DAYS
customDays: number[]
```

Cuando el usuario seleccione la frecuencia **Días personalizados**, muestra un
selector que permita elegir uno o varios números del 1 al 31.

Ejemplo:

```json
{
  "frequency": "CUSTOM_DAYS",
  "customDays": [5, 15, 31]
}
```

Reglas:

- `customDays` es obligatorio únicamente con `CUSTOM_DAYS`.
- No envíes números repetidos.
- Las fechas elegidas se repiten cada mes.
- Si un día no existe en un mes, se usa el último día disponible.
- Por ejemplo, el día 31 se convierte en 28 de febrero de 2026.
- Si dos números terminan convertidos en la misma fecha, existe una sola
  ocurrencia.
- Al cambiar a otra frecuencia, no envíes `customDays`.
- Al editar los días, el cambio afecta las ocurrencias futuras. El historial ya
  registrado se conserva.

## Bloqueo de registros futuros

Aplica estas reglas en los tres módulos:

- Nunca habilites una ocurrencia cuya fecha sea posterior a la fecha real actual.
- El backend también rechaza fechas futuras con estado HTTP `409`.
- Cada ocurrencia se registra una sola vez.
- Después de registrarla, bloquea o elimina su botón de registro.
- Una ocurrencia vencida sí puede registrarse posteriormente.
- Los vencidos se registran individualmente porque cada uno puede tener un monto
  real diferente.
- Pueden registrarse varios vencidos el mismo día, pero uno por uno.
- Cada registro conserva su `occurrenceDate` original.
- La fecha real del registro puede ser hoy aunque la ocurrencia sea anterior.

Comportamiento por frecuencia:

- Diaria: como máximo una ocurrencia por día.
- Semanal: después de registrar la ocurrencia disponible, la siguiente permanece
  bloqueada hasta que llegue su fecha.
- Días personalizados: solo pueden registrarse los días elegidos que ya hayan
  llegado.

En el formulario de registro muestra el monto esperado unitario como valor
inicial, permite cambiar el monto real y envía la fecha original en
`occurrenceDate`.

## Pagos Recurrentes

Frecuencias disponibles:

```text
DAILY
WEEKLY
BIWEEKLY
MONTHLY
QUARTERLY
ANNUAL
CUSTOM_DAYS
```

Orden exacto de pestañas:

1. Pendientes
2. Pausados
3. Vencidos
4. Finalizados
5. Todos

Consultas:

```text
GET /api/recurring-payments?period=YYYY-MM&view=pending
GET /api/recurring-payments?period=YYYY-MM&view=paused
GET /api/recurring-payments?period=YYYY-MM&view=overdue
GET /api/recurring-payments?period=YYYY-MM&view=finished
GET /api/recurring-payments?period=YYYY-MM&view=all
```

Cada tarjeta debe mostrar:

- `expectedAmount`: monto esperado unitario.
- `periodExpectedAmount`: monto esperado total del mes.
- `confirmedCount`, `expectedConfirmations` y `progress`.
- `pendingCount` y `overdueCount`.
- `nextOccurrenceDate`.
- `viewStatus`: `PENDING`, `PAUSED`, `OVERDUE` o `FINISHED`.

Una tarjeta es `OVERDUE` cuando conserva al menos una ocurrencia pendiente cuya
fecha ya pasó. Después de registrar todos los vencidos:

- pasa a Pendientes si todavía existen fechas futuras del mes;
- pasa a Finalizados si completó todas las ocurrencias del mes.

## Pagos Variables

Frecuencias disponibles:

```text
DAILY
WEEKLY
MONTHLY
CUSTOM_DAYS
```

Los días personalizados se calculan únicamente dentro del `period` de la
tarjeta. La tarjeta no se replica en meses posteriores.

Orden exacto de pestañas:

1. Pendientes
2. Confirmados
3. Vencidos
4. Todos

Consultas:

```text
GET /api/variable-payments?period=YYYY-MM&view=pending
GET /api/variable-payments?period=YYYY-MM&view=confirmed
GET /api/variable-payments?period=YYYY-MM&view=overdue
GET /api/variable-payments?period=YYYY-MM&view=all
```

Cada tarjeta debe mostrar:

- `expectedAmount`: monto esperado unitario.
- `periodExpectedAmount`: total esperado del mes.
- `confirmedCount`, `expectedConfirmations` y `progress`.
- `pendingCount` y `overdueCount`.
- `nextOccurrenceDate`.
- `viewStatus`: `PENDING`, `CONFIRMED` u `OVERDUE`.

Un pago variable de un mes anterior puede registrarse tarde. El movimiento real
se crea en la fecha de registro y conserva su relación con la ocurrencia original.

## Suscripciones

Conserva la opción existente `CUSTOM_MONTHS` con `intervalMonths` y añade por
separado `CUSTOM_DAYS` con `customDays`.

Frecuencias disponibles:

```text
DAILY
WEEKLY
MONTHLY
ANNUAL
CUSTOM_MONTHS
CUSTOM_DAYS
```

No envíes `intervalMonths` con `CUSTOM_DAYS` ni `customDays` con
`CUSTOM_MONTHS`.

Orden exacto de pestañas:

1. Pendientes
2. Pausadas
3. Vencidas
4. Finalizadas
5. Todas

Consultas:

```text
GET /api/subscriptions?period=YYYY-MM&view=pending
GET /api/subscriptions?period=YYYY-MM&view=paused
GET /api/subscriptions?period=YYYY-MM&view=overdue
GET /api/subscriptions?period=YYYY-MM&view=finished
GET /api/subscriptions?period=YYYY-MM&view=all
```

El backend todavía acepta `view=confirmed` por compatibilidad, pero el frontend
nuevo debe utilizar `view=finished`.

Cada tarjeta debe mostrar:

- `expectedAmount`: monto unitario.
- `periodExpectedAmount`: total esperado del mes.
- `projectedCostNext12Months`.
- `confirmedCount`, `expectedConfirmations` y `progress`.
- `pendingCount` y `overdueCount`.
- `nextRenewalDate`.
- `viewStatus`: `PENDING`, `PAUSED`, `OVERDUE`, `FINISHED` o `NO_DUE`.

`NO_DUE` aparece únicamente en Todas cuando una suscripción anual o por cantidad
de meses no tiene renovación en el mes seleccionado.

## Rendimiento en el detalle de Cuentas

Consulta el detalle usando el mes seleccionado en el calendario global:

```http
GET /api/accounts/:id?period=YYYY-MM
```

Para cuentas que no son tarjetas de crédito, la respuesta incluye
`account.performance`:

```json
{
  "period": "2026-08",
  "openingBalance": 3000,
  "expectedIncome": 500,
  "expectedExpense": 1000,
  "expectedClosingBalance": 2500,
  "actualBalanceToDate": 2700,
  "projectedClosingBalance": 2550,
  "cushion": 50,
  "chart": []
}
```

Significado:

- `openingBalance`: saldo con el que comenzó el mes.
- `expectedIncome`: ingresos planificados del mes.
- `expectedExpense`: gastos planificados del mes.
- `expectedClosingBalance`: resultado esperado al cerrar el mes.
- `actualBalanceToDate`: saldo real alcanzado con movimientos confirmados.
- `projectedClosingBalance`: saldo actual más ingresos y gastos pendientes.
- `cushion`: proyección menos resultado esperado.

Muestra `cushion`:

- positivo: verde y con signo `+`;
- negativo: rojo;
- cero: color neutral.

No sumes `cushion` nuevamente al saldo actual. Solo es un indicador comparativo.

### Gráfico

`chart` contiene exactamente seis meses y termina en el mes seleccionado.

Dibuja dos líneas:

- Esperado: `expectedClosingBalance`, gris y discontinua.
- Real/proyectado: `projectedClosingBalance`, línea principal.

Cada punto puede ser verde si `cushion` es positivo, rojo si es negativo y
neutral si es cero.

El backend ya incluye los ingresos, gastos y transferencias reales en la
proyección. Las transferencias no forman parte del monto esperado porque no son
planificaciones.

Para tarjetas de crédito, `performance` es `null`. No dibujes este gráfico para
esas cuentas.

## Actualización de interfaz

- Usa el mes global en todas las consultas.
- Al cambiar de mes, vuelve a consultar los tres módulos y el detalle de Cuentas.
- Después de una operación exitosa, vuelve a consultar la lista; no calcules los
  estados manualmente.
- Si el backend responde `409` por una fecha futura o duplicada, muestra su
  mensaje y actualiza la tarjeta.
- Conserva el monto real de Cuentas; las diferencias y el colchón son indicadores
  separados.

## Pruebas obligatorias del frontend

- Crear y editar días personalizados en los tres módulos.
- Verificar que el día 31 use el último día de un mes corto.
- No habilitar ni enviar fechas futuras.
- Registrar individualmente varios vencidos el mismo día.
- Verificar el cambio Vencido a Pendiente o Finalizado.
- Respetar el orden exacto de las pestañas.
- Mostrar monto unitario y total del periodo por separado.
- Cambiar el mes global y actualizar estados y rendimiento.
- Dibujar los seis meses del gráfico.
- Aplicar colores correctos al colchón positivo, negativo y cero.
- No dibujar rendimiento cuando `performance` sea `null`.
