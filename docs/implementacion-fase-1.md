# Implementación fase 1: finanzas e inversiones en Notion

Fecha: 24/09/2026. La auditoría está en `auditoria-finanzas-notion.md` y los respaldos del estado anterior están en `respaldos/`.

No se borró ninguna propiedad, fila, relación, vista anterior, embed ni subpágina. Lo que ya no se usa quedó en la sección de administración de cada página.

## 1. Qué cambió

### Finanzas (💰 Mis finanzas personales)

**Mis movimientos**
- Nuevo tipo **Transferencia**.
- Nuevas columnas **Capital**, **Intereses** y **Seguros y otros**.
- Nuevas categorías "Aporte de mi esposa" y "Costos financieros".
- Fórmulas corregidas: Entra, Sale, Solo gastos y Abono.
- Fórmulas nuevas: Solo ingresos, Pago a deudas, Cargo a tarjeta y Pagado en el mes actual.
- Nueva validación **Revisar**, que muestra ✅ o ⚠️ con el motivo.

**Mis meses**
- Nuevos rollups de Ingresos y Pagado en deudas.
- "Balance" se renombró a **Disponible**.
- Nuevos textos KPI: 💰 Ingresos, 💸 Gastos, 💳 Pagado en deudas y ✅ Disponible.
- "Presupuesto usado" ahora es una barra.

**Mis deudas y pagos fijos** (antes "Mis deudas y préstamos")
- Nuevo tipo **Pago fijo** y nueva fila **Arriendo**.
- Nueva casilla **Finalizado**.
- Nuevos rollups: Cargos (compras con tarjeta) y Pagado este mes.
- Nuevas fórmulas: Saldo pendiente, Pendiente este mes, Próxima fecha y Progreso.
- Nuevo **Semáforo** en texto: 🟢 Al día, 🟡 Vence en N días, 🔴 Vencido hace N días, ✅ Pagada.
- Nuevos textos 💰 Saldo, 📌 Cuota y ⏳ Por pagar este mes.

**Página principal**, en este orden:
1. 📊 Este mes (tarjeta con los KPI).
2. 🧾 Obligaciones por pagar este mes (gráfico de número).
3. ⚡ Registrar (ingreso, gasto y pago de deuda).
4. 📅 Próximos pagos.
5. 🔁 Movimientos, con las pestañas Este mes, Gastos, Ingresos, Pagos de deuda, Historial y Revisar.
6. Mi plan y 📘 Cómo registrar.
7. ⚙️ Administración, con las bases completas y el tablero HTML anterior.

### Inversiones (💼 Mi portafolio de inversiones)

**Movimientos de inversión** (antes "Movimientos")
- Nuevo tipo **Reinversión**.
- Nuevas columnas: Activo, Unidades, **Pagado a mi banco** y Moneda de la inversión.
- Monto en COP corregido.
- Nuevos indicadores: Es aporte, Es retiro, Es rendimiento, Aporte COP, Salidas COP y Revisar.

**Inversiones**
- Nuevos rollups: Capital aportado, Retirado, Dividendos e intereses, Cobrado a mi banco, Aportado en COP y Salidas en COP.
- Nuevos cálculos: Ganancia, Rentabilidad, Ganancia en COP y Rentabilidad en COP.
- Nuevos textos: 💵 Aportado, 📈 Valor actual, 🪙 Dividendos, ↩️ Retirado, ± Ganancia y 🇨🇴 En pesos.
- La columna manual "Aportado" se conservó como "Aportado (manual, ya no se usa)".

**Nueva base 📸 Historial de valor**
- Guarda una foto mensual: Valor, TRM, Aportado a la fecha y Retirado a la fecha.
- Calcula Ganancia, Rentabilidad, Valor en COP y sus textos.

**Aportes mensuales ARQ**
- Nueva relación **Movimiento**, que lo conecta con Movimientos de inversión.

**Página principal**, en este orden:
1. 📈 Cómo va mi portafolio (tarjetas KPI y TRM).
2. ⚡ Registrar (aporte, dividendo y retiro).
3. 🔁 Movimientos, con las pestañas Aportes, Dividendos, Retiros e Historial.
4. 📸 Evolución.
5. 🎯 Mi plan.
6. 📐 Cómo decido (sin cambios).
7. 🗄️ Administración.

La página usa colores verde y morado, distintos de los de finanzas.

## 2. Por qué y qué problema resolvió cada cambio

| Cambio | Problema que resolvió |
|---|---|
| Compra con tarjeta = gasto, pero no resta del disponible. El pago de la tarjeta = pago de deuda | Antes, pagar la tarjeta contaba como un segundo gasto (**doble conteo**) |
| Tipo Transferencia | Mover plata entre mis cuentas se veía como gasto o como ingreso |
| Capital / Intereses / Seguros | Toda la cuota bajaba el saldo de la deuda, aunque los intereses y seguros no lo bajan |
| Cargos de tarjeta suben el saldo | El saldo de la tarjeta no cambiaba al usarla |
| Pagado este mes, Pendiente este mes y Semáforo | Había que actualizar a mano "Estado" y "Próximo pago" |
| Pago fijo (Arriendo) | El arriendo no aparecía entre mis obligaciones del mes |
| Revisar (✅/⚠️) | No había aviso cuando faltaba el mes, la deuda o el medio de pago |
| Rollups de aportes y retiros | "Aportado" era un número manual que se desactualizaba |
| Ganancia = valor + retirado + cobrado − aportado | Un retiro parecía pérdida y los dividendos se mezclaban con los aportes |
| Pagado a mi banco | No se distinguía el dividendo que se quedó invertido del que llegó a mi cuenta |
| COP con la TRM histórica de cada aporte | Se mezclaban USD y COP, o se usaba una sola TRM para todo |
| Historial de valor | Al actualizar el valor actual se perdía el pasado |
| Vistas con pocas columnas y bases completas escondidas | Las páginas eran anchas y difíciles de usar en el celular |
| Vistas filtradas por tipo y mes | Registrar un movimiento pedía llenar Tipo y Mes a mano (ahora se llenan solos) |

## 3. Qué NO se implementó (a propósito)

- Patrimonio neto, activos personales y contabilidad completa (fuera del alcance).
- Borrar las columnas viejas (Estado, Próximo pago, Avance, Faltan y Aportado manual). Se conservan porque tienen datos.
- Botones nativos y plantillas de base de datos: la API no permite crearlos. Los pasos para hacerlos a mano están en "Cómo registrar".
- Calendario por "Próxima fecha": la API no deja crear calendarios sobre fórmulas. Sigue existiendo el calendario anterior en Administración.
- Filtro automático "este mes": el lenguaje de vistas de la API no acepta fechas relativas ni filtros por fórmula. Por eso las pestañas del mes usan un filtro fijo que se cambia el día 1.
- Más gráficos: Notion gratis limita los gráficos. Solo se creó uno, el de obligaciones por pagar.
- Automatizaciones de base de datos: son de pago.

## 4. Ideas para la fase 2

- Si pasas a un plan de pago: automatizaciones para crear el mes nuevo, la foto mensual del portafolio y filtros "este mes" automáticos.
- Una rutina con Claude o Cowork el día 1: cambiar los filtros de mes, crear el mes siguiente y tomar la foto del portafolio.
- Presupuesto por categoría (una base pequeña de límites mensuales).
- Actualizar la TRM y el valor del portafolio con un script externo, solo si algún día vale la pena.
- Importar extractos bancarios en CSV (sin sincronización bancaria).

## 5. Limitaciones de Notion gratis (y de la API) encontradas

- Una fórmula no puede leer propiedades de una página relacionada. Solo pueden hacerlo los rollups.
- Las vistas creadas por API no pueden filtrar por fórmulas ni por fechas relativas ("este mes").
- No se pueden crear botones, plantillas ni calendarios sobre fórmulas por API.
- Los gráficos son limitados en el plan gratis. Si el gráfico de obligaciones aparece bloqueado, bórralo: la lista 📅 Próximos pagos muestra lo mismo por deuda.
- Los resultados de las fórmulas no se pueden leer por API. Se validaron la sintaxis y los tipos, no los valores mostrados.

## Mantenimiento mensual (2 minutos)

1. **Día 1:** cambiar el filtro **Mes** en las pestañas 📅 Este mes, 💸 Gastos, 💰 Ingresos y 💳 Pagos de deuda, y el nombre del mes en 📊 Este mes.
2. **Último día:** actualizar el valor actual y la TRM de cada inversión, y crear la foto en 📸 Evolución.
3. **Cuando se acaben los meses creados:** agregar el siguiente mes en Mis meses.
