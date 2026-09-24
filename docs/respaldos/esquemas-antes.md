# Esquemas y datos antes de los cambios (24/09/2026)

Los textos de las fórmulas no son legibles por la API; se listan solo sus nombres.

## Mis movimientos — collection://42cfb3a7-f88a-47f0-be6f-27cf648abb27
Propiedades: Concepto (título), Fecha, Tipo (Ingreso, Gasto, Pago de deuda, Ahorro e inversión, Préstamo que recibí, Préstamo que hice, Cobro de préstamo), Monto (COP), Categoría (25 opciones), Detalle (24 opciones), Medio de pago (Efectivo, Tarjeta débito, Tarjeta de crédito, Nequi, Daviplata, Transferencia, Bre-B), ¿Qué tan necesario?, Soporte, Notas, Mes (→ Mis meses), Deuda o préstamo (→ Deudas). Fórmulas: Entra, Sale, Solo gastos, Abono.

Filas:
| Concepto | Fecha | Tipo | Monto | Categoría | Detalle | Necesario | Mes |
|---|---|---|---|---|---|---|---|
| Arriendo | 2026-09-15 | Gasto | 1.450.000 | Hogar | Arriendo | Necesario | Septiembre 2026 |

## Mis meses — collection://edce4fe8-200d-437d-a12e-8ee3b33e90e7
Propiedades: Mes (título), Inicio del mes, Ingreso esperado, Presupuesto de gastos, Cómo me fue, Movimientos. Rollups: Entró (Entra), Salió (Sale), Gastos (Solo gastos). Fórmulas: Balance, Presupuesto usado.
Filas: Septiembre 2026 (2026-09-01), Octubre 2026, Noviembre 2026, Diciembre 2026. Sin ingreso esperado ni presupuesto.

## Mis deudas y préstamos — collection://d206714a-040e-4677-9d3a-3638da7b1601
Propiedades: Nombre, Entidad o persona, Dirección, Tipo, Saldo al empezar, Cuota mensual, Tasa E.A., Día de pago, Próximo pago, Estado, Cuotas pendientes al empezar, Notas, Movimientos. Rollup: Abonado (Abono). Fórmulas: Saldo pendiente, Avance, Faltan.

| Nombre | Tipo | Saldo al empezar | Cuota | Tasa E.A. | Día | Próximo pago | Estado |
|---|---|---|---|---|---|---|---|
| American Express (pesos) | Tarjeta de crédito | 4.114.775 | 323.108 | 29,2215 % | 5 | 2026-10-05 | Al día |
| American Express (dólares) | Tarjeta de crédito | 64.173 | 64.173 | 29,2215 % | 5 | 2026-10-05 | Al día |
| Crédito del carro | Crédito de vehículo | 36.402.260 | 1.435.961 | 15,8 % | 11 | 2026-10-11 | Al día (36 cuotas) |
| Tarjeta Visa | Tarjeta de crédito | 1.805.170 | 1.184.291 | 0 % | 5 | 2026-10-05 | Al día |

## Inversiones — collection://dedb856b-ef51-4a57-b8af-87499485ba86
Propiedades: Inversión, Aportado, Valor actual, Moneda, TRM referencia, Estado, Tipo, Plataforma, Riesgo, Liquidez, Rentabilidad esperada, Fecha de inicio, Vencimiento, Última actualización, Página del plan, Notas, Movimientos. Fórmulas: Valor en COP, Ganancia, Rentabilidad.
Fila: "ARQ · ETFs de EE. UU. (VOO, QQQM, SMH)" — Planeada, USD, Aportado 0, Valor actual 0, Rentabilidad esperada 9,3 %, inicio 2026-10-01.

## Movimientos (inversión) — collection://6d1480f1-d7ab-4f93-8fb1-90f3c5a97e6b
Propiedades: Movimiento, Fecha, Tipo (Aporte, Retiro, Dividendo, Interés, Comisión, Ajuste de valor), Monto, Moneda, TRM, Notas, Inversión. Fórmula: Monto en COP. Sin filas.

## Aportes mensuales ARQ — collection://60cf4a43-dddf-4d4c-bbeb-fdb9418d5790
12 filas pendientes (2026-10 a 2027-09), Meta 300 USD cada una.

## Mis revisiones trimestrales — collection://54a99d5a-0ca3-4d16-a7f4-cb95fee7db13
Sin filas.
