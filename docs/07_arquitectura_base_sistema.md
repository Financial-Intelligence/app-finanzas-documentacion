# 07 - Arquitectura base del sistema

## Estado del documento

Actualizado con `FINANCE_APP_DATABASE_AND_API_SPEC.md` y el avance real del modulo de cuentas.

## Objetivo

Definir la arquitectura base del sistema para guiar el backend.

## Principio principal

`movements` es la fuente central de verdad financiera.

Todos los modulos que mueven dinero deben crear o relacionarse con un movimiento.

## Capas funcionales

### Acceso

- `users`
- `refresh_tokens`

### Dinero base

- `accounts`
- `movements`

### Clasificacion

- `categories`
- `subcategories`

### Planificacion y compromisos

- `recurring_payments`
- `recurring_occurrences`
- `variable_payments`
- `subscriptions`
- `subscription_renewals`

### Obligaciones y objetivos

- `loans`
- `loan_collections`
- `debts`
- `debt_payments`
- `goals`
- `goal_contributions`

### Analisis mensual

- `monthly_plans`
- `budgets`

## Estado actual de implementacion

### Implementado

- Login base.
- Modulo cuentas base.

### Cuentas implementado

Endpoints:

```text
GET    /api/accounts
GET    /api/accounts/summary
POST   /api/accounts
GET    /api/accounts/:id
PATCH  /api/accounts/:id
PATCH  /api/accounts/:id/status
```

Reglas confirmadas:

- Cuentas por usuario autenticado.
- Creacion activa por defecto.
- Inactivacion en lugar de eliminacion fisica.
- Cuenta principal automatica para la primera cuenta.
- Una sola cuenta principal por usuario.
- Tarjetas de credito separadas del saldo total.

## Modelo de datos objetivo

Ver:

- [Modelo fisico](database/03_modelo_fisico.md)
- [Diccionario de datos](database/04_diccionario_datos.md)
- [ERD completo visual](diagrams/erd/sistema_completo.md)
- [DBML completo](diagrams/erd/sistema_completo.dbml)

## Flujo general

```text
Usuario
  -> Modulo funcional
  -> Validacion de reglas
  -> Tabla especifica del modulo
  -> movements si afecta dinero
  -> accounts si modifica saldo real
  -> dashboard/reportes consultan datos consolidados
```

## Reglas de arquitectura

1. No duplicar movimientos.
2. No calcular saldos desde varias fuentes distintas.
3. No borrar fisicamente informacion critica.
4. Usar `user_id` para separar datos del usuario autenticado.
5. Usar `period` para separar meses.
6. Usar transacciones cuando una accion toque varias tablas.
7. Mantener documentacion actualizada cuando el backend cambie.

