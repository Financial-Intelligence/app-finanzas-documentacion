# ERD visual

Diagramas actualizados con la especificacion de base de datos/API.

## Diagrama completo recomendado

- [ERD completo del sistema](sistema_completo.md)
- [Fuente Mermaid completa](sistema_completo.mmd)
- [Fuente DBML completa](sistema_completo.dbml)

## Diagrama reducido

```mermaid
erDiagram
    users ||--o{ accounts : owns
    users ||--o{ movements : registers
    accounts ||--o{ movements : affects
    categories ||--o{ movements : classifies
    recurring_payments ||--o{ recurring_occurrences : generates
    recurring_occurrences ||--o| movements : creates
    subscriptions ||--o{ subscription_renewals : generates
    subscription_renewals ||--o| movements : creates
    loans ||--o{ loan_collections : receives
    debts ||--o{ debt_payments : pays
    goals ||--o{ goal_contributions : receives
```

## Archivos fuente

- [Mermaid](erd.mmd)
- [DBML](erd.dbml)
