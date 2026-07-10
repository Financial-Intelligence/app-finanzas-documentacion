# 05 - Decisiones de arquitectura

## Estado del documento

Documento para registrar decisiones tecnicas y conceptuales importantes.

## Objetivo

Evitar que las decisiones se pierdan durante el avance del proyecto.

## Decisiones actuales

- La documentacion se mantiene separada del frontend y backend.
- La documentacion se guarda en un repositorio propio.
- La planificacion mensual no se considera modulo visible independiente.
- La planificacion mensual se trata como reglas internas de calculo mensual.
- El sistema es personal y solo maneja login de acceso.
- Por ahora se manejara una sola moneda comun para todas las cuentas.
- Las cuentas no se eliminaran fisicamente; se inactivaran.
- Las cuentas de credito no sumaran al saldo total.
- Las cuentas de credito se sumaran en un indicador separado de credito disponible.
- La cuenta principal se maneja en backend con `isPrimary`.
- Solo una cuenta principal debe existir por usuario.
- La accion para pagos planificados sera registrar pago, no confirmar recurrente.
- El monto previsto se conservara aunque el monto real registrado sea distinto.

## Decisiones pendientes

- Definir resultado real.
- Definir reglas finales de cierre mensual.
- Definir alcance final de dashboard.
- Definir importacion desde Excel.
- Definir reglas avanzadas de tarjetas de credito.
- Definir si en el futuro se subiran comprobantes como archivos locales o externos.
