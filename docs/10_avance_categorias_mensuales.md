# 10 - Avance backend de Categorias Mensuales

Fecha de corte: 2026-07-15

## Estado real

El modulo de Categorias esta implementado para administrar categorias de ingreso y egreso por mes.

## API

```text
GET    /api/categories?period=YYYY-MM&type=INCOME|EXPENSE
POST   /api/categories
GET    /api/categories/:id
PATCH  /api/categories/:id
DELETE /api/categories/:id
```

## Independencia mensual

Al abrir o preparar un mes por primera vez se copian las categorias y subcategorias activas del ultimo mes anterior. `CategoryPeriod` registra que el mes ya fue inicializado. Despues, editar o eliminar en agosto no cambia julio.

## Proteccion del historial

Una categoria o subcategoria usada por movimientos visibles del mes no puede eliminarse. El usuario debe cambiar la clasificacion o eliminar esos movimientos. Los movimientos de otros meses no bloquean cambios porque cada mes utiliza su propia copia.

## Totales

`monthTotal` suma solamente `actualAmount` de movimientos confirmados del periodo. Pendientes, cancelados y eliminados no participan. La respuesta tambien incluye cantidades de categorias de ingreso y egreso para las pestañas.

## Alcance aplazado

Importar y exportar categorias no forma parte de este avance y no tiene codigo provisional.
