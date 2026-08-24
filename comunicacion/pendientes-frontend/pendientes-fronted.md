# Prompts pendientes para el frontend

Este archivo contiene unicamente trabajo que debe realizarse en el frontend.
Cada punto esta redactado como un prompt independiente para entregarlo a una IA
o a un desarrollador. No modificar el backend desde el proyecto frontend.

## 1. Edicion de categorias por alcance

Implementa completamente en el frontend la edicion de categorias por alcance,
manteniendo el diseño y las convenciones que ya utiliza el modulo de
Categorias.

### Objetivo visual y funcional

Al pulsar Editar en una tarjeta, abre el formulario autorrellenado con el
nombre, color, icono y subcategorias de la categoria del mes abierto. El usuario
debe elegir donde se aplicaran los cambios:

1. `Solo este mes`
   - Valor para la API: `CURRENT`.
   - Disponible para cualquier categoria.
   - Modifica solamente el mes abierto.

2. `Meses personalizados`
   - Valor para la API: `CUSTOM`.
   - Disponible para cualquier categoria.
   - Mostrar el mismo selector anual de meses usado al crear categorias.
   - El mes abierto siempre debe estar seleccionado y no se puede desmarcar.
   - Enviar los meses seleccionados mediante `targetPeriods` con formato
     `YYYY-MM`.

3. `Desde este mes en adelante`
   - Valor para la API: `FROM_CURRENT`.
   - Mostrarlo solamente cuando `category.isPermanent === true`.
   - Explicar: "Actualiza este mes, los meses futuros preparados y los meses
     nuevos. Los meses anteriores no cambiaran".
   - No enviar `targetPeriods`.

Una categoria no permanente no debe mostrar ni intentar usar
`FROM_CURRENT`.

### Datos que debe enviar

El cuerpo de la vista previa y del guardado definitivo tiene la misma forma:

```json
{
  "scope": "CUSTOM",
  "targetPeriods": ["2026-08", "2026-09"],
  "name": "Alimentacion",
  "color": "#F97316",
  "icon": "utensils",
  "subcategories": [
    { "id": 3, "name": "Mercado" },
    { "name": "Delivery" }
  ]
}
```

Reglas del cuerpo:

- enviar `targetPeriods` exclusivamente con `scope=CUSTOM`;
- conservar el `id` de cada subcategoria existente;
- una subcategoria nueva se envia sin `id`;
- omitir una subcategoria existente significa que el usuario intenta
  eliminarla;
- no enviar propiedades adicionales;
- las categorias de ingreso no usan subcategorias.

### Vista previa obligatoria

Para `CUSTOM` y `FROM_CURRENT`, antes de guardar llama a:

```http
POST /api/categories/:id/preview-update
Authorization: Bearer <token>
```

No agregues el token manualmente si el interceptor actual ya lo hace.

La respuesta tiene esta forma:

```json
{
  "categoryId": 13,
  "sourcePeriod": "2026-08",
  "scope": "FROM_CURRENT",
  "targetPeriods": ["2026-08", "2026-09", "2026-12"],
  "affectedPeriods": ["2026-08", "2026-09", "2026-12"],
  "canApply": false,
  "conflicts": [
    {
      "period": "2026-09",
      "categoryId": 18,
      "code": "SUBCATEGORY_IN_USE",
      "movementsCount": 2,
      "message": "No se pueden quitar subcategorias usadas por 2 movimiento(s)"
    }
  ]
}
```

Si `canApply=true`, muestra una confirmacion con los meses de
`affectedPeriods` y habilita el boton `Confirmar cambios`.

Si `canApply=false`:

- lista todos los conflictos agrupados por mes;
- muestra el `message` entregado por el backend;
- no permitas confirmar;
- conserva el formulario para que el usuario pueda corregirlo;
- no ofrezcas omitir meses: la operacion debe ejecutarse completa o no
  ejecutarse.

Codigos que debe reconocer la interfaz:

| Codigo | Mensaje que debe comunicar |
| --- | --- |
| `CATEGORY_NOT_FOUND` | La categoria correspondiente no existe en ese mes. |
| `SUBCATEGORIES_DIVERGED` | Ese mes fue editado independientemente y debe revisarse por separado. |
| `CATEGORY_NAME_IN_USE` | Otra categoria del mes ya usa el nuevo nombre. |
| `SUBCATEGORY_IN_USE` | Hay movimientos que usan una subcategoria que se intenta quitar. |
| `PERMANENT_NAME_IN_USE` | Otra categoria permanente ya usa el nuevo nombre. |

### Guardado definitivo

Cuando la vista previa permita continuar, envia el mismo cuerpo a:

```http
PATCH /api/categories/:id
Authorization: Bearer <token>
```

Para `CURRENT` se puede llamar directamente al `PATCH`. Si responde `409`,
conserva el formulario y muestra el mensaje del backend.

Respuesta exitosa:

```json
{
  "message": "Categorias actualizadas correctamente",
  "category": {},
  "categories": [],
  "scope": "CUSTOM",
  "updatedPeriods": ["2026-08", "2026-09"]
}
```

Despues del exito:

- cerrar el formulario y la confirmacion;
- mostrar el mensaje de exito;
- invalidar todas las consultas/cache de categorias, no solamente el mes
  visible;
- volver a consultar las categorias del mes abierto;
- no modificar manualmente la cache suponiendo que todos los meses quedaron
  iguales.

### Compatibilidad con lo que ya existe

- conservar sin cambios el flujo de creacion `CURRENT`, `CUSTOM` y
  `PERMANENT`;
- conservar la eliminacion como una accion exclusiva del mes abierto;
- reutilizar el selector anual, componentes, estilos, cliente HTTP y sistema de
  notificaciones actuales;
- no crear una pantalla para editar o desactivar directamente plantillas
  permanentes;
- no modificar el backend.

### Pruebas requeridas

Agregar o actualizar pruebas para comprobar:

- autorrellenado del formulario de edicion;
- `CURRENT` envia el `PATCH` sin `targetPeriods`;
- `CUSTOM` obliga a usar la vista previa y envia los meses elegidos;
- `FROM_CURRENT` solo aparece para categorias permanentes;
- una vista previa con conflictos bloquea la confirmacion;
- una vista previa sin conflictos permite confirmar;
- los errores `400` y `409` mantienen el formulario abierto;
- despues del exito se invalidan todas las consultas de categorias.

No se considera terminada la tarea mientras estas pruebas y el flujo visual no
funcionen con el backend real.

## 2. Omitir un modulo visual independiente de Transferencias

No crear una pagina, ruta, opcion de menu, seccion de navegacion ni modulo
visual independiente llamado `Transferencias`.

Las transferencias ya forman parte del modulo de Movimientos y deben mantenerse
integradas alli. Omitir el modulo independiente no significa eliminar la
funcionalidad.

El frontend debe conservar dentro de Movimientos:

- el boton `Transferir`;
- el formulario de nueva transferencia;
- la seleccion de cuenta origen y cuenta destino;
- el monto, fecha, estado y descripcion;
- las transferencias como filas de la tabla de movimientos;
- el filtro `TRANSFER`;
- la tarjeta superior con el total y cantidad de transferencias;
- las acciones permitidas para confirmar, editar, cancelar o eliminar;
- la visualizacion de una transferencia en el historial de la cuenta origen y
  de la cuenta destino.

Usar exclusivamente los endpoints existentes del modulo de Movimientos:

```http
POST /api/movements/transfer
GET /api/movements?type=TRANSFER&period=YYYY-MM
GET /api/movements/:id
PATCH /api/movements/:id
POST /api/movements/:id/confirm
POST /api/movements/:id/cancel
DELETE /api/movements/:id
```

No crear endpoints nuevos ni duplicar estados, hooks, servicios o tipos bajo un
feature separado de transferencias. Reutilizar los componentes, consultas,
cache y manejo de errores de Movimientos.

La navegacion esperada debe mantenerse asi:

```text
Movimientos
├── Ingresos
├── Egresos
└── Transferencias
```

No debe existir una opcion adicional `Transferencias` en el menu principal.

### Pruebas requeridas

Comprobar que:

- no existe pagina ni enlace independiente de Transferencias;
- el boton `Transferir` abre el formulario dentro de Movimientos;
- una transferencia creada aparece en la tabla de Movimientos;
- el filtro de transferencias funciona;
- el historial de ambas cuentas muestra la transferencia;
- no se elimina ni se rompe ninguna funcionalidad de transferencias existente.

## 3. Implementar el modulo visual de Deudas

Implementa el modulo de Deudas consumiendo el backend real. Conserva la linea
visual del prototipo: tarjetas de resumen, pestanas `Activas`, `Pagadas` y
`Todas`, tarjetas de deuda, formulario lateral de nueva deuda y formulario
lateral para registrar pagos. No uses datos simulados ni dupliques calculos que
ya entrega la API.

Todas las rutas requieren JWT mediante `Authorization: Bearer <token>`. Reutiliza
el cliente HTTP o interceptor existente; no escribas el token manualmente en
cada componente.

### Listado y calendario

Al abrir Deudas o cambiar el mes global, consulta:

```http
GET /api/debts?status=active&period=YYYY-MM
```

Mapeo de pestanas: `Activas=active`, `Pagadas=paid`, `Todas=all`. Las deudas son
globales: cambiar de mes no crea ni copia deudas. `period` solo cambia
`paidThisPeriod`, es decir, cuanto se pago en el mes seleccionado.

La respuesta contiene `period`, `debts`, `counts` y `summary`. Usa:

- `counts.active`, `counts.paid` y `counts.all` en las pestanas;
- `summary.totalOwed` para Total que debes;
- `summary.totalPaid` y `summary.paidPercentage` para Ya pagado;
- `summary.monthlyInstallment` y `summary.paidThisPeriod` para Cuota mensual;
- `summary.nextDueDebt` para Proximo vencimiento.

`summary.byCurrency` contiene un resumen por moneda. Con una sola moneda los
campos superiores traen numeros. Con varias monedas llegan como `null`: no sumes
monedas distintas y muestra cada elemento de `byCurrency` por separado.

Cada deuda ya trae `paidAmount`, `paidPercentage`, `paymentsCount`, `isOverdue`,
`paymentAccount` y `paidThisPeriod`. Usa esos valores, sin recalcularlos, para
mostrar acreedor, tipo, cuenta, saldo pendiente, monto original, barra de
progreso, cuota, vencimiento, cantidad de pagos y las etiquetas `Vencida` o
`Pagada`. Los botones `Registrar pago` y `Pagar cuota` aparecen solo en activas.

### Formulario Nueva deuda

Usa `POST /api/debts`. Campos: acreedor, tipo, monto total, cuota estimada, tasa
mensual informativa, cuenta de pago, proximo vencimiento opcional, descripcion
opcional e interruptor `¿Recibiste este dinero ahora?`, desactivado por defecto.

Tipos y etiquetas:

| API | Etiqueta |
| --- | --- |
| `CREDIT_CARD` | Tarjeta |
| `BANK_LOAN` | Prestamo bancario |
| `PERSONAL_DEBT` | Deuda personal |
| `INSTALLMENT_PURCHASE` | Compra en cuotas |
| `MORTGAGE` | Hipoteca |
| `OTHER` | Otra |

Deuda anterior, sin cambiar una cuenta:

```json
{
  "creditor": "Prestamo BCP",
  "type": "BANK_LOAN",
  "originalAmount": 6000,
  "installment": 540,
  "monthlyInterestRate": 1.9,
  "paymentAccountId": 15,
  "nextDueDate": "2026-09-05",
  "description": "Prestamo bancario",
  "createIncome": false
}
```

Si se activa `¿Recibiste este dinero ahora?`, muestra cuenta receptora y fecha,
y envia tambien `createIncome=true`, `receiveAccountId` y `receivedOn`:

```json
{
  "creditor": "Prestamo familiar",
  "type": "PERSONAL_DEBT",
  "originalAmount": 800,
  "paymentAccountId": 15,
  "createIncome": true,
  "receiveAccountId": 15,
  "receivedOn": "2026-08-24"
}
```

El backend crea un ingreso confirmado y aumenta la cuenta receptora. Las cuentas
de recepcion y pago deben tener la misma moneda. No muestres cuentas inactivas
ni tarjetas de credito en esos selectores.

### Editar una deuda

Autorrellena el formulario y envia solo los cambios a `PATCH /api/debts/:id`.
Campos permitidos: `creditor`, `type`, `originalAmount`, `installment`,
`monthlyInterestRate`, `paymentAccountId`, `nextDueDate` y `description`.

Bloquea `originalAmount` cuando `paymentsCount > 0` o `receiveMovementId` no sea
`null`; el backend tambien lo protege con `409`. Los demas campos siguen
editables. Pagar no modifica la cuota o tasa y no avanza el vencimiento.

### Registrar un pago

El boton general primero permite elegir una deuda activa. El boton de una tarjeta
abre el mismo formulario con esa deuda seleccionada. Para obtener el historial
usa `GET /api/debts/:id`.

Muestra saldo pendiente, monto original, progreso, cuota sugerida, cuenta y
fecha. `Cuota` coloca `installment` sin superar `balance`; `Saldo total` coloca
`balance`. Envia:

```http
POST /api/debts/:id/payments
```

```json
{
  "amount": 400,
  "accountId": 15,
  "date": "2026-08-24"
}
```

`accountId` puede omitirse para usar la cuenta predeterminada. El pago queda
confirmado inmediatamente. El backend bloquea monto superior al saldo, saldo
insuficiente, moneda diferente, cuenta inactiva o tarjeta de credito.

### Corregir pagos y eliminar deudas

En el historial agrega `Revertir pago`, con confirmacion, usando:

```http
DELETE /api/debts/:id/payments/:paymentId
```

Esto devuelve el dinero a la cuenta, restaura el saldo pendiente y vuelve la
deuda a `ACTIVE` si estaba pagada.

Permite eliminar solo cuando `paymentsCount === 0` mediante
`DELETE /api/debts/:id`. La eliminacion es logica. Si hubo ingreso inicial,
advierte que ese dinero tambien se descontara de la cuenta. Si llega `409`,
conserva la pantalla y muestra el `message` del backend.

### Integracion con Movimientos y cache

Los ingresos y pagos de Deudas aparecen en Movimientos con `sourceType=DEBT` y
`sourceId=<debtId>`. Deben verse, pero no deben ofrecer Editar, Cancelar ni
Eliminar directamente; las correcciones se hacen desde Deudas.

Despues de crear, editar, pagar, revertir o eliminar, invalida y vuelve a
consultar deudas, detalle de deuda, cuentas, resumen de cuentas, movimientos del
periodo afectado e historial de la cuenta afectada. Usa fechas `YYYY-MM-DD` sin
conversiones que cambien el dia y envia montos como numeros.

Maneja `400`, `401`, `404` y `409` usando `message`. Ante `401`, aplica el flujo
global para borrar sesion y volver al login. No cierres formularios con error.

### Pruebas requeridas

Comprobar:

- pestanas, contadores y periodo enviado;
- que las deudas no se copian al cambiar de mes;
- creacion con y sin ingreso inicial;
- campos obligatorios al activar el ingreso;
- autorrellenado y bloqueo del monto original con historial;
- pago manual, por cuota y por saldo total;
- invalidacion de deudas, cuentas y movimientos despues de pagar;
- bloqueo de acciones directas sobre movimientos `DEBT`;
- reversion de pago y eliminacion con confirmacion;
- errores `400`, `401`, `404`, `409` y resumen de una o varias monedas.

No se considera terminado hasta probar todo contra el backend real, sin mocks y
sin crear endpoints diferentes.

## 4. Implementar el modulo visual de Prestamos por cobrar

Implementa Prestamos consumiendo exclusivamente el backend real. Mantiene el
diseño del prototipo: tarjetas superiores, pestanas `Activos`, `Cobrados` y
`Todos`, tarjetas con progreso, formulario lateral `Nuevo prestamo` y formulario
lateral `Registrar cobro`. No uses mocks ni guardes prestamos en `localStorage`.

Todas las rutas requieren JWT. Reutiliza el interceptor actual para enviar
`Authorization: Bearer <token>`.

### Listado y calendario

Al abrir Prestamos o cambiar el mes global, consulta:

```http
GET /api/loans?status=active&period=YYYY-MM
```

Mapeo de pestanas:

| Pestana | Query `status` |
| --- | --- |
| Activos | `active` |
| Cobrados | `collected` |
| Todos | `all` |

Los prestamos son globales. Cambiar de mes no crea ni copia registros. El
`period` solo modifica `collectedThisPeriod`, que representa lo cobrado dentro
del mes seleccionado.

Usa la respuesta de la API sin recalcular sus valores:

- `counts.active`, `counts.collected` y `counts.all` para las pestanas;
- `summary.totalOutstanding` para Por cobrar;
- `summary.totalCollected` y `summary.collectedPercentage` para Ya cobrado;
- `summary.totalLent` y `summary.collectedThisPeriod` para Total prestado;
- `summary.nextDueLoan` para Proximo a vencer.

`summary.byCurrency` contiene los resumenes separados por moneda. Si hay varias,
los campos superiores llegan como `null`: no sumes soles y dolares; muestra cada
elemento de `byCurrency` por separado.

Cada prestamo incluye `collectedAmount`, `collectedPercentage`,
`collectedThisPeriod`, `collectionsCount`, `isOverdue`, `sourceAccount` y
`returnAccount`. Usa esos datos en la tarjeta para mostrar deudor, concepto,
cuenta de cobro, pendiente, total prestado, progreso, vencimiento, cantidad de
cobros y etiquetas `Vencido` o `Cobrado`.

Los botones `Registrar cobro` y `Cobrar todo` aparecen solo cuando
`status=ACTIVE`.

### Formulario Nuevo prestamo

Usa:

```http
POST /api/loans
```

Campos visuales:

- a quien le prestaste;
- concepto opcional;
- monto prestado;
- fecha del prestamo;
- fecha limite opcional;
- cuenta de origen;
- cuenta predeterminada donde entraran los cobros;
- descripcion opcional;
- interruptor `Descontar de mi cuenta ahora`.

La fecha del prestamo es obligatoria. La cuenta de cobro puede usar inicialmente
la misma cuenta de origen. No muestres cuentas inactivas ni tarjetas de credito.

Ejemplo:

```json
{
  "debtor": "Rivaldo",
  "concept": "Prestamo personal",
  "originalAmount": 300,
  "sourceAccountId": 15,
  "returnAccountId": 10,
  "loanDate": "2026-08-24",
  "dueDate": "2026-09-10",
  "description": null,
  "createExpense": true
}
```

Siempre envia `createExpense`:

- `true`: se crea un egreso confirmado y se descuenta el monto de la cuenta;
- `false`: se guarda un prestamo anterior sin cambiar saldos.

Si `returnAccountId` se omite, el backend usa `sourceAccountId`. Ambas cuentas
deben utilizar la misma moneda. Si llega `409` por saldo insuficiente, conserva
el formulario abierto y muestra `message`.

### Editar un prestamo

Autorrellena el formulario y envia solo los cambios a:

```http
PATCH /api/loans/:id
```

Campos permitidos: `debtor`, `concept`, `originalAmount`, `sourceAccountId`,
`returnAccountId`, `loanDate`, `dueDate` y `description`.

Cuando `collectionsCount > 0` o `expenseMovementId` no sea `null`, bloquea
visualmente `originalAmount`, `sourceAccountId` y `loanDate`. El backend tambien
los protege con `409`. Los demas campos siguen editables. Registrar un cobro no
cambia automaticamente la fecha limite.

### Registrar un cobro

El boton general `Registrar cobro` permite seleccionar un prestamo activo. El
boton de una tarjeta abre el mismo formulario con ese prestamo seleccionado.
Para cargar el historial usa:

```http
GET /api/loans/:id
```

La respuesta incluye `collections`, ordenados del mas reciente al mas antiguo.
El formulario muestra pendiente, total prestado, progreso, cuenta receptora y
fecha. Envia:

```http
POST /api/loans/:id/collections
```

```json
{
  "amount": 100,
  "toAccountId": 10,
  "date": "2026-08-24"
}
```

`toAccountId` puede omitirse para usar la cuenta de cobro predeterminada. El
cobro se confirma inmediatamente, crea un ingreso y reduce lo pendiente. El
backend rechaza montos mayores que `outstanding`, cuentas inactivas, tarjetas o
monedas diferentes.

`Cobrar todo` no debe ejecutar la API inmediatamente. Debe abrir el formulario
de cobro con `amount=outstanding` autorrellenado para que el usuario confirme
cuenta, fecha y monto.

### Revertir cobros y eliminar prestamos

En el historial agrega `Revertir cobro` con confirmacion:

```http
DELETE /api/loans/:id/collections/:collectionId
```

El backend descuenta ese dinero de la cuenta, restaura `outstanding` y devuelve
el prestamo a `ACTIVE`. Puede responder `409` si la cuenta ya no tiene saldo
suficiente; no cierres la pantalla en ese caso.

Permite eliminar solo cuando `collectionsCount === 0`:

```http
DELETE /api/loans/:id
```

La eliminacion es logica. Si `expenseMovementId` no es `null`, muestra en la
confirmacion que el dinero prestado regresara a la cuenta de origen. Con cobros
activos el backend responde `409`.

### Integracion con Movimientos y cache

El egreso inicial y los cobros aparecen en Movimientos con `sourceType=LOAN` y
`sourceId=<loanId>`. Deben verse en la tabla, pero no deben ofrecer Editar,
Cancelar ni Eliminar directamente. Las correcciones se realizan desde
Prestamos.

Despues de crear, editar, cobrar, revertir o eliminar, invalida y vuelve a
consultar:

- listado y detalle de prestamos;
- cuentas y resumen de cuentas;
- movimientos de los periodos afectados;
- historial de las cuentas afectadas.

Usa fechas `YYYY-MM-DD` sin conversiones que cambien el dia. Envia montos como
numeros. Maneja `400`, `401`, `404` y `409` mostrando `message`; ante `401`, usa
el flujo global actual para cerrar la sesion.

### Pruebas requeridas

Comprobar:

- pestanas, contadores, KPIs y periodo enviado;
- que los prestamos no se copian al cambiar de mes;
- creacion con `createExpense=true` y `false`;
- validacion de fecha, cuenta y saldo;
- autorrellenado y bloqueo de campos con historial;
- cobro parcial y cobro total con confirmacion;
- cambio automatico a `COLLECTED` al llegar a cero;
- reversion de un cobro y errores por saldo insuficiente;
- eliminacion con y sin egreso inicial;
- movimientos `LOAN` visibles pero sin acciones manuales;
- invalidacion de prestamos, cuentas y movimientos;
- resumen correcto con una o varias monedas.

No se considera terminado hasta probar el flujo completo contra el backend real
y sin endpoints diferentes de los indicados.
