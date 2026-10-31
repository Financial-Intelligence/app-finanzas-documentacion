# Pendientes del backend solicitados por frontend

Fecha de corte: 2026-07-15

## Ya resuelto

1. Modulo Movimientos implementado.
2. `movementsCount` agregado a `GET /api/accounts/:id`.
3. Historial disponible en `GET /api/accounts/:id/movements`.
4. Eliminacion de cuenta bloqueada cuando existe historial.
5. Los movimientos confirmados actualizan saldos; cancelarlos o eliminarlos logicamente revierte el efecto.
6. Categorias y subcategorias mensuales implementadas con copia automatica y proteccion por movimientos.
7. Ingresos y egresos nacen pendientes y conservan monto esperado y monto real.

## Resuelto: POST /:id/skip respondia 500 en los tres modulos

Corregido en la migracion `20260826170000_allow_skipped_occurrences_zero_amount` (relaja los 3 CHECK constraints para permitir `actualAmount = 0` cuando `resolutionStatus = 'SKIPPED'`, exactamente como se sugirio abajo). Re-verificado en vivo en `recurring-payments`, `subscriptions` y `variable-payments`: skip y deshacer omision funcionan, saldo y movimientos no se alteran, calendario y presupuestos reflejan el cambio. Sin hallazgos nuevos.

## [Detalle historico, ya resuelto] POST /:id/skip respondia 500 en los tres modulos

Afecta a `recurring-payments`, `variable-payments` y `subscriptions`. Reproducido en vivo: crear cualquier tarjeta, ejecutar `POST /api/recurring-payments/:id/skip` con `{"occurrenceDate": "<fecha vigente>"}` devuelve 500:

```text
DriverAdapterError: new row for relation "RecurringConfirmation" violates
check constraint "RecurringConfirmation_amounts_positive"
```

Causa: las tablas `RecurringConfirmation`, `VariableConfirmation` y `SubscriptionPayment` tienen, desde su migracion de creacion, un CHECK:

```sql
CONSTRAINT "RecurringConfirmation_amounts_positive" CHECK (
    "expectedAmount" > 0 AND "actualAmount" > 0
)
```

(mismo patron en `VariableConfirmation_amounts_positive` y `SubscriptionPayment_amounts_positive`, en `prisma/migrations/20260825120000_create_recurring_payments`, `20260825160000_create_variable_payments` y `20260825170000_create_subscriptions`).

La migracion que agrego omitir/deshacer ocurrencia (`20260826150000_add_skipped_occurrences_and_pause_history`) inserta `actualAmount: 0` a proposito cuando `resolutionStatus: "SKIPPED"` (es el comportamiento documentado: una omision no crea movimiento ni monto real), pero nunca relajo estos tres constraints. El insert siempre viola la regla y la transaccion revierte con 500.

`POST /:id/skip` queda inutilizable en los tres modulos hasta que se corrija. `DELETE /:id/skip/:occurrenceDate` (deshacer) no se vio afectado porque nunca llega a insertar.

Arreglo sugerido: una migracion nueva que reemplace los tres constraints para permitir `actualAmount = 0` solo cuando `resolutionStatus = 'SKIPPED'`, por ejemplo:

```sql
ALTER TABLE "RecurringConfirmation" DROP CONSTRAINT "RecurringConfirmation_amounts_positive";
ALTER TABLE "RecurringConfirmation" ADD CONSTRAINT "RecurringConfirmation_amounts_positive" CHECK (
    "expectedAmount" > 0 AND (
        "actualAmount" > 0 OR ("actualAmount" = 0 AND "resolutionStatus" = 'SKIPPED')
    )
);
-- Repetir para VariableConfirmation_amounts_positive y SubscriptionPayment_amounts_positive.
```

El frontend ya tiene el boton "Saltar..." y el flujo de deshacer construidos, con tests unitarios en verde, esperando este fix para poder verificarse en vivo.

## Bug bloqueante: un usuario nuevo (registro) no recibe NINGUNA categoria

Reproducido en vivo: `POST /api/auth/register` con un usuario nuevo, sin ningun dato previo, y luego `GET /api/categories?period=<mes actual>` devuelve `{"categories": [], "counts": {"income": 0, "expense": 0}}`. Cero categorias, cero subcategorias. El usuario no puede categorizar ningun movimiento hasta crear cada categoria a mano.

Causa raiz confirmada en codigo:

- `registerUser` (`src/modules/auth/auth.service.js`) solo hace `prisma.user.create({...})`. Nunca crea `CategoryTemplate` ni `Category`.
- `listCategories` (`src/modules/categories/categories.service.js`) llama `initializeCategoryPeriod(userId, period)` en cada `GET`, que:
  1. Si ya existe `CategoryPeriod` para ese user+period, no hace nada.
  2. Si existe un periodo ANTERIOR del usuario, copia esas categorias hacia el periodo actual (`copyCategoryToPeriod`).
  3. Llama `addPermanentTemplatesToPeriod`, que solo copia templates de `categoryTemplate.findMany({where: {userId, isActive: true}})` — es decir, unicamente plantillas que el usuario YA tiene.

Para un usuario recien registrado no hay periodo anterior ni `CategoryTemplate` alguno (nadie los creo nunca), asi que los 3 pasos no hacen nada y el usuario queda con cero categorias para siempre, salvo que las cree manualmente una por una.

El UNICO lugar en todo el backend que crea `CategoryTemplate`/`Category` por default es `DEFAULT_CATEGORIES` dentro de `resetOwnFinancialData` (`src/modules/users/users.service.js`, el endpoint `POST /users/me/reset-data`) — y ese codigo solo corre cuando un usuario YA EXISTENTE ejecuta un reset, nunca en el registro. O sea: hoy la unica forma de que un usuario termine con categorias base es haber reseteado sus datos alguna vez; el registro normal (el flujo que usa el 100% de los usuarios nuevos) no bootstrapea nada.

Reproducido y limpiado (usuario de prueba `test-categorias-verify@example.com`, creado y eliminado vía Prisma tras confirmar el `categories: []`).

Pedido: que `registerUser` (o el primer `initializeCategoryPeriod` de un usuario sin `CategoryTemplate` alguno) tambien corra la misma logica de `DEFAULT_CATEGORIES` que hoy solo usa el reset — idealmente extrayendo `DEFAULT_CATEGORIES` a un helper compartido entre `auth.service.js`/`categories.service.js` y `users.service.js`, para no duplicar la lista en dos lugares y que quede sincronizada.

Aprovechando el mismo fix: como ya reportamos antes, ese `DEFAULT_CATEGORIES` tampoco setea `icon`/`color`/subcategorias (`create` solo manda `type`/`name`/`sortOrder`) — el frontend ya soporta ambos campos con fallback correcto (`CategoryCard` muestra un icono generico "dot" cuando `icon` es `null`, comportamiento esperado, nada que corregir ahi), asi que conviene resolver los dos pedidos juntos: que el seed default (usado tanto en registro como en reset) incluya icono y color por categoria, usando estos valores ya soportados por el frontend:

- Iconos disponibles (`IconName`): `tag, home, utensils, plug, film, car, heart, bag, book, store, gift, dot`.
- Colores disponibles (hex): `#8b5cf6, #3b82f6, #10b981, #f59e0b, #ec4899, #06b6d4, #f43f5e, #84cc16`.

Subcategorias por default son un plus deseable (el piloto `lumen-next` las tiene) pero no bloqueante — lo bloqueante es que hoy un usuario nuevo no tiene absolutamente ninguna categoria.

## Notificaciones: acoplar generacion y lectura no escala para SaaS

Revisamos `GET /api/notifications` (`src/modules/notifications/notifications.service.js`). Hoy `listNotifications` llama `synchronizeNotifications(userId)` **de forma sincronica en cada request**, antes de devolver la lista. `synchronizeNotifications` carga eventos de calendario (ventana de 33 dias), recalcula presupuestos completos y metas completas, y recien despues lee la tabla `Notification`. O sea: generar y leer estan pegados en el mismo endpoint.

Con el flujo que ya recomendamos para el frontend (consultar al cargar la app, al abrir la campana, tras cada mutacion de pagos, y opcionalmente cada 60s mientras este abierta), eso significa recalcular calendario+presupuesto+metas completos hasta una vez por minuto por cada pestaña activa. Funciona bien con pocos usuarios de prueba; con trafico real de SaaS (muchos usuarios, muchas pestañas abiertas en simultaneo) escala mal — cada poll de cada usuario dispara la misma agregacion pesada.

Dos hallazgos concretos, no opiniones:

**1. `createOrRefreshNotification` deja el `message`/`amount` desactualizado dentro del mismo nivel de alerta.**

```js
if (existing.type !== data.type) {
    await prisma.notification.update({ where: { id: existing.id }, data: { ...data, isRead: false, readAt: null } });
}
// si existing.type === data.type, no actualiza nada
```

Analogia: es como la luz de bateria baja del carro — se prende con "15%" y aunque la bateria siga bajando a 8%, 5%, 2%, el tablero sigue mostrando "15%" porque solo se refresca cuando la luz CAMBIA de color, no mientras siga siendo la misma alerta. Aca pasa igual: una notificacion `BUDGET_NEAR_LIMIT` creada cuando el gasto real era S/120 se queda mostrando "S/120" para siempre, aunque el usuario ya vaya en S/450, mientras siga sin cruzar a `EXCEEDED`.

Pedido: que el `update` corra siempre que cambien `message`/`amount`/`dueDate` (no solo cuando cambia `type`), no solo cuando escala de nivel.

**2. Generacion y lectura deberian desacoplarse antes de escalar trafico.**

Analogia: hoy es como pedirle al mozo que, cada vez que un cliente pregunta "¿mi mesa ya esta?", recuente TODAS las mesas del restaurante de cero antes de contestar — en vez de mantener una lista de "mesas listas" que se actualiza sola cuando una mesa se libera, y que el mozo solo consulta. Con 5 clientes preguntando no se nota; con 5000 preguntando cada minuto, el mozo no hace otra cosa.

Estrategia sugerida (arquitectura estandar de "servicio de notificaciones" en SaaS — no requiere websocket, la lectura sigue siendo polling del lado frontend):

- **Generar en el momento del evento real** (event-driven): cuando una accion del propio usuario cambia el estado (confirmar un movimiento que empuja un presupuesto a 80%, por ejemplo), generar la notificacion ahi mismo, dentro de esa misma transaccion — ya se tienen los datos cargados, sale gratis y es mas instantaneo que ahora.
- **Un job programado** (cron corriendo para TODOS los usuarios de una sola pasada, ej. cada hora o una vez al dia) para lo que depende solo del paso del tiempo — pagos que se vuelven vencidos, metas que se atrasan sin que el usuario haga nada.
- **`GET /notifications` pasa a ser una lectura pura** de la tabla `Notification` (query indexada, barata), sin recalcular nada — puede pollearse cada 60s sin problema sin importar cuantos usuarios esten activos.

Paso intermedio barato si no quieren meter cron todavia: un `lastSyncedAt` por usuario, y solo correr `synchronizeNotifications` si paso, por ejemplo, mas de 5 minutos desde la ultima vez — sin infraestructura nueva, tapa el peor caso mientras deciden si construyen el job de verdad.

No bloqueante para empezar a integrar notificaciones en el frontend ahora mismo (funciona correctamente con el volumen actual), pero conviene resolverlo antes de que el sistema reciba trafico real de SaaS.

## Resuelto: GET /api/dashboard - emptyState.hasBudget siempre daba false

Corregido en PR #25 exactamente como se sugirio: `report.budget?.general?.limitAmount != null` -> `report.budget?.limitAmount != null`. El frontend ya dejo de usar el workaround local y consume el flag real.

## Bug: notificaciones quedan huerfanas cuando se borra su entidad origen

Reproducido en vivo: se creo un movimiento de prueba (`Movement id 25`), el backend genero su notificacion `PAYMENT_DUE` (`sourceModule: "movements", sourceId: 25`) al listar `/notifications`. Se borro el movimiento (`DELETE`/script), pero la fila en `Notification` nunca se elimino ni se marco como invalida — quedo huerfana para siempre, apuntando a un `sourceId` que ya no existe, y siguio apareciendo en la campana (titulo/monto congelados desde la ultima sincronizacion) hasta que se borro a mano directo en la base.

Mismo problema aplica a cualquier entidad que genera notificacion y luego se borra/cancela: pagos recurrentes, variables, suscripciones, metas eliminadas. `synchronizeNotifications` solo crea o actualiza (`createOrRefreshNotification`), nunca borra una notificacion cuya entidad origen desaparecio.

Pedido: cuando se borra (o cancela) la entidad que origino una notificacion (`Movement`, `RecurringPayment`, `VariablePayment`, `Subscription`, `Goal`), borrar tambien las filas de `Notification` con ese `sourceModule`+`sourceId` (o, si se prefiere mantener historial, marcarlas invalidas/expiradas en vez de seguir mostrandolas). No bloqueante para uso normal (rara vez se borra algo con notificacion activa), pero se noto en vivo integrando la campana.

## Pedido: endpoint para eliminar/descartar una notificacion individual

Hoy `notifications.routes.js` solo tiene `GET /`, `GET/PATCH /preferences`, `PATCH /:id/read`, `PATCH /read-all` — no hay forma de que el usuario saque una notificacion puntual de la lista sin que sea "no leida". El frontend quiere ofrecer un boton "x" (o gesto de deslizar) para descartar una card individual de la campana.

Pedido: `DELETE /api/notifications/:id` (borra o marca como descartada esa notificacion puntual del usuario autenticado, 404 si no existe o no es suya). Mientras no exista, el frontend va a implementar "descartar" como un alias de "marcar como leida" + ocultarla de la vista (la campana solo listara no-leidas) — es honesto con lo que el backend ofrece hoy, pero no es un borrado real; cuando este el endpoint, cambiamos a un borrado de verdad.

## Pendiente principal

### GET /api/accounts/:id/summary

Continua pendiente. El modulo Movimientos ya permite calcular ingresos, gastos, transferencias y resultado real, pero todavia no existen los datos necesarios para:

- saldo heredado planificado;
- ingresos fijos del mes;
- resultado esperado;
- serie historica esperado contra real.

Estos valores dependen de `monthly_plans` y pagos recurrentes. El endpoint no se implementa con ceros o datos inventados.

Contrato esperado:

```text
GET /api/accounts/:id/summary?period=YYYY-MM
```

El objeto se devolvera directamente, sin envoltorio `{ summary }`, cuando sus dependencias existan.

## Decisiones aun pendientes fuera de Movimientos

1. Definir si inactivar la cuenta principal debe bloquearse o reasignar otra automaticamente.
2. Definir el uso final o retiro de `expectedMonthlyAmount`.
3. Mantener `monthlyPlan` solamente en `/accounts/:id/summary` para evitar duplicar logica.

## Resuelto: cantidad de ocurrencias por dia (`customDayCounts`)

Migracion corregida (`DROP INDEX IF EXISTS` en vez de `DROP CONSTRAINT`, todo idempotente) y aplicada. Probado en vivo end-to-end contra un pago recurrente real: `customDayCounts: {"27": 3}` crea 3 ocurrencias independientes el mismo dia, cada una se registra por separado con su `occurrenceNumber`, `occurrenceProgress` en la respuesta refleja `resolvedCount`/`pendingCount` en tiempo real, la 4ta ocurrencia se rechaza con 409 limpio ("La cantidad configurada para esta fecha ya fue completada"), y al completar las 3 el pago pasa a `viewStatus: FINISHED`. Sin hallazgos nuevos — el frontend ya puede consumir esto.

## [Detalle historico, ya resuelto] Pedido: cantidad de ocurrencias por dia (`customDayCounts`) — migracion rota, bloquea todo

Producto quiere que, al elegir un dia especifico dentro de `CUSTOM_DAYS` (Recurrentes y Variables — no aplica a Suscripciones, una suscripcion es un solo servicio con un ciclo fijo, no tiene sentido "3 Netflix el mismo dia"), el usuario pueda indicar cuantas veces se repite ese pago ese mismo dia (ej. "sabado 8 con cantidad 3" = 3 pagos independientes, cada uno registrable/saltable por separado, con progreso tipo "1 de 3 registrados").

**Actualizacion 2 (la mas reciente)**: el commit `ca0a405 contador` ya escribio la capa de aplicacion completa — schemas Zod (`customDayCounts`/`occurrenceNumber` en ambos modulos), `occurrenceLimitForDate` + expansion de slots pendientes en `service.js`, `register`/`skip`/`undoSkip` ya validan y usan `occurrenceNumber` (via query param `?occurrenceNumber=` en el DELETE, via body en los POST). El codigo en si esta bien — usan `findFirst` con where plano en vez de `findUnique` con compound key, asi que no dependen del nombre exacto del constraint. Buen trabajo ahi.

**Pero la migracion `20260828110000_add_custom_day_occurrences` esta rota y nunca se aplico de verdad.** Al correr `npx prisma migrate deploy` (recien, en este ambiente local) fallo a la mitad:

```
ERROR: constraint "RecurringConfirmation_recurringPaymentId_occurrenceDate_key" of relation "RecurringConfirmation" does not exist
```

Las 4 lineas de `ADD COLUMN` (`customDayCounts` x2, `occurrenceNumber` x2) **si corrieron** — confirmado directo contra Postgres, las columnas existen. Pero el `DROP CONSTRAINT "RecurringConfirmation_recurringPaymentId_occurrenceDate_key"` que sigue **fallo y ahi se corto todo**.

**Causa raiz**: ese objeto existe en la base como **indice unico** (`CREATE UNIQUE INDEX`, asi lo genero Prisma en la migracion original de creacion de la tabla), no como **constraint** (`ADD CONSTRAINT`). Son objetos distintos en Postgres — `DROP CONSTRAINT` no encuentra un indice suelto, hace falta `DROP INDEX`. Mismo problema espera en `VariableConfirmation`.

**Estado en el que queda la base ahora mismo** (no se toco nada mas despues de esto):

- Prisma marca esta migracion como **"failed"** — bloquea `migrate dev`/`migrate deploy` de cualquier otra migracion hasta que alguien la resuelva con `prisma migrate resolve --rolled-back "20260828110000_add_custom_day_occurrences"` (lo mas correcto, ya que el rename del constraint nunca se completo) o el codigo nuevo se ajusta y se corre de nuevo.
- El indice unico VIEJO de 2 campos sigue activo (el `DROP` fallo, nunca se borro) — el constraint nuevo de 3 campos nunca se creo. Si alguien intenta registrar una 2da ocurrencia el mismo dia ahora mismo con el codigo nuevo: la validacion propia (`findFirst`) no encuentra duplicado y deja pasar, pero el `INSERT` choca contra el indice viejo y tira un 500 crudo de Postgres en vez del 409 esperado.
- El cliente Prisma tampoco se regenero todavia (`npx prisma generate` pendiente, no lo corri porque la base quedo en un estado intermedio raro).

**Arreglo necesario**: nueva migracion correctiva que reemplace las 2 lineas de `DROP CONSTRAINT` (Recurring y Variable) por `DROP INDEX IF EXISTS "..."`. Despues: `prisma migrate resolve --rolled-back` para destrabar el historial, `migrate deploy` de nuevo, y `prisma generate`. Recien ahi se puede probar el feature de verdad.

No bloqueante para lo que ya esta en produccion (dias personalizados con cantidad 1 implicita sigue funcionando exactamente igual, esto solo bloquea migraciones nuevas en los ambientes que ya intentaron aplicar esta). El frontend sigue sin mandar `customDayCounts` — esperando a que esto se resuelva.

## Resuelto: los movimientos manuales (gasto/ingreso "hormiga") ya afectan el cumplimiento de la cuenta

Verificado en `accounts.service.js` (2026-10-01): `complianceAmount` ya suma `manualChange` (movimientos MANUAL, sin transferencias). Detalle histórico abajo.

### [Detalle histórico]

Hoy `complianceByPeriodForAccount` (`accounts.service.js`) solo suma `RecurringConfirmation` + `VariableConfirmation` + `SubscriptionPayment` con `resolutionStatus:"REGISTERED"` — nunca toca la tabla `Movement`. Un gasto o ingreso manual (el que crea el formulario de "Nuevo gasto/ingreso hormiga", `sourceType:"MANUAL"`) hoy **no mueve `complianceAmount`/`cushion` para nada**, aunque se confirme.

Producto quiere que si lo haga: un gasto manual confirmado debe bajar el cumplimiento por su monto real completo; un ingreso manual confirmado debe subirlo por su monto real completo. A diferencia de los otros 3 origenes (que comparan contra un monto esperado previo via `difference`), un movimiento manual no tiene "plan" previo — no hay una diferencia que calcular, es el monto real completo el que mueve la aguja.

Propuesta concreta: sumar al mismo calculo de `complianceByPeriodForAccount` los `Movement` con `type IN (INCOME, EXPENSE)`, `sourceType:"MANUAL"`, `status:"CONFIRMED"`, `deletedAt: null`, dentro del rango de `occurredOn` del periodo — sumando `actualAmount` para INCOME y restandolo para EXPENSE.

No bloqueante — el frontend ya avisa en el formulario que estos movimientos "no pertenecen a pagos recurrentes/variables/suscripciones" pero deliberadamente **no** promete que afecten el cumplimiento todavia, justamente porque hoy no es cierto.

## Resuelto: "Resetear mis datos" no borra las notificaciones

Verificado por código: `tx.notification.deleteMany({ where: { userId } })` agregado dentro de la misma transacción de `resetOwnFinancialData`, justo antes del `tx.account.deleteMany(...)`, exacto como se pidió. No se probó en vivo (ejecutar el reset borraría cuentas/categorías/todo del usuario real, no solo notificaciones) — la revisión de código alcanza, es un cambio de una línea sin ambigüedad.

## [Detalle histórico, ya resuelto] Bug: "Resetear mis datos" no borra las notificaciones

Usuario reporta: reseteo sus datos desde la config de la cuenta y las notificaciones del historial anterior siguen apareciendo (capturas de pantalla mostrando cobros/pagos vencidos de pagos recurrentes que ya no existen).

**Confirmado leyendo el codigo**: `resetOwnFinancialData` en `src/modules/users/users.service.js:90-179` borra, dentro de la misma transaccion, TODAS las tablas financieras del usuario (`debtPayment`, `debtInstallment`, `loanCollection`, `loanInstallment`, `recurringConfirmation`, `variableConfirmation`, `subscriptionPayment`, `goalContribution`, `recurringPayment`, `variablePayment`, `subscription`, `goal`, `debt`, `loan`, `movement`, `categoryBudget`, `monthlyBudget`, `subcategory`, `category`, `categoryTemplate`, `categoryPeriod`, `account`) pero **nunca** llama `tx.notification.deleteMany({ where: { userId } })`. El modelo `Notification` (`prisma/schema.prisma:189-212`) tiene `userId` directo, sin ninguna dependencia de las tablas que si se borran (no hay FK entre ambos), asi que no se borra en cascada tampoco.

**Fix exacto**: agregar una linea dentro de la transaccion de `resetOwnFinancialData` (`src/modules/users/users.service.js`, antes de la linea 175 `await tx.account.deleteMany(...)` o en cualquier punto de la transaccion, el orden no importa ya que Notification no tiene FK hacia estas tablas):

```js
await tx.notification.deleteMany({ where: { userId } });
```

No bloqueante para nada mas, pero rompe la promesa de "reset de fabrica" — el usuario ve notificaciones de pagos/cobros que ya no existen despues de resetear.

## Bug: `expectedDebtPayments`/`expectedLoanCollections` quedan en 0 para deudas/prestamos sin plan de cuotas

Contrato original (ya implementado y verificado en vivo por el frontend): `account.performance` debe incluir en `expectedDebtPayments`/`expectedLoanCollections` **todas** las cuotas de deuda/cobros de prestamo pendientes con vencimiento dentro del periodo, asignadas a esa cuenta — confirmadas o no, ya que "pueden ser pagadas en cualquier momento".

**Confirmado leyendo el codigo**: `debtAndLoanTotalsForAccount` (`src/modules/accounts/accounts.service.js:76-111`) solo lee de `prisma.debtInstallment`/`prisma.loanInstallment` — filas que solo se generan si la deuda/prestamo tiene plan de cuotas real (`hasInstallmentPlan(monthlyInterestRate, installment)` en `src/utils/installments.js:17`, exige `monthlyInterestRate > 0 || installment > 0`). Una deuda o prestamo simple — sin cuota mensual configurada, solo con `nextDueDate`/`dueDate` y un `balance`/`outstanding` total (el caso comun de "fulano me debe X, vence tal fecha", sin desglose de cuotas) **nunca genera esas filas** via `syncDebtInstallments`/el equivalente de loans. Resultado: `expectedDebtPayments`/`expectedLoanCollections` quedan en 0 permanentemente para esos registros, sin importar que tan cerca este el vencimiento, aunque el usuario los vea claramente listados como pendientes en `/deudas` y `/prestamos`.

Verificado con datos TEST reales: una deuda/prestamo creados CON `installment > 0` si generan la fila de instalment y si cuentan correctamente en la proyeccion (frontend ya listo y probado para este caso). El problema es especificamente el caso sin plan de cuotas.

**Fix propuesto**: en `debtAndLoanTotalsForAccount`, ademas de sumar `debtInstallment`/`loanInstallment` como ya hace, sumar tambien las deudas/prestamos ACTIVOS de esa cuenta que **no tengan plan de cuotas** (`monthlyInterestRate = 0 AND installment = 0`) cuyo `nextDueDate`/`dueDate` caiga dentro del periodo — usando su `balance`/`outstanding` completo como el monto pendiente de ese mes:

```js
const [lumpSumDebts, lumpSumLoans] = await Promise.all([
    prisma.debt.findMany({
        where: {
            userId, paymentAccountId: accountId, status: "ACTIVE", deletedAt: null,
            monthlyInterestRate: 0, installment: 0,
            nextDueDate: { gte: start, lte: end },
        },
        select: { balance: true },
    }),
    prisma.loan.findMany({
        where: {
            userId, status: "ACTIVE", deletedAt: null,
            monthlyInterestRate: 0, installment: 0,
            OR: [{ returnAccountId: accountId }, { returnAccountId: null, sourceAccountId: accountId }],
            dueDate: { gte: start, lte: end },
        },
        select: { outstanding: true },
    }),
]);
const lumpSumDebtTotal = lumpSumDebts.reduce((sum, d) => sum + Number(d.balance), 0);
const lumpSumLoanTotal = lumpSumLoans.reduce((sum, l) => sum + Number(l.outstanding), 0);
return {
    expectedDebtPayments: roundMoney(remaining(debtInstallments) + lumpSumDebtTotal),
    expectedLoanCollections: roundMoney(remaining(loanInstallments) + lumpSumLoanTotal),
};
```

No toca nada del frontend — `AccountDetailPage.tsx` ya lee `performance.expectedDebtPayments`/`performance.expectedLoanCollections` tal cual vienen y los muestra en el desglose de "Resultado esperado" (oculta la linea si el valor es 0). En cuanto el backend los llene correctamente para deudas/prestamos sin plan de cuotas, aparecen solos, sin ningun cambio adicional de UI.

## Resuelto (2026-10-31, corregido desde frontend con permiso): editar el monto de un pago recurrente no se reflejaba si existía una versión de horario con fecha posterior

Fix aplicado en `updateRecurringPayment`: `deleteMany` de versiones con `effectiveFrom > effectiveFrom` antes del `upsert`. Test: `tests/recurring-payments.update-versions.test.js`.

Reproducido con datos reales (2026-10-31): "SUELDO CONFLUX" (id 36), monto 1329.45 → editado a 1329.40. `PATCH` responde OK y `RecurringPayment.expectedAmount` queda en 1329.40, pero la tarjeta, el monto del periodo y la proyección siguen mostrando 1329.45.

Estado en la base después de guardar:

| `RecurringScheduleVersion` | `effectiveFrom` | `scheduleStartDate` | `expectedAmount` |
|---|---|---|---|
| id 5 (creada por la edición) | 2026-10-01 | 2026-10-31 | 1329.40 |
| id 1 (versión original) | 2026-10-31 | 2026-10-31 | 1329.45 |

Causa raíz (`src/utils/recurring-schedule.js`, `scheduleForDate`): para una fecha toma la versión con el `effectiveFrom` más reciente que sea `<=` esa fecha. Para la ocurrencia del 31-oct (y todas las siguientes), la versión 1 (31-oct) le gana a la versión 5 (1-oct). La edición queda "tapada" por una versión anterior que tiene fecha efectiva posterior.

En `updateRecurringPayment` (`src/modules/recurring-payments/recurring-payments.service.js`, bloque `if (scheduleChanged)`), el `upsert` crea la versión en `effectiveFrom` (inicio del periodo editado, `${period}-01`) pero no toca las versiones con `effectiveFrom` mayor. Pasa cada vez que la versión original se creó con `effectiveFrom = startDate` a mitad de mes (create sin `effectiveFrom`, o backfill de la migración `20261030010000_recurring_schedule_versions` usando `scheduleAnchorDate`).

Fix propuesto: una edición aplica "desde este periodo en adelante", así que antes del `upsert` borrar las versiones posteriores:

```js
await tx.recurringScheduleVersion.deleteMany({
    where: { recurringPaymentId: id, effectiveFrom: { gt: effectiveFrom } },
});
```

El frontend no necesita cambios: ya manda `expectedAmount`, `startDate` y `effectiveFrom: "${period}-01"`, y muestra el monto que devuelve el backend.
