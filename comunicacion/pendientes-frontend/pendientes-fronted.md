# Pagos recurrentes y variables: calendario, scroll y cantidad por día

Aplica a **pagos recurrentes** y **pagos variables**. No aplicar esta función a
suscripciones, porque una suscripción tiene una única renovación por fecha.

## Calendario para días específicos del mes

- Al elegir “Días específicos del mes”, mostrar un calendario del período que
  se está preparando, no botones planos del 1 al 31.
- Mostrar la cabecera `Lun, Mar, Mié, Jue, Vie, Sáb, Dom` y colocar cada fecha
  bajo el día de semana correcto.
- Los días fuera del mes deben verse atenuados y no ser seleccionables.
- Al seleccionar una fecha, crearla con cantidad `1` por defecto.

## Cantidad de ocurrencias por día

- Después de seleccionar una fecha, mostrar un control `[-] 1 [+]` para esa
  fecha. El mínimo es 1; al bajar de 1 se deselecciona la fecha.
- Cada fecha guarda su propia cantidad. Ejemplo: lunes `1`, sábado `3`.
  Al seleccionar otra fecha, conservar la cantidad anterior y comenzar la nueva
  en `1`.
- Enviar al backend, junto con `customDays`, el mapa `customDayCounts`, por
  ejemplo: `{ "3": 1, "8": 3 }`. Para días de la semana, usar la misma regla
  con `1` para lunes y `6` para sábado.
- Mostrar un resumen claro: `Sábado 8: 3 ocurrencias · S/ 24.00 previsto`,
  calculado como cantidad por monto unitario.

## Registro en la fecha programada

- Cuando llegue una fecha con cantidad mayor a uno, mostrar el avance, por
  ejemplo `Sábado 8 · 1 de 3 registrados`.
- Cada acción “Registrar pago” o “Saltar uno” resuelve una sola ocurrencia.
- Cuando el avance llegue a `3 de 3`, deshabilitar las acciones de registro y
  mostrar “Completado”.

## Modal desplazable

- El modal debe tener una altura máxima basada en la ventana visible y su área
  central debe usar desplazamiento vertical (`overflow-y`).
- Mantener visibles el encabezado y los botones Cancelar/Crear al desplazarse,
  preferiblemente fijos dentro del modal.
- No permitir que el formulario quede cortado fuera de pantalla ni que dependa
  del desplazamiento de la página de fondo.

## Nuevo movimiento: gastos e ingresos hormiga

En el menú superior **Nuevo**, mantener las opciones existentes, pero hacer
explícito que son movimientos manuales fuera de la planificación:

- `Añadir gasto` debe mostrarse como **Nuevo gasto hormiga** o **Añadir gasto
  manual**.
- `Añadir ingreso` debe mostrarse como **Nuevo ingreso hormiga** o **Añadir
  ingreso manual**.
- Mantener `Añadir transferencia` y `Añadir cuenta` sin cambios.

En los botones principales de Movimientos, usar las mismas etiquetas: **Nuevo
gasto manual** y **Nuevo ingreso manual**. Debajo del título del formulario
mostrar una ayuda breve: “No pertenece a pagos recurrentes, variables ni
suscripciones; afectará tu cumplimiento cuando se confirme”.

Reglas de interfaz:

- No mostrar estos movimientos como pagos programados ni sumarlos al monto
  esperado de la cuenta.
- Una vez confirmados, sí deben reflejarse en el cumplimiento del mes.
- Para un gasto manual confirmado, el cumplimiento baja por el monto real; para
  un ingreso manual confirmado, sube por el monto real.
- El formulario debe conservar fecha, cuenta, categoría, descripción, monto
  esperado y monto real según el flujo actual de Movimientos.
- No crear un módulo separado: “hormiga” es una etiqueta comprensible para el
  usuario, mientras que el backend los conserva como movimientos manuales.
