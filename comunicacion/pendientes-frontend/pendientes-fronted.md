# Pendiente frontend: Reportes sin datos

Cuando el usuario recién reinició sus datos financieros o todavía no registró
movimientos, la pantalla **Reportes** debe funcionar sin errores.

- Antes de calcular o dibujar un gráfico, convertir cada monto a un número
  seguro. Si llega vacío, `null`, `undefined` o no es numérico, usar `0`.
- No dividir entre cero para calcular porcentajes, escalas ni posiciones.
- Si `charts.trend` tiene puntos sin movimientos, mostrar el gráfico con valores
  en `0`.
- Si `charts.trend` llega vacío, mostrar: **“Aún no hay movimientos para este
  período”**, en lugar del gráfico.
- Validar que las coordenadas SVG (`x`, `y`, `x1`, `y1`, ancho y alto) sean
  números finitos antes de renderizarlas.
- No usar `Math.max(...[])` con arreglos vacíos.
- En la dona de categoría, si no hay gastos, mostrar un estado vacío y no
  calcular porcentajes ni trazos SVG.

Esto evita el error: `Received NaN for the y1 attribute`.
