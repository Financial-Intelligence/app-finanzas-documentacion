# Pendientes y contrato de integración del Frontend

## Propósito de este documento

Este archivo es la guía que debe leer la IA o la persona que continúe el frontend. Su objetivo es conectar las pantallas existentes con el backend real sin inventar rutas, campos ni reglas.

La referencia principal es el backend ubicado en:

```text
D:\Programacion\PROYECTOS FREELANCE\APP_FINANZAS\Backend\finanzas-app-backend
```

La documentación interactiva de la API se puede consultar con el backend encendido en:

```text
http://localhost:3000/api/swagger
```

El contrato JSON está disponible en:

```text
http://localhost:3000/api/openapi.json
```

## Estado actual de la integración

El frontend ya tiene una base de conexión:

- Next.js.
- Axios en `lib/api/client.ts`.
- `NEXT_PUBLIC_API_URL` en `.env.example`.
- Envío automático del token Bearer.
- Redirección al login cuando el backend responde `401`.
- React Query para refrescar datos.
- Pantallas visuales de Cuentas, Movimientos y Categorías.

La tarea pendiente es reemplazar los datos de ejemplo de `lib/data.ts` por consultas reales al backend y conectar las acciones de los formularios.

## 1. Configuración para ejecutar frontend y backend

### Backend local

En el backend:

```powershell
pnpm run dev
```

El backend debe estar disponible en:

```text
http://localhost:3000
```

### Frontend local

En el frontend:

```powershell
pnpm run dev
```

El frontend funciona en:

```text
http://localhost:3001
```

El archivo `.env.local` del frontend debe contener:

```env
NEXT_PUBLIC_API_URL=http://localhost:3000/api
```

No escribir la URL del backend directamente dentro de los componentes. Siempre usar `env.apiUrl` y `apiClient`.

### Preparación para subirlo

Cuando el backend esté publicado, cambiar solamente la variable de entorno:

```env
NEXT_PUBLIC_API_URL=https://dominio-del-backend/api
```

Después de cambiar una variable `NEXT_PUBLIC_*`, se debe volver a generar el build del frontend:

```powershell
pnpm run build
```

El backend debe permitir en `CORS_ORIGINS` el dominio exacto del frontend publicado. Ejemplo:

```env
CORS_ORIGINS=https://dominio-del-frontend.com
```

No colocar secretos del backend dentro del frontend.

## 2. Reglas generales de comunicación

### Token

Después de `POST /api/auth/login`, guardar `response.data.token` en el almacenamiento de autenticación que ya usa el frontend.

El `apiClient` debe enviar automáticamente:

```text
Authorization: Bearer TOKEN
```

No enviar el token como query parameter ni dentro del body.

### Respuestas y errores

El backend responde normalmente con:

```json
{
  "message": "Texto descriptivo"
}
```

En errores de validación, mostrar el `message` recibido en una notificación clara.

Reglas mínimas:

- `400`: mostrar que los datos del formulario son inválidos.
- `401`: borrar sesión y enviar al login.
- `404`: mostrar que el registro ya no existe.
- `409`: mostrar la regla que impide la operación, por ejemplo una categoría usada por movimientos.
- `429`: indicar que se hicieron demasiados intentos y esperar antes de volver a intentar.
- `500`: mostrar un mensaje general y permitir reintentar; no mostrar el stack trace.

### Refresco de datos

Después de crear, editar, confirmar, cancelar o eliminar, invalidar y volver a consultar las queries relacionadas.

Ejemplos:

- Un movimiento confirmado debe refrescar movimientos, cuentas, resumen de cuentas y categorías.
- Una categoría editada debe refrescar categorías y los selectores de los formularios de movimientos.
- Una cuenta editada debe refrescar listado, resumen y detalle de cuenta.

El backend actual es REST y no tiene WebSocket. “Tiempo real” significa que la interfaz debe actualizarse inmediatamente después de una operación exitosa. No prometer actualizaciones automáticas desde otro navegador hasta que exista un canal en tiempo real.

## 3. Autenticación y perfil

### Rutas

```text
POST   /api/auth/register
POST   /api/auth/login
GET    /api/auth/me
POST   /api/auth/logout
GET    /api/users/me
PATCH  /api/users/me
DELETE /api/users/me
```

### Login

Body:

```json
{
  "email": "usuario@example.com",
  "password": "Segura123"
}
```

Guardar `token` y `user` de la respuesta. Si el login falla, no entrar al layout privado.

### Registro

Body:

```json
{
  "name": "Luis",
  "email": "usuario@example.com",
  "password": "Segura123"
}
```

La contraseña debe tener entre 8 y 72 caracteres.

### Logout

Ejecutar `POST /api/auth/logout` y después borrar el token localmente. El JWT actual es stateless: cerrar sesión en el frontend es obligatorio.

### Perfil

`GET /api/users/me` obtiene el perfil completo. `PATCH /api/users/me` acepta solamente campos enviados:

```json
{
  "name": "Luis Angel",
  "phone": "+51 999 888 777",
  "timezone": "America/Lima",
  "avatar": "https://example.com/avatar.png"
}
```

La zona horaria del proyecto debe ser `America/Lima`.

## 4. Cuentas

### Rutas disponibles

```text
GET    /api/accounts
GET    /api/accounts/summary
POST   /api/accounts
GET    /api/accounts/:id
PATCH  /api/accounts/:id
DELETE /api/accounts/:id
PATCH  /api/accounts/:id/status
GET    /api/accounts/:id/movements
```

### Crear cuenta

```json
{
  "name": "BCP Sueldo",
  "type": "BANK_ACCOUNT",
  "institution": "BCP",
  "currency": "PEN",
  "initialBalance": 1000,
  "isPrimary": true
}
```

Tipos válidos:

```text
BANK_ACCOUNT
SAVINGS
CASH
DIGITAL_WALLET
CREDIT_CARD
INVESTMENT
OTHER
```

Si se crea una cuenta con `isPrimary: true`, la cuenta principal anterior pasa a `false`. Solo debe existir una cuenta principal por usuario.

Una cuenta `CREDIT_CARD` necesita `creditLimit`. No debe sumarse al saldo total; sí debe aparecer en crédito disponible.

### Campo antiguo que no debe usar el frontend

No enviar ni mostrar `expectedMonthlyAmount`. Es un campo antiguo que todavía existe en la base de datos con valor predeterminado `0`, pero ya no pertenece al formulario ni al flujo actual. Se eliminará en una limpieza posterior del backend.

### Listado y detalle

El detalle `GET /api/accounts/:id` devuelve también:

```json
{
  "account": {
    "id": 14,
    "movementsCount": 3
  }
}
```

Regla para el botón eliminar:

- Mostrar eliminar solamente cuando `movementsCount === 0` y la cuenta no sea principal.
- Si `movementsCount > 0`, mostrar inactivar.
- No permitir eliminar una cuenta principal.

### Resumen

Usar `GET /api/accounts/summary` para saldo total y crédito disponible.

Todavía no existe:

```text
GET /api/accounts/:id/summary
```

No llamar esa ruta desde el frontend porque responde `404`. El panel esperado versus real de la cuenta queda pendiente del módulo de planificación mensual y de ingresos fijos.

### Historial de una cuenta

```text
GET /api/accounts/:id/movements?period=YYYY-MM
```

Usar esta ruta en la pestaña de historial del detalle de cuenta.

## 5. Selector anual de meses

El backend trabaja con periodos en formato:

```text
YYYY-MM
```

Ejemplos:

```text
2026-07
2026-08
```

### Diseño obligatorio

El selector debe funcionar como un calendario de meses, no como una lista vertical larga.

Al presionar el periodo de la barra superior debe abrirse un cuadro con:

```text
              <      2026      >

Enero       Febrero      Marzo        Abril
Mayo        Junio        Julio        Agosto
Septiembre  Octubre      Noviembre    Diciembre
```

En escritorio debe usar una cuadrícula de 4 columnas por 3 filas. En pantallas pequeñas puede adaptarse a 2 columnas por 6 filas para mantener botones cómodos.

El encabezado debe incluir:

- Flecha izquierda para ver el año anterior.
- Año visible en el centro.
- Flecha derecha para ver el año siguiente.
- Acción “Ir al mes actual” cuando se esté viendo otro año o mes.

Estados visuales:

- El mes seleccionado debe quedar marcado claramente.
- El mes actual de Perú debe tener un indicador diferente, incluso si no está seleccionado.
- Los meses deben ser botones accesibles mediante teclado.
- El cuadro debe cerrarse después de seleccionar un mes.

### Regla para no crear datos innecesarios

Mostrar los 12 meses no significa consultar ni crear los 12 periodos.

No hacer 12 llamadas a `/categories` o `/movements` para llenar el calendario. Eso prepararía meses que el usuario todavía no abrió.

Cambiar solamente el año visible tampoco debe llamar al backend. La consulta ocurre únicamente cuando el usuario selecciona un mes.

Ejemplo:

1. El usuario abre el calendario y cambia el encabezado de 2026 a 2027.
2. El frontend solo muestra los nombres de enero a diciembre de 2027.
3. Todavía no consulta ni crea información de 2027.
4. El usuario selecciona marzo.
5. El frontend construye `selectedPeriod = "2027-03"`.
6. Recién entonces consulta las rutas que dependen del periodo.

### Estado compartido del periodo

Mantener un único `selectedPeriod` compartido por las pantallas financieras. Debe iniciar con el mes actual calculado en la zona horaria `America/Lima`.

Cuando se selecciona otro mes:

```text
selectedPeriod = `${selectedYear}-${monthNumber}`
```

El número del mes siempre debe llevar dos dígitos:

```text
Enero 2027  -> 2027-01
Agosto 2027 -> 2027-08
```

El periodo debe formar parte de las claves de React Query:

```text
["categories", selectedPeriod]
["movements", selectedPeriod, filters, page]
["account-movements", accountId, selectedPeriod, filters, page]
```

Así, cambiar de julio a agosto provoca una consulta nueva y volver a julio permite reutilizar su caché sin mezclar datos.

### Consultas al seleccionar un mes

En Categorías:

```text
GET /api/categories?period=2027-03
```

En Movimientos:

```text
GET /api/movements?period=2027-03&page=1&pageSize=20
```

En el historial de una cuenta:

```text
GET /api/accounts/:id/movements?period=2027-03&page=1&pageSize=20
```

El listado general de cuentas y `GET /api/accounts/summary` no reciben periodo porque representan saldos actuales, no snapshots mensuales.

### Reglas funcionales

- La fecha visible debe respetar la hora de Perú.
- Al cambiar de mes, reiniciar la página de movimientos a `1`.
- Mantener los filtros si continúan siendo válidos; limpiar categoría o subcategoría si no existe en el nuevo periodo.
- Se puede preparar un mes futuro, por ejemplo agosto mientras se está en julio.
- La primera consulta de categorías de un mes prepara ese mes copiando el último mes anterior disponible.
- No modificar datos de julio al editar agosto.
- En las rutas de movimientos, la fecha `date` determina el periodo; no enviar un campo `period` al crear.
- Si el formulario crea un movimiento para un mes distinto al seleccionado, pedir confirmación o actualizar la vista al periodo de la fecha guardada.

## 6. Categorías mensuales

### Rutas

```text
GET    /api/categories?period=YYYY-MM&type=INCOME|EXPENSE
POST   /api/categories
GET    /api/categories/:id
PATCH  /api/categories/:id
DELETE /api/categories/:id
```

### Comportamiento de la pantalla

La pantalla debe conservar el diseño mostrado:

- Título “Categorías”.
- Pestañas “Egresos” e “Ingresos”.
- Tarjetas con color, icono, nombre y subcategorías.
- Total confirmado del mes en cada tarjeta.
- Botón “Nueva categoría”.
- Botón “Importar” visible solamente si se deja como función futura; no conectarlo todavía.
- Acción de editar en cada tarjeta o mediante la apertura del formulario con datos precargados.
- Acción de eliminar en cada tarjeta.

### Listar

Siempre incluir el periodo seleccionado:

```text
GET /api/categories?period=2026-07
```

La primera consulta de un periodo nuevo hace que el backend copie las categorías del último mes disponible. Después, cada mes queda independiente.

### Crear egreso

```json
{
  "period": "2026-07",
  "type": "EXPENSE",
  "name": "Comida",
  "color": "#F59E0B",
  "icon": "utensils",
  "subcategories": [
    "Mercado",
    "Restaurante"
  ]
}
```

### Crear ingreso

```json
{
  "period": "2026-07",
  "type": "INCOME",
  "name": "Sueldo",
  "color": "#10B981",
  "icon": "briefcase",
  "subcategories": []
}
```

Las categorías de ingreso no aceptan subcategorías.

### Editar

Primero consultar `GET /api/categories/:id` para autorrellenar el formulario. Al enviar las subcategorías:

- Mantener el `id` de las existentes.
- Enviar sin `id` las nuevas.
- No enviar las que se quieran quitar.

```json
{
  "name": "Alimentación",
  "color": "#F97316",
  "icon": "utensils",
  "subcategories": [
    { "id": 3, "name": "Mercado" },
    { "name": "Delivery" }
  ]
}
```

### Eliminar

Si la categoría tiene movimientos visibles asociados, el backend responde `409`. Mostrar una alerta con esta idea:

```text
Esta categoría tiene movimientos asociados. Reasigna o elimina esos movimientos antes de eliminarla.
```

No borrar la categoría localmente si el backend rechazó la operación.

### Totales de tarjetas

`monthTotal` y `confirmedMovementsCount` consideran solamente movimientos `CONFIRMED` y su `actualAmount`. Los movimientos pendientes o cancelados no deben sumarse.

## 7. Pantalla de Movimientos

La pantalla debe conservar el diseño de las imágenes compartidas:

- Tres tarjetas superiores: Total ingresos, Total egresos y Transferencias.
- Cantidad de movimientos por tarjeta.
- Selector de mes.
- Buscador por descripción.
- Filtro por tipo.
- Filtro por cuenta.
- Filtro por categoría.
- Filtro por subcategoría cuando corresponda.
- Filtro por estado.
- Botón “Limpiar”.
- Cantidad de resultados.
- Tabla con fecha, tipo, descripción, cuenta, categoría/subcategoría, estado, monto y acciones.
- Montos de egreso en rojo.
- Montos de ingreso en verde.
- Transferencias en azul.
- Botones “Transferir”, “Ingreso” y “Nuevo gasto”.

### Rutas

```text
GET    /api/movements
POST   /api/movements/income
POST   /api/movements/expense
POST   /api/movements/transfer
GET    /api/movements/:id
PATCH  /api/movements/:id
POST   /api/movements/:id/confirm
POST   /api/movements/:id/cancel
DELETE /api/movements/:id
```

### Consulta de historial

Ejemplo:

```text
GET /api/movements?period=2026-07&page=1&pageSize=20
```

Filtros válidos:

```text
period
type=INCOME|EXPENSE|TRANSFER
accountId
categoryId
subcategoryId
status=PENDING|CONFIRMED|CANCELLED
search
page
pageSize
```

La respuesta contiene:

```json
{
  "data": [],
  "meta": {
    "total": 0,
    "page": 1,
    "pageSize": 20,
    "totalPages": 0
  },
  "totals": {
    "income": 0,
    "expense": 0,
    "transfersIn": 0,
    "transfersOut": 0,
    "incomeCount": 0,
    "expenseCount": 0,
    "transferCount": 0
  }
}
```

Las tarjetas superiores deben usar `totals`, no sumar solamente las filas filtradas de `data`. Los totales son del periodo completo y solamente consideran confirmados.

### Crear ingreso

```json
{
  "amount": 3500,
  "accountId": 14,
  "categoryId": 3,
  "date": "2026-07-15",
  "description": "Sueldo esperado de julio"
}
```

El backend ignora cualquier `status` enviado para ingresos y los crea como `PENDING`.

### Crear gasto

```json
{
  "amount": 500,
  "accountId": 14,
  "categoryId": 4,
  "subcategoryId": 3,
  "date": "2026-07-16",
  "description": "Mercado mensual esperado"
}
```

`amount` es el monto esperado. Al crear no se debe mostrar como confirmado ni descontar del saldo.

### Confirmar ingreso o gasto

Un gasto creado queda pendiente hasta que el usuario registra lo que realmente pagó. La pantalla de confirmación debe permitir corregir:

- Monto real.
- Fecha.
- Cuenta.
- Categoría.
- Subcategoría si es egreso.
- Descripción.

Ejemplo:

```http
POST /api/movements/15/confirm
```

```json
{
  "actualAmount": 400,
  "date": "2026-07-16",
  "accountId": 14,
  "categoryId": 4,
  "subcategoryId": 3,
  "description": "Mercado realmente pagado"
}
```

La respuesta conserva:

```json
{
  "expectedAmount": 500,
  "actualAmount": 400,
  "status": "CONFIRMED"
}
```

El frontend debe mostrar ambos valores si el usuario necesita revisar la diferencia. Confirmar S/ 400 no cambia el monto esperado de S/ 500 para futuros pagos fijos.

### Transferencia

Formulario:

- Cuenta origen.
- Cuenta destino.
- Monto.
- Fecha.
- Estado.
- Descripción.

Body para transferencia inmediata:

```json
{
  "amount": 100,
  "accountId": 14,
  "toAccountId": 10,
  "date": "2026-07-17",
  "status": "CONFIRMED",
  "description": "Transferencia a ahorros"
}
```

Body para transferencia programada:

```json
{
  "amount": 100,
  "accountId": 14,
  "toAccountId": 10,
  "date": "2026-07-25",
  "status": "PENDING",
  "description": "Transferencia programada"
}
```

Una transferencia no es ingreso ni egreso. Debe aparecer en azul y solamente modifica saldos cuando está confirmada.

### Cancelar

```text
POST /api/movements/:id/cancel
```

Cancelar cambia el estado a `CANCELLED`, conserva el registro visible y no afecta saldos ni totales.

### Eliminar o archivar

```text
DELETE /api/movements/:id
```

La eliminación es lógica. El movimiento desaparece del historial normal, no afecta cálculos y queda archivado internamente. Si era confirmado, el backend revierte su efecto sobre la cuenta.

En la tabla:

- Movimiento pendiente: mostrar “Confirmar” y “Eliminar”.
- Movimiento confirmado: mostrar “Eliminar” y, si el diseño lo permite, editar.
- Movimiento cancelado: mostrar estado cancelado; no mostrar confirmar.

## 8. Recurrentes, ingresos fijos y casuales

Todavía no existe una API para ingresos fijos, ingresos casuales ni pagos recurrentes.

Por eso, por ahora:

- No enviar `isRecurring` al endpoint de ingresos o gastos.
- No enviar `sourceType` al crear manualmente.
- No inventar `/api/recurring` ni `/api/fixed-income`.
- El interruptor “¿Es recurrente?” de las imágenes debe quedar oculto, deshabilitado o marcado como función futura.
- No modificar el monto esperado futuro al confirmar un movimiento con otro monto real.

Cuando ese módulo exista, generará ocurrencias y movimientos pendientes usando el mismo flujo esperado versus real.

Los ingresos casuales serán entradas ocasionales y los ingresos fijos serán entradas que pueden repetirse. Ninguno debe duplicar movimientos en la tabla central.

## 9. Importación

El botón “Importar” de Categorías queda reservado para una fase futura. No conectarlo ni enviar archivos todavía. Si se muestra, debe indicar que la función está pendiente.

## 10. Estados visuales obligatorios

Cada pantalla conectada al backend debe contemplar:

- Cargando datos.
- Error con botón “Reintentar”.
- Lista vacía con una explicación y botón de creación.
- Guardado en progreso para evitar doble clic.
- Confirmación visual después de guardar.
- Modal de confirmación antes de eliminar.
- Mensaje específico cuando el backend responde `409`.

No actualizar la interfaz como si la operación hubiera funcionado antes de recibir una respuesta exitosa.

## 11. Comprobación de integración

La IA o persona que implemente el frontend debe comprobar este recorrido:

1. Registrar o iniciar sesión.
2. Confirmar que `Authorization` se envía automáticamente.
3. Crear dos cuentas.
4. Marcar una como principal y comprobar que la anterior queda en `false`.
5. Crear una categoría de ingreso y una de egreso con subcategorías.
6. Abrir el calendario 4×3, cambiar de año y comprobar que eso no realiza consultas mensuales.
7. Seleccionar un mes futuro y comprobar que recién entonces se consultan categorías y movimientos.
8. Comprobar que las categorías se copian al entrar por primera vez al mes.
9. Editar una categoría futura y comprobar que el mes anterior no cambia.
10. Crear un ingreso pendiente.
11. Crear un gasto pendiente.
12. Confirmar el gasto con un monto real menor al esperado.
13. Comprobar que el saldo usa el monto real.
14. Comprobar que la categoría usa solamente el total confirmado.
15. Crear una transferencia confirmada.
16. Crear una transferencia pendiente y luego confirmarla.
17. Probar filtros, búsqueda y paginación.
18. Cancelar un movimiento y comprobar que no aparece en totales.
19. Eliminar un movimiento y comprobar que se archiva.
20. Intentar eliminar una categoría usada y mostrar el error `409`.
21. Intentar eliminar una cuenta con historial y ofrecer inactivarla.
22. Ejecutar `pnpm run build` en el frontend.

## 12. Criterios para considerar lista la integración

La integración se considera lista cuando:

- No quedan datos de `lib/data.ts` en las pantallas conectadas.
- Todas las consultas usan `apiClient`.
- Todas las rutas usan `NEXT_PUBLIC_API_URL`.
- El token se envía sin duplicar lógica en cada componente.
- Los formularios respetan exactamente los nombres del backend.
- Ingresos y egresos nacen pendientes.
- La confirmación usa `actualAmount`.
- Los totales usan solamente confirmados.
- Los periodos usan `YYYY-MM` y hora de Perú.
- El selector muestra un año con 12 meses en cuadrícula 4×3.
- Cambiar el año visible no crea ni consulta periodos.
- Solo seleccionar un mes dispara las consultas correspondientes.
- Las categorías se mantienen independientes por mes.
- Los errores `401`, `404` y `409` se muestran correctamente.
- Las acciones actualizan las queries relacionadas.
- El build de producción termina sin errores.
- El frontend funciona con una URL de backend configurada por entorno.
