# Flujos visuales

Estos diagramas muestran flujos generales del sistema.

## Flujo de movimientos

```mermaid
flowchart TD
    Origen[Modulo origen]
    Validacion[Validar datos]
    Movimiento[Crear o actualizar movimiento]
    Cuenta[Actualizar cuenta si corresponde]
    Historial[Guardar historial]

    Origen --> Validacion
    Validacion --> Movimiento
    Movimiento --> Cuenta
    Movimiento --> Historial
```

## Flujo de cuentas

```mermaid
flowchart TD
    Movimiento[Movimiento confirmado]
    Tipo{Tipo de movimiento}
    Ingreso[Aumenta saldo]
    Egreso[Disminuye saldo]
    Transferencia[Mueve saldo entre cuentas]

    Movimiento --> Tipo
    Tipo --> Ingreso
    Tipo --> Egreso
    Tipo --> Transferencia
```

## Flujo de cierre de mes

```mermaid
flowchart TD
    Inicio[Fin del mes calendario]
    Pendientes[Revisar pendientes]
    Confirmados[Revisar confirmados]
    Real[Calcular resultado real]
    Historico[Guardar vista historica]

    Inicio --> Pendientes
    Inicio --> Confirmados
    Pendientes --> Real
    Confirmados --> Real
    Real --> Historico
```

## Flujo de resultado esperado

```mermaid
flowchart TD
    Inicio[Monto inicial]
    Ingresos[Ingresos previstos]
    Gastos[Gastos previstos]
    Ahorros[Ahorros como transferencias]
    Deudas[Deudas previstas]
    Prestamos[Prestamos previstos]
    Resultado[Sobrante esperado]

    Inicio --> Resultado
    Ingresos --> Resultado
    Gastos --> Resultado
    Ahorros --> Resultado
    Deudas --> Resultado
    Prestamos --> Resultado
```

## Archivos fuente

- [Flujo de movimientos](flujo_movimientos.mmd)
- [Flujo de cuentas](flujo_cuentas.mmd)
- [Flujo de cierre de mes](flujo_cierre_mes.mmd)
- [Flujo de resultado esperado](flujo_resultado_esperado.mmd)

