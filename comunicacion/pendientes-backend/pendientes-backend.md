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

## Bug menor: GET /api/dashboard - emptyState.hasBudget siempre da false

Reproducido integrando el nuevo endpoint en el frontend. `dashboard.service.js` calcula `emptyState.hasBudget` como `report.budget?.general?.limitAmount != null`, pero `report.budget` (reusa el shape de `getReport`) no tiene una propiedad `.general` — `limitAmount` esta directo en la raiz (`report.budget.limitAmount`). Esa expresion siempre da `undefined != null` -> `false`, aunque el usuario tenga un presupuesto real configurado con limite.

No bloqueante: el frontend ya lo esquiva derivando el flag localmente (`data.budget && data.budget.limitAmount !== null`, mismo patron que ya usa Reportes), asi que la card de presupuesto del dashboard funciona bien. Pero el campo `emptyState.hasBudget` en si mismo esta mal y cualquier otro consumidor futuro de este endpoint que confie en el va a fallar igual.

Arreglo sugerido: en `dashboard.service.js`, cambiar `report.budget?.general?.limitAmount != null` por `report.budget?.limitAmount != null`.

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
