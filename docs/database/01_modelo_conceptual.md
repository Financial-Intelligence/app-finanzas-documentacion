# 01 - Modelo conceptual

## Estado del documento

Actualizado con la especificacion de base de datos/API.

## Objetivo

Representar las entidades principales del negocio en lenguaje simple.

## Entidades principales

| Entidad | Significado |
| --- | --- |
| Usuario | Persona que usa el sistema y posee sus datos financieros. |
| Cuenta | Lugar donde existe o se mueve dinero. |
| Categoria | Clasificacion de ingresos o egresos. |
| Subcategoria | Detalle dentro de una categoria. |
| Movimiento | Registro central de ingreso, egreso o transferencia. |
| Pago recurrente | Ingreso o egreso que se repite. |
| Ocurrencia recurrente | Aparicion de un recurrente en un periodo. |
| Pago variable | Ingreso o egreso planificado solo para un periodo. |
| Suscripcion | Servicio con renovacion periodica. |
| Renovacion de suscripcion | Pago o renovacion de una suscripcion en un periodo. |
| Prestamo por cobrar | Dinero prestado a terceros. |
| Cobro de prestamo | Registro de dinero recuperado. |
| Deuda por pagar | Dinero recibido que debe devolverse. |
| Pago de deuda | Registro de pago de una deuda. |
| Meta | Objetivo financiero con monto y fecha. |
| Aporte a meta | Dinero aportado a una meta. |
| Presupuesto | Limite mensual general o por categoria. |
| Plan mensual | Snapshot de planificacion por periodo y cuenta. |
| Refresh token | Registro tecnico para mantener sesion. |

## Concepto central

El movimiento es la entidad que une el sistema.

Todo modulo que afecte dinero debe terminar generando o relacionandose con un movimiento.

