# Decisiones del proyecto

Cada decisión registra la pregunta, lo que sabemos, las opciones, cómo se elige y qué se decidió.

Estado: `pendiente` · `en curso` · `decidida`.

---

## 1 · Target: cómo lo construimos a partir de `is_suspended`

**Estado:** `decidida`

**Pregunta.** El modelo ordena a los merchants activos: ¿cuáles se parecen a los que una revisión termina suspendiendo? Devuelve un ranking, no la probabilidad de que la suspensión caiga en un plazo.

`is_suspended` es el estado diario, no un evento. Para armar el target hay que definir quién entra y qué día se mira de cada merchant.

### Lo que sabemos

Fuente: `notebooks/02_suspensiones.ipynb`, `docs/suspensiones.png`, `docs/suspensiones_dia_a_dia.md`, `docs/variables/`.

| Hecho | Valor |
|---|---|
| Merchants alguna vez suspendidos | 26 |
| Con inicio observado (un día activo seguido de un día suspendido) | 19, todos con un solo inicio |
| Sin inicio observado | 7: 3 ya están suspendidos el 1-abr (primer día de datos); 3 aparecen en los datos ya suspendidos, 1 día, y vuelven a operar; 1 (`MEBF6C9…`) tiene 153 días de estado nulo antes de su suspensión |
| Duración de la suspensión (los 19) | de 1 a 92 días |
| Vuelven a operar | 19 de 26 |
| Período con etiqueta | abril–septiembre; octubre–diciembre `is_suspended` es 100% nulo |

Cómo se ven los 19 inicios día por día (lectura de los gráficos; es una interpretación sobre pocos casos):

| Patrón | Merchants | Qué implica |
|---|---:|---|
| Ráfaga de intentos y caída de aprobación 1 a 3 días antes del inicio | ~7 | Hay señal, pero con muy poca anticipación |
| Rechazos solo en la fila del primer día suspendido | ~4 | Es actividad del día anterior, previa a la suspensión: no hay rechazos por suspensión |
| Sin nada raro antes (casi sin actividad o recién dados de alta) | ~8 | Su comportamiento no anticipa la suspensión |

Otros hechos que pesan en la decisión:
- Varias suspensiones caen el mismo día (4 el 23-ago, 2 el 11-sep, 2 el 22-abr): parecen revisiones en lote.
- Los merchants suspendidos operan muy poco: de 0 a 2 transacciones por día.
- Mientras el merchant está suspendido, sus features reflejan la suspensión (volumen en cero; no hay rechazos por suspensión). Usar esas filas es fuga.

### Decisión 1a · Qué casos entran

**Clasificación de los 19 inicios observados** según su comportamiento antes de la suspensión. Es una lectura de los gráficos de `docs/suspensiones.png` y `docs/variables/` (de 30 días antes a 10 después), no una regla automática. Antes se descartaron las variables que no se mueven antes del inicio: red, canales open platform / pocket / recurrente / SPEI, flags, MCC, claims y contracargos. Los percentiles de industria describen a la industria, no al merchant.

| Grupo | Merchant | Inicio | Qué se ve antes |
|---|---|---|---|
| **A · Deterioro durante días o semanas** | `MA52FE20E571E5DCA` | 15-jul | Ticket de 5.000 a 12.000, volumen creciente, aprobación por TPV de 90% a 27% en 3 semanas |
| | `M2971A5DEEED25520` | 23-ago | Intentos a 30 días de 10 a 90 en un mes, aprobación por TPV cerca de 0 |
| | `M331E694E1EECD91A` | 23-ago | Intentos en crecimiento continuo; el día −7 el ticket se duplica y la aprobación baja a 75% |
| | `MB351DC99D4064375` | 22-abr | Aprobación ~10%; salto de intentos el día −10 |
| | `MC45B94C90F7E27CD` | 25-jul | Aprobación ~30% a 30 días; el 4-jul (día −21) 9 intentos y 7 rechazados |
| | `M46B66AE0E309595B` | 14-jun | Aparece 4 días antes y pasa de 4 a 43 intentos con aprobación en caída; 256 fraudes en transacciones |
| **B · Ráfaga corta de rechazos 1 a 3 días antes** | `MB2ECDC9763AB4E54` | 23-ago | Pico de intentos y aprobación de 100% a 44% en los días −2 y −1 |
| | `MEB47BBD227407EA5` | 23-ago | 6 intentos rechazados el día −1 |
| | `MFECEBCA0010F3873` | 22-abr | Intentos rechazados el día −1, ticket salta a 5.500 |
| | `M7F9108A69784504E` | 29-abr | El 26-abr (día −3) 3 intentos, los 3 rechazados, ticket ~17 |
| **C · Merchant nuevo, todo pasa el día 0** | `MF6C1EBC5EE0342DB` | 29-may | Aparece el día −1; su red suma un suspendido el día 0 |
| | `MED4E6F81B6D71742` | 1-jul | Primera transacción online, de 10.000, rechazada |
| **D · Sin riesgo visible antes** | `MEA1EE5CA0A25553F` | 12-sep | Cuenta nueva (12 días), 0 transacciones de por vida |
| | `M002D3A13420843D9` | 7-abr | Solo 6 días de historia (los datos empiezan el 1-abr) |
| **Fuera del análisis** | `ME9BD50FCE697D4A7` | 25-jun | Historial de aprobación normal y después silencio. No cuenta como caso |
| | `MF000975FCCCF8010` | 11-sep | Antigüedad 0, una transacción de alrededor de 1 peso. No cuenta como caso |
| | `M486B452BBACA6CA4` | 11-sep | Casi inactivo: dos transacciones online de ~3.000, aprobadas. Probablemente lo suspendió el propio merchant. No cuenta como caso |
| | `M3182C56E1D13EC33` | 24-sep | No hizo nada raro. Lo suspendieron por otro motivo. No cuenta como caso |
| | `M315DBFD11714A6D5` | 30-sep | Antes del bloqueo: 1 a 3 intentos por día, todos aprobados, y 4 días sin actividad. No hay actividad sospechosa. Los claims (mensuales) saltan el 1-oct, de 0 a 17 |

Lectura:
- **A y B (10):** muestran algo antes de la suspensión. Es lo único que un modelo podría anticipar.
- **C (2):** el día anterior no tienen actividad. Los intentos están en la fila del primer día suspendido, que es la foto de todos los positivos. `MED4E6F` tiene un intento rechazado de unos 10.000. `MF6C1EB` tiene unos 7 intentos, 3 rechazados.
- **D:** no muestran riesgo antes del inicio. `M002D3A…` queda fuera por censura al inicio. `MEA1EE5…` queda fuera del dataset de entrenamiento y de test: nunca operó (0 transacciones de por vida). `M3182C5…` no cuenta: no hizo nada raro.

Cuidados:
- Excluir a D por "no tener actividad inusual" sería usar como criterio lo mismo que el modelo aprende, y eso infla los resultados (opción D de la tabla siguiente). El criterio tiene que ser independiente del comportamiento, por ejemplo poca antigüedad o casi sin transacciones de toda su vida, y confirmarse con la empresa (¿hay suspensiones por revisión de alta?).
- Hay datos inconsistentes: `MEA1EE5…` tiene 0 transacciones de por vida pero ~2 intentos a 30 días.
- Los gráficos cubren de −30 a +10 días; no se miró historia más vieja.

**Decidido**
- **Positivos: 12 merchants con inicio observado**, todos menos `M002D3A13420843D9`, `ME9BD50FCE697D4A7`, `MF000975FCCCF8010`, `M315DBFD11714A6D5`, `M486B452BBACA6CA4`, `M3182C56E1D13EC33` y `MEA1EE5CA0A25553F`.
- **`M002D3A13420843D9` queda fuera del todo**, también de la sensibilidad. Su historia empieza el 1-abr, primer día de datos, y lo suspenden el 7-abr. Esos días existen porque ahí arranca el dataset, no porque hayamos visto al merchant antes. No se puede separar una suspensión de un borde de la ventana.
- **`ME9BD50FCE697D4A7`, `MF000975FCCCF8010`, `M315DBFD11714A6D5`, `M486B452BBACA6CA4`, `M3182C56E1D13EC33` y `MEA1EE5CA0A25553F` quedan fuera del análisis**, de entrenamiento y de test. `M3182` no hizo nada raro: lo suspendieron por otro motivo. `MEA1EE5` nunca operó. `M315DBFD11714A6D5` tampoco muestra actividad sospechosa antes de la suspensión: 1 a 3 intentos por día, todos aprobados, y 4 días sin actividad. El criterio común es ese: no hay nada en la actividad que un modelo pueda aprender, y la empresa dice que las suspensiones pueden ser por documentación o cumplimiento, sin dar el motivo.
- **Límite del recall.** Se sacaron 6 de los 18 casos con inicio observado porque antes del inicio no muestran actividad sospechosa. El recall que se reporta es sobre las suspensiones que se ven en la actividad. La empresa evalúa contra todas las suspensiones de octubre a diciembre, incluidas las que no tienen señal, así que el recall real va a ser menor.
- **Los 7 sin inicio observado quedan fuera.** No hay un día activo anterior desde el que rankearlos. `M214FEFB687B0BF24`, `ME05F52FFA78E2A54` y `ME180F1798534B082` ya están suspendidos el 1-abr. `M610AE7B7698D91D4`, `MBEAC57FE29695D34` y `MD6E9ABDB55CED9C5` aparecen el mismo día en que los suspenden. `MEBF6C90BC93B6B23` tiene 153 días de estado nulo antes de su suspensión.

**Opciones**

| Opción | Detalle | A favor | En contra |
|---|---|---|---|
| A | Solo los 19 con inicio observado | Se sabe cuándo empieza; sus filas previas son limpias | Pierde 7 positivos |
| B | Los 19 + los 4 que entran desde un nulo | 23 positivos | El inicio real cae en un día sin dato |
| C | Los 26 | Máximos positivos | Los 3 ya suspendidos el 1-abr no tienen historia previa |
| D | Excluir además los que no muestran señal previa | Target más limpio | Es elegir los casos que el modelo puede ver: sesga la evaluación |

### Decisión 1b · Hasta dónde se usa la historia de cada merchant

**Decidido: una fila por merchant.**

- **Positivo:** su primer día suspendido. Esa fila resume la actividad hasta el día anterior, cuando el merchant todavía operaba, y un suspendido no genera transacciones: no hay rechazos por suspensión. Así quedan `M46B66A` (7 rechazos), `MA52FE2` (15), `MB351DC` (6), `MED4E6F` y `MF6C1EB`, sin elegir el día mirando el comportamiento del merchant.
- **La regla** ordena por la peor de tres señales: la peor aprobación entre 7 y 30 días, los rechazos del día de la foto, o el peor día de rechazos en los 30 días hasta la foto. Ese pico es el que tiene `MC45B94` el 4-jul (7 rechazos) y `M7F9108` el 26-abr (3 rechazos).
- **Negativo:** un merchant que no fue suspendido en abril–septiembre, en un día activo. La fecha se sortea con la misma mezcla que las fotos de los positivos (semilla 0), para que el calendario no sea la señal. La mezcla queda cerca: agosto pesa un poco más en los negativos porque hay más merchants con fila ese día. Si no tiene fila activa en ninguna de esas fechas, queda fuera: 6.739 merchants, de 40.239 nunca suspendidos. Entran 33.500.
- **Fuera:** lo posterior al primer día suspendido, los días con estado nulo, octubre–diciembre, y la historia anterior a la foto. Esos días ya no se etiquetan como negativos: si el merchant ya tenía aprobación baja, esa fila le enseñaba al modelo que ese perfil es sano.
- La fila del primer día suspendido no tiene fuga: la empresa confirmó que la fila con fecha *t* resume la actividad hasta *t*−1 y que un suspendido no genera transacciones (ver el anexo). Por eso no se usa el día anterior ni se elige entre los dos.

### Decisión 1c · Horizonte de 30 días (descartado)

Se construyó un target diario: la fila del día *t* era positiva si la suspensión caía en (*t*, *t* + 30]. Se dejó de lado.

La señal medida en el paso 3 de features es de nivel (aprobación más baja y ticket más alto que el resto), no un cambio dentro de los 30 días. Con esa etiqueta, los días −31 y anteriores de un positivo quedaban como negativos teniendo el mismo perfil. Además varias suspensiones caen el mismo día: parecen una revisión en lote, no un plazo de 30 días.

### Decisión

El dataset (`notebooks/03_dataset.ipynb`, `data/processed/dataset.parquet`) tiene **33.512 filas, una por merchant**:

| | Filas | Qué es |
|---|---:|---|
| Positivo | 12 | Su primer día suspendido. `MEA1EE5` no está: nunca operó |
| Negativo | 33.500 | Nunca suspendidos en abril–septiembre, un día activo con la fecha de los positivos |

`group_d` queda vacío.

---

## 2 · Validación: cómo separamos train y test

**Estado:** `decidida`

**Decidido: el modelo es la logística B** (`notebooks/05_logistica.ipynb`). Validación: `StratifiedKFold` de 5 folds, repetido con 10 semillas. Cada merchant se predice con un modelo que no lo vio. Hay una fila por merchant, así que train y test de un fold no comparten ninguno.

La métrica es el ranking en el test: PR-AUC y recall en el top 1% y 5%. Con 12 positivos el test queda con unos 2 o 3, así que el número es ruidoso. `MEA1EE5` no entra ni a entrenamiento ni a test.

---

## 3 · Features: cuáles entran al modelo

**Estado:** `en curso`

Las mediciones de abajo se hicieron sobre el panel diario, antes de pasar a una fila por merchant. Siguen sirviendo para elegir columnas: el modelo las lee en la foto del primer día suspendido.

El análisis se hizo sobre el panel diario, con una versión anterior de `notebooks/04_features.ipynb`. Esos gráficos y CSV ya no están: el dataset dejó de ser un panel y la lectura quedó escrita acá.

### Paso 1 · Fuera por definición (decidido)

| Columna | Motivo |
|---|---|
| `me_merchant_created_at_timestamp` | Fecha; la antigüedad ya está en `merchant_tenure_days` |
| `config_segment` | 100% nula |
| `me_attempted_pocket_txn_sum_7d` | Un solo valor en todo el dataset |
| `me_industry_mcc` | Igual a `industry_mcc` en el 99,7% de las filas; se queda `industry_mcc` |
| `vip0_segment` | No nula exactamente donde `is_veto_vip0` no es nula: repite el flag |
| `me_suspended_merchants_in_network`, `me_suspended_merchants_in_network_share` | Fuga: suben justo cuando se suspende el propio merchant y bajan cuando vuelve |

Quedan 61 features. Los flags `is_*` y los contracargos / disputas acumulados (`me_lifetime_chargeback_tpv`, `me_lifetime_dispute_tpv`) son 97–99% nulos, pero se quedan por ahora: el nulo probablemente significa "no tiene". Cómo tratar ese nulo queda pendiente.

### Paso 2 · Qué se mueve al acercarse la suspensión (lectura preliminar)

Para cada feature: mediana de los 13 positivos según los días hasta su suspensión (−60 a −1), contra el rango p25–p75 de los merchants nunca suspendidos.

**Cuidado al leer los gráficos:** la mediana no se calcula siempre sobre los mismos merchants. Entre −60 y −31 solo aportan los que tienen historia larga; en −28 y −21 entran merchants con poca historia, y en los últimos 4 días los que casi no tienen. Varios saltos en −30, −28 y −21 son cambios de composición, no del comportamiento.

| Lectura | Features |
|---|---|
| **Se mueven al acercarse la suspensión** | Tasas de aprobación (por transacción y por TPV, a 1, 7, 15 y 30 días): caen bajo el rango normal en los últimos ~20 días, más fuerte cerca del día −1 (a 7 días, de ~100% a 30–50%). Ticket promedio a 7 y 30 días: por encima del rango y en aumento. Intentos a 7, 15 y 30 días: suben al final, aunque dentro del rango normal |
| **Distintas en nivel, pero no se mueven** | Aprobación histórica (`me_lifetime_*_approval_rate`: 0,6–0,7 contra 0,9 normal) y ticket histórico. Describen cómo es el merchant, no un cambio |
| **Sin señal** | Transacciones aprobadas (1, 7, 15, 30 y 90 días), TPV aprobado |
| **Sin variación ni datos en los positivos** | `me_attempted_openplatform_txn_sum_7d`, `me_attempted_recurring_txn_sum_7d`, `me_attempted_spei_txn_sum_7d`, `me_network_avg_tpv_chargeback_rate`, `me_network_avg_tpv_dispute_rate`, `ratio_claims_tpv_180d`, flags `is_*`, contracargos y disputas acumulados (solo 1 positivo tiene datos) |
| **Describen la industria, no al merchant** | `industry_mcc`, `ind_*` y percentiles `p*_approved_*_90d`: sus saltos vienen de qué merchants entran en la mediana |
| **Ruido o artefacto** | `merchant_tenure_days` (salta sin sentido), `me_network_size` y `me_network_avg_tpv_approval_rate` (solo 4 positivos tienen red; el salto del final es un merchant que entra) |
| **La mediana no sirve** | `me_attempted_ecommerce_txn_sum_7d`: la mayoría vale 0, así que la mediana esconde a los pocos que sí tienen ecommerce. Hay que mirarla con la proporción de merchants con valor > 0 |

### Paso 3 · Medición con números

Se comparó la **ventana target** (días −30 a −1) con la **ventana base** (días −60 a −31), contra 3.000 merchants nunca suspendidos con una fecha de "suspensión" al azar. AUC 0,5 = sin diferencia; más lejos de 0,5, más señal. Con 7 y ~11 positivos, los números sirven para ordenar, no como prueba.

- **Dentro del merchant** (cambio entre ventana base y target; 7 positivos con historia suficiente): **ninguna feature muestra un cambio fuerte**. Las AUC van de 0,36 a 0,68. Las tasas de aprobación quedan entre 0,42 y 0,59, menos de lo que sugerían los gráficos: la caída se concentra en los últimos días y la mediana de 30 días la diluye. Además, en el grupo A el deterioro ya empezó en la ventana base.
- **Entre merchants** (ventana target; ~11 positivos con datos): los positivos tienen **menor aprobación** (actual: AUC 0,29–0,41; histórica: 0,29–0,30), **ticket más alto** (a 7 días 0,70; a 30 días 0,67; histórico 0,69) y algo más de intentos (~0,60).
- **Industria:** `ind_ratio_claims_tpv_180d` (0,76), `industry_mcc` (0,31) y percentiles `p*_approved_*_90d` (hasta 0,27) separan entre merchants, pero porque los positivos se concentran en pocos MCC. Con 13 casos es fácil que el modelo memorice industrias.
- **Red:** solo 2 a 4 positivos tienen datos; no se puede medir.
- **Sin información:** `me_attempted_openplatform_txn_sum_7d`, `me_attempted_recurring_txn_sum_7d`, `me_attempted_spei_txn_sum_7d`, `ratio_claims_tpv_180d`, flags `is_*`, contracargos y disputas acumulados (0 o 1 positivo con datos, o sin variación).

Conclusión preliminar: la señal más clara es **entre merchants** (aprobación baja y ticket alto). El cambio dentro del merchant no se ve con ventanas de 30 días; se podría volver a medir comparando los últimos 7 días contra la base.

### Paso 4 · Features relativas a la industria

En vez de usar las variables de industria en crudo, se compara al merchant con su industria:

| Feature | Definición | AUC dentro | AUC entre |
|---|---|---:|---:|
| `tpv_vs_industry_p50` | TPV aprobado 90 días / mediana del TPV de su industria | 0,57 | 0,60 |
| `tpv_vs_industry_p95` | TPV aprobado 90 días / percentil 95 de su industria | 0,61 | 0,58 |
| `txns_vs_industry_p50` | Transacciones aprobadas 90 días / mediana de su industria | 0,63 | 0,46 |

Separan poco, parecido al TPV en crudo. La separación que daban las variables de industria en crudo (hasta 0,76) desaparece al comparar contra la propia industria: venía de que los positivos se concentran en pocos MCC, no del comportamiento del merchant.

### Decisión (lista inicial)

**Se quedan 25 features:**

| Grupo | Features |
|---|---|
| Aprobación actual (8) | `me_txn_approval_rate_1d`, `_7d`, `_15d`, `_30d`; `me_tpv_approval_rate_1d`, `_7d`, `_15d`, `_30d` |
| Aprobación histórica (2) | `me_lifetime_txn_approval_rate`, `me_lifetime_tpv_approval_rate` |
| Ticket (4) | `me_attempted_tpv_avg_1d`, `_7d`, `_30d`; `me_lifetime_tpv_avg` |
| Intentos (4) | `me_attempted_txn_sum_1d`, `_7d`, `_15d`, `_30d` |
| Ecommerce (1) | `me_attempted_ecommerce_txn_sum_7d` |
| Tamaño (2) | `approved_tpv_90d`, `me_lifetime_approved_txn` |
| Relativas a la industria (3, nuevas) | `tpv_vs_industry_p50`, `tpv_vs_industry_p95`, `txns_vs_industry_p50` |
| Pico de rechazos (1) | `max_rechazos_1d_30d`: el máximo de intentos rechazados en un día, en los 30 días hasta la foto |

**Salen 40 de las originales** (además de las 7 del paso 1):

| Grupo | Cuántas | Motivo |
|---|---:|---|
| Sin información | 12 | `me_attempted_openplatform_txn_sum_7d`, `me_attempted_recurring_txn_sum_7d`, `me_attempted_spei_txn_sum_7d`, `ratio_claims_tpv_180d`, `is_closed_won_ever`, `is_veto_vip0`, `is_veto_config`, `is_portfolio_managed`, `me_lifetime_chargeback_tpv`, `me_lifetime_dispute_tpv`, `me_network_avg_tpv_chargeback_rate`, `me_network_avg_tpv_dispute_rate`: sin datos o sin variación en los positivos |
| Red | 2 | `me_network_size`, `me_network_avg_tpv_approval_rate`: solo 2 a 4 positivos tienen red |
| Industria en crudo | 15 | `industry_mcc`, `ind_*` y percentiles `p*_approved_*_90d`: con 13 casos el modelo memorizaría industrias; se reemplazan por las relativas |
| Aprobadas sin señal | 7 | `me_approved_txn_sum_1d`, `_7d`, `_15d`, `_30d`, `me_approved_tpv_sum_30d`, `approved_txns_90d`, `me_lifetime_approved_tpv`: no separan; el tamaño queda con dos features |
| Canal de adquisición | 3 | `me_acquisition_channel`, `me_acquisition_sub_channel`, `me_business_acquisition_channel`: identidad del merchant, no comportamiento |
| Antigüedad | 1 | `merchant_tenure_days`: ruido, salta sin sentido |

Cuidados:
- Es una lista decidida analizando cada feature por separado, no con un modelo. Hay que validarla después entrenando con y sin cada grupo.
- Con 12 positivos conviene tener pocas features; 25 todavía es bastante y varias están correlacionadas (las 8 de aprobación).
- Las relativas a la industria separan poco; se mantienen por ser la forma correcta de usar la industria, a validar con el modelo.

### Lags y diferencias (descartados)

Con una fila por merchant, la pregunta es si el perfil de ese día se parece al de los suspendidos. Las tasas y los tickets a 7, 15 y 30 días ya son la ventana reciente. Un lag o un delta mediría si el merchant cambió respecto de sí mismo, y esa señal no apareció. Además sumaban 25 columnas para 14 positivos, y las tasas diarias quedan nulas cerca del 75% de las veces porque la mayoría no opera todos los días.

**Total: 25 features** (21 originales + 3 relativas a la industria + `max_rechazos_1d_30d`). Las genera `notebooks/04_features.ipynb` en `data/processed/features.parquet`.

Nulos: en las tasas, el valor −1 (sin transacciones) se toma como nulo. La tasa a 1 día queda nula la mayoría de las veces: el merchant no operó ese día. La logística imputa esos nulos con la mediana.

### Nulos y valores fuera de rango (resuelto)

- **Tasas con −1** (sin transacciones): se toman como nulas (`04_features.ipynb`).
- **Tasas fuera de [0, 1]:** se recortan a ese rango en la logística (`05_logistica.ipynb`).
- **Nulos en las variables del modelo:** `aprobacion_7d` y `aprobacion_historica` se imputan con la mediana dentro de cada fold, para no usar información del fold de test. La falta de actividad a 7 días queda como variable propia (`sin_actividad_7d`). El ticket y los intentos nulos valen 0 (no hubo actividad).
- **Conteos con decimales:** se dejan como vienen; es un ruido chico frente a la escala de los conteos.

### Retención por tipo de pago y por monto

Fuente: `notebooks/07_reglas.ipynb` (últimas celdas). Salidas: `docs/reglas/retencion_cnp_monto.csv` y `docs/reglas/rechazo_cnp_monto.csv`.

El score es por merchant y no ve el fraude de los merchants chicos que venden sin tarjeta presente (CNP). Estas reglas miran la transacción: el tipo de pago y el monto, que están disponibles al autorizar. En lo aprobado, el fraude es 17 veces más frecuente en CNP (0,55% de las transacciones contra 0,03% en tarjeta presente) y se concentra en los montos altos.

**Rechazar por tipo de pago o monto no conviene.** El equilibrio queda entre 0,6% y 0,8%, por debajo del 1,5% que exige el criterio de la sección 5. El fraude de los montos altos pesa mucho en dólares, pero rechazar esas transacciones cuesta casi lo mismo en TPV bueno.

| Regla de rechazo | Fraude evitado (% del $) | TPV bueno rechazado | Equilibrio |
|---|---:|---:|---:|
| Monto ≥ 5.000 | 75,0% | 29,7% | 0,62% |
| Monto ≥ 20.000 | 32,3% | 12,0% | 0,66% |
| Todo CNP | 27,6% | 8,5% | 0,80% |
| CNP con monto ≥ 500 | 27,5% | 8,4% | 0,81% |

**Retener sí.** No se pierde TPV, solo se demora. Se retiene, sobre lo que queda aprobado después del rechazo, toda transacción que cumpla alguna de dos condiciones: **CNP con monto de 500 o más**, o **monto de 20.000 o más** (cualquier tipo).

| Retención | TPV retenido | Fraude retenido (% del $) | Demora promedio | Transacciones |
|---|---:|---:|---:|---:|
| Actual: revisión manual + automática | 18,3% | 33,3% | 1,28 días | 995.861 |
| Nueva: CNP con monto ≥ 500 | 8,0% | 23,7% | 0,56 días | 40.297 |
| Nueva: monto ≥ 20.000 | 11,8% | 34,0% | 0,82 días | 8.767 |
| Nuevas: ambas | 16,4% | 48,3% | 1,15 días | 47.057 |
| **Actual + nuevas** | **30,6%** | **68,6%** | **2,13 días** | 1.031.095 |

Las nuevas suman **12,3 puntos de TPV demorado y 35,3 puntos de fraude retenido** (2,9 veces), y llevan el fraude evitado o retenido del 41,5% a cerca del 72% (12,3% evitado por el rechazo, más 68,6% de lo que queda). Lo que más aporta es el monto alto: son 8.767 transacciones en tres meses (unas 100 por día), un volumen manejable para revisión manual. La regla CNP son 40.297, y toma el fraude que el score no ve: el 91% de los merchants con CNP no está en el 5% más riesgoso.

**Límites**
- Los cortes (500 y 20.000) se eligieron mirando estas mismas transacciones, igual que el resto de las reglas.
- Las transacciones no tienen fecha, así que no se puede probar estabilidad en el tiempo.
- Entre mitades de merchants el monto alto se sostiene (36,8% y 31,9% del fraude retenido); el CNP varía más (27,3% y 21,0%). Sin el merchant con más fraude (`M46B66AE…`), la retención completa pasa de 68,6% a 65,8%.
- **El fraude CNP está concentrado:** 138 merchants con fraude, y los 10 mayores suman el 58% del $ (el mayor, el 18%). Es la parte más frágil.
- Retener cuesta en operación y en demora: la regla completa demora 2,13 días en promedio el TPV bueno, contra 1,28 de la actual. Depende del supuesto de 7 días de retención, que no está en los datos.
- Cerca del 28% del $ de fraude sigue pasando sin ningún control (antes, 58,5%).

---

## Anexo · Respuestas de la empresa

La empresa respondió las dudas del challenge. Lo que confirmó:

| Tema | Respuesta |
|---|---|
| `is_fraud` y `approved` | Se pueden usar para lo que se quiera. El challenge pide separar la evaluación del modelo del impacto de negocio: si se entrena con ellas, no queda un conjunto independiente para medir las reglas, y hay que explicar cómo se resuelve |
| Transacciones | Son de **octubre a diciembre de 2025**, posteriores al dataset de merchants, y se espera que midan el impacto del modelo entrenado por merchant. Se les quitó la fecha a propósito: el score es por comercio y se une por `merchant_id` |
| Qué es una suspensión | Un cambio de estado de la cuenta decidido por el equipo de riesgo y fraude, a veces manual y a veces por reglas automáticas en batch. Puede ser por fraude confirmado, riesgo preventivo, cumplimiento o documentación. **El dataset no trae el motivo** |
| Suspensión y transacciones | Un merchant suspendido no puede generar una transacción: **no existen rechazos por suspensión** |
| 19 de 26 vuelven a operar | Es esperable. `is_suspended` es un estado que se prende y se apaga por días, no un evento único. Las agrupaciones por día son reales |
| Calendario | El dataset de merchants tiene una fila por comercio y día del 1-abr al 31-dic de 2025. `is_suspended` está informada hasta el 30-sep y es nula de octubre a diciembre: ocultaron las etiquetas de ese período para la evaluación |
| Ventana de las features | **La fila con fecha *t* no incluye la actividad de ese mismo día.** Las columnas `me_`, `ind_`, `me_network_` y `me_lifetime_` resumen la actividad hasta *t*−1 inclusive, en fecha de Ciudad de México |
| Columnas mensuales | `merchant_tenure_days`, `ratio_claims_tpv_180d`, `approved_txns_90d`, `approved_tpv_90d`, las `p*_`, `ind_ratio_claims_tpv_180d` y las `node_*` son mensuales: para cualquier día del mes *M* reflejan datos hasta el fin del mes *M*−1. Sus ventanas de 90 y 180 días son 3 y 6 meses calendario |

### Lo que cambia (a revisar en las decisiones de arriba)

- **El día de la suspensión no tiene fuga.** La fila del primer día suspendido resume actividad hasta el día anterior, cuando el merchant todavía operaba, y un suspendido no genera rechazos. Los rechazos de esa fila son previos a la suspensión. Eso deja sin fundamento la duda de "el día 0 no se usa" (decisión 1b) y el criterio de elegir "el día con más rechazos". Lo coherente es una sola regla: la foto de cada positivo es la fila de su primer día suspendido. La fila del día anterior resume actividad de dos días antes.
- **Octubre a diciembre se puntúa con la logística B, sin reentrenar con las transacciones.** Una fila por merchant que aparece en `txn_sample`: la del 1-oct, que resume la actividad hasta el 30-sep. Si ese día no tiene fila, la última con fecha ≤ 1-oct. El score se une por `merchant_id`. `is_fraud` y `approved` no entran al ajuste. Lo escribe `notebooks/05_logistica.ipynb` en `data/processed/score_octubre.parquet`.
- **Las columnas mensuales explican el "ruido".** El salto de `merchant_tenure_days` es la actualización mensual, no un error. El salto de claims de `M315DBFD11714A6D5` ocurre al cambiar de mes y puede reflejar actividad de septiembre, previa a la suspensión. El argumento "lo raro aparece después del bloqueo" no se sostiene: el motivo para dejarlo fuera es otro, que antes de la suspensión no hay actividad sospechosa (ver la decisión 1a).
- **Los motivos de suspensión están mezclados** (fraude, riesgo preventivo, cumplimiento, documentación) y el dataset no los separa. El grupo D puede ser cumplimiento o documentación, pero no se puede confirmar caso por caso.

---

## 4 · Score por merchant y join a las transacciones

**Estado:** `decidida`

Fuente: `notebooks/06_transacciones_con_score.ipynb`. El score sale de `05_logistica.ipynb` (logística B, fila del 1-oct; ver el anexo).

- **Join por `merchant_id`**, muchos a uno. Las transacciones no tienen fecha, así que cada merchant tiene un único score para todas sus transacciones de octubre a diciembre.
- **Cobertura 100%:** 7.079.949 transacciones y 40.265 merchants, sin ninguna transacción ni merchant sin score. Sale en `data/processed/transacciones_con_score.parquet` (no se versiona: `data/` está ignorado).
- **`risk_pct`:** percentil del score entre los 40.265 merchants (1 = más riesgo). **Los umbrales de las reglas se definen por percentil, no por probabilidad:** la logística se entrena con pesos balanceados y su probabilidad no está calibrada.
- **Empates en el extremo bajo:** 952 merchants (2,4%) comparten el score mínimo, así que un corte en los percentiles bajos no es preciso. El extremo alto sí se distingue.
- `is_fraud` y `approved` no entraron al modelo: quedan para medir el impacto de negocio.

**Chequeo descriptivo** (no es la evaluación de las reglas). Tabla en `docs/score/riesgo_por_tramo.csv` y gráfico en `docs/score/riesgo_por_tramo.png`:

| Tramo de riesgo | Merchants | % transacciones | % TPV | Aprobación | Fraude |
|---|---:|---:|---:|---:|---:|
| 0–50% | 20.132 | 6,1% | 17,0% | 89,5% | 0,081% |
| 50–90% | 16.106 | 55,5% | 49,2% | 94,3% | 0,025% |
| 90–95% | 2.013 | 24,4% | 14,9% | 93,7% | 0,020% |
| 95–99% | 1.611 | 11,7% | 12,8% | 92,2% | 0,093% |
| 99–100% | 403 | 2,4% | 6,1% | 86,2% | 0,098% |

- El 5% de merchants más riesgoso (2.014) concentra el 36% de los fraudes (931 de 2.600) con el 14% de las transacciones y el 19% del TPV. Su tasa de fraude es de 3,7 a 4,9 veces la de los tramos 50–95%.
- **El tramo de menor riesgo también tiene tasa alta (0,081%),** con apenas el 6% de las transacciones: son merchants chicos o con poca actividad. El score se entrenó con suspensión, no con fraude, así que no ordena todo el fraude. Hay que tenerlo en cuenta al fijar las reglas.

### Limitación del score: no ordena bien el fraude

El score ordena a los merchants con algún fraude (AUC por merchant 0,76), pero casi no ordena las transacciones fraudulentas (AUC por transacción 0,57; el azar da 0,50). El gráfico de la tasa de fraude por tramo no es monótono:

- **El tramo 0–50% tiene tasa alta (0,081%)** porque son merchants con muy poca actividad: mediana de 5 transacciones por merchant, contra 60 en el resto, y el 37% tiene entre 1 y 3. Hay 171 merchants con fraude ahí; los tres mayores son solo el 12%. Con tan pocas transacciones, un fraude mueve mucho la tasa.
- **El tramo 95–99% está inflado por un merchant:** `M46B66AE0E309595B` tiene 256 de los 767 fraudes de ese tramo. Sin él, la tasa baja de 0,093% a 0,062%.
- **Por qué:** el score se entrenó con suspensión, no con fraude, y son poblaciones casi distintas (de los 26 suspendidos, solo 1 tenía fraude alto).

**Qué implica para las reglas:** la señal está en el extremo alto (el 5% más riesgoso concentra el 36% de los fraudes), no en el resto. Una regla que mire solo el score deja pasar el fraude de merchants chicos de bajo riesgo, y la tasa observada de un merchant con pocas transacciones no es confiable. Por eso las reglas de la sección 5 combinan el score con el volumen del merchant.

---

## 5 · Reglas de pre y post autorización

**Estado:** `decidida`, con límites declarados abajo.

Fuente: `notebooks/07_reglas.ipynb`. Salidas en `docs/reglas/`: `kpis_reglas.csv`, `tabla_de_cortes.csv`, `chequeos.csv`, `segmentos.csv`, `reglas.png` y `trade_off.png`.

Las transacciones (octubre a diciembre) no se usaron para entrenar: son el conjunto independiente para medir el impacto. Las reglas combinan `risk_pct` con el **volumen** del merchant (intentos en los 30 días hasta el 30-sep: sin actividad, baja 1–9, media 10–99, alta 100 o más), porque el score ordena el fraude solo en parte (sección 4). En volumen alto casi no separa: con riesgo ≥ 95% la tasa de fraude es 0,05%–0,07%, contra 0,4%–1,1% en volumen bajo y medio.

**Supuestos que no están en los datos:** margen de Clip sobre el TPV de 1% y retención de pagos de 7 días.

### Las reglas

| Etapa | Regla | Por qué |
|---|---|---|
| **Pre-autorización: rechazar** | `risk_pct` ≥ 0,90 y volumen **bajo**; o `risk_pct` ≥ 0,99 y volumen **medio** | Donde el score discrimina y el equilibrio supera con holgura el margen (1,56% y 2,34% contra 1%) |
| | No se rechaza a volumen **alto** | El score no separa el fraude y rechazar costaría mucho TPV bueno (con corte 0,95: 12% del TPV bueno para evitar 21% del fraude) |
| **Post-autorización: retener** | **Revisión manual:** volumen alto y `risk_pct` ≥ 0,95 | 244 merchants y mucho TPV: se retiene y se revisa, sin rechazar |
| | **Retención automática de 7 días:** merchants sin actividad en 30 días | El score los pone en el riesgo más bajo (el coeficiente de `sin_actividad_7d` es negativo), pero su tasa de fraude es 2,4 veces el promedio |
| | El resto se paga de inmediato | |

**Criterio de los cortes:** una regla de rechazo debe tener un *equilibrio* (fraude evitado en $ / TPV bueno rechazado) de al menos 1,5%, con el margen supuesto de 1%. Los cortes se compararon en `tabla_de_cortes.csv`.

### KPIs de negocio

Sobre lo aprobado (6.615.456 transacciones, TPV $3.000 millones, fraude $7,4 millones = 0,247% del TPV):

| | Merchants | Resultado |
|---|---:|---|
| **Pre: rechazar** | 1.477 | 19.193 transacciones rechazadas. **Fraude evitado: 165 transacciones, 12,3% del $.** TPV bueno rechazado: **1,78%**. Equilibrio 1,71%. Beneficio neto de $0,38 millones con el margen supuesto. Aprobación final 93,17% (antes 93,44%) |
| **Post: revisión manual** | 244 | Retiene el 12,5% del TPV y el **23,6% del $ de fraude**. Demora promedio de 0,88 días |
| **Post: retención automática** | 8.620 | Retiene el 5,7% del TPV y el 9,7% del $ de fraude. Demora de 0,40 días |
| **Post: ambas** | 8.864 | Retiene el 18,3% del TPV y el **33,3% del $ de fraude**. Demora de 1,28 días |

Entre rechazo y retención se evita o retiene el **41,5% del $ de fraude**. El **58,5% pasa sin ningún control**.

### Trade-off (`docs/reglas/trade_off.png`)

El gráfico barre el corte de `risk_pct` y muestra cuánto fraude se gana y cuánto TPV se paga.

- **Rechazar, volumen bajo y medio:** las curvas quedan por encima de la línea de equilibrio con margen de 1%. La regla elegida (1,78% del TPV bueno rechazado, 12,3% del fraude evitado) cae entre las líneas de equilibrio de 1% y 2%: rechazar conviene si el margen de Clip es menor que 1,71%.
- **Rechazar, volumen alto:** la curva queda muy por debajo de las líneas de equilibrio y cerca del azar. Por eso no se rechaza.
- **Retener:** las curvas quedan apenas por encima del azar (retener por riesgo el 15,6% del TPV capta el 26,7% del fraude). La mejora de "ambas" (18,3% del TPV, 33,3% del fraude) viene sobre todo de sumar a los merchants sin actividad, que el score no ve. Retener no pierde TPV, solo lo demora, así que no hay un equilibrio en dinero: el costo es la demora y la carga de revisión.

### Segmentos para los emisores (bonus)

Se agrupan las transacciones aprobadas en tres paquetes: **alto** (merchants de las reglas de rechazo o `risk_pct` ≥ 0,99), **medio** (revisión, retención automática o `risk_pct` entre 0,95 y 0,99) y **bajo** (el resto).

| Segmento | Merchants | % transacciones | % TPV | Tasa de fraude |
|---|---:|---:|---:|---:|
| Bajo | 28.360 | 84,0% | 77,0% | 0,026% |
| Medio | 9.582 | 13,6% | 16,2% | 0,090% |
| Alto | 1.411 | 2,4% | 6,8% | 0,146% |

El paquete principal (bajo) tiene una tasa de fraude 31% menor que el promedio de lo aprobado (0,037%). Mandar los otros dos paquetes por separado cuida la reputación frente a los emisores.

### Chequeos y límites

- **Estabilidad por mitades de merchants.** Pre: el fraude evitado es de 11,2% y 13,4%, y el equilibrio de 1,35% y 2,20%, así que la regla de rechazo se sostiene. **Post: el fraude retenido va de 45,0% a 21,5%.** La retención depende de pocos merchants.
- **Sin el merchant con más fraude** (`M46B66AE0E309595B`): el rechazo no cambia (13,3% del $) y la retención baja de 33,3% a 27,4%.
- **La estructura y los cortes se eligieron mirando estas mismas transacciones.** Las mitades muestran estabilidad, pero no son una evaluación independiente: no hay otro período.
- **El equilibrio de 1,71% depende del margen supuesto.** Si el margen real de Clip es mayor, rechazar deja de convenir.
- **El 58,5% del $ de fraude pasa sin control:** fraude de merchants de volumen alto o de riesgo bajo que el score no ve.
- **Hallazgo sin usar:** el fraude es 13 veces más frecuente en transacciones sin tarjeta presente (CNP: 0,41%, contra 0,03% en tarjeta presente), y pesa más en monto (mediana de $684 contra $147). No es parte del score por merchant y no se usó en las reglas. Una regla por tipo de pago o por monto es la mejora más clara.
