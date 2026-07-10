# 03 - Modelo fisico

## Estado del documento

Actualizado parcialmente con la tabla real `Account` del backend.

## Tabla Account

| Columna | Tipo | Reglas |
| --- | --- | --- |
| id | Int | PK autoincremental |
| userId | Int | FK hacia User |
| name | VarChar(120) | Unico junto con userId |
| type | AccountType | BANK_ACCOUNT, SAVINGS, CASH, DIGITAL_WALLET, CREDIT_CARD, INVESTMENT, OTHER |
| institution | VarChar(120), null | Banco o entidad |
| currency | VarChar(3) | Default PEN |
| initialBalance | Decimal(14,2) | Default 0 |
| currentBalance | Decimal(14,2) | Default 0 |
| expectedMonthlyAmount | Decimal(14,2) | Default 0, reservado para calculos mensuales futuros |
| creditLimit | Decimal(14,2), null | Solo credito |
| availableCredit | Decimal(14,2), null | Solo credito |
| isActive | Boolean | Default true |
| isPrimary | Boolean | Default false |
| createdAt | DateTime | Default now |
| updatedAt | DateTime | updatedAt automatico |

Indices y restricciones:

1. `@@unique([userId, name])`.
2. `@@index([userId])`.
3. `@@index([userId, isActive])`.
4. `@@index([userId, isPrimary])`.
5. `@@index([userId, type])`.
6. Indice unico parcial SQL: una sola cuenta con `isPrimary = true` por usuario.