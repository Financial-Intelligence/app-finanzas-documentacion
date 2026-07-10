# Secuencias visuales

Estos diagramas muestran flujos iniciales entre usuario, modulos y movimientos.

## Registrar pago recurrente

```mermaid
sequenceDiagram
    participant Usuario
    participant Recurrentes as Pagos recurrentes
    participant Movimientos
    participant Cuentas

    Usuario->>Recurrentes: Registra pago
    Recurrentes->>Movimientos: Registra movimiento confirmado
    Movimientos->>Cuentas: Actualiza saldo relacionado
```

## Registrar prestamo

```mermaid
sequenceDiagram
    participant Usuario
    participant Prestamos
    participant Movimientos
    participant Cuentas

    Usuario->>Prestamos: Registra prestamo
    Prestamos->>Movimientos: Crea salida de dinero
    Movimientos->>Cuentas: Descuenta saldo de cuenta origen
```

## Registrar deuda

```mermaid
sequenceDiagram
    participant Usuario
    participant Deudas
    participant Movimientos
    participant Cuentas

    Usuario->>Deudas: Registra deuda recibida
    Deudas->>Movimientos: Crea ingreso de dinero
    Movimientos->>Cuentas: Aumenta saldo de cuenta de ingreso
```

## Archivos fuente

- [Registrar pago recurrente Mermaid](confirmar_pago_recurrente.mmd)
- [Registrar pago recurrente PlantUML](confirmar_pago_recurrente.puml)
- [Registrar prestamo Mermaid](registrar_prestamo.mmd)
- [Registrar prestamo PlantUML](registrar_prestamo.puml)
- [Registrar deuda Mermaid](registrar_deuda.mmd)
- [Registrar deuda PlantUML](registrar_deuda.puml)
