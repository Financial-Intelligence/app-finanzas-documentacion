# UML visual

Diagramas actualizados con los modulos definidos en la especificacion.

## Componentes principales

```mermaid
flowchart LR
    Auth[Acceso personal]
    Accounts[Cuentas]
    Categories[Categorias]
    Movements[Movimientos]
    Recurring[Pagos recurrentes]
    Variables[Pagos variables]
    Subscriptions[Suscripciones]
    Loans[Prestamos]
    Debts[Deudas]
    Goals[Metas]
    Budgets[Presupuestos]
    MonthlyPlans[Planes mensuales]
    Dashboard[Dashboard y reportes]

    Auth --> Accounts
    Auth --> Categories
    Auth --> Movements
    Accounts --> Movements
    Categories --> Movements
    Recurring --> Movements
    Variables --> Movements
    Subscriptions --> Movements
    Loans --> Movements
    Debts --> Movements
    Goals --> Movements
    Categories --> Budgets
    Accounts --> MonthlyPlans
    Movements --> MonthlyPlans
    Movements --> Dashboard
    Budgets --> Dashboard
    MonthlyPlans --> Dashboard
```

## Modelo simple de clases

```mermaid
classDiagram
    class Account {
        +id
        +name
        +type
        +currentBalance
        +status
    }

    class Movement {
        +id
        +type
        +status
        +amount
        +period
        +sourceType
    }

    class Category {
        +id
        +type
        +name
    }

    Account "1" --> "0..*" Movement
    Category "1" --> "0..*" Movement
```

## Archivos fuente

- [Modelo de clases Mermaid](modelo_clases.mmd)
- [Modelo de clases PlantUML](modelo_clases.puml)
- [Componentes Mermaid](componentes_modulos.mmd)
- [Componentes PlantUML](componentes_modulos.puml)
- [Secuencias visuales](secuencias/README.md)

