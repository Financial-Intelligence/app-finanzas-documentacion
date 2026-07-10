# 04 - Diccionario de datos

## Estado del documento

Actualizado parcialmente con base en el backend real.

## Objetivo

Explicar el significado de cada tabla y cada campo.

## Formato sugerido por tabla

| Campo | Tipo | Obligatorio | Descripcion | Reglas |
| --- | --- | --- | --- | --- |
| id | Int | Si | Identificador interno de la cuenta | Autoincremental |
| userId | Int | Si | Usuario propietario | Relaciona cuenta con usuario |
| name | String | Si | Nombre visible de la cuenta | Unico por usuario |
| type | AccountType | Si | Tipo de cuenta | BANK_ACCOUNT, SAVINGS, CASH, DIGITAL_WALLET, CREDIT_CARD, INVESTMENT, OTHER |
| institution | String | No | Banco, billetera o entidad | Ejemplo: BCP, Yape, Interbank |
| currency | String | Si | Moneda de la cuenta | Por ahora se usa moneda comun, normalmente PEN |
| initialBalance | Decimal | Si | Saldo inicial registrado | En cuenta normal inicia el saldo actual |
| currentBalance | Decimal | Si | Saldo actual de la cuenta | En el futuro lo afectaran movimientos confirmados |
| expectedMonthlyAmount | Decimal | Si | Resultado esperado mensual asociado a la cuenta | Sirve para comparar esperado vs real |
| creditLimit | Decimal | No | Limite de credito | Solo aplica a CREDIT_CARD |
| availableCredit | Decimal | No | Credito disponible | Solo aplica a CREDIT_CARD |
| isActive | Boolean | Si | Indica si la cuenta esta activa | Se inactiva en vez de borrar |
| isPrimary | Boolean | Si | Indica si es la cuenta principal del usuario | Solo una cuenta principal por usuario |
| createdAt | DateTime | Si | Fecha de creacion | Automatico |
| updatedAt | DateTime | Si | Fecha de actualizacion | Automatico |
