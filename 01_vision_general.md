# 01 - Vision general del sistema financiero

## Estado del documento

Este documento resume la vision general entendida hasta ahora.

Se debe actualizar conforme avance la conversacion y se confirmen nuevas reglas, cambios o decisiones del sistema.

## Proposito del sistema

El sistema sera una herramienta de planificacion financiera personal.

No sera solamente un registro de gastos. Su objetivo principal sera ayudar al usuario a planificar cada mes, registrar lo que realmente ocurre y comparar lo planificado contra lo real.

El sistema debe responder preguntas como:

- Con cuanto dinero empiezo el mes.
- Cuanto planeo recibir.
- Cuanto planeo gastar.
- Cuanto planeo ahorrar.
- Cuanto dinero deberia sobrar al final del mes.
- Cuanto dinero debo.
- Cuanto dinero me deben.
- Si estoy cumpliendo o no mi planificacion.
- Como evoluciona mi situacion financiera mes a mes.

## Idea central

Cada mes se trabajara con una planificacion financiera.

Al iniciar o preparar un mes, el usuario podra definir ingresos, gastos, ahorros, deudas, prestamos, suscripciones y otros compromisos previstos.

Con esa informacion, el sistema calculara un resultado esperado. Ese resultado esperado representa el sobrante estimado que deberia quedar al finalizar el mes.

Al terminar el mes, el sistema tambien debera permitir calcular un resultado real. Este concepto se seguira definiendo con mas detalle durante el avance del proyecto.

## Calendario y meses

El sistema debe estar alineado con el calendario real.

Si hoy es el ultimo dia del mes, todavia se considera el mes actual. Cuando la fecha real cambie al dia 1, el sistema ya debe considerar el nuevo mes.

Ejemplo:

- Si hoy es 30 de junio, el mes actual sigue siendo junio.
- Si hoy es 1 de julio, el mes actual pasa a ser julio.

El usuario tambien podra planificar meses futuros.

Ejemplo:

- Si esta en junio, puede preparar la planificacion de julio.
- Esa planificacion futura debe poder verse como una vista futura separada del mes actual.

La planificacion mensual no se tratara por ahora como un modulo principal independiente. Se entendera como una regla interna de calculo que se refleja dentro de cuentas, pagos recurrentes, pagos variables, suscripciones, metas, prestamos, deudas y dashboard.

## Resultado esperado

El resultado esperado es un calculo de planificacion.

Representa cuanto dinero se espera que sobre al final del mes despues de considerar:

- Monto inicial.
- Ingresos previstos.
- Gastos previstos.
- Ahorros planificados.
- Deudas previstas.
- Prestamos que afecten el mes.
- Suscripciones.
- Pagos recurrentes.
- Pagos variables.

Este resultado es solo una proyeccion. Puede cambiar cuando los movimientos pendientes se confirmen, se editen o se eliminen.

## Resultado real

El resultado real se ira definiendo conforme avance el sistema.

Por ahora se entiende que:

- Se calculara al cerrar o finalizar el mes.
- Debe basarse en lo que realmente ocurrio.
- Debe considerar movimientos confirmados.
- Debe permitir comparar lo real contra lo esperado.
- Deudas y prestamos tambien deben afectar este resultado.

## Monto inicial del mes

El monto inicial del mes representa con cuanto dinero se planifica iniciar un periodo.

Puede estar formado por:

- Sobrante del mes anterior.
- Ingresos previstos del mes.
- Otros saldos relevantes.

Si un ingreso todavia no fue recibido, se manejara como pendiente.

## Estados de movimientos

Los movimientos del sistema tendran estados importantes:

- Pendiente: el movimiento esta planificado, pero todavia no ocurrio.
- Confirmado: el movimiento ya ocurrio.
- Eliminado: el movimiento fue eliminado, pero sigue visible en el historial.

Un movimiento pendiente ayuda a calcular lo esperado.

Un movimiento confirmado ayuda a calcular lo real.

Un movimiento eliminado no desaparece completamente, porque debe poder verse en el historial.

## Edicion de movimientos

Un movimiento confirmado podra editarse.

Cuando se edite, el dato se reemplazara. Tambien se debe poder actualizar la fecha cuando corresponda.

Todo movimiento del sistema debe tener una descripcion.

## Eliminacion de movimientos

Eliminar desde un modulo no debe borrar automaticamente el historial.

Si un movimiento se elimina, debe quedar marcado como eliminado y seguir visible.

Por ahora, si se elimina desde el modulo de movimientos, tambien debe quedar visible como eliminado.

## Moneda

Por ahora, todo el sistema manejara una sola moneda comun para todas las cuentas.

La moneda podra elegirse, pero no se manejara conversion entre monedas en esta etapa.

## Usuario y acceso

El sistema sera de uso personal.

Por ahora solo se manejara login para acceder al sistema. No habra administracion de otros usuarios ni control entre multiples usuarios.

El login ya esta avanzado dentro de configuraciones y guarda la informacion en base de datos.

## Cuentas

Las cuentas representan lugares donde existe o se mueve dinero.

Ejemplos:

- Cuenta bancaria.
- Cuenta de ahorros.
- Billetera digital.
- Efectivo.
- Tarjeta de credito.

Las tarjetas de credito se consideran para futuro. Por ahora podran estar visibles o contempladas, pero no tendran logica funcional completa.

Las cuentas deben permitir saber:

- Cuanto dinero hay.
- Como se movio el dinero.
- Que movimientos estan asociados.
- Como participa la cuenta en la planificacion mensual.

## Ahorro

El ahorro no debe tratarse como gasto.

El ahorro mensual debe trabajarse como transferencia, porque representa mover dinero de una cuenta a otra.

Ejemplo:

- Sale dinero de una cuenta principal.
- Ingresa dinero a una cuenta de ahorros.

Aunque reduzca el dinero disponible en una cuenta, no significa que el dinero se haya perdido.

## Metas

Las metas representan objetivos financieros.

Ejemplos:

- Fondo de emergencia.
- Viaje.
- Compra importante.
- Reposicion de ahorros.
- Pago de deuda.

Las metas deben considerar dinero real depositado en una cuenta y tambien medir el progreso hacia un objetivo.

Cada aporte a una meta debe quedar registrado como movimiento.

## Pagos recurrentes

Los pagos recurrentes representan ingresos o gastos que se repiten en el tiempo.

Ejemplos:

- Sueldo.
- Internet.
- Servicios.
- Alimentacion de mascota.
- Pagos fijos.

Los pagos recurrentes sirven para proyectar automaticamente cada mes.

Si un pago recurrente todavia no ocurre, estara pendiente.

Cuando se confirme, pasara a representar un movimiento real.

Si el monto real es distinto al previsto, debe poder editarse o registrarse la diferencia.

Ejemplo:

- Se planifica comida de gato por S/10.
- Al comprar, realmente cuesta S/8.
- La diferencia positiva es S/2.

Esa diferencia no es un ingreso nuevo. Es dinero que sobro respecto al plan.

Si se planifica S/10 y se gasta S/12, la diferencia es negativa porque se gasto mas de lo esperado.

## Pagos variables

Los pagos variables son ingresos o gastos planificados para un mes especifico.

No se copian automaticamente al siguiente mes.

Ejemplos:

- Compra de zapatillas.
- Concierto.
- Viaje.
- Regalo.
- Venta ocasional.
- Proyecto freelance.

Los pagos variables ayudan a calcular la proyeccion de un mes concreto.

## Suscripciones

Las suscripciones seran un modulo separado.

Aunque son similares a pagos recurrentes, se manejaran de forma independiente.

Ejemplos:

- Netflix.
- Spotify.
- Disney+.
- ChatGPT.
- Google Drive.
- Hosting.

Las suscripciones deben participar en la planificacion financiera y en los gastos previstos del mes.

## Prestamos por cobrar

Los prestamos por cobrar representan dinero que el usuario entrega a otra persona o entidad y espera recuperar.

Por ahora, los prestamos se manejaran con pago completo.

Si alguien devuelve solo una parte, se creara otro registro separado. En el futuro se podra mejorar para manejar pagos parciales.

Un prestamo no debe tratarse como gasto normal, porque el dinero deberia regresar.

Debe afectar cuentas, movimientos y resultado financiero.

## Deudas por pagar

Las deudas por pagar representan dinero que el usuario recibe y debe devolver.

Por ahora, las deudas se manejaran con pago completo.

No se manejaran cuotas ni intereses en esta primera etapa.

Una deuda aumenta el saldo cuando se recibe el dinero, pero tambien crea una obligacion futura.

Debe afectar cuentas, movimientos y resultado financiero.

## Categorias

Las categorias sirven para clasificar ingresos y egresos.

Existiran categorias de ingresos y categorias de egresos.

Tambien podran existir subcategorias.

Ejemplos:

- Sueldo.
- Transporte.
- Alimentacion.
- Entretenimiento.
- Servicios.
- Suscripciones.
- Mascotas.
- Familia.

Las categorias ayudan a ordenar movimientos, hacer filtros y preparar futuros reportes.

## Movimientos

El modulo de movimientos sera el historial general del sistema.

Debe concentrar ingresos, egresos, transferencias, pagos de deudas, recuperacion de prestamos, aportes a metas, pagos recurrentes, pagos variables y suscripciones.

Cada movimiento debe incluir como minimo:

- Fecha.
- Tipo.
- Cuenta relacionada.
- Monto.
- Estado.
- Descripcion.

Los movimientos son importantes porque permiten reconstruir lo que paso con el dinero.

## Historial mensual

La informacion historica principal sera el historial de movimientos de mes en mes.

Cada mes debe tener su propio registro.

Los pagos fijos o recurrentes se pueden duplicar como parte del calculo mensual, pero no deben contarse como confirmados hasta que realmente ocurran.

Los meses pasados podran modificarse, pero el sistema debe cuidar que el historial siga siendo entendible.

## Riesgos conceptuales detectados

Estos riesgos deben controlarse durante el diseño:

- Duplicar ingresos al contar un ingreso pendiente y luego volverlo a contar al confirmarlo.
- Duplicar gastos si una suscripcion tambien se registra como pago recurrente normal.
- Confundir ahorro con gasto.
- Confundir prestamo con gasto definitivo.
- Confundir deuda con ingreso real.
- Mezclar dinero propio con dinero que se debe devolver.
- Perder historial al eliminar registros.
- Cambiar meses pasados sin dejar claro que fueron modificados.
- Mezclar planificacion futura con movimientos reales del mes actual.
- Calcular distinto el mismo dato en diferentes partes del sistema.

## Reglas para evitar riesgos

- Separar siempre movimientos pendientes de movimientos confirmados.
- No contar un movimiento dos veces.
- Tratar el ahorro como transferencia.
- Tratar prestamos como dinero por cobrar.
- Tratar deudas como obligaciones por pagar.
- Mantener movimientos eliminados visibles como historial.
- Separar vista actual de vista futura.
- Usar una sola logica de calculo para todo el sistema.
- Evitar que suscripciones y pagos recurrentes dupliquen el mismo gasto.

## Temas pendientes para definir mas adelante

- Definicion exacta del resultado real.
- Reglas completas de cierre mensual.
- Forma final de mostrar diferencias positivas y negativas.
- Reglas de modificacion de meses pasados.
- Dashboard.
- Reportes.
- Presupuestos.
- Logica futura de tarjetas de credito.
- Posible manejo de pagos parciales en prestamos y deudas en versiones futuras.
- Importacion desde Excel.
