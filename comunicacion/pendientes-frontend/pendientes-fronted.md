# Pendiente del frontend segun el backend actual

Documento de entrega para continuar la aplicacion web sin adivinar el contrato.
Se comparo con el backend y con el frontend existentes el 2026-08-21.

## 1. Estado real

### Ya conectado en el frontend

- registro y login;
- guardado local de `{ accessToken, user }` bajo la clave `lumen-auth`;
- envio automatico de `Authorization: Bearer <token>` mediante Axios;
- restauracion de sesion con `GET /auth/me`;
- limpieza de sesion y redireccion a login ante un `401`;
- consulta y edicion del perfil;
- listado, resumen, creacion, edicion, activacion, inactivacion y eliminacion de
  cuentas.

### Aun no terminado en el frontend

- la pagina Movimientos es una pantalla vacia;
- la pagina Categorias es una pantalla vacia;
- el historial del detalle de cuenta no consume el backend;
- las metricas del detalle de cuenta muestran valores fijos;
- el detalle llama a `GET /accounts/:id/summary`, ruta que no existe;
- el filtro global conserva un periodo fijo antiguo (`2026-05`);
- el selector actual usa una lista de meses y no el calendario anual acordado;
- el Dashboard usa datos de demostracion.

## 2. Conexion y autenticacion

La direccion del backend se configura en el frontend con:

```env
VITE_API_BASE_URL=http://localhost:3000/api
```

Registro y login devuelven:

```json
{
  "message": "...",
  "token": "jwt...",
  "user": { "id": 2, "name": "Luis", "email": "..." }
}
```

El adaptador actual del frontend convierte `token` a `accessToken`. Despues del
login debe:

1. guardar `accessToken` y `user` en el store de autenticacion;
2. persistirlos con `authStorage`;
3. redirigir a una pantalla protegida;
4. permitir que el interceptor de Axios agregue el encabezado:

```text
Authorization: Bearer <token>
```

Al recargar la pagina, `AuthGuard` lee el almacenamiento y consulta
`GET /auth/me`. Si responde `200`, actualiza el usuario; si responde `401`,
limpia la sesion y vuelve a login.

No se debe agregar el token manualmente en cada funcion de API. El interceptor
existente ya lo hace.

## 3. Endpoints listos para consumir

Todos usan el prefijo `/api`. Solo `health`, `register` y `login` son publicos.

| Metodo | Ruta | Uso del frontend |
| --- | --- | --- |
| GET | `/health` | Comprobar disponibilidad. |
| POST | `/auth/register` | Crear usuario. |
| POST | `/auth/login` | Iniciar sesion. |
| GET | `/auth/me` | Restaurar y validar sesion. |
| POST | `/auth/logout` | Confirmar cierre; la limpieza real del token es local. |
| GET/PATCH/DELETE | `/users/me` | Consultar, editar o eliminar perfil. |
| GET/POST | `/accounts` | Listar o crear cuentas. |
| GET | `/accounts/summary` | Tarjetas de resumen de cuentas. |
| GET/PATCH/DELETE | `/accounts/:id` | Detalle, edicion y eliminacion. |
| PATCH | `/accounts/:id/status` | Activar o inactivar. |
| GET | `/accounts/:id/movements` | Historial paginado de la cuenta. |
| GET/POST | `/categories` | Listar/inicializar o crear categorias del mes. |
| GET/PATCH/DELETE | `/categories/:id` | Detalle, edicion o eliminacion. |
| GET | `/movements` | Tabla, filtros, paginacion y totales del mes. |
| POST | `/movements/income` | Crear ingreso pendiente. |
| POST | `/movements/expense` | Crear egreso pendiente. |
| POST | `/movements/transfer` | Crear transferencia. |
| GET/PATCH/DELETE | `/movements/:id` | Consultar, editar o eliminar logicamente. |
| POST | `/movements/:id/confirm` | Confirmar un pendiente con monto real. |
| POST | `/movements/:id/cancel` | Cancelar y revertir. |

El contrato completo de cuerpos y respuestas esta en
`Backend/finanzas-app-backend/docs/API.md`.

## 4. Periodo global que debe implementar el frontend

Usar un unico estado `selectedPeriod` con formato `YYYY-MM`, inicialmente el
mes actual en `America/Lima`. Debe compartirse entre Cuentas, Categorias,
Movimientos y, en el futuro, Dashboard.

El selector debe abrir un calendario de meses:

- encabezado con año anterior, año visible y año siguiente;
- cuadricula de 4 columnas por 3 filas con enero a diciembre;
- cambiar de año no llama al backend;
- seleccionar un mes actualiza `selectedPeriod` y cierra el selector;
- resaltar el mes actual peruano y el mes seleccionado;
- permitir meses pasados y futuros.

Al seleccionar agosto de 2026:

```http
GET /categories?period=2026-08
GET /movements?period=2026-08&page=1&pageSize=20
```

La consulta de categorias inicializa ese mes si aun no existe. Desde entonces,
editar agosto no toca julio. El frontend no necesita copiar categorias.

Eliminar el valor fijo `2026-05` del estado actual. No hacer doce peticiones
para renderizar los doce botones del calendario.

## 5. Pantalla Movimientos

### Datos

Consumir:

```http
GET /movements?period=YYYY-MM&page=1&pageSize=20
```

Filtros admitidos:

| Filtro | Valores |
| --- | --- |
| `period` | `YYYY-MM` |
| `type` | `INCOME`, `EXPENSE`, `TRANSFER` |
| `accountId` | ID positivo |
| `categoryId` | ID positivo |
| `subcategoryId` | ID positivo |
| `status` | `PENDING`, `CONFIRMED`, `CANCELLED` |
| `search` | Texto o monto exacto |
| `sourceType` | `MANUAL`, `RECURRING`, `VARIABLE`, `SUBSCRIPTION`, `LOAN`, `DEBT`, `GOAL`, `TRANSFER` |
| `page` | Desde 1 |
| `pageSize` | De 1 a 100 |

La respuesta contiene:

- `data`: filas de la pagina;
- `meta`: `total`, `page`, `pageSize` y `totalPages`;
- `totals`: `income`, `incomeCount`, `expense`, `expenseCount`, `transfers` y
  `transferCount`.

Importante: el backend devuelve un solo total `transfers`. No devuelve
`transfersIn` ni `transfersOut`. Las tarjetas superiores deben mostrar ese
contrato actual o esperar una ampliacion del backend.

Los totales consideran solo confirmados del periodo y no cambian por los
filtros de la tabla. La cantidad de resultados visibles sale de `meta.total`.

### Interfaz requerida

- tarjetas: total ingresos, total egresos y transferencias, con sus cantidades;
- buscador;
- filtros por tipo, cuenta, categoria, subcategoria y estado;
- boton Limpiar;
- paginacion;
- tabla con fecha, tipo, descripcion, cuenta, categoria/subcategoria, estado,
  monto y acciones;
- ingreso en verde, egreso en rojo y transferencia en azul;
- acciones segun estado: confirmar pendientes, cancelar y eliminar cuando el
  backend lo permita;
- formularios para nuevo ingreso, nuevo egreso y transferencia.

### Reglas de formularios

- ingreso y egreso no deben enviar `status`: el backend siempre los crea como
  `PENDING`;
- usar `amount` al crear; el backend lo guarda como `expectedAmount`;
- al confirmar enviar `actualAmount`, que puede ser mayor, menor o igual;
- no actualizar `expectedAmount` al confirmar;
- enviar la fecha como `date=YYYY-MM-DD`; el backend obtiene el periodo de esa
  fecha;
- mostrar solo cuentas activas que no sean tarjetas de credito;
- una transferencia necesita cuentas distintas y con la misma moneda;
- categoria y subcategoria son opcionales, pero si se envian deben corresponder
  al tipo y periodo del movimiento;
- no mostrar controles de recurrencia: esos modulos todavia no existen.

Despues de crear, editar, confirmar, cancelar o eliminar, invalidar y volver a
consultar movimientos, cuentas y categorias del periodo afectado. El backend no
emite actualizaciones en tiempo real.

## 6. Pantalla Categorias

Consultar siempre el mes seleccionado:

```http
GET /categories?period=YYYY-MM
```

La respuesta trae listas de ingreso y egreso. Cada categoria incluye
`subcategories`, `monthTotal` y `confirmedMovementsCount`. `monthTotal` suma
solo movimientos confirmados de ese mes.

La pantalla debe tener:

- pestañas Egresos e Ingresos con cantidad de categorias;
- tarjetas con icono, color, nombre, subcategorias y total mensual;
- alta, edicion autorrellenada y eliminacion;
- mensajes vacios y estados de carga/error.

Reglas:

- categoria: nombre de 2 a 80 caracteres;
- color: hexadecimal `#RRGGBB` o `null`;
- icono: hasta 40 caracteres o `null`;
- solo egresos admiten subcategorias;
- al editar, `subcategories` representa la lista final completa;
- si eliminar una categoria o quitar una subcategoria usada responde `409`,
  mostrar el mensaje del backend y pedir reasignar o eliminar el movimiento;
- el boton Importar queda fuera por ahora.

## 7. Cuentas y detalle

Conservar lo ya conectado, con estos ajustes:

1. Quitar o desactivar la consulta a `/accounts/:id/summary`; hoy produce `404`.
2. Conectar el historial a
   `GET /accounts/:id/movements?period=YYYY-MM&page=1&pageSize=20`.
3. Usar `movementsCount` de `GET /accounts/:id` para ocultar o bloquear
   eliminacion cuando sea mayor que cero.
4. Habilitar los botones Ingreso, Egreso y Transferir usando los endpoints de
   movimientos.
5. Volver a consultar detalle, resumen e historial despues de cada operacion.

El resumen mensual de planificacion, esperado contra real y grafico todavia
depende de trabajo del backend. No inventar esos valores en el frontend.

## 8. Errores HTTP

| Estado | Comportamiento recomendado |
| --- | --- |
| `400` | Mostrar el `message` junto al formulario o filtro invalido. |
| `401` | Limpiar sesion y volver a login; el interceptor ya lo hace. |
| `404` | Mostrar recurso no encontrado; no reintentar indefinidamente. |
| `409` | Mostrar conflicto y conservar el formulario para corregirlo. |
| `429` | Informar que hubo demasiados intentos y pedir esperar. |
| `500` | Mostrar un error general y permitir reintentar manualmente. |

Los errores actuales tienen normalmente esta forma:

```json
{ "message": "Descripcion del problema" }
```

Zod rechaza campos adicionales en la mayoria de cuerpos. El frontend debe
enviar solo las propiedades documentadas.

## 9. Orden recomendado de implementacion

1. Corregir el periodo global y construir el calendario anual 4 x 3.
2. Implementar tipos, API, consultas y pantalla de Categorias.
3. Implementar tipos, API, consultas y pantalla de Movimientos.
4. Conectar el historial y las acciones del detalle de cuenta.
5. Retirar la llamada inexistente al resumen de cuenta y dejar su seccion como
   pendiente del backend.
6. Reemplazar datos de demostracion del Dashboard cuando existan sus endpoints.

## 10. Lista de aceptacion

- [ ] No queda ningun periodo fijo de demostracion.
- [ ] Todas las rutas protegidas usan el interceptor existente.
- [ ] Un `401` elimina la sesion local.
- [ ] Cambiar de mes recarga categorias y movimientos.
- [ ] Cambiar solo de año no llama al backend.
- [ ] Ingresos y egresos se crean pendientes.
- [ ] Confirmar usa un monto real sin cambiar el esperado.
- [ ] Los saldos y totales se vuelven a consultar despues de cada cambio.
- [ ] Los filtros se envian como query params y la tabla usa `meta`.
- [ ] Los conflictos `409` muestran el mensaje del backend.
- [ ] No se consume `/accounts/:id/summary`.
- [ ] No se muestran recurrencia, importacion ni modulos que aun no existen.

## 11. Funcionalidades que dependen del backend

- resumen mensual de planificacion de una cuenta;
- ingresos fijos y casuales;
- pagos recurrentes y suscripciones;
- presupuestos, metas, deudas y prestamos;
- Dashboard y reportes reales;
- movimientos de tarjetas de credito;
- separacion de transferencias recibidas y enviadas en los totales;
- carga de comprobantes e importacion;
- sincronizacion en vivo mediante WebSocket.

Todo lo demas descrito en las secciones de Categorias, Movimientos e historial
de cuenta ya puede construirse con el backend actual.
