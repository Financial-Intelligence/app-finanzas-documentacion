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
