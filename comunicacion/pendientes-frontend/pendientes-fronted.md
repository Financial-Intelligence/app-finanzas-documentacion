# Prompt para frontend: implementar únicamente Pagos Recurrentes

Implementa el módulo visual de **Pagos Recurrentes** consumiendo el backend
real. No uses mocks, no inventes rutas y no calcules ocurrencias en el frontend.

## Rutas disponibles

Todas requieren el token JWT:

```http
GET    /api/recurring-payments?period=YYYY-MM&view=pending|paused|finished|all&type=EXPENSE|INCOME
GET    /api/recurring-payments/:id?period=YYYY-MM
POST   /api/recurring-payments
PATCH  /api/recurring-payments/:id
POST   /api/recurring-payments/:id/register
POST   /api/recurring-payments/:id/pause
POST   /api/recurring-payments/:id/resume
DELETE /api/recurring-payments/:id
```

Frecuencias y etiquetas:

```text
DAILY       Diaria
WEEKLY      Semanal
BIWEEKLY    Quincenal
MONTHLY     Mensual
QUARTERLY   Trimestral
ANNUAL      Anual
```

El backend es la fuente de verdad para fechas, ocurrencias, contadores,
progreso, estado visual, próxima fecha y diferencias. Si una recurrencia usa
el día 29, 30 o 31, el backend ajusta automáticamente los meses cortos y vuelve
al día original cuando sea posible.

## Pantalla principal

Mostrar los resúmenes usando únicamente estos campos del backend:

- egresos esperados: `summary.expensesExpected`;
- egresos registrados: `summary.expensesActual`;
- ingresos esperados: `summary.incomeExpected`;
- ingresos registrados: `summary.incomeActual`;
- diferencia total: `summary.difference`;
- próximo registro: `summary.nextRecurringPayment` y
  `summary.nextOccurrenceDate`.

Pestañas:

- **Pendientes**: enviar `view=pending`;
- **Pausados**: enviar `view=paused`;
- **Finalizados**: enviar `view=finished`.

Usar `counts.pending`, `counts.paused`, `counts.finished` y `counts.all`.
Finalizado significa que completó las ocurrencias del período consultado; no
significa que terminó para siempre.

Cada tarjeta debe mostrar:

- nombre y descripción;
- tipo, cuenta y categoría;
- frecuencia y monto esperado;
- `progress`;
- `nextOccurrenceDate`;
- `periodExpectedAmount`;
- `periodActualAmount`;
- `periodDifference`.

Acciones de la tarjeta:

- si es `EXPENSE`: **Registrar gasto**;
- si es `INCOME`: **Registrar ingreso**;
- **Editar**;
- **Pausar** o **Reanudar**;
- **Eliminar**.

No utilizar el texto “Confirmar pago”.

## Crear una recurrencia

El formulario debe exigir nombre, tipo, monto esperado, frecuencia, fecha de
inicio, cuenta y categoría. La descripción es opcional. Filtrar las categorías
para que coincidan con el tipo y no mostrar tarjetas de crédito como cuentas
disponibles.

```json
{
  "name": "Internet",
  "description": "Servicio del hogar",
  "type": "EXPENSE",
  "expectedAmount": 100,
  "frequency": "MONTHLY",
  "startDate": "2026-08-10",
  "accountId": 15,
  "categoryId": 19
}
```

Después de crear, volver a consultar Pagos Recurrentes.

## Editar

Usar `PATCH /api/recurring-payments/:id` enviando solamente los campos que
cambiaron. Los cambios se aplican a las ocurrencias futuras; el frontend no
debe modificar ni recalcular registros históricos.

Después de editar, volver a consultar la lista y el detalle.

## Registrar gasto o ingreso

Al pulsar la acción, abrir un segundo formulario con:

- nombre de la recurrencia;
- ocurrencia: `nextOccurrenceDate`;
- monto esperado como referencia;
- monto real editable, inicialmente igual al esperado;
- fecha real del movimiento, inicialmente hoy;
- vista previa de la diferencia.

Enviar:

```json
{
  "occurrenceDate": "2026-08-10",
  "actualAmount": 80,
  "registeredDate": "2026-08-25"
}
```

`registeredDate` es opcional. El backend crea el movimiento confirmado y
devuelve `confirmation` con `expectedAmount`, `actualAmount`, `difference`,
`occurrenceDate`, `registeredDate`, cuenta y movimiento.

Mostrar la diferencia que entrega el backend:

- gasto: esperado menos real;
- ingreso: real menos esperado;
- positivo: favorable;
- negativo: desfavorable.

Ejemplo visual:

```text
Internet
Esperado: S/ 100.00
Pagado: S/ 80.00
Diferencia: +S/ 20.00
```

No modificar saldos manualmente. Después del registro, refrescar Pagos
Recurrentes, Cuentas, resumen de Cuentas y Movimientos. Si el backend responde
`409`, mostrar su `message`; puede indicar que la ocurrencia ya fue registrada.

## Pausar, reanudar y eliminar

`pause` y `resume` no llevan body. Una tarjeta pausada no debe mostrar la acción
de registro. Al reanudar, el backend continúa desde la fecha actual sin generar
los registros que pasaron durante la pausa.

Antes de eliminar mostrar:

> Se eliminará esta recurrencia y dejará de aparecer. Los gastos o ingresos ya registrados se conservarán en el historial.

Después de eliminar, retirar la tarjeta y refrescar Movimientos. Nunca borrar
los movimientos históricos desde el frontend.

## Cambio real de día y mes

Al abrir la pantalla, usar el `period` que responde el backend. Programar una
nueva consulta al llegar las 00:00 en la zona horaria del usuario y también al
recuperar el foco o visibilidad de la pestaña.

Cuando cambie el día o mes, refrescar Pagos Recurrentes, Cuentas, sus resúmenes
y Movimientos. El frontend solo solicita nuevamente los datos; el backend hace
todos los cálculos y conserva el historial anterior.

## Integración nueva con Cuentas

Consumir:

```http
GET /api/accounts?period=YYYY-MM
GET /api/accounts/summary?period=YYYY-MM
```

Cada cuenta incluye `recurringDifference` y el resumen incluye
`summary.recurringDifference`. Mostrarlo separado con el título **Diferencia
frente a lo planificado**:

- positivo: **A favor**;
- negativo: **Exceso / faltante**;
- cero: **Sin diferencia**.

No sumar `recurringDifference` a `currentBalance`; el movimiento real ya cambió
el saldo y sumarlo otra vez duplicaría el resultado.

## Actualización de datos y errores

Después de crear, editar, registrar, pausar, reanudar o eliminar, invalidar y
consultar nuevamente los datos afectados. Usar fechas `YYYY-MM-DD`, montos
numéricos y mostrar el `message` del backend ante respuestas `400`, `401`,
`404` o `409`. No cerrar el formulario si hubo un error.

## Pruebas obligatorias

- Mensual: `0 / 1`, registro, paso a Finalizados y regreso a Pendientes en el
  siguiente mes.
- Semanal: respetar exactamente las ocurrencias devueltas por el backend.
- Diaria: meses de 28, 29, 30 y 31 días e inicio en mitad del mes.
- Quincenal, trimestral y anual.
- Inicio el día 31, ajuste en febrero y regreso al día 31 en marzo.
- Gasto e ingreso con monto menor, igual y mayor al esperado.
- Mensaje ante registro duplicado.
- Pausa y reanudación sin registros atrasados.
- Edición sin alterar el historial.
- Eliminación conservando Movimientos.
- Actualización al cambiar de mes y al recuperar el foco.
- Diferencia visible en Cuentas sin modificar dos veces el saldo.

No se considera terminado hasta probar todos estos casos contra el backend
real, sin mocks ni rutas distintas de las indicadas.
