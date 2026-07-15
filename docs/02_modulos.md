# 02 - Modulos del sistema financiero

## Estado del documento

Actualizado con el avance real de cuentas y movimientos al 2026-07-15.

No describe pantallas solamente. Describe responsabilidades reales del sistema.

## Lista de modulos principales

1. Acceso personal.
2. Cuentas.
3. Categorias.
4. Movimientos.
5. Ingresos fijos e ingresos casuales.
6. Pagos recurrentes.
7. Pagos variables.
8. Suscripciones.
9. Prestamos por cobrar.
10. Deudas por pagar.
11. Metas.
12. Presupuestos.
13. Dashboard y reportes.

## 1. Acceso personal

### Responsabilidad

Permitir que el usuario ingrese al sistema y que sus datos financieros queden separados de los datos de otros usuarios.

### Que hace

- Login.
- Registro, si esta habilitado.
- Refresco de sesion con `refresh_tokens`.
- Consulta del usuario autenticado.

### Que no hace

- No administra otros usuarios.
- No maneja roles avanzados.
- No calcula saldos.
- No registra movimientos financieros.

## 2. Cuentas

### Estado actual

Primera version funcional implementada en backend.

### Responsabilidad

Administrar las cuentas donde existe o se mueve dinero.

### Que hace

- Crear cuentas.
- Editar cuentas.
- Listar cuentas.
- Ver detalle de cuenta.
- Activar o inactivar cuentas.
- Eliminar una cuenta que no sea principal.
- Marcar una cuenta como principal.
- Calcular resumen general de cuentas.
- Separar saldo total de credito disponible.

### Que no hace

- No permite eliminar una cuenta principal.
- No permite eliminar cuentas con historial de movimientos.
- No calcula por si solo todo el resultado mensual.
- Los movimientos confirmados modifican su saldo actual.

### Tipos soportados

```text
BANK_ACCOUNT
SAVINGS
CASH
DIGITAL_WALLET
CREDIT_CARD
INVESTMENT
OTHER
```

## 3. Categorias

### Estado actual

Modulo mensual implementado en backend. Permite listar, crear, consultar, editar y eliminar categorias con sus subcategorias.

### Responsabilidad

Clasificar ingresos y egresos.

### Que hace

- Gestiona categorias de ingreso.
- Gestiona categorias de egreso.
- Gestiona subcategorias.
- Permite calcular totales por categoria usando movimientos confirmados.
- Copia la configuracion del ultimo mes al preparar uno nuevo.
- Mantiene cada mes independiente despues de inicializarlo.
- Bloquea eliminar categorias o subcategorias usadas por movimientos del mes.

### Que no hace

- No mueve dinero.
- No modifica saldos.
- No elimina historial.

## 4. Movimientos

### Estado actual

Primera version funcional implementada en backend. Incluye ingresos, gastos, transferencias, consulta, filtros, edicion, confirmacion, cancelacion y eliminacion logica.

### Responsabilidad

Ser la fuente central de verdad financiera.

### Que hace

- Registra ingresos.
- Registra egresos.
- Registra transferencias.
- Guarda estado pendiente, confirmado o cancelado.
- Conserva monto esperado y monto real confirmado.
- Vincula cada movimiento con su modulo de origen mediante `source_type` y `source_id`.
- Alimenta cuentas, dashboard, reportes y categorias.

### Que no hace

- No reemplaza las reglas especificas de cada modulo.
- No debe duplicar movimientos ya generados por otro modulo.

## 5. Ingresos fijos e ingresos casuales

### Estado actual

Modulo planificado y pendiente de implementacion.

### Responsabilidad

Separar los ingresos que se esperan regularmente de los ingresos que aparecen de manera ocasional.

### Que hara

- Los ingresos fijos guardaran una planificacion repetible, como el sueldo.
- Los ingresos casuales registraran entradas ocasionales que no deben repetirse automaticamente.
- Ambos crearan movimientos pendientes.
- Al confirmar se conservara el monto esperado y se guardara por separado el monto realmente recibido.
- Confirmar un monto diferente no cambiara el valor definido para futuros ingresos fijos.

## 6. Pagos recurrentes

### Responsabilidad

Gestionar ingresos o egresos que se repiten en el tiempo.

### Que hace

- Define pagos recurrentes.
- Genera ocurrencias por periodo.
- Permite registrar o confirmar una ocurrencia.
- Crea movimiento al registrarse el pago.

### Cambio de lenguaje confirmado

Para el flujo futuro se preferira hablar de **registrar pago** en vez de usar como concepto principal **confirmar recurrente**.

## 7. Pagos variables

### Responsabilidad

Gestionar ingresos o egresos planificados para un periodo especifico.

### Que hace

- Pertenece a un solo periodo `YYYY-MM`.
- No se copia automaticamente al mes siguiente.
- Al confirmarse crea movimiento.

## 8. Suscripciones

### Responsabilidad

Gestionar servicios o membresias con renovacion periodica.

### Que hace

- Controla monto, cuenta, frecuencia y proxima renovacion.
- Permite pausar, reactivar o finalizar.
- Al registrar pago crea egreso en `movements`.

### Que no hace

- No debe duplicarse como pago recurrente normal.

## 9. Prestamos por cobrar

### Responsabilidad

Controlar dinero prestado a terceros.

### Que hace

- Registra prestamos.
- Controla saldo por cobrar.
- Permite registrar cobros.
- Cada cobro genera ingreso.

## 10. Deudas por pagar

### Responsabilidad

Controlar dinero recibido que debe devolverse.

### Que hace

- Registra deudas.
- Controla saldo pendiente.
- Permite registrar pagos.
- Cada pago genera egreso.

## 11. Metas

### Responsabilidad

Controlar objetivos financieros con aportes y fecha limite.

### Que hace

- Crea metas.
- Registra aportes.
- Calcula progreso.
- Marca completada cuando llega al objetivo.

## 12. Presupuestos

### Responsabilidad

Definir limites mensuales generales o por categoria.

### Que hace

- Guarda presupuesto general o por categoria.
- Calcula lo usado desde movimientos confirmados.

### Estado

Incluido en la especificacion, pendiente de implementacion.

## 13. Dashboard y reportes

### Responsabilidad

Mostrar resumenes y analisis del sistema.

### Que hace

- Lee datos de movimientos, cuentas, metas, deudas, prestamos, suscripciones y presupuestos.
- Muestra esperado contra real.
- Muestra proximos vencimientos.
- Muestra distribucion por categorias.

### Que no hace

- No debe inventar calculos propios separados.
- No registra movimientos directamente.

## Reglas internas de calculo mensual

No se consideran modulo visible independiente.

Sirven para que todo el sistema calcule igual:

- Periodo actual.
- Periodos futuros.
- Resultado esperado.
- Resultado real.
- Diferencias entre previsto y real.

## Relacion simple entre modulos

- Acceso protege los datos del usuario.
- Cuentas guardan el dinero.
- Categorias clasifican movimientos.
- Movimientos centraliza todo lo ocurrido.
- Ingresos fijos, ingresos casuales, recurrentes, variables, suscripciones, prestamos, deudas y metas generan movimientos.
- Dashboard y reportes leen informacion consolidada.
