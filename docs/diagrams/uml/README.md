# UML visual

Estos diagramas son versiones iniciales para representar el sistema de forma general.

## Modelo de clases

```mermaid
classDiagram
    class Cuenta {
        +id
        +nombre
    }

    class Movimiento {
        +id
        +monto
        +estado
    }

    Cuenta "1" --> "0..*" Movimiento
```

## Componentes principales

```mermaid
flowchart LR
    Acceso[Acceso personal]
    Cuentas[Cuentas]
    Movimientos[Movimientos]
    Categorias[Categorias]
    Recurrentes[Pagos recurrentes]
    Variables[Pagos variables]
    Suscripciones[Suscripciones]
    Metas[Metas]
    Prestamos[Prestamos por cobrar]
    Deudas[Deudas por pagar]
    Dashboard[Dashboard y analisis]

    Acceso --> Cuentas
    Cuentas --> Movimientos
    Categorias --> Movimientos
    Recurrentes --> Movimientos
    Variables --> Movimientos
    Suscripciones --> Movimientos
    Metas --> Movimientos
    Prestamos --> Movimientos
    Deudas --> Movimientos
    Movimientos --> Dashboard
```

## Archivos fuente

- [Modelo de clases Mermaid](modelo_clases.mmd)
- [Modelo de clases PlantUML](modelo_clases.puml)
- [Componentes Mermaid](componentes_modulos.mmd)
- [Componentes PlantUML](componentes_modulos.puml)
- [Secuencias visuales](secuencias/README.md)

