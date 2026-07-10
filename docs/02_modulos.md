# 02 - Modulos del sistema financiero

## Estado del documento

Este documento define los modulos principales del sistema desde una mirada de arquitectura.

No describe pantallas ni diseno visual. Describe responsabilidades reales del sistema.

Debe actualizarse cuando se confirmen nuevas reglas o se corrijan decisiones durante el avance del proyecto.

## Criterio usado para definir modulos

Un modulo existe si tiene una responsabilidad clara, propia y suficientemente importante dentro del sistema.

Se evitan modulos que sean solo:

- Pantallas.
- Botones.
- Formularios.
- Filtros.
- Reportes pequenos.
- Funciones sueltas.

## Lista de modulos principales

Los modulos principales detectados son:

1. Acceso personal.
2. Cuentas.
3. Movimientos.
4. Categorias.
5. Pagos recurrentes.
6. Pagos variables.
7. Suscripciones.
8. Metas.
9. Prestamos por cobrar.
10. Deudas por pagar.
11. Importacion y exportacion.
12. Dashboard y analisis.
13. Tarjetas de credito futuras.

## 1. Modulo Acceso personal

### Responsabilidad

Administrar el ingreso del usuario al sistema mediante login.

El sistema sera de uso personal. Por ahora no se administraran otros usuarios.

### Que hace

- Gestiona login.
- Permite acceder al sistema.
- Guarda la informacion de acceso en base de datos.
- Forma parte de configuraciones.

### Que no hace

- No administra otros usuarios.
- No maneja roles o permisos avanzados.
- No calcula saldos.
- No registra movimientos financieros.
- No planifica meses.
- No administra cuentas bancarias.

### Justificacion

Aunque el sistema sea personal, necesita un acceso basico para entrar y proteger la informacion financiera.

Este modulo existe porque el login ya forma parte del sistema y guarda datos en base de datos.

### Submodulos posibles

- Inicio de sesion.
- Configuracion de acceso.

## 2. Modulo Cuentas

### Responsabilidad

Administrar los lugares donde existe o se mueve el dinero.

Ejemplos:

- Cuenta bancaria.
- Cuenta de ahorros.
- Billetera digital.
- Efectivo.
- Tarjeta de credito.

### Que hace

- Crea cuentas.
- Edita cuentas.
- Mantiene saldo de cuentas.
- Permite activar o inactivar cuentas.
- Permite marcar una cuenta principal por usuario.
- Permite identificar desde donde sale o hacia donde entra el dinero.
- Participa en transferencias.
- Sirve como base para calcular dinero disponible.
- Separa saldo total de credito disponible.

### Que no hace

- No decide si un gasto esta bien o mal.
- No define categorias.
- No calcula por si solo todas las reglas del mes.
- No debe guardar reglas propias de suscripciones, deudas o prestamos.

### Justificacion

Todo movimiento de dinero necesita una cuenta de origen, una cuenta de destino o ambas.

Este modulo existe porque las cuentas son la base real del dinero disponible.

### Submodulos posibles

- Cuentas activas.
- Cuentas inactivas.
- Saldos.
- Transferencias entre cuentas.
- Cuenta principal.
- Cuentas de credito.

### Regla confirmada sobre credito

Las cuentas de credito forman parte del modulo de cuentas, pero no suman al saldo total.

El credito disponible se muestra como un indicador separado.

## 3. Modulo Movimientos

### Responsabilidad

Ser el historial financiero central del sistema.

Un movimiento representa algo que afecta o explica el dinero.

### Que hace

- Registra ingresos.
- Registra egresos.
- Registra transferencias.
- Registra pagos de deudas.
- Registra recuperacion de prestamos.
- Registra aportes a metas.
- Registra pagos de suscripciones.
- Registra pagos recurrentes confirmados.
- Registra pagos variables confirmados.
- Mantiene estados como pendiente, confirmado y eliminado.
- Permite consultar el historial por mes.

### Que no hace

- No reemplaza a los modulos origen.
- No decide la regla especifica de una deuda, prestamo, meta o suscripcion.
- No debe duplicar movimientos que ya vienen de otro modulo.

### Justificacion

El sistema necesita una fuente central para saber que ocurrio con el dinero.

Este modulo evita que cada parte del sistema tenga su propio historial separado y dificil de reconciliar.

### Submodulos posibles

- Ingresos.
- Egresos.
- Transferencias.
- Historial mensual.
- Movimientos eliminados.
- Busqueda y filtros.

## 4. Modulo Categorias

### Responsabilidad

Clasificar ingresos y egresos.

### Que hace

- Administra categorias de ingresos.
- Administra categorias de egresos.
- Administra subcategorias.
- Permite ordenar movimientos.
- Ayuda a futuros reportes y analisis.

### Que no hace

- No registra dinero.
- No modifica saldos.
- No confirma pagos.
- No calcula por si sola el resultado mensual.

### Justificacion

Las categorias permiten entender en que se gasta o de donde viene el dinero.

Sin este modulo, los movimientos existirian, pero seria dificil analizarlos.

### Submodulos posibles

- Categorias de ingresos.
- Categorias de egresos.
- Subcategorias.
- Importacion y exportacion de categorias.

## Reglas internas de calculo mensual

Estas reglas no se consideran un modulo principal visible.

No representan una pantalla independiente ni un dashboard. Son la logica interna que permite que cuentas, pagos recurrentes, pagos variables, suscripciones, metas, prestamos, deudas y dashboard calculen el mes de la misma manera.

### Responsabilidad

Centralizar los calculos mensuales para evitar que cada modulo calcule de forma distinta.

### Que hace

- Identifica el mes actual segun el calendario real.
- Permite que existan vistas futuras de meses planificados.
- Calcula totales esperados.
- Ayuda a calcular el sobrante esperado.
- Mas adelante ayudara a calcular el resultado real.
- Separa movimientos pendientes, confirmados y eliminados.
- Permite comparar monto previsto contra monto real.

### Que no hace

- No es una pantalla propia.
- No reemplaza al dashboard.
- No reemplaza al modulo de cuentas.
- No reemplaza al historial de movimientos.
- No confirma movimientos por si sola.

### Justificacion

La planificacion mensual aparece distribuida en varios modulos. Por ejemplo, se ve en cuentas, gastos recurrentes, gastos variables, suscripciones, prestamos y deudas.

Por eso no se documenta como modulo principal independiente. Sin embargo, la regla de calculo debe existir para mantener consistencia en todo el sistema.

## 5. Modulo Pagos recurrentes

### Responsabilidad

Gestionar ingresos y gastos que se repiten en el tiempo.

### Que hace

- Registra ingresos recurrentes.
- Registra egresos recurrentes.
- Proyecta pagos futuros.
- Genera movimientos pendientes para cada periodo.
- Permite registrar cuando el pago realmente ocurre.
- Permite registrar diferencias entre monto previsto y monto real.

### Que no hace

- No administra suscripciones, porque estas tendran modulo propio.
- No debe duplicar movimientos ya generados.
- No maneja gastos unicos de un mes.

### Justificacion

Muchos ingresos y gastos se repiten. Si no existe este modulo, el usuario tendria que crearlos manualmente cada mes.

Este modulo ayuda a construir los calculos mensuales de forma automatica.

La accion operativa no se llamara confirmar recurrente. Se llamara registrar pago.

Al registrar pago, el monto previsto se carga por defecto, pero el usuario puede ingresar un monto real distinto.

### Submodulos posibles

- Ingresos recurrentes.
- Egresos recurrentes.
- Frecuencias.
- Confirmaciones.
- Diferencias entre monto previsto y real.

## 6. Modulo Pagos variables

### Responsabilidad

Gestionar ingresos y gastos planificados para un mes especifico, pero que no se repiten automaticamente.

### Que hace

- Registra gastos variables del mes.
- Registra ingresos variables del mes.
- Participa en el resultado esperado.
- Permite confirmar cuando el movimiento ocurre.
- Permite eliminar o editar planificaciones del mes.

### Que no hace

- No se copia automaticamente al mes siguiente.
- No administra pagos fijos.
- No administra suscripciones.

### Justificacion

No todos los gastos son recurrentes. Existen compras, viajes, eventos o ingresos ocasionales que deben planificarse solo para un mes.

Este modulo separa lo extraordinario de lo fijo.

### Submodulos posibles

- Ingresos variables.
- Egresos variables.
- Planificacion por mes.
- Confirmacion de variables.

## 7. Modulo Suscripciones

### Responsabilidad

Administrar servicios o membresias con renovaciones periodicas.

### Que hace

- Registra suscripciones.
- Controla proxima fecha de cobro.
- Controla frecuencia.
- Permite confirmar pagos.
- Participa en los gastos previstos del mes.
- Genera movimientos cuando se confirma una suscripcion.

### Que no hace

- No administra todos los pagos recurrentes.
- No debe duplicarse como pago recurrente normal.
- No maneja deudas ni prestamos.

### Justificacion

Aunque se parece a pagos recurrentes, una suscripcion tiene una logica especifica de renovacion, estado y seguimiento.

Separarla evita mezclar servicios con otros gastos fijos generales.

### Submodulos posibles

- Suscripciones activas.
- Suscripciones pausadas.
- Renovaciones.
- Confirmacion de pago.

## 8. Modulo Metas

### Responsabilidad

Gestionar objetivos financieros con monto, plazo y progreso.

### Que hace

- Crea metas.
- Registra aportes.
- Mide progreso.
- Relaciona aportes con cuentas reales.
- Permite saber cuanto falta para completar una meta.
- Participa en los calculos mensuales cuando hay aportes previstos.

### Que no hace

- No debe tratar los aportes como gastos normales.
- No administra deudas por si sola.
- No reemplaza a las cuentas donde se guarda el dinero.

### Justificacion

El sistema necesita representar objetivos como ahorro, fondo de emergencia, viaje o compra importante.

Las metas combinan dinero real guardado con seguimiento de avance.

### Submodulos posibles

- Metas activas.
- Metas completadas.
- Metas pausadas.
- Aportes.
- Progreso.

## 9. Modulo Prestamos por cobrar

### Responsabilidad

Gestionar dinero que el usuario presto y espera recuperar.

### Que hace

- Registra prestamos entregados.
- Descuenta dinero de una cuenta al prestar.
- Mantiene estado pendiente o pagado.
- Registra recuperacion completa del prestamo.
- Afecta cuentas, movimientos y resultado financiero.

### Que no hace

- No maneja pagos parciales en esta primera etapa.
- No debe tratar el prestamo como gasto definitivo.
- No administra deudas del usuario.

### Justificacion

El dinero prestado no es igual a un gasto normal. Sale de una cuenta, pero se espera que vuelva.

Este modulo permite controlar cuanto dinero esta pendiente de recuperar.

### Submodulos posibles

- Prestamos pendientes.
- Prestamos pagados.
- Recuperacion de prestamos.

## 10. Modulo Deudas por pagar

### Responsabilidad

Gestionar dinero que el usuario recibio y debe devolver.

### Que hace

- Registra deudas recibidas.
- Aumenta saldo cuando se recibe dinero.
- Mantiene obligacion pendiente.
- Registra pago completo de la deuda.
- Afecta cuentas, movimientos y resultado financiero.

### Que no hace

- No maneja cuotas ni intereses en esta primera etapa.
- No maneja pagos parciales por ahora.
- No debe tratar el dinero recibido como ingreso real libre.

### Justificacion

Una deuda aumenta el dinero disponible, pero tambien crea una obligacion.

Este modulo evita confundir dinero propio con dinero que se debe devolver.

### Submodulos posibles

- Deudas pendientes.
- Deudas pagadas.
- Pago de deudas.

## 11. Modulo Importacion y exportacion

### Responsabilidad

Permitir entrada y salida de informacion desde archivos externos.

### Que hace

- Importa informacion desde Excel.
- Exporta informacion para respaldo o revision.
- Permite reutilizar datos claros en el sistema.
- Puede apoyar categorias, pagos recurrentes, pagos variables e historiales.

### Que no hace

- No calcula saldos por si solo.
- No decide reglas financieras.
- No debe importar datos ambiguos sin validacion.

### Justificacion

El sistema nace desde una logica ya trabajada en Excel. Por eso debe existir una forma ordenada de traer informacion externa sin romper los calculos.

### Submodulos posibles

- Importacion desde Excel.
- Exportacion a Excel.
- Exportacion a CSV o JSON.
- Validacion previa de datos importados.

## 12. Modulo Dashboard y analisis

### Responsabilidad

Mostrar resumenes y lectura general de la situacion financiera.

### Que hace

- Muestra indicadores generales.
- Muestra comparaciones entre esperado y real.
- Muestra evolucion mensual.
- Muestra proximos pagos.
- Muestra deudas, prestamos, metas y suscripciones relevantes.

### Que no hace

- No debe ser la fuente principal de los datos.
- No registra movimientos directamente.
- No reemplaza las reglas internas de calculo mensual.

### Justificacion

El usuario necesita una vista clara para entender su situacion sin revisar cada modulo por separado.

Este modulo debe consultar informacion ya calculada o registrada por otros modulos.

### Submodulos posibles

- Resumen mensual.
- Evolucion financiera.
- Proximos vencimientos.
- Cumplimiento de metas.
- Alertas futuras.

## 13. Modulo Tarjetas de credito futuras

### Responsabilidad

Reservar espacio conceptual para una funcionalidad futura de tarjetas de credito.

### Que hace por ahora

- Puede permitir visualizar o registrar el tipo de cuenta como tarjeta de credito.
- Deja claro que la logica completa no estara activa en la primera etapa.

### Que no hace por ahora

- No calcula linea de credito.
- No calcula fechas de pago.
- No calcula intereses.
- No maneja cuotas.
- No genera alertas de deuda de tarjeta.

### Justificacion

Las tarjetas de credito ya tienen una primera version funcional dentro del modulo de cuentas: limite de credito, credito disponible y exclusion del saldo total. Las reglas avanzadas quedan para una etapa futura.

Mantenerlas separadas evita mezclar una logica incompleta con cuentas normales.

### Submodulos posibles a futuro

- Linea de credito.
- Consumos.
- Cuotas.
- Fecha de corte.
- Fecha de pago.
- Intereses.
- Alertas.

## Relacion simple entre modulos

Esta relacion muestra dependencias conceptuales, no una base de datos ni una arquitectura tecnica.

### Acceso personal

Base de ingreso al sistema.

Usan este acceso:

- Cuentas.
- Movimientos.
- Categorias.
- Pagos recurrentes.
- Pagos variables.
- Suscripciones.
- Metas.
- Prestamos.
- Deudas.

### Cuentas

Base del dinero real.

Usan cuentas:

- Movimientos.
- Pagos recurrentes.
- Pagos variables.
- Suscripciones.
- Metas.
- Prestamos.
- Deudas.

### Movimientos

Historial central.

Generan o consultan movimientos:

- Cuentas.
- Pagos recurrentes.
- Pagos variables.
- Suscripciones.
- Metas.
- Prestamos.
- Deudas.
- Dashboard y analisis.

### Categorias

Clasificacion del dinero.

Usan categorias:

- Movimientos.
- Pagos recurrentes.
- Pagos variables.
- Suscripciones.
- Dashboard y analisis.

### Reglas internas de calculo mensual

Logica compartida para calcular el mes de forma consistente.

Usan estas reglas:

- Cuentas.
- Movimientos.
- Pagos recurrentes.
- Pagos variables.
- Suscripciones.
- Metas.
- Prestamos.
- Deudas.

### Dashboard y analisis

Modulo de lectura.

Consulta informacion de:

- Reglas internas de calculo mensual.
- Movimientos.
- Cuentas.
- Metas.
- Prestamos.
- Deudas.
- Suscripciones.

## Modulos que no se crean por ahora

No se crean como modulos independientes por ahora:

- Filtros: son parte de los modulos que muestran listas.
- Formularios: son parte de la operacion de cada modulo.
- Reportes avanzados: quedan pendientes hasta definir mejor dashboard y analisis.
- Alertas: quedan pendientes porque todavia no hay reglas completas.
- Presupuestos: quedan pendientes porque aun no estan definidos.
- Auditoria de cambios: queda pendiente, porque por ahora la edicion reemplaza el dato.

## Riesgos de modularizacion

### Riesgo: duplicar pagos entre recurrentes y suscripciones

Una suscripcion no debe registrarse tambien como pago recurrente normal.

### Riesgo: convertir ahorro en gasto

El ahorro debe tratarse como transferencia entre cuentas, no como egreso definitivo.

### Riesgo: mezclar planificacion con realidad

Los movimientos pendientes sirven para planificar.

Los movimientos confirmados sirven para representar lo que ya ocurrio.

### Riesgo: que dashboard calcule distinto

Dashboard no debe inventar calculos propios. Debe usar las reglas internas de calculo mensual, movimientos y cuentas.

### Riesgo: perder historial

Los movimientos eliminados deben seguir visibles como eliminados.

## Decision actual

Los modulos principales quedan definidos de forma inicial, pero este documento se mantiene abierto a cambios.

Cuando se confirme una nueva regla, se debera actualizar este archivo si afecta responsabilidades, limites o relaciones entre modulos.
