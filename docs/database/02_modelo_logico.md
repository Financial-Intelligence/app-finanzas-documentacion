# 02 - Modelo logico

## Estado del documento

Actualizado parcialmente con el modulo real de cuentas del backend.

## Cuenta

Entidad que representa un lugar donde el usuario guarda o mueve dinero.

Atributos principales:

1. `id`: identificador interno.
2. `userId`: usuario propietario.
3. `name`: nombre visible unico por usuario.
4. `type`: tipo de cuenta.
5. `institution`: banco, billetera o entidad.
6. `currency`: moneda.
7. `initialBalance`: saldo inicial.
8. `currentBalance`: saldo actual.
9. `expectedMonthlyAmount`: campo reservado para calculos mensuales futuros.
10. `creditLimit`: limite de credito si aplica.
11. `availableCredit`: credito disponible si aplica.
12. `isActive`: indica si la cuenta esta activa.
13. `isPrimary`: indica si es la cuenta principal.

Reglas logicas:

1. Una cuenta pertenece a un usuario.
2. Un usuario puede tener muchas cuentas.
3. El nombre de cuenta es unico por usuario.
4. La primera cuenta del usuario se marca como principal automaticamente.
5. Solo una cuenta por usuario debe quedar como principal.
6. Una cuenta inactiva no debe quedar como principal.
7. Las cuentas `CREDIT_CARD` no suman al saldo total.
8. Las cuentas `CREDIT_CARD` suman al credito disponible.