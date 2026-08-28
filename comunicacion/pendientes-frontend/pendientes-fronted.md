# Corrección de campos monetarios en el frontend

Corrige todos los campos donde el usuario ingresa dinero. Esto incluye cuentas,
movimientos, deudas, préstamos, metas, presupuestos, pagos recurrentes, pagos
variables y suscripciones.

El formulario de cuentas rechaza valores como `956.63` porque está configurado
para aceptar solo enteros (`step="1"`). El backend sí acepta decimales; esta
corrección es únicamente del frontend.

## Configuración

Usar en cada campo monetario una configuración equivalente a:

```html
<input type="number" min="0" step="0.01" />
```

Reglas:

- Permitir enteros y hasta dos decimales: `956.63`, `1000.50`, `0.25`.
- No usar `step="1"` ni redondear mientras el usuario escribe.
- Permitir temporalmente `956.` para que pueda terminar de escribir.
- Validar de forma definitiva al enviar el formulario.
- Rechazar negativos, valores vacíos, texto y números no finitos.
- Rechazar valores con más de dos decimales.
- Convertir a número antes de enviarlo a la API.
- No enviar `NaN`, `Infinity`, `undefined` ni cadenas vacías.
- Mantener el formato monetario visual sin alterar el valor real enviado.

## Mensajes

Mostrar mensajes claros en español:

- `Ingresa un monto válido.`
- `El monto no puede ser negativo.`
- `El monto puede tener como máximo dos decimales.`

No cambiar endpoints, nombres de propiedades ni contratos del backend. Mantener
los estilos y componentes actuales.

## Pruebas obligatorias

Confirmar que todos los formularios permitan guardar:

```text
956.63
1000.50
0.25
1000
```

Y rechacen correctamente:

```text
-10
12.345
texto
```

No deben aparecer advertencias de React ni mensajes de validación del navegador
al ingresar un monto válido con dos decimales.

# Días personalizados en pagos recurrentes, variables y suscripciones

Cuando el usuario seleccione **Días personalizados**, mostrar primero dos
modalidades:

```text
Días de la semana
Días específicos del mes
```

## Días de la semana

Mostrar botones seleccionables en este orden:

```text
Lun · Mar · Mié · Jue · Vie · Sáb · Dom
```

Esta opción sirve para reglas como **Pasajes laborales**. Si el usuario marca
lunes a viernes, la recurrencia debe calcularse automáticamente en cada mes y
no debe crear ocurrencias los sábados ni domingos.

Enviar los días usando valores ISO de lunes a domingo:

```json
{
  "frequency": "CUSTOM_DAYS",
  "customDaysMode": "WEEK_DAYS",
  "customDays": [1, 2, 3, 4, 5]
}
```

Valores permitidos: `1=Lun`, `2=Mar`, `3=Mié`, `4=Jue`, `5=Vie`, `6=Sáb`,
`7=Dom`.

## Días específicos del mes

Mostrar el calendario numérico del 1 al 31 para pagos que ocurren en fechas
concretas. Por ejemplo, los días 3, 11, 18 y 27:

```json
{
  "frequency": "CUSTOM_DAYS",
  "customDaysMode": "MONTH_DAYS",
  "customDays": [3, 11, 18, 27]
}
```

Si se selecciona el día 31 y el mes no tiene 31 días, el backend usará el
último día disponible de ese mes.

## Reglas de edición y visualización

- Al crear, seleccionar una modalidad y al menos un día.
- Al editar, conservar la modalidad actual y permitir cambiarla.
- La regla se repite automáticamente en los meses futuros.
- Editar la regla debe afectar solo ocurrencias futuras; no modificar el
  historial ya registrado.
- Mostrar en la tarjeta la próxima fecha calculada según la modalidad.
- Mantener el bloqueo de una ocurrencia ya registrada para no duplicarla.
- Permitir **Saltar pago** para una ocurrencia vencida o del día actual, sin
  crear movimiento.
- No permitir registrar una fecha futura.
- Si el usuario cambia de “días de la semana” a “días del mes” (o viceversa),
  reemplazar la selección anterior y enviar la nueva modalidad completa.

Para frecuencias que no sean `CUSTOM_DAYS`, enviar `customDays: []` y
`customDaysMode: "MONTH_DAYS"`.

## Nombre de la fecha de inicio

En los formularios de **pagos recurrentes**, **pagos variables** y
**suscripciones**, cambiar la etiqueta:

```text
Primera ocurrencia
```

por:

```text
Fecha de inicio de los pagos
```

Mostrar debajo la ayuda:

```text
El pago comenzará a generarse desde esta fecha.
```

Si el usuario está viendo agosto pero selecciona septiembre para preparar un
pago, la fecha debe pertenecer a septiembre y nunca usar automáticamente la
fecha actual de agosto. Para pagos variables, tanto el período seleccionado
como la fecha de inicio deben pertenecer al mismo mes.

## Mostrar monto unitario y total del período

En cada tarjeta de pagos recurrentes, pagos variables y suscripciones mostrar
por separado:

- **Monto por ocurrencia**: usar `expectedAmount`. Es el valor que corresponde a
  cada día, semana, fecha programada o renovación.
- **Total esperado del período**: usar `periodExpectedAmount`. Es la suma del
  monto unitario multiplicado por todas las ocurrencias del período seleccionado.
- Mostrar también cuántas ocurrencias existen, usando `expectedConfirmations`.

Ejemplo para pasajes laborales:

```text
Monto por día: S/ 2.60
Total esperado de agosto: S/ 54.60
21 fechas programadas
```

No mostrar únicamente `periodExpectedAmount` como si fuera el monto de un solo
pago, porque puede confundir al usuario. El formulario de “Registrar pago” debe
mostrar inicialmente `expectedAmount` como monto sugerido, pero permitir que el
usuario ingrese el monto real de esa ocurrencia.

Si la tarjeta tiene confirmaciones, mantener separados:

```text
Monto por ocurrencia: expectedAmount
Total esperado: periodExpectedAmount
Total registrado: periodActualAmount
Diferencia: periodDifference
```

Adaptar las etiquetas según la frecuencia: “por día”, “por semana”, “por fecha”
o “por renovación”. Para pagos variables, añadir “del mes seleccionado” y no
mostrar totales de meses futuros.

## Vista inicial de los módulos de pagos

Esta regla aplica a **pagos recurrentes**, **pagos variables** y
**suscripciones**:

- Al entrar por primera vez a cualquiera de los tres módulos, seleccionar y
  mostrar primero la pestaña **Pendientes**.
- Consultar la API con la vista pendiente (`view=pending`) o usar la respuesta
  pendiente predeterminada del backend.
- No abrir inicialmente en “Todos”, “Finalizados”, “Confirmados”, “Pausados” ni
  “Vencidos”, aunque existan registros en esos estados.
- El usuario podrá cambiar manualmente a las demás pestañas después de entrar.
- Si no hay pendientes, mantener seleccionada la pestaña “Pendientes” y mostrar
  un estado vacío claro, sin cambiar automáticamente a otra pestaña.

## Regla especial: pagos variables

Para **pagos variables** usar las mismas dos modalidades de días personalizados:

- Días de la semana: botones `Lun` a `Dom`.
- Días específicos del mes: calendario del 1 al 31, en el orden del mes
  seleccionado.

Ejemplo: un pago variable de pasajes laborales creado para agosto con lunes a
viernes solo genera ocurrencias de lunes a viernes **dentro de agosto**.

- No copiarlo a septiembre ni a ningún mes futuro.
- Al cambiar el selector global a septiembre, no mostrar la tarjeta creada para
  agosto.
- Para crear una tarjeta en septiembre, el usuario debe crearla desde septiembre
  y volver a seleccionar los días que correspondan a ese mes.
- La edición y el registro de ocurrencias solo afectan el mes de la tarjeta.
- No mostrar fechas de otro mes, aunque se usen días de la semana.

## Corrección: cumplimiento de la cuenta

En el detalle de una cuenta, separar siempre la **proyección** del
**cumplimiento real**.

- El resultado esperado (`performance.expectedResult`) sigue siendo una
  proyección: saldo inicial del mes - todos los gastos esperados + ingresos que
  ya se registraron.
- Para la tarjeta “Cumplimiento de [mes]” y la gráfica de cumplimiento, usar
  `performance.complianceAmount`. Como compatibilidad también está disponible
  como `performance.cushion`.
- **No** calcular el cumplimiento restando el saldo real menos el resultado
  esperado. Ese cálculo hace parecer positivo un gasto que sólo fue planificado
  y todavía no se pagó.

Comportamiento visual:

- Sin pagos confirmados: mostrar `S/ 0.00`, estado neutro/amarillo y texto
  “Aún no hay pagos confirmados”.
- Pago confirmado exactamente por el monto esperado: `S/ 0.00`, estado
  neutro/amarillo y texto “Vas según lo esperado”.
- Para un gasto, pagar menos de lo esperado produce un valor positivo y color
  verde; pagar más produce un valor negativo y color rojo.
- Para un ingreso, el backend ya aplica la regla inversa: recibir más es verde
  y recibir menos es rojo.
- Usar `performance.isOnTarget` para el caso cero e `isAboveExpected` sólo para
  el caso positivo.

La proyección puede incluir gastos pendientes; el cumplimiento nunca los debe
contar hasta que estén confirmados.

## Corrección visual: panel de notificaciones

Ampliar el panel desplegable de notificaciones del encabezado a un ancho
aproximado de **380 a 420 px** en escritorio. El objetivo es que cada aviso se
entienda sin texto excesivamente comprimido.

- El título, descripción y fecha deben aprovechar el ancho disponible antes de
  pasar a otra línea.
- El monto y el botón para cerrar/eliminar una notificación no deben invadir ni
  reducir el texto principal.
- Mantener una altura mínima uniforme por notificación, con separación clara
  entre avisos.
- El control “Marcar todas como leídas” debe tener espacio suficiente para
  mostrarse completo y ser fácil de pulsar.
- Conservar un ancho adaptable en pantallas pequeñas: no provocar desborde
  horizontal y usar márgenes laterales seguros.

## Pendientes incluye pagos vencidos

Esta regla aplica a **pagos recurrentes**, **pagos variables** y
**suscripciones**.

Al abrir la pestaña **Pendientes**, consultar `view=pending`. Ahora la API
devuelve juntos los pagos próximos y los vencidos, porque ambos requieren una
acción del usuario. La respuesta los entrega con los vencidos primero.

- En la misma pestaña, crear primero la sección **“Vencidos (N)”** con estilo
  de alerta y acciones para registrar o saltar cada pago.
- Debajo, mostrar **“Próximos pendientes (N)”** con los pagos aún no vencidos.
- Seguir mostrando la pestaña **Vencidos** como filtro independiente, para ver
  sólo los atrasados cuando el usuario lo necesite.
- El contador de la pestaña Pendientes es `counts.pending` e incluye vencidos.
  Para el contador de próximos usar `counts.pendingUpcoming`.
- Si `counts.pending` es cero, mostrar el estado vacío de pendientes. No cambiar
  automáticamente a otra pestaña.
