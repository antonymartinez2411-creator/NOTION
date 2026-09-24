# Auditoría del sistema de finanzas personales en Notion

**Fecha:** 24 de septiembre de 2026
**Alcance:** solo lectura. No se modificó nada en Notion.
**Qué revisé:** las páginas *Mis finanzas personales*, *Mi plan de finanzas*, *Mi portafolio de inversiones* y *Mi plan de inversión en ARQ*, y la página antigua *Presupuesto mensual*. También las 9 bases de datos (esquema, relaciones, rollups, vistas y filas), el código HTML de los 3 tableros embebidos y la descripción de las automatizaciones de Cowork.

**Límites del análisis (importante):**

- La API de Notion **no expone el texto de las fórmulas** ni sus resultados. Solo da una referencia opaca a cada una. Lo que digo de `Entra`, `Sale`, `Solo gastos`, `Abono`, `Balance`, `Presupuesto usado`, `Saldo pendiente`, `Avance`, `Faltan`, `Valor en COP`, `Ganancia`, `Rentabilidad`, `Monto en COP`, `Invertido neto`, `Acciones` y `Avance meta` sale de sus nombres, del código de los tableros (que sí pude leer) y de las notas de las páginas. Donde algo depende del texto exacto de la fórmula, lo marco con **(verificar)**.
- Las tareas programadas de Cowork no viven en Notion. Las conozco solo por la tabla "Lo que tengo automatizado" de la página de ARQ.
- El sistema es muy nuevo: hay **1 movimiento** (arriendo de septiembre), **4 meses** creados (sep–dic 2026), **4 deudas**, **1 inversión** en estado *Planeada* con valor 0, **12 aportes de ARQ** pendientes y **0 movimientos de inversión**. Por eso casi ningún problema se ve todavía en los números. **Es el mejor momento para corregir el diseño**: antes del primer aporte a ARQ (1 de octubre) y antes de cerrar el primer mes completo.

---

## 0. Resumen ejecutivo

**Lo que está bien:**

- Hay una sola tabla de movimientos, con un campo `Tipo` que ya separa ingreso, gasto, pago de deuda, ahorro/inversión y préstamos. Es la base correcta.
- Las deudas se calculan desde el registro de movimientos (saldo inicial menos abonos), no a mano.
- En inversiones ya existen los tipos *Aporte, Retiro, Dividendo, Interés, Comisión*, y la TRM se guarda en cada operación.
- *Aportes mensuales ARQ* y *Mis revisiones trimestrales* guardan fotos con fecha (valor del portafolio, TRM). Ese es justo el patrón que debería usar todo el sistema.

**Los 5 problemas que más pesan:**

1. **Las compras con tarjeta de crédito se cuentan dos veces como salida.** Una vez como *Gasto* al comprar y otra como *Pago de deuda* al pagar la tarjeta. Además, esas compras no suben el saldo de la tarjeta en *Mis deudas*.
2. **La cuota del carro y la de la Amex no son todo capital.** Si `Abono` suma el monto completo de la cuota, el saldo pendiente baja más rápido que en la realidad. En el carro serían unos **$626.000 de más cada mes**, unos $7,5 millones de error en un año. Los intereses y seguros son gasto; solo el capital reduce la deuda **(verificar)**.
3. **No existe una entidad "Cuenta"** (Bancolombia, Nequi, Daviplata, efectivo, saldo en USDc de ARQ). Sin ella no hay dónde anotar transferencias entre tus propias cuentas, no puedes cuadrar saldos y no se puede calcular el patrimonio neto.
4. **No se calcula el patrimonio neto.** El KPI que dice "PATRIMONIO" en *Mi tablero* es en realidad el **valor del portafolio de inversiones**. No descuenta los ~$42,4 millones de deuda ni suma tu liquidez.
5. **No hay históricos congelados.** `Valor actual`, `TRM referencia`, el estado de las deudas y los datos de los tableros se sobrescriben. Las cifras de meses pasados salen de rollups en vivo que cambian si editas un movimiento viejo. Hoy no podrías reconstruir tu patrimonio de un mes anterior.

**Recomendación central:** mantener Notion como lugar de **registro y consulta**, y mover los **cálculos que deben ser exactos** a un script pequeño y determinista en este repositorio: cierre mensual, conversión de monedas, rentabilidad, rebalanceo y validaciones. Hoy esos cálculos los hace Claude en cada ejecución y los escribe en el JSON de los tableros. Claude debe seguir siendo la interfaz conversacional, pero no la calculadora.

---

## 1. Arquitectura actual detectada

### 1.1 Mapa

```
MIS FINANZAS PERSONALES
┌──────────────────────┐   Mes (relación manual)   ┌────────────────────────┐
│  Mis movimientos     │──────────────────────────▶│  Mis meses             │
│  (registro de caja)  │                           │  rollups: Entró, Salió,│
│  fórmulas: Entra,    │                           │  Gastos; fórmulas:     │
│  Sale, Solo gastos,  │                           │  Balance, Presupuesto  │
│  Abono               │                           │  usado                 │
│                      │   Deuda o préstamo        ┌────────────────────────┐
│                      │──────────────────────────▶│  Mis deudas y préstamos│
└──────────────────────┘                           │  rollup: Abonado;      │
                                                   │  fórmulas: Saldo pend.,│
                                                   │  Avance, Faltan        │
                                                   └────────────────────────┘
  Tableros HTML: "Mi mes en números", "Mi plan en números" (JSON escrito por Claude)
  Página "Mi plan de finanzas": tablas estáticas de ingresos, fijos y deudas

MI PORTAFOLIO DE INVERSIONES
┌──────────────────────┐   Movimientos (relación,  ┌────────────────────────┐
│  Inversiones         │◀── SIN rollups) ─────────│  Movimientos (inv.)    │
│  Aportado y Valor    │                           │  Aporte/Retiro/Divid./ │
│  actual A MANO;      │                           │  Interés/Comisión/     │
│  fórmulas: Valor en  │                           │  Ajuste; Monto en COP  │
│  COP, Ganancia,      │                           └────────────────────────┘
│  Rentabilidad        │
└──────────────────────┘   (sin relación)          ┌────────────────────────┐
                                                   │  Aportes mensuales ARQ │ ← isla
                                                   │  Pesos, TRM, USDc,     │
                                                   │  comisión, precio,     │
                                                   │  valor del portafolio  │
                                                   └────────────────────────┘
                                                   ┌────────────────────────┐
                                                   │  Revisiones trimestral.│ ← isla (bien)
                                                   └────────────────────────┘
  Tablero HTML "Mi tablero" (JSON escrito por Claude) + widget TRM

RESTO
  "Presupuesto mensual" (plantilla de 2025, tablas Ingresos/Gastos en formato dólar), sin uso
```

**Entre las dos áreas no hay ninguna relación.** Una recarga a ARQ anotada como "Ahorro e inversión" en *Mis movimientos* no se conecta con el *Aporte* de *Movimientos (inv.)* ni con la fila de *Aportes mensuales ARQ*.

### 1.2 Inventario de bases de datos

| Base | Filas hoy | Propiedades clave | Relaciones | Rollups / fórmulas |
|---|---|---|---|---|
| Mis movimientos | 1 | Concepto, Fecha, Monto (COP), Tipo (7), Categoría (25), Detalle (24), Medio de pago (7), ¿Qué tan necesario?, Soporte, Notas | Mes → Mis meses; Deuda o préstamo → Deudas | Entra, Sale, Solo gastos, Abono |
| Mis meses | 4 | Mes, Inicio del mes, Ingreso esperado, Presupuesto de gastos, Cómo me fue | Movimientos | Entró, Salió, Gastos (rollups); Balance, Presupuesto usado |
| Mis deudas y préstamos | 4 | Dirección, Tipo, Saldo al empezar, Cuota mensual, Tasa E.A., Día de pago, Próximo pago, Estado (manual), Cuotas pendientes al empezar | Movimientos | Abonado (rollup de `Abono`); Saldo pendiente, Avance, Faltan |
| Inversiones | 1 | Aportado (manual), Valor actual (manual), Moneda, TRM referencia, Estado, Tipo, Plataforma, Riesgo, Liquidez, Rentabilidad esperada, Fechas | Movimientos (inv.) | Valor en COP, Ganancia, Rentabilidad (**sin rollups**) |
| Movimientos (inv.) | 0 | Movimiento, Fecha, Tipo (6), Monto, Moneda, TRM | Inversión | Monto en COP |
| Aportes mensuales ARQ | 12 | Mes, Fecha, Estado, Meta (USD), Pesos (COP), TRM, USDc recibidos, Comisión ARQ, ETF sugerido/comprado, Precio por acción, Valor del portafolio | — | Invertido neto, Acciones, Avance meta |
| Mis revisiones trimestrales | 0 | Revisión, Fecha, Decisión, Resumen, TRM, Valor del portafolio (COP) | — | — |
| Ingresos / Gastos (mensuales) | ? | Nombre, Monto (formato **USD**) | — | — (plantilla vieja) |

### 1.3 Flujo de información y automatizaciones (según la página de ARQ)

| Cuándo | Qué hace Claude/Cowork | Dónde escribe |
|---|---|---|
| Día 1, 8:00 | Busca la TRM y los precios de VOO/QQQM/SMH, calcula el ETF más atrasado y pregunta cuánto invertir | Aportes ARQ (TRM, ETF sugerido) |
| Cuando respondes | Convierte pesos a USDc, calcula comisión y acciones y registra la compra | Aportes ARQ + "mi portafolio" |
| Día 5, 9:00 | Recordatorio si el mes sigue *Pendiente* | — |
| Cada lunes | Actualiza la TRM y el valor estimado de ARQ | Inversiones (sobrescribe) + JSON de *Mi tablero* |
| Trimestral (día 2) | Revisión con datos | Revisiones trimestrales |
| Al registrar movimientos (inferido) | Regenera los JSON de *Mi mes* y *Mi plan* | Adjuntos HTML |

Los tableros son archivos HTML adjuntos. Cada actualización reemplaza su bloque `data` completo. En ese reemplazo **Claude recalcula por su cuenta** los KPIs (entró, salió, deuda, patrimonio, ganancia). Esos cálculos no son las fórmulas de Notion, así que existen **dos fuentes de verdad para el mismo número**.

---

## 2. Evaluación punto por punto

### 1) ¿Datos duplicados o propiedades innecesarias? — ⚠️ Sí, varios

- **Un aporte a ARQ se anota en hasta 4 lugares:** *Mis movimientos* (Tipo "Ahorro e inversión"), *Movimientos (inv.)* (Aporte), *Aportes mensuales ARQ* (Pesos/USDc) y *Inversiones.Aportado* (número manual). Nada obliga a que coincidan.
- **La TRM está en 5 lugares:** Inversiones.TRM referencia, Movimientos (inv.).TRM, Aportes ARQ.TRM, Revisiones.TRM y el JSON de los tableros. La TRM de cada operación **sí debe guardarse**, porque es histórica y la necesitas para la declaración de renta. La "TRM referencia" que se sobrescribe cada lunes es la que sobra, porque es un dato global y no de cada fila.
- **El mismo hecho se codifica tres veces** en Tipo, Categoría y Detalle. Ejemplo: Tipo "Pago de deuda", Categoría "Deudas", Detalle "Cuota" o "Abono a capital". Lo mismo con "Ahorro"/"Inversión"/"Préstamos". Cuando no coincidan (y pasará), cada vista y cada rollup dará un número distinto.
- **`Categoría` mezcla ingresos (UTS, UDI, Clases) con gastos**, y `Detalle` mezcla Salario/Honorarios con Arriendo/Netflix. Falta "Aporte de mi esposa": el tablero lo tiene como color, pero no es una opción de Categoría, así que caería en "Otros ingresos".
- **`Medio de pago` mezcla conceptos distintos:** métodos (Transferencia, Bre-B), cuentas (Nequi, Daviplata, Efectivo) y un instrumento de deuda genérico ("Tarjeta de crédito", ¿Visa o Amex?). Está haciendo el trabajo de una base *Cuentas* que no existe.
- **Las deudas están copiadas a mano en otros sitios:** las tablas estáticas de *Mi plan de finanzas* (cuotas, tasas y el total de mínimos, $1.571.572 = 323.108 + 64.173 + 1.184.291) y el JSON de dos tableros. Esas copias se van a desactualizar.
- **`Estado` de deudas (Al día/Atrasada) es manual** aunque se puede deducir de `Próximo pago` y de si hay un pago registrado en el mes.
- **La página "Presupuesto mensual" es de 2025** y usa tablas en formato dólar. Es un residuo: conviene archivarla para que ni tú ni Claude la confundan con la real.
- **Hay dos bases llamadas casi igual:** "Mis movimientos" y "Movimientos". Es un riesgo real de que una automatización escriba en la equivocada.

### 2) ¿Relaciones bien diseñadas? — ⚠️ Parcialmente

- ✅ Movimientos → Deudas, con rollup de abonos, es el patrón correcto (el saldo se deriva del registro).
- ⚠️ **Movimientos → Mes es una relación manual.** Si falta, el movimiento no suma en ningún mes. Si está mal (fecha 30 de septiembre enlazada a octubre), suma en el mes equivocado. Nada lo valida.
- ❌ **Inversiones ↔ Movimientos (inv.) existe pero no tiene rollups.** Por eso `Aportado` y `Valor actual` se escriben a mano, y la relación hoy no sirve para nada.
- ❌ **Aportes mensuales ARQ no tiene relación con nada.** Tiene los datos más ricos (pesos pagados, TRM, USDc, comisión, precio, ETF), pero no alimenta a *Inversiones*.
- ❌ **No hay puente entre finanzas e inversiones**, ni una base *Cuentas* que conecte origen y destino del dinero.
- ❌ **Las compras con tarjeta no se relacionan con la deuda de la tarjeta** (ver punto 5).

### 3) ¿Cálculos que pueden producir inconsistencias? — ❌ Sí, los más importantes

| # | Cálculo | Problema | Ejemplo con tus datos |
|---|---|---|---|
| C1 | `Abono` → `Abonado` → `Saldo pendiente` | Si usa el monto completo del pago, cuenta intereses y seguros como capital **(verificar)** | Carro: cuota $1.435.961, de la cual ~$448.000 son interés (1,23 % mensual sobre $36,4 M) y ~$178.000 seguros. Solo ~$810.000 son capital, así que el saldo bajaría ~$626.000/mes de más. Amex: ~$89.000/mes de interés sobre $4,1 M |
| C2 | `Saldo pendiente` de tarjetas | Solo resta: las compras nuevas con la tarjeta no suben el saldo | Visa: pagas $1.805.170, el saldo llega a 0, pero ya hiciste compras nuevas que no aparecen |
| C3 | `Salió` y `Balance` en Mis meses | Si `Sale` suma Gasto + Pago de deuda, una compra con tarjeta sale dos veces. Además el ahorro/inversión aparece como "salió", junto a los gastos | El tablero lo confirma: `salió = gastos + pagos_deudas + ahorro` |
| C4 | `Entra` y `Entró` | Si incluye "Préstamo que recibí" o "Cobro de préstamo", un préstamo aparece como ingreso **(verificar)** | Un avance de la Amex o un préstamo familiar inflan "lo que entró" |
| C5 | `Ganancia` / `Rentabilidad` de Inversiones | `Valor − Aportado` con `Aportado` manual. Ignora retiros y dividendos cobrados. En COP usa la TRM **de hoy** para ambos lados, así que no refleja los pesos que realmente pagaste | Si compras con TRM 4.100 y hoy está en 3.800, en dólares ganas pero en pesos pierdes, y el tablero no lo muestra |
| C6 | "PATRIMONIO" en *Mi tablero* | Es solo el valor de las inversiones activas. No resta deudas ni suma liquidez | Hoy mostraría $0 con ~$42,4 M de deuda |
| C7 | Rebalanceo (ETF más atrasado) | El JSON guarda `actual_usd` por ETF. Si es lo **aportado** por ETF y no **unidades × precio actual**, el rebalanceo se hace sobre el costo y no sobre el valor de mercado **(verificar)** | Si SMH sube 40 %, su peso real pasa del tope del 25 %, pero por costo seguiría en 20 % |
| C8 | `Presupuesto usado` | `Presupuesto de gastos` está vacío en los 4 meses, así que la fórmula divide por vacío o cero | — |
| C9 | KPIs de los tableros | Los calcula Claude al generar el JSON, fuera de las fórmulas de Notion, y pueden no coincidir con `Balance` | Dos "saldos del mes" distintos |
| C10 | Deudas en USD | La Amex en dólares está como $64.173 COP fijos | Si el dólar sube, la deuda real en pesos sube y no se refleja |
| C11 | Fórmulas con `now()` (si `Faltan` o `Avance` lo usan) | El resultado cambia con el paso del tiempo, aunque los datos no cambien **(verificar)** | — |

### 4) ¿Ingresos, gastos y transferencias bien diferenciados? — ⚠️ Casi

`Tipo` distingue bien *Ingreso*, *Gasto*, *Pago de deuda*, *Ahorro e inversión* y los tres tipos de préstamo. Faltan:

- **Transferencia entre mis cuentas** (Bancolombia → Nequi, retiro de cajero, recarga de ARQ).
- **Retiro de inversión** (volver de ARQ al banco). Hoy solo se podría anotar como "Ingreso".
- **Cargo a deuda / avance** (avance en efectivo de la Amex: el dinero entra a tu cuenta, pero es deuda, no ingreso).
- **Reembolso** (alguien te devuelve un gasto). Debería restar gasto, no sumar ingreso.
- **Una regla para salario bruto o neto.** El plan anota UDI y UTS "bruto". Si registras el bruto como ingreso, debes registrar también EPS, pensión y retención como gasto o deducción. Si registras el neto, **no** debes anotar la EPS de nómina como gasto. Hoy existe `Detalle = EPS` como gasto, que se presta a contarla dos veces.
- **Una regla para el aporte de tu esposa:** ¿es un ingreso del hogar o una transferencia para gastos compartidos? Cualquiera sirve, pero hay que decidirlo y aplicarlo siempre igual.

### 5) ¿Las transferencias inflan ingresos o gastos? — ❌ Sí, en los casos más comunes

| Caso | Cómo se anotaría hoy | Efecto |
|---|---|---|
| Compra de $200.000 con la Visa y luego pago de la Visa | Gasto $200.000 + Pago de deuda $200.000 | **Salió $400.000** por un consumo de $200.000. La deuda de la Visa no sube con la compra |
| Paso $500.000 de Bancolombia a Nequi | No hay tipo; o no se anota, o se anota como Gasto + Ingreso | O el saldo por cuenta queda descuadrado, o infla ingresos y gastos en $500.000 |
| Avance de la Amex de $1.000.000 | "Préstamo que recibí" (o Ingreso) | Si `Entra` lo cuenta, el ingreso queda inflado |
| Recarga de $1.200.000 a ARQ | "Ahorro e inversión" | ✅ No es gasto, pero `Salió` la cuenta junto con los gastos |
| Retiro de ARQ al banco | Solo cabría como "Ingreso" | Infla el ingreso: es tu propio dinero que vuelve |
| Dividendo que se queda en ARQ | Si además se anota en *Mis movimientos* | Se cuenta como ingreso de caja aunque el dinero nunca llegó a tu cuenta |

### 6) ¿El patrimonio neto se calcula bien? — ❌ No se calcula

Hoy **no existe** un cálculo de patrimonio neto. Para tenerlo hacen falta:

- **Activos líquidos:** saldo de cada cuenta (no existe la base *Cuentas*).
- **Inversiones:** valor de mercado en COP a la TRM **de la fecha de corte** (existe, pero se sobrescribe).
- **Cuentas por cobrar:** deudas con *Dirección = Me deben* (existe).
- **Otros activos (opcional):** el carro, depósitos de arriendo.
- **Menos pasivos:** saldo **real** de cada deuda. El actual tiene los errores C1, C2 y C10.

El KPI "PATRIMONIO" de *Mi tablero* debería llamarse **"Valor del portafolio"**.

### 7) ¿Las inversiones distinguen aportes, valor, ganancias realizadas y no realizadas, dividendos e intereses? — ⚠️ Los tipos existen, el cálculo no

| Concepto | ¿Se registra? | ¿Se calcula bien? |
|---|---|---|
| Aportes de capital | ✅ Tipo *Aporte* (y en Aportes ARQ) | ❌ `Aportado` manual, sin rollup |
| Valor actual | ✅ Campo manual/estimado | ⚠️ Se sobrescribe cada lunes, sin histórico |
| Ganancia realizada | ⚠️ Tipo *Retiro* existe | ❌ No hay costo base (costo promedio). No se puede saber qué parte de un retiro es capital y qué parte es ganancia |
| Ganancia no realizada | — | ❌ `Ganancia = Valor − Aportado` mezcla realizada y no realizada en cuanto haya un retiro |
| Dividendos | ✅ Tipo *Dividendo* | ❌ No se suman. No se define si son brutos o netos de retención (para residentes en Colombia, EE. UU. suele retener 30 % por falta de tratado; confírmalo en el reporte de ARQ). No se define si se reinvierten |
| Intereses | ✅ Tipo *Interés* | ❌ No se suman |
| Comisiones | ✅ Tipo *Comisión* + campo en Aportes ARQ | ⚠️ Duplicadas y sin regla clara (¿suben el costo base o bajan la rentabilidad?) |
| Efecto cambiario | ⚠️ Hay TRM por operación | ❌ No se separa el rendimiento en USD del efecto del peso |
| Posición por ETF (unidades) | ⚠️ `Acciones` aproximadas por compra | ❌ No hay unidades acumuladas por ETF ni valor de mercado por ETF |

### 8) ¿Se puede obtener rentabilidad absoluta y porcentual? — ⚠️ Solo una versión simplificada

- **Absoluta correcta:** `Valor actual + Retiros + Dividendos e intereses cobrados − Aportes − Comisiones`. Hoy falta todo menos valor y aportes.
- **Porcentual:** con aportes mensuales (DCA), `Ganancia / Aportado` engaña: un aporte hecho ayer pesa igual que uno de hace un año. Conviene mostrar dos cifras:
  - **Rentabilidad del mes (Modified Dietz)**, calculable en Notion con los cierres mensuales: `(V_fin − V_ini − Flujos) / (V_ini + 0,5 × Flujos)`. Si encadenas los meses obtienes la rentabilidad por periodo (TWR), comparable con el S&P 500.
  - **TIR/XIRR (tu rentabilidad real como inversionista)**: requiere código, porque Notion no la puede calcular.
- **En dos monedas:** en USD (cómo va el activo) y en COP (qué te pasó a ti, con el efecto cambiario). Para declarar renta importa la cifra en COP.

### 9) ¿Se conservan los históricos mensuales sin que cambien? — ❌ No

| Dato | Comportamiento hoy |
|---|---|
| `Valor actual`, `TRM referencia`, `Valor en COP` | Se sobrescriben cada lunes. El valor de agosto se pierde en septiembre |
| Entró / Salió / Gastos de Mis meses | Rollups en vivo: si corriges o recategorizas un movimiento viejo, el mes cerrado cambia sin dejar rastro |
| Saldos de deudas | Se recalculan desde todo el registro. No hay "saldo al cierre de agosto" guardado |
| Tableros HTML | El JSON se reemplaza entero. `historial: []` está vacío |
| Estado de deudas | Manual y sobrescrito |
| ✅ Aportes ARQ (`Valor del portafolio` por mes) y Revisiones (valor y TRM) | **Buen patrón:** números fijos con fecha |

**Solución:** hacer un **cierre mensual** que copie los valores a campos **número** (no rollups ni fórmulas) en *Mis meses* y marque `Cerrado ✓`. Desde ese momento, los meses cerrados no se recalculan.

### 10) ¿Qué dejar en Notion y qué automatizar con código? — ver sección 6

En resumen: **Notion** para registrar, clasificar, consultar, presupuestar y anotar decisiones. **Código** para todo lo que deba dar siempre el mismo resultado: cierres, conversiones de moneda, rentabilidades, rebalanceo, validaciones, respaldos y reporte anual.

---

## 3. Problemas encontrados

| ID | Problema | Severidad |
|---|---|---|
| P1 | Compras con tarjeta: doble salida y la deuda de la tarjeta no sube | Alta |
| P2 | Pagos de deuda cuentan intereses y seguros como capital **(verificar `Abono`)** | Alta |
| P3 | No hay base de Cuentas, así que no hay transferencias internas ni saldos por cuenta | Alta |
| P4 | No se calcula el patrimonio neto; el KPI "PATRIMONIO" está mal nombrado | Alta |
| P5 | No hay cierres mensuales congelados; valores sobrescritos | Alta |
| P6 | El aporte a ARQ se registra en 4 lugares sin relación entre ellos | Alta |
| P7 | `Aportado` y `Valor actual` manuales; relación Inversiones↔Movimientos sin rollups | Media-alta |
| P8 | Ganancia sin retiros, dividendos ni costo base; COP a TRM actual | Media-alta |
| P9 | Rebalanceo posiblemente por costo y no por valor de mercado **(verificar)** | Media |
| P10 | KPIs calculados por Claude en el JSON: segunda fuente de verdad | Media |
| P11 | Tipo, Categoría y Detalle redundantes; faltan tipos de transferencia, retiro, avance y reembolso | Media |
| P12 | Relación Movimiento→Mes manual y sin validar | Media |
| P13 | Sin regla de salario bruto/neto ni para el aporte de tu esposa | Media |
| P14 | Deuda en USD guardada como COP fijos | Baja |
| P15 | Tablas estáticas en *Mi plan de finanzas* que copian datos de las bases | Baja |
| P16 | Página vieja "Presupuesto mensual" y dos bases llamadas "Movimientos" | Baja |
| P17 | `Presupuesto de gastos` vacío, así que `Presupuesto usado` no se puede calcular | Baja |

---

## 4. Riesgos de integridad de datos

| ID | Riesgo | Cómo se materializa | Qué lo previene |
|---|---|---|---|
| R1 | **Deuda subestimada** | Saldo pendiente menor que el real (P1, P2). Decides abonos, o crees que ya terminaste de pagar, con datos falsos | Registrar cargos a la tarjeta, separar el capital y cuadrar cada mes contra el extracto |
| R2 | **Tasa de ahorro inflada o desinflada** | Transferencias y pagos de tarjeta mezclados con gastos | Tipo *Transferencia* y métricas bien definidas (sección 7.3) |
| R3 | **Números que no cuadran entre vistas** | `Balance` de Notion ≠ "Me queda" del tablero ≠ lo que dice Claude | Una sola fuente de cálculo |
| R4 | **Pérdida de historia** | Sobrescritura semanal y rollups vivos | Cierre mensual congelado |
| R5 | **Error de la IA en aritmética o clasificación** | Claude calcula conversiones, comisiones y rebalanceo en cada ejecución; un error queda guardado como dato | Cálculo determinista por script; Claude solo propone y registra |
| R6 | **Escritura en la base equivocada** | "Mis movimientos" vs "Movimientos"; plantilla vieja de presupuesto | Renombrar y archivar |
| R7 | **Movimiento huérfano** | Sin Mes, sin Tipo o sin Deuda cuando corresponde, así que no suma en ningún lado | Validador semanal |
| R8 | **Duplicados** | Una misma compra anotada por ti y por Claude | Validador (misma fecha + monto + concepto) |
| R9 | **Dependencia de un solo proveedor sin respaldo** | Notion no versiona por fila; un borrado masivo o un error de automatización no se puede deshacer fácilmente | Exportación semanal a CSV/JSON en este repositorio |
| R10 | **Datos fiscales incompletos** | Para la renta necesitas patrimonio y deudas al 31 de diciembre, dividendos en COP y retenciones | Cierre de diciembre + reporte anual |
| R11 | **Automatizaciones que no corren** | Si las tareas de Cowork dependen de que tu computador o la app estén abiertos, un día 1 puede pasar sin registro **(verificar)** | Validador que detecte meses sin cierre o aportes vencidos |

---

## 5. Mejoras recomendadas

Impacto: 🔴 alto · 🟠 medio · 🟢 bajo. Dificultad: **B** baja (minutos, solo Notion) · **M** media (algunas horas o una migración pequeña) · **A** alta (código o rediseño).

### Fase 1: antes del 1 de octubre (solo Notion, sin código)

| ID | Mejora | Impacto | Dificultad | Resuelve |
|---|---|---|---|---|
| M1 | **Definir y escribir las reglas de registro** (Anexo A) en una página corta que lean tanto tú como Claude | 🔴 | B | P1, P11, P13, R2 |
| M2 | Agregar a `Tipo` en Mis movimientos: **Transferencia**, **Retiro de inversión**, **Cargo a deuda (avance)** y **Reembolso**. Unificar en uno los tres préstamos y usar la dirección de la deuda | 🔴 | B | P11, R2 |
| M3 | Ligar las **compras con tarjeta** a su deuda (relación `Deuda o préstamo` que ya existe) y agregar la fórmula `Cargo`. `Saldo pendiente = Saldo al empezar + Cargos − Abonos a capital` | 🔴 | B | P1, R1 |
| M4 | Agregar el número **`Parte capital`** en Mis movimientos. `Abono` usa `Parte capital` si está lleno. El resto de la cuota (intereses y seguros) se registra como gasto de categoría "Costos financieros" | 🔴 | B | P2, R1 |
| M5 | Redefinir las fórmulas del mes (ver 7.3): **Ingresos, Gastos, Ahorro, Tasa de ahorro**. Dejar "flujo de caja" como métrica aparte | 🔴 | B | C3, C4, R2 |
| M6 | Fórmula de control `Mes OK` en Mis movimientos (mes de la Fecha = mes relacionado) y una vista "⚠ Revisar" con los errores | 🟠 | B | P12, R7 |
| M7 | Limpiar la taxonomía. `Categoría` solo para Ingreso y Gasto: quitar "Deudas", "Ahorro", "Inversión" y "Préstamos", que ya son Tipo. Agregar "Aporte de mi esposa" (según la regla elegida) y "Costos financieros". `Detalle` pasa a ser opcional | 🟠 | B | P11 |
| M8 | Renombrar "Movimientos" a **"Movimientos de inversión"**, archivar "Presupuesto mensual" y cambiar "PATRIMONIO" a **"Valor del portafolio"** en *Mi tablero* | 🟢 | B | P4, P16, R6 |
| M9 | Rollups en Inversiones: **Aportes, Retiros, Dividendos, Intereses y Comisiones** desde Movimientos de inversión. `Aportado` deja de ser manual | 🔴 | B | P7, C5 |
| M10 | Un aporte se registra **una sola vez** en Movimientos de inversión, con los campos de ARQ (ETF, unidades, precio, pesos pagados, TRM, comisión). *Aportes mensuales ARQ* queda como **checklist del plan** (Pendiente/Invertido/Omitido, ETF sugerido) relacionado 1 a 1 con el movimiento, sin repetir montos | 🔴 | M | P6 |
| M11 | Reemplazar las tablas estáticas de *Mi plan de finanzas* por vistas enlazadas de Deudas | 🟢 | B | P15 |

### Fase 2: estructura (Notion)

| ID | Mejora | Impacto | Dificultad | Resuelve |
|---|---|---|---|---|
| M12 | Nueva base **Cuentas** (Bancolombia, Nequi, Daviplata, Efectivo, ARQ-USDc…) con `Saldo inicial`, `Moneda`, rollups de entradas y salidas, `Saldo calculado` y `Saldo según banco` (manual, mensual) → `Diferencia`. En Mis movimientos, `Cuenta` y `Cuenta destino` reemplazan a `Medio de pago` | 🔴 | M | P3, R2, R7 |
| M13 | **Cierre mensual en Mis meses**: campos número congelados (Ingresos, Gastos, Ahorro, Tasa de ahorro, Liquidez, Inversiones USD/COP, TRM de cierre, Deudas, Me deben, **Patrimonio neto**, Aportes del mes, Rentabilidad del mes) + checkbox `Cerrado` | 🔴 | M | P4, P5, R4, R10 |
| M14 | Base pequeña **Activos** (VOO, QQQM, SMH): unidades (rollup), precio actual, valor, peso actual vs. meta. El rebalanceo usa valor de mercado | 🟠 | M | P9, C7 |
| M15 | Rentabilidad del portafolio con **dos cifras**: acumulada simple (USD y COP) y del mes (Modified Dietz desde los cierres) | 🟠 | M | P8, C5 |
| M16 | `Estado` de deudas como fórmula (vencida si `Próximo pago` < hoy y no hay pago en el mes) | 🟢 | B | — |
| M17 | Deudas en USD: campo `Moneda` y cálculo en COP con la TRM del cierre | 🟢 | B | P14 |
| M18 | Reemplazar los KPIs de los tableros HTML por **vistas de gráfico nativas de Notion** donde se pueda (ingresos vs. gastos por mes, gastos por categoría, patrimonio en el tiempo). Dejar HTML solo para simuladores (pago de deudas, ARQ) | 🟠 | M | P10, R3 |

### Fase 3: código (este repositorio)

| ID | Mejora | Impacto | Dificultad | Resuelve |
|---|---|---|---|---|
| M19 | **Script de cierre mensual** (día 1): calcula y congela las cifras de M13 y cuadra el patrimonio (ver 7.4) | 🔴 | A | P5, R3, R4, R5 |
| M20 | **Validador de integridad** semanal (lista en 6.1) que escribe un resumen en Notion | 🔴 | M | R1, R7, R8, R11 |
| M21 | **TRM y precios** obtenidos por código (TRM oficial de datos.gov.co; precios de VOO/QQQM/SMH) y **cálculo de rebalanceo** determinista. Claude solo muestra y pregunta | 🟠 | M | R5, C7 |
| M22 | **Respaldo semanal** de todas las bases a CSV/JSON en el repositorio (con historial en git) | 🟠 | B | R9 |
| M23 | **Reporte anual para la renta** (febrero): patrimonio y deudas al 31/12, aportes, retiros, dividendos y retenciones en COP a la TRM de cada fecha | 🟠 | M | R10 |
| M24 | **XIRR** del portafolio | 🟢 | M | — |

---

## 6. Automatizaciones que sí aportan valor

### 6.1 Recomendadas

| Automatización | Frecuencia | Qué hace | Valor | Dónde |
|---|---|---|---|---|
| **Cierre mensual** | Día 1 | Congela cifras del mes anterior, calcula patrimonio neto, crea el mes nuevo y marca `Cerrado` | Históricos confiables; base para todo lo demás | Código |
| **Validador** | Semanal (domingo) | Detecta movimientos sin Mes o con Mes ≠ Fecha, sin Tipo, gastos sin Categoría, compras con tarjeta sin deuda, pagos de deuda sin `Parte capital`, posibles duplicados, deudas vencidas, cuentas con diferencia frente al banco, inversiones con valor de más de 7 días y meses sin cerrar | Atrapa errores cuando todavía es fácil corregirlos | Código → página "Salud de mis datos" |
| **TRM + precios + rebalanceo** | Día 1 y lunes | Datos oficiales y cálculo exacto del ETF a comprar | Elimina el error de cálculo de la IA | Código; Claude conversa |
| **Respaldo** | Semanal | Exporta todo a este repositorio | Recuperación ante borrados o errores | Código |
| **Reporte fiscal** | Febrero | Cifras en COP listas para la declaración | Ahorra horas y reduce errores | Código |
| **Registro conversacional** (lo que ya tienes) | A demanda | "Gasté 45 mil en almuerzo con Nequi" → fila bien clasificada según las reglas del Anexo A | Menos fricción al registrar | Claude/Cowork |

### 6.2 No recomendadas (complejidad sin retorno)

- Precios en tiempo real o fotos diarias del portafolio: con aportes mensuales y horizonte de 5 años o más, basta con el cierre mensual y el lunes.
- Conexión automática con el banco o scraping: frágil, riesgosa con credenciales y con mucho mantenimiento. Si más adelante quieres importar, que sea desde el **CSV del extracto**, revisado por ti.
- Categorización automática con modelos entrenados: con un solo registro al día no compensa. Las reglas y Claude bastan.
- Más tableros HTML con datos copiados: cada uno es una copia más que se desactualiza.

---

## 7. Arquitectura ideal propuesta

La idea es **simple y confiable**: 7 bases (una nueva: Cuentas; una opcional pequeña: Activos), un solo registro de caja, un solo registro de inversiones, un cierre mensual congelado y un script que calcula.

### 7.1 Mapa

```
                         ┌────────────┐
                         │  Cuentas   │  (nueva) liquidez por cuenta + cuadre con el banco
                         └─────▲──────┘
                 Cuenta / Cuenta destino
┌──────────────────────────────┴───────────────┐   Mes     ┌──────────────────────────┐
│  Mis movimientos  (ÚNICO registro de caja)    │─────────▶│  Mis meses               │
│  Tipo: Ingreso · Gasto · Transferencia ·      │           │  en vivo: rollups del mes│
│        Pago de deuda · Cargo a deuda ·        │           │  cierre: números fijos,  │
│        Aporte a inversión · Retiro de inv. ·  │           │  Patrimonio neto,        │
│        Reembolso · Préstamo (dar/recibir/cob.)│           │  Cerrado ✓               │
│  Parte capital (para pagos de deuda)          │           └──────────────────────────┘
└───────┬──────────────────────────┬────────────┘
        │ Deuda                    │ Movimiento de inversión (1:1, solo aportes/retiros)
        ▼                          ▼
┌──────────────────┐     ┌──────────────────────────┐   Inversión  ┌──────────────┐
│ Deudas y préstamos│     │ Movimientos de inversión │─────────────▶│ Inversiones  │
│ Saldo = inicial + │     │ Aporte/Retiro/Dividendo/ │              │ rollups:     │
│ cargos − capital  │     │ Interés/Comisión          │   Activo     │ aportes, ret.│
└──────────────────┘     │ ETF, unidades, precio,    │─────────────▶│ divid., com. │
                         │ USD, COP pagados, TRM     │  ┌─────────┐ └──────────────┘
                         └───────────▲──────────────┘  │ Activos │ (opcional)
                                     │ 1:1              │ VOO/QQQM│ unidades, precio,
                         ┌───────────┴──────────────┐  │ /SMH    │ peso vs. meta
                         │ Plan ARQ (checklist)      │  └─────────┘
                         │ Pendiente/Invertido/Omit. │
                         └──────────────────────────┘
          Revisiones trimestrales (sin cambios)

   ┌─────────────── Script en este repositorio (GitHub Actions) ───────────────┐
   │ cierre mensual · validador · TRM/precios/rebalanceo · respaldo · fiscal   │
   │ lee y escribe Notion por API; Claude/Cowork conversa y registra           │
   └────────────────────────────────────────────────────────────────────────────┘
```

**Por qué esto y no algo más complejo:** no hace falta contabilidad de partida doble completa. Con **Cuenta / Cuenta destino** en cada movimiento consigues lo esencial de la partida doble (las transferencias no inflan nada y cada cuenta cuadra) sin libro diario ni plan de cuentas.

### 7.2 Qué vive dónde

| Queda en Notion | Pasa a código |
|---|---|
| Registrar movimientos (tú o Claude) | Cálculos que deben ser exactos y repetibles: cierre, patrimonio, rentabilidad, rebalanceo |
| Catálogos: cuentas, deudas, inversiones, activos, categorías | Datos externos: TRM, precios |
| Presupuesto del mes y "Cómo me fue" | Validaciones de integridad |
| Fórmulas simples **por fila** (signo, conversión con la TRM de esa fila, "Mes OK") | Cálculos entre muchas filas o entre meses (XIRR, costo promedio, Modified Dietz encadenado) |
| Vistas y gráficos nativos | Respaldo y reporte anual |
| Revisiones y decisiones | — |
| Simuladores HTML (son herramientas de planeación, no datos) | — |

**Por qué código y no solo Claude/Cowork para calcular:** un script da siempre el mismo resultado, se puede probar y queda versionado en git. Además, la API de Notion con un token de integración no tiene el límite de consultas SQL que tu plan actual aplica al conector. Claude sigue siendo la interfaz: te pregunta, registra y explica los resultados que calcula el script.

### 7.3 Definición de métricas (una sola fuente de verdad)

| Métrica | Definición |
|---|---|
| **Ingresos** | Σ Tipo = Ingreso (salario **neto**, honorarios, clases, aporte de tu esposa si es ingreso, dividendos o intereses **cobrados a una cuenta tuya**) |
| **Gastos** | Σ Tipo = Gasto − Σ Reembolso (incluye intereses, seguros y comisiones financieras) |
| **Ahorro del mes** | Ingresos − Gastos |
| **Tasa de ahorro** | Ahorro / Ingresos |
| **Destino del ahorro** | Capital abonado a deudas + Aportes netos a inversión + Aumento de liquidez |
| **Flujo de caja** | Entradas − salidas de tus cuentas. **No** mide qué tan bien te fue; sirve para saber si te alcanza |
| **Patrimonio neto** | Liquidez + Inversiones (valor de mercado en COP a la TRM de cierre) + Me deben − Deudas (saldo real) [+ otros activos si decides incluirlos] |
| **Ganancia del portafolio** | Valor + Retiros + Dividendos e intereses cobrados − Aportes (en USD y en COP a TRM histórica) |
| **Rentabilidad del mes** | Modified Dietz: `(V_fin − V_ini − F) / (V_ini + 0,5·F)` |

Las transferencias, los aportes a inversión, los retiros de inversión, los préstamos y los cargos o pagos de deuda **no son ingreso ni gasto**. Mueven dinero entre bolsillos tuyos o cambian una deuda.

### 7.4 Cuadre automático (la prueba de que todo está bien)

Cada cierre mensual, el script comprueba:

```
Patrimonio(fin) − Patrimonio(inicio)
  ≈ Ahorro del mes
  + Ganancia/pérdida de inversiones del mes (incluye efecto cambiario)
  − Variación de deudas no explicada por el registro (intereses no registrados, etc.)
```

Si la diferencia pasa de un umbral (por ejemplo $50.000), algo quedó mal registrado y el validador lo avisa. Con esta sola regla se detectan casi todos los errores de P1, P2, P3 y R8.

---

## 8. Plan sugerido y lo que necesito de ti

**Orden recomendado:**

1. **Esta semana (Fase 1, solo Notion):** M1–M9 y M11, antes del primer aporte a ARQ y del primer mes completo. La mayoría toma minutos.
2. **Octubre (Fase 2):** M12 (Cuentas), M13 (cierre en Mis meses) y M10. Después M14 a M18.
3. **Noviembre (Fase 3, código):** M22 (respaldo) → M20 (validador) → M19 (cierre) → M21 → M23 antes de febrero.

**Para confirmar el diagnóstico, necesito:**

1. El texto de estas fórmulas (en Notion, abre la propiedad → *Editar fórmula* → copia y pega): `Entra`, `Sale`, `Solo gastos`, `Abono`, `Balance`, `Saldo pendiente`, `Faltan`, `Ganancia`, `Rentabilidad`, `Valor en COP`.
2. **Salario:** ¿vas a registrar UDI y UTS en neto (lo que llega a la cuenta) o en bruto con sus deducciones? Recomiendo **neto**.
3. **Aporte de tu esposa:** ¿ingreso del hogar o transferencia? Recomiendo **ingreso**, con su propia categoría, si el presupuesto es del hogar.
4. **Carro:** ¿lo quieres como activo en el patrimonio (valor comercial, actualizado una vez al año) o solo contar su deuda? Recomiendo incluirlo en una línea aparte: patrimonio **con** y **sin** activos fijos.
5. **ARQ:** ¿los dividendos se reinvierten solos o quedan como USDc? ¿`actual_usd` del tablero es costo o valor de mercado?
6. ¿Las tareas programadas de Cowork corren aunque tu computador esté apagado?

---

## Anexo A: reglas de registro (una fila por situación)

| Situación | Tipo | Monto | Cuenta → Cuenta destino | Deuda | Categoría | ¿Ingreso o gasto? |
|---|---|---|---|---|---|---|
| Me pagan UDI (neto) | Ingreso | neto | → Bancolombia | — | UDI | Ingreso |
| Almuerzo con Nequi | Gasto | valor | Nequi → | — | Mercado y comida | Gasto |
| Compra con la Visa | Gasto | valor | (tarjeta) | Visa | la del gasto | Gasto (una sola vez) |
| Pago de la Visa | Pago de deuda | pagado; `Parte capital` = pagado | Bancolombia → | Visa | — | No |
| Cuota del carro | Pago de deuda | cuota; `Parte capital` = según extracto | Bancolombia → | Carro | — | No (el capital) |
| …intereses y seguros de esa cuota | Gasto | cuota − capital | (misma fila o fila aparte) | Carro | Costos financieros | Gasto |
| Avance de la Amex | Cargo a deuda | valor | → Bancolombia | Amex | — | No |
| Bancolombia → Nequi | Transferencia | valor | Bancolombia → Nequi | — | — | No |
| Recarga PSE a ARQ | Aporte a inversión | pesos | Bancolombia → ARQ | — | — | No (+ fila en Movimientos de inversión) |
| Retiro de ARQ al banco | Retiro de inversión | pesos | ARQ → Bancolombia | — | — | No |
| Dividendo que queda en ARQ | (solo en Movimientos de inversión) | USD neto | — | — | — | No toca caja |
| Me devuelven un gasto compartido | Reembolso | valor | → cuenta | — | la del gasto original | Resta gasto |
| Presto $300.000 a un amigo | Préstamo (dar) | valor | Nequi → | "Préstamo a X" (Me deben) | — | No |
| Me paga el amigo | Préstamo (cobro) | valor | → Nequi | "Préstamo a X" | — | No |
