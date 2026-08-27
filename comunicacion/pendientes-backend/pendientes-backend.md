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
