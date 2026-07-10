# Flujos visuales

Diagramas actualizados con el avance de cuentas y la especificacion de movimientos.

## Flujo actual de cuentas

```mermaid
flowchart TD
    Usuario[Usuario autenticado]
    API[API cuentas]
    Validacion[Validar datos]
    Accounts[accounts]
    Summary[Resumen de cuentas]

    Usuario --> API
    API --> Validacion
    Validacion --> Accounts
    Accounts --> Summary
```

## Flujo futuro de movimientos

```mermaid
flowchart TD
    Origen[Modulo origen]
    Movement[movements]
    Estado{Estado}
    Proyeccion[Participa en proyeccion]
    Saldo[Actualiza saldo real]
    Dashboard[Dashboard y reportes]

    Origen --> Movement
    Movement --> Estado
    Estado -->|pending| Proyeccion
    Estado -->|confirmed| Saldo
    Movement --> Dashboard
```

## Flujo futuro de registrar pago recurrente

```mermaid
flowchart TD
    Recurrente[recurring_payments]
    Ocurrencia[recurring_occurrences]
    Formulario[Formulario con monto previsto]
    Real[Monto real registrado]
    Movimiento[movements]
    Cuenta[accounts]

    Recurrente --> Ocurrencia
    Ocurrencia --> Formulario
    Formulario --> Real
    Real --> Movimiento
    Movimiento --> Cuenta
```

## Archivos fuente

- [Flujo de movimientos](flujo_movimientos.mmd)
- [Flujo de cuentas](flujo_cuentas.mmd)
- [Flujo de cierre de mes](flujo_cierre_mes.mmd)
- [Flujo de resultado esperado](flujo_resultado_esperado.mmd)

