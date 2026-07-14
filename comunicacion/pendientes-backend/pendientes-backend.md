# 09 - Pendientes del backend solicitados por frontend

Este documento reúne lo que el frontend ya tiene implementado y está esperando del backend. El frontend está listo: en cuanto el backend cumpla cada punto, la funcionalidad se enciende sola sin tocar frontend.

Mientras un endpoint o campo no exista, el frontend muestra un estado vacío limpio y nunca inventa datos.

## Estado general

El frontend cubre hoy: autenticación, perfil, y el módulo de Cuentas completo (listado, detalle, crear, editar, inactivar/activar, eliminar, cuenta principal, y el panel de planificación mensual del detalle). El backend cubre autenticación, usuarios y cuentas base.

Los pendientes de abajo son las piezas que faltan para que el módulo de Cuentas quede 100% conectado.

---

## Bloqueantes (frontend hecho, esperando backend)

### 1. GET /api/accounts/:id/summary

No existe (responde 404). Alimenta el panel de planificación del detalle de cuenta (descripciones dinámicas: heredado + ingresos fijos, resultado esperado, cumplimiento del mes y gráfico esperado vs real).

Ruta:

```text
GET /api/accounts/:id/summary?period=YYYY-MM
```

Respuesta esperada (sin envoltorio `{ summary }`, el objeto va directo):

```json
{
  "period": "2026-05",
  "income": 4150,
  "expense": 1245.5,
  "transfersOut": 700,
  "transfersIn": 200,
  "plan": {
    "carriedOver": 2600,
    "recurringIncome": 3500,
    "initialAmount": 6100,
    "expectedResult": 900,
    "realResult": 950,
    "fulfillment": 50,
    "months": [
      { "label": "Abr", "expected": 850, "real": 1100 },
      { "label": "May", "expected": 900, "real": 950 }
    ]
  }
}
```

Significado de cada campo del `plan`:

1. `carriedOver`: saldo heredado del mes anterior.
2. `recurringIncome`: ingresos fijos del mes.
3. `initialAmount`: `carriedOver + recurringIncome`.
4. `expectedResult`: meta de sobrante al cerrar el mes.
5. `realResult`: resultado real del último mes.
6. `fulfillment`: `realResult - expectedResult` (positivo = sobre la meta).
7. `months`: serie esperado vs real para el gráfico.

Este endpoint depende del módulo de movimientos, porque `realResult` y `months` se calculan con movimientos confirmados.

### 2. movementsCount en GET /api/accounts/:id

El frontend oculta el botón Eliminar cuando la cuenta tiene historial. Hoy el detalle no devuelve ningún dato de movimientos, así que el botón siempre aparece.

Con agregar un campo `movementsCount: number` al objeto cuenta del detalle (y opcionalmente en el listado), el frontend oculta el botón solo cuando `movementsCount > 0`.

---

## Módulo grande (desbloquea lo anterior)

### 3. Módulo Movimientos

Es la dependencia real. `realResult`, `months[]` y `movementsCount` se calculan a partir de movimientos confirmados. Sin este módulo, los puntos 1 y 2 no tienen datos reales.

---

## Reglas y consistencia a corregir

### 4. DELETE: físico vs soft delete

El contrato define DELETE como soft delete (`status='archived'`, respuesta `204`, conserva movimientos). El backend actual hace borrado físico con `prisma.account.delete()`.

Cuando existan movimientos, el borrado físico reventará por llave foránea. Decidir: pasar a soft delete o definir `onDelete` explícito en la relación.

### 5. Regla de eliminación por movimientos

El contrato dice que una cuenta solo se puede eliminar si no tiene historial. El backend hoy solo valida que no sea la cuenta principal. Falta agregar la validación de "sin movimientos" cuando exista el módulo.

### 6. Inactivar la cuenta principal deja al usuario sin principal

`updateAccountStatus`, al inactivar, hace `isPrimary: false` sin reasignar otra cuenta. Esto es inconsistente: `delete` bloquea borrar la principal, pero inactivar sí la deja sin ninguna principal.

Decidir: bloquear inactivar la principal (pedir reasignar primero), o auto-reasignar principal a otra cuenta activa.

---

## Limpieza y alineación

### 7. Campo expectedMonthlyAmount huérfano

El frontend ya no envía `expectedMonthlyAmount` (se quitó del formulario de cuenta). Sigue en la base de datos con default 0. Limpiar el campo o darle un uso definido.

### 8. Ambigüedad del contrato: monthlyPlan en dos sitios

El contrato ubica `monthlyPlan` en dos endpoints: `GET /accounts/:id` ("cuenta + monthlyPlan") y `GET /accounts/:id/summary`. El frontend consume `/accounts/:id/summary`.

Confirmar cuál es la fuente oficial para no duplicar la lógica.

---

## Resumen de prioridad

1. Módulo Movimientos (habilita todo lo de cuentas conectado).
2. `GET /accounts/:id/summary` con el shape de arriba.
3. `movementsCount` en el detalle de cuenta.
4. Decisiones de reglas: soft delete, regla por movimientos, inactivar principal.
5. Limpieza: `expectedMonthlyAmount`, alineación de `monthlyPlan`.
