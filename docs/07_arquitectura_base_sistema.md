# 07 - Arquitectura base del sistema

## Estado del documento

Propuesta inicial basada en la vision general y los modulos definidos hasta ahora.

Esta arquitectura sirve como base para empezar a construir el backend, pero debe validarse cuando se implemente el codigo y cuando se definan reglas faltantes.

## Objetivo

Definir una base completa de tablas y relaciones para sostener el sistema financiero personal.

La idea principal es evitar duplicidad de datos, mantener historial y separar claramente:

- Lo planificado.
- Lo confirmado.
- Lo eliminado.
- Los movimientos reales.
- Las reglas mensuales.
- Los modulos que generan movimientos.

## Principio central

El sistema debe tener una tabla central de movimientos.

Los demas modulos pueden crear, planificar o confirmar informacion, pero el historial financiero debe pasar por movimientos.

Ejemplo:

- Un pago recurrente confirmado genera un movimiento.
- Una suscripcion confirmada genera un movimiento.
- Un aporte a una meta genera un movimiento.
- Un prestamo genera salida y recuperacion.
- Una deuda genera ingreso y pago.

## Tablas principales propuestas

### Acceso y configuracion

- `usuarios`
- `configuraciones_usuario`

### Calendario financiero

- `periodos_financieros`

Esta tabla no representa un modulo visible. Sirve para ordenar los calculos por mes y permitir vistas futuras.

### Dinero y clasificacion

- `cuentas`
- `categorias`
- `subcategorias`
- `movimientos`

### Planificacion y compromisos

- `pagos_recurrentes`
- `pagos_recurrentes_ocurrencias`
- `pagos_variables`
- `suscripciones`
- `suscripciones_ocurrencias`

### Objetivos y obligaciones

- `metas`
- `aportes_metas`
- `prestamos_cobrar`
- `deudas_pagar`

## Responsabilidad de cada tabla

| Tabla | Responsabilidad |
| --- | --- |
| `usuarios` | Guarda el acceso personal del sistema. |
| `configuraciones_usuario` | Guarda configuraciones generales como moneda principal. |
| `periodos_financieros` | Representa cada mes de trabajo, actual, pasado o futuro. |
| `cuentas` | Guarda las cuentas donde existe o se mueve dinero. |
| `categorias` | Clasifica ingresos y egresos. |
| `subcategorias` | Da mas detalle dentro de una categoria. |
| `movimientos` | Historial central de ingresos, egresos y transferencias. |
| `pagos_recurrentes` | Define pagos o ingresos que se repiten. |
| `pagos_recurrentes_ocurrencias` | Representa cada aparicion mensual o periodica de un recurrente. |
| `pagos_variables` | Guarda ingresos o gastos planificados para un mes especifico. |
| `suscripciones` | Define servicios con renovacion periodica. |
| `suscripciones_ocurrencias` | Representa cada renovacion planificada o confirmada. |
| `metas` | Guarda objetivos financieros. |
| `aportes_metas` | Registra aportes realizados a metas. |
| `prestamos_cobrar` | Controla dinero prestado a terceros. |
| `deudas_pagar` | Controla dinero recibido que debe devolverse. |

## Estados base

### Estados de movimientos

- `pendiente`
- `confirmado`
- `eliminado`

### Estados operativos

Aplican a recurrentes, suscripciones y metas cuando corresponda.

- `activo`
- `pausado`
- `finalizado`

### Estados de prestamos y deudas

- `pendiente`
- `pagado`
- `eliminado`

## Reglas importantes

- El ahorro no es gasto. Se registra como transferencia.
- Las suscripciones no deben duplicarse como pagos recurrentes normales.
- Los pagos recurrentes son plantillas; sus ocurrencias representan cada aparicion en el tiempo.
- Los pagos variables pertenecen a un periodo financiero especifico.
- Un movimiento pendiente afecta el calculo esperado.
- Un movimiento confirmado afecta el resultado real.
- Un movimiento eliminado queda visible, pero no debe contarse como activo.
- Prestamos y deudas afectan cuentas, movimientos y calculos mensuales.
- Por ahora no se manejan pagos parciales de prestamos ni deudas.
- Por ahora se maneja una sola moneda general.

## Diagrama ERD completo

Ver tambien:

- [ERD visual completo](diagrams/erd/sistema_completo.md)
- [Fuente Mermaid](diagrams/erd/sistema_completo.mmd)
- [Fuente DBML](diagrams/erd/sistema_completo.dbml)

```mermaid
erDiagram
    USUARIOS {
        int id PK
        string nombre
        string email UK
        string password_hash
        string estado
        datetime creado_en
        datetime actualizado_en
    }

    CONFIGURACIONES_USUARIO {
        int id PK
        int usuario_id FK
        string moneda_principal
        string zona_horaria
        datetime creado_en
        datetime actualizado_en
    }

    PERIODOS_FINANCIEROS {
        int id PK
        int usuario_id FK
        int anio
        int mes
        string estado
        decimal monto_inicial_estimado
        decimal resultado_esperado
        decimal resultado_real
        datetime creado_en
        datetime actualizado_en
    }

    CUENTAS {
        int id PK
        int usuario_id FK
        string nombre
        string tipo
        string banco
        string moneda
        decimal saldo_inicial
        decimal saldo_actual
        string estado
        datetime creado_en
        datetime actualizado_en
    }

    CATEGORIAS {
        int id PK
        int usuario_id FK
        string nombre
        string tipo
        string color
        string icono
        string estado
        datetime creado_en
        datetime actualizado_en
    }

    SUBCATEGORIAS {
        int id PK
        int categoria_id FK
        string nombre
        string estado
        datetime creado_en
        datetime actualizado_en
    }

    MOVIMIENTOS {
        int id PK
        int usuario_id FK
        int periodo_id FK
        int cuenta_origen_id FK
        int cuenta_destino_id FK
        int categoria_id FK
        int subcategoria_id FK
        string tipo
        decimal monto_previsto
        decimal monto_real
        string estado
        date fecha
        time hora
        string descripcion
        string modulo_origen
        int referencia_id
        datetime creado_en
        datetime actualizado_en
    }

    PAGOS_RECURRENTES {
        int id PK
        int usuario_id FK
        int cuenta_id FK
        int categoria_id FK
        int subcategoria_id FK
        string nombre
        string tipo
        decimal monto_previsto
        string frecuencia
        date fecha_inicio
        date fecha_fin
        string estado_operativo
        string descripcion
        datetime creado_en
        datetime actualizado_en
    }

    PAGOS_RECURRENTES_OCURRENCIAS {
        int id PK
        int pago_recurrente_id FK
        int periodo_id FK
        int movimiento_id FK
        date fecha_programada
        decimal monto_previsto
        decimal monto_real
        string estado
        string descripcion
        datetime creado_en
        datetime actualizado_en
    }

    PAGOS_VARIABLES {
        int id PK
        int usuario_id FK
        int periodo_id FK
        int cuenta_id FK
        int categoria_id FK
        int subcategoria_id FK
        int movimiento_id FK
        string nombre
        string tipo
        decimal monto_previsto
        decimal monto_real
        date fecha_prevista
        string estado
        string descripcion
        datetime creado_en
        datetime actualizado_en
    }

    SUSCRIPCIONES {
        int id PK
        int usuario_id FK
        int cuenta_id FK
        int categoria_id FK
        string nombre
        decimal monto
        string frecuencia
        date proximo_cobro
        string estado_operativo
        string descripcion
        datetime creado_en
        datetime actualizado_en
    }

    SUSCRIPCIONES_OCURRENCIAS {
        int id PK
        int suscripcion_id FK
        int periodo_id FK
        int movimiento_id FK
        date fecha_cobro
        decimal monto_previsto
        decimal monto_real
        string estado
        string descripcion
        datetime creado_en
        datetime actualizado_en
    }

    METAS {
        int id PK
        int usuario_id FK
        int cuenta_destino_id FK
        string nombre
        string tipo
        decimal monto_objetivo
        decimal monto_actual
        date fecha_limite
        string estado
        string descripcion
        datetime creado_en
        datetime actualizado_en
    }

    APORTES_METAS {
        int id PK
        int meta_id FK
        int movimiento_id FK
        int cuenta_origen_id FK
        int cuenta_destino_id FK
        decimal monto
        date fecha
        string descripcion
        datetime creado_en
        datetime actualizado_en
    }

    PRESTAMOS_COBRAR {
        int id PK
        int usuario_id FK
        int cuenta_origen_id FK
        int cuenta_retorno_id FK
        int movimiento_salida_id FK
        int movimiento_retorno_id FK
        string nombre
        string persona_entidad
        decimal monto
        date fecha_prestamo
        date fecha_devolucion_estimada
        string estado
        string descripcion
        datetime creado_en
        datetime actualizado_en
    }

    DEUDAS_PAGAR {
        int id PK
        int usuario_id FK
        int cuenta_ingreso_id FK
        int cuenta_pago_id FK
        int movimiento_ingreso_id FK
        int movimiento_pago_id FK
        string nombre
        string persona_entidad
        decimal monto
        date fecha_recepcion
        date fecha_pago_estimada
        string estado
        string descripcion
        datetime creado_en
        datetime actualizado_en
    }

    USUARIOS ||--|| CONFIGURACIONES_USUARIO : configura
    USUARIOS ||--o{ PERIODOS_FINANCIEROS : tiene
    USUARIOS ||--o{ CUENTAS : posee
    USUARIOS ||--o{ CATEGORIAS : define
    USUARIOS ||--o{ MOVIMIENTOS : registra
    USUARIOS ||--o{ PAGOS_RECURRENTES : configura
    USUARIOS ||--o{ PAGOS_VARIABLES : planifica
    USUARIOS ||--o{ SUSCRIPCIONES : administra
    USUARIOS ||--o{ METAS : crea
    USUARIOS ||--o{ PRESTAMOS_COBRAR : registra
    USUARIOS ||--o{ DEUDAS_PAGAR : registra

    PERIODOS_FINANCIEROS ||--o{ MOVIMIENTOS : agrupa
    PERIODOS_FINANCIEROS ||--o{ PAGOS_RECURRENTES_OCURRENCIAS : contiene
    PERIODOS_FINANCIEROS ||--o{ PAGOS_VARIABLES : contiene
    PERIODOS_FINANCIEROS ||--o{ SUSCRIPCIONES_OCURRENCIAS : contiene

    CATEGORIAS ||--o{ SUBCATEGORIAS : contiene
    CATEGORIAS ||--o{ MOVIMIENTOS : clasifica
    SUBCATEGORIAS ||--o{ MOVIMIENTOS : detalla

    CUENTAS ||--o{ MOVIMIENTOS : origen
    CUENTAS ||--o{ MOVIMIENTOS : destino
    CUENTAS ||--o{ PAGOS_RECURRENTES : usa
    CUENTAS ||--o{ PAGOS_VARIABLES : usa
    CUENTAS ||--o{ SUSCRIPCIONES : paga
    CUENTAS ||--o{ METAS : guarda

    PAGOS_RECURRENTES ||--o{ PAGOS_RECURRENTES_OCURRENCIAS : genera
    PAGOS_RECURRENTES_OCURRENCIAS ||--o| MOVIMIENTOS : confirma

    PAGOS_VARIABLES ||--o| MOVIMIENTOS : confirma

    SUSCRIPCIONES ||--o{ SUSCRIPCIONES_OCURRENCIAS : genera
    SUSCRIPCIONES_OCURRENCIAS ||--o| MOVIMIENTOS : confirma

    METAS ||--o{ APORTES_METAS : recibe
    APORTES_METAS ||--|| MOVIMIENTOS : registra

    PRESTAMOS_COBRAR ||--o| MOVIMIENTOS : salida
    PRESTAMOS_COBRAR ||--o| MOVIMIENTOS : retorno

    DEUDAS_PAGAR ||--o| MOVIMIENTOS : ingreso
    DEUDAS_PAGAR ||--o| MOVIMIENTOS : pago
```

