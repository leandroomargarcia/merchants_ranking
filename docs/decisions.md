# Decisiones del proyecto

Cada decisión registra la pregunta, lo que sabemos, las opciones, cómo se elige y qué se decidió.

---

## 1 · Target: cómo lo construimos a partir de `is_suspended`

**Pregunta.** El modelo ordena a los merchants activos: ¿cuáles se parecen a los que una revisión termina suspendiendo? Devuelve un ranking, no la probabilidad de que la suspensión caiga en un plazo.

`is_suspended` es el estado diario, no un evento. Para armar el target hay que definir quién entra y qué día se mira de cada merchant.

### Lo que sabemos

Fuente: `notebooks/02_suspensiones.ipynb`, `docs/suspensiones.png`, `docs/suspensiones_dia_a_dia.md`, `docs/variables/`.


| Hecho                                                             | Valor                                                                                                                                                                                            |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Merchants alguna vez suspendidos                                  | 26                                                                                                                                                                                               |
| Con inicio observado (un día activo seguido de un día suspendido) | 19, todos con un solo inicio                                                                                                                                                                     |
| Sin inicio observado                                              | 7: 3 ya están suspendidos el 1-abr (primer día de datos); 3 aparecen en los datos ya suspendidos, 1 día, y vuelven a operar; 1 (`MEBF6C9…`) tiene 153 días de estado nulo antes de su suspensión |
| Duración de la suspensión (los 19)                                | de 1 a 92 días                                                                                                                                                                                   |
| Vuelven a operar                                                  | 19 de 26                                                                                                                                                                                         |
| Período con etiqueta                                              | abril–septiembre; octubre–diciembre `is_suspended` es 100% nulo                                                                                                                                  |


Cómo se ven los 19 inicios día por día (lectura de los gráficos; es una interpretación sobre pocos casos):


| Patrón                                                               | Merchants | Qué implica                                                                           |
| -------------------------------------------------------------------- | --------- | ------------------------------------------------------------------------------------- |
| Ráfaga de intentos y caída de aprobación 1 a 3 días antes del inicio | ~7        | Hay señal, pero con muy poca anticipación                                             |
| Rechazos solo en la fila del primer día suspendido                   | ~4        | Es actividad del día anterior, previa a la suspensión: no hay rechazos por suspensión |
| Sin nada raro antes (casi sin actividad o recién dados de alta)      | ~8        | Su comportamiento no anticipa la suspensión                                           |


Otros hechos que pesan en la decisión:

- Varias suspensiones caen el mismo día (4 el 23-ago, 2 el 11-sep, 2 el 22-abr): parecen revisiones en lote.
- Los merchants suspendidos operan muy poco: de 0 a 2 transacciones por día.
- Mientras el merchant está suspendido, sus features reflejan la suspensión (volumen en cero; no hay rechazos por suspensión).

### Decisión 1a · Qué casos entran

**Clasificación de los 19 inicios observados** según su comportamiento antes de la suspensión. Es una lectura de los gráficos de `docs/suspensiones.png` y `docs/variables/` (de 30 días antes a 10 después), no una regla automática. Antes se descartaron las variables que no se mueven antes del inicio: red, canales open platform / pocket / recurrente / SPEI, flags, MCC, claims y contracargos. Los percentiles de industria describen a la industria, no al merchant.


| Grupo                                             | Merchant            | Inicio | Qué se ve antes                                                                                                                                                    | Entra      | Motivo                                                                      |
| ------------------------------------------------- | ------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | --------------------------------------------------------------------------- |
| **A · Deterioro durante días o semanas**          | `MA52FE20E571E5DCA` | 15-jul | Ticket de 5.000 a 12.000, volumen creciente, aprobación por TPV de 90% a 27% en 3 semanas                                                                          | Se incluye | Hay deterioro visible antes del inicio                                      |
|                                                   | `M2971A5DEEED25520` | 23-ago | Intentos a 30 días de 10 a 90 en un mes, aprobación por TPV cerca de 0                                                                                             | Se incluye | Hay deterioro visible antes del inicio                                      |
|                                                   | `M331E694E1EECD91A` | 23-ago | Intentos en crecimiento continuo; el día −7 el ticket se duplica y la aprobación baja a 75%                                                                        | Se incluye | Hay deterioro visible antes del inicio                                      |
|                                                   | `MB351DC99D4064375` | 22-abr | Aprobación ~10%; salto de intentos el día −10                                                                                                                      | Se incluye | Hay deterioro visible antes del inicio                                      |
|                                                   | `MC45B94C90F7E27CD` | 25-jul | Aprobación ~30% a 30 días; el 4-jul (día −21) 9 intentos y 7 rechazados                                                                                            | Se incluye | Hay deterioro visible antes del inicio                                      |
|                                                   | `M46B66AE0E309595B` | 14-jun | Aparece 4 días antes y pasa de 4 a 43 intentos con aprobación en caída; 256 fraudes en transacciones                                                               | Se incluye | Hay deterioro visible antes del inicio                                      |
| **B · Ráfaga corta de rechazos 1 a 3 días antes** | `MB2ECDC9763AB4E54` | 23-ago | Pico de intentos y aprobación de 100% a 44% en los días −2 y −1                                                                                                    | Se incluye | Hay una ráfaga de rechazos antes del inicio                                 |
|                                                   | `MEB47BBD227407EA5` | 23-ago | 6 intentos rechazados el día −1                                                                                                                                    | Se incluye | Hay una ráfaga de rechazos antes del inicio                                 |
|                                                   | `MFECEBCA0010F3873` | 22-abr | Intentos rechazados el día −1, ticket salta a 5.500                                                                                                                | Se incluye | Hay una ráfaga de rechazos antes del inicio                                 |
|                                                   | `M7F9108A69784504E` | 29-abr | El 26-abr (día −3) 3 intentos, los 3 rechazados, ticket ~17                                                                                                        | Se incluye | Hay una ráfaga de rechazos antes del inicio                                 |
| **C · Merchant nuevo, todo pasa el día 0**        | `MF6C1EBC5EE0342DB` | 29-may | Aparece el día −1; su red suma un suspendido el día 0                                                                                                              | Se incluye | Los intentos están en la fila de la foto                                    |
|                                                   | `MED4E6F81B6D71742` | 1-jul  | Primera transacción online, de 10.000, rechazada                                                                                                                   | Se incluye | Los intentos están en la fila de la foto                                    |
| **D · Sin riesgo visible antes**                  | `MEA1EE5CA0A25553F` | 12-sep | Cuenta nueva (12 días), 0 transacciones de por vida                                                                                                                | Se quita   | Nunca operó                                                                 |
|                                                   | `M002D3A13420843D9` | 7-abr  | Solo 6 días de historia (los datos empiezan el 1-abr)                                                                                                              | Se quita   | Censura al inicio: la historia empieza con el dataset                       |
| **Fuera del análisis**                            | `ME9BD50FCE697D4A7` | 25-jun | Historial de aprobación normal y después silencio. No cuenta como caso                                                                                             | Se quita   | Aprobación normal y después silencio: no hay actividad que aprender         |
|                                                   | `MF000975FCCCF8010` | 11-sep | Antigüedad 0, una transacción de alrededor de 1 peso. No cuenta como caso                                                                                          | Se quita   | Una transacción de alrededor de 1 peso, sin actividad sospechosa            |
|                                                   | `M486B452BBACA6CA4` | 11-sep | Casi inactivo: dos transacciones online de ~3.000, aprobadas. Probablemente lo suspendió el propio merchant. No cuenta como caso                                   | Se quita   | Casi inactivo y transacciones aprobadas; probable autosuspensión            |
|                                                   | `M3182C56E1D13EC33` | 24-sep | No hizo nada raro. Lo suspendieron por otro motivo. No cuenta como caso                                                                                            | Se quita   | No hay actividad sospechosa                                                 |
|                                                   | `M315DBFD11714A6D5` | 30-sep | Antes del bloqueo: 1 a 3 intentos por día, todos aprobados, y 4 días sin actividad. No hay actividad sospechosa. Los claims (mensuales) saltan el 1-oct, de 0 a 17 | Se quita   | No hay actividad sospechosa; el salto de claims es la actualización mensual |


### Decisión 1b · ¿Cómo se construyó la target?

Cada merchant tiene una fila por día. El modelo ve **una sola**: la foto. En esa fila, **positivo** es `target = 1` y **negativo** es `target = 0`.

**Positivo (**`target = 1`**): el primer día suspendido de uno de los 12 que entran.** La empresa confirmó que la fila con fecha *t* resume la actividad hasta el día anterior, y que un merchant suspendido no puede transaccionar. Los números de esa fila son, entonces, los del último día en que todavía operaba. No hay rechazos causados por la suspensión.

Usar el día anterior perdería ese último día: esa fila resume la actividad de dos días antes. El día tampoco se elige mirando qué fila tiene más rechazos. Con esta regla, en la foto se ven los 7 rechazos de `M46B66A`, los 15 de `MA52FE2`, los 6 de `MB351DC`, y los intentos de `MED4E6F` y `MF6C1EB`.

La regla sin entrenar mira tres cosas de esa foto y se queda con la peor: la aprobación entre 7 y 30 días, los rechazos de la propia foto, y el peor día de rechazos en los 30 días que terminan en la foto. Ese pico es el que alcanza a `MC45B94` (7 rechazos el 4-jul, 21 días antes) y a `M7F9108` (3 rechazos el 26-abr, 3 días antes).

**Negativo (**`target = 0`**): un merchant que no fue suspendido entre abril y septiembre, en un día en el que estaba activo.** La fecha se sortea con la misma mezcla de días que las fotos de los positivos (semilla 0), para que el calendario no se vuelva la señal. Agosto pesa un poco más entre los negativos porque ese mes hay más merchants con fila. Si un merchant no tiene ningún día activo en esas fechas, queda afuera: 6.739 de los 40.239 que nunca se suspendieron. Entran 33.500.

**No se usa el resto de la historia.**

- Los días posteriores al primer día suspendido: el merchant ya está bloqueado y las features muestran volumen en cero.
- Los días con estado nulo.
- Octubre a diciembre: las etiquetas de ese período están ocultas.
- Los días anteriores a la foto, como filas extra de un merchant sano. Si en uno de esos días el merchant ya se veía mal y lo etiquetáramos como negativo, el modelo aprendería que ese perfil es sano.

---

## 2 · Validación: cómo separamos train y test

El corte es un `StratifiedKFold` de 5 folds, repetido con 10 semillas (`notebooks/05_logistica.ipynb`). Cada merchant se predice con un modelo entrenado sin él. Hay una fila por merchant, así que el train y el test de un fold no comparten ninguno. Se repite con varias semillas porque un solo corte deja el test con unos 2 o 3 positivos (`target = 1`), de 12, y con tan pocos el ranking cambia según cómo se parta.

La métrica es el ranking en el test: PR-AUC y recall en el top 1% y 5%. Con ese tamaño de test el número es ruidoso.

Este mismo corte se usa para comparar los sets de variables. Cuál quedó elegido está en la sección 3.

---

## 3 · Features: cuáles se arman y cuáles entran al modelo

Los pasos de abajo dicen cómo se eligieron esas columnas. 

### Paso 1 · Fuera por definición


| Columna                                                                                                     | Motivo                                                                                                                                      |
| ----------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `me_merchant_created_at_timestamp`                                                                          | Fecha; la antigüedad ya está en `merchant_tenure_days`                                                                                      |
| `config_segment`                                                                                            | 100% nula                                                                                                                                   |
| `me_attempted_pocket_txn_sum_7d`                                                                            | Un solo valor en todo el dataset                                                                                                            |
| `me_industry_mcc`                                                                                           | Igual a `industry_mcc` en el 99,7% de las filas. `industry_mcc` se queda en este paso y sale después, con el resto de la industria en crudo |
| `vip0_segment`                                                                                              | No nula exactamente donde `is_veto_vip0` no es nula: repite el flag                                                                         |
| `me_suspended_merchants_in_network`, `me_suspended_merchants_in_network_share`                              | Fuga: suben justo cuando se suspende el propio merchant y bajan cuando vuelve                                                               |
| `is_closed_won_ever`, `is_veto_vip0`, `is_veto_config`, `is_portfolio_managed`                              | 97–99% nulas en esa medición; casi ningún positivo (`target = 1`) tenía dato                                                                |
| `me_lifetime_chargeback_tpv`, `me_lifetime_dispute_tpv`                                                     | 97–99% nulas en esa medición; casi ningún positivo (`target = 1`) tenía dato                                                                |
| `me_network_size`, `me_network_avg_tpv_approval_rate`                                                       | Solo 2 a 4 suspendidos tienen red. No alcanza para medir                                                                                    |
| `me_network_avg_tpv_chargeback_rate`, `me_network_avg_tpv_dispute_rate`                                     | Casi no tienen valores en los suspendidos                                                                                                   |
| `me_attempted_openplatform_txn_sum_7d`, `me_attempted_recurring_txn_sum_7d`, `me_attempted_spei_txn_sum_7d` | Los canales open platform, recurrente y SPEI casi no tienen valores en los suspendidos                                                      |
| `ratio_claims_tpv_180d`                                                                                     | Casi no tiene valores en los suspendidos                                                                                                    |


### Paso 2 · Qué se mueve al acercarse la suspensión

Acá se buscan features que cambien en los días anteriores a la suspensión. Para cada una se sigue la mediana de los 13 positivos de esa medición, desde 60 días antes hasta el día anterior, y se la compara con el rango habitual (percentil 25 a 75) de los merchants que nunca se suspendieron. Si al acercarse el día la mediana sale de ese rango, la feature se mueve.


| Lectura                                     | Features                                                                                                                                                                                                                                                                                                                          |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Se mueven al acercarse la suspensión**    | Tasas de aprobación (por transacción y por TPV, a 1, 7, 15 y 30 días): caen bajo el rango normal en los últimos ~20 días, más fuerte cerca del día −1 (a 7 días, de ~100% a 30–50%). Ticket promedio a 7 y 30 días: por encima del rango y en aumento. Intentos a 7, 15 y 30 días: suben al final, aunque dentro del rango normal |
| **Distintas en nivel, pero no se mueven**   | Aprobación histórica (`me_lifetime_*_approval_rate`: 0,6–0,7 contra 0,9 normal) y ticket histórico. Describen cómo es el merchant, no un cambio                                                                                                                                                                                   |
| **Sin señal**                               | Transacciones aprobadas (1, 7, 15, 30 y 90 días), TPV aprobado                                                                                                                                                                                                                                                                    |
| **Sin variación ni datos en los positivos** | `me_attempted_openplatform_txn_sum_7d`, `me_attempted_recurring_txn_sum_7d`, `me_attempted_spei_txn_sum_7d`, `me_network_avg_tpv_chargeback_rate`, `me_network_avg_tpv_dispute_rate`, `ratio_claims_tpv_180d`, flags `is_`*, contracargos y disputas acumulados (solo 1 positivo tiene datos)                                     |
| **Describen la industria, no al merchant**  | `industry_mcc`, `ind_`* y percentiles `p*_approved_*_90d`: sus saltos vienen de qué merchants entran en la mediana                                                                                                                                                                                                                |
| **Ruido o artefacto**                       | `merchant_tenure_days` (salta sin sentido), `me_network_size` y `me_network_avg_tpv_approval_rate` (solo 4 positivos tienen red; el salto del final es un merchant que entra)                                                                                                                                                     |
| **La mediana no sirve**                     | `me_attempted_ecommerce_txn_sum_7d`: la mayoría vale 0, así que la mediana esconde a los pocos que sí tienen ecommerce. Hay que mirarla con la proporción de merchants con valor > 0                                                                                                                                              |


### Paso 3 · Qué le sirve al modelo: el cambio, o cómo es el merchant

Esto se hizo para elegir qué mirar. Había dos posibilidades. Una: el merchant empeora en el mes anterior a la suspensión, y entonces hay que medir el cambio. La otra: ya era distinto de los que nunca se suspenden, y entonces alcanza con cómo está en la foto.

El paso anterior miraba los gráficos día por día. Acá se compara el mes justo antes de la suspensión con el mes anterior a ese, y con 3.000 merchants que nunca se suspendieron, a los que se les puso una fecha al azar para poder hacer la misma cuenta. Había pocos casos: 7 con historia suficiente para ver si cambiaron, y unos 11 con datos en el mes previo. Con tan pocos, los números sirven para ordenar las ideas.

- **El cambio.** En los 7 que tenían historia suficiente, ninguna variable cambia fuerte de un mes al otro. La aprobación baja, pero menos de lo que parecía en los gráficos: la caída está en los últimos días y un promedio de 30 días la esconde. En el grupo A ese deterioro ya había empezado en el mes más lejano.
- **Cómo es el merchant.** En el mes previo, los que se suspendieron aprueban menos que los demás, ahora y en toda su historia. También mueven montos más altos y tienen algo más de intentos.

La medición decidió qué no construir. No se arma una variable de "cuánto cambió respecto de sí mismo": de un mes al otro ese cambio no aparece, y un promedio de 30 días esconde la caída de los últimos días. Lo que sí se lleva a la foto es la diferencia contra los demás: aprueba menos y mueve montos más altos. Por eso el modelo mira el nivel del merchant ese día, no su cambio.

### Embudo

El panel trae 71 columnas. Tres no son features: `merchant_id`, `observation_date` e `is_suspended`. El embudo empieza con las otras 68, deja 25 en `features.parquet` y de esas entrena con 4.


| Paso                        | Qué sale                                                                                                                                                                | Salen | Quedan |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- | ------ |
| Panel                       | —                                                                                                                                                                       | —     | 68     |
| 1. Por definición           | Las 21 de la tabla de arriba: fecha, vacías, duplicadas, fuga, flags, contracargos, red, canales sin datos y claims                                                     | 21    | 47     |
| 2. Industria en crudo       | `industry_mcc`, las `ind_*` y los percentiles `p*_`. Con 13 suspendidos el modelo aprendería el rubro. Entran 3 columnas nuevas: el merchant contra su propia industria | 15    | 35     |
| 3. Aprobadas que no separan | Conteos y TPV aprobado (`me_approved_txn_sum_*`, `me_approved_tpv_sum_30d`, `approved_txns_90d`, `me_lifetime_approved_tpv`). El tamaño queda en dos columnas           | 7     | 28     |
| 4. Canal y antigüedad       | De dónde vino el merchant (3) y `merchant_tenure_days`, que salta al cambiar de mes y no describe el día                                                                | 4     | 24     |
| 5. Pico de rechazos         | Entra `max_rechazos_1d_30d`: el peor día de rechazos en los 30 días hasta la foto. Abajo está la lista                                                                  | —     | 25     |


En el paso 5 quedan estas 25, que son las de `features.parquet`:


| Grupo                    | Features                                                                                           |
| ------------------------ | -------------------------------------------------------------------------------------------------- |
| Aprobación actual (8)    | `me_txn_approval_rate_1d`, `_7d`, `_15d`, `_30d`; `me_tpv_approval_rate_1d`, `_7d`, `_15d`, `_30d` |
| Aprobación histórica (2) | `me_lifetime_txn_approval_rate`, `me_lifetime_tpv_approval_rate`                                   |
| Ticket (4)               | `me_attempted_tpv_avg_1d`, `_7d`, `_30d`; `me_lifetime_tpv_avg`                                    |
| Intentos (4)             | `me_attempted_txn_sum_1d`, `_7d`, `_15d`, `_30d`                                                   |
| Ecommerce (1)            | `me_attempted_ecommerce_txn_sum_7d`                                                                |
| Tamaño (2)               | `approved_tpv_90d`, `me_lifetime_approved_txn`                                                     |
| Contra su industria (3)  | `tpv_vs_industry_p50`, `tpv_vs_industry_p95`, `txns_vs_industry_p50`                               |
| Pico de rechazos (1)     | `max_rechazos_1d_30d`                                                                              |


Desde esas 25 se sigue descartando:


| Paso                             | Qué sale                                                                                                                                                                                                                                     | Salen | Quedan |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----- | ------ |
| 5                                | Las 25 de arriba                                                                                                                                                                                                                             | —     | 25     |
| 6. Aprobación repetida           | Las 8 tasas y la histórica por TPV dicen lo mismo. La de 1 día está vacía casi siempre, la de 15 queda entre la de 7 y la de 30, y las de TPV se mueven con las de transacción. Quedan la de 7 días, la de 30 y la histórica por transacción | 7     | 18     |
| 7. Ticket e intentos             | Se probaron sumados a la aprobación y al pico. El PR-AUC sube de 0,022 a 0,023 y el top 5% baja de 10,6 a 9,9, con más variación entre cortes. No entran                                                                                     | 8     | 10     |
| 8. Ecommerce, tamaño e industria | Con 12 positivos, columnas de más que en la lectura separaban poco. No entran                                                                                                                                                                | 6     | 4      |


Con 12 positivos no se entrena con las 25: varias cuentan la misma historia. La que importa, según el paso 3, es que el merchant aprueba menos que los demás y que tuvo un día de muchos rechazos. Por eso el set que entrena es la aprobación (la peor entre 7 y 30 días, más la histórica), un indicador de si hubo actividad a 7 días, y el pico de rechazos. Ese set pone 10,6 de los 12 en el top 5%. Solo la aprobación llega a 8,3.

Quedan cuatro columnas y con esas se arman las cuatro que entrenan:


| Entrenan               | Cómo se arma                                                                                           |
| ---------------------- | ------------------------------------------------------------------------------------------------------ |
| `aprobacion_7d`        | El mínimo entre `me_txn_approval_rate_7d` y `_30d`. Si las dos faltan, `me_lifetime_txn_approval_rate` |
| `aprobacion_historica` | `me_lifetime_txn_approval_rate`                                                                        |
| `sin_actividad_7d`     | 1 si `me_txn_approval_rate_7d` es nula                                                                 |
| `log_max_rechazos_30d` | El log de `max_rechazos_1d_30d`                                                                        |


---

## 4 · Modeling

Con las 25 columnas se arman tres sets y se comparan con el corte de la sección 2 (`notebooks/05_logistica.ipynb`).


| Set               | Variables                                                                | PR-AUC | Top 5% (de 12) |
| ----------------- | ------------------------------------------------------------------------ | ------ | -------------- |
| A                 | `aprobacion_7d`                                                          | 0,0060 | 8,3            |
| **B, la elegida** | A + `aprobacion_historica` + `sin_actividad_7d` + `log_max_rechazos_30d` | 0,0216 | 10,6           |
| C                 | B + `log_ticket_30d` + `log_intentos_7d`                                 | 0,0234 | 9,9            |


C sube el PR-AUC y captura menos positivos en el top 5%, con más desvío entre semillas. Se queda B.


| Variable de B          | Definición                                                                                                             |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `aprobacion_7d`        | Mínimo entre la aprobación por transacción a 7 y a 30 días. Si las dos faltan, la aprobación histórica por transacción |
| `aprobacion_historica` | `me_lifetime_txn_approval_rate`                                                                                        |
| `sin_actividad_7d`     | 1 si la tasa a 7 días es nula                                                                                          |
| `log_max_rechazos_30d` | Log de `max_rechazos_1d_30d`                                                                                           |


### Resultados

Fuente del join: `notebooks/06_transacciones_con_score.ipynb`. El score sale de la logística B, en la fila del 1-oct.

El puesto va de 0 a 1. **1 es el más riesgoso.** Acá se miran las transacciones de octubre a diciembre según ese puesto, para ver si el fraude queda cerca del 1. La tabla está en `docs/score/riesgo_por_tramo.csv`.


| Puesto (`risk_pct`)                   | Merchants | % de las transacciones | % del dinero | % de los fraudes | Aprobación | Tasa de fraude |
| ------------------------------------- | --------- | ---------------------- | ------------ | ---------------- | ---------- | -------------- |
| 0–0,50                                | 20.132    | 6,1%                   | 17,0%        | 13,4%            | 89,5%      | 0,081%         |
| 0,50–0,90                             | 16.106    | 55,5%                  | 49,2%        | 37,5%            | 94,3%      | 0,025%         |
| 0,90–0,95                             | 2.013     | 24,4%                  | 14,9%        | 13,3%            | 93,7%      | 0,020%         |
| 0,95–0,99                             | 1.611     | 11,7%                  | 12,8%        | 29,5%            | 92,2%      | 0,093%         |
| 0,99–1, el percentil 1                | 403       | 2,4%                   | 6,1%         | 6,3%             | 86,2%      | 0,098%         |
| **0,95–1, suma de las dos de arriba** | **2.014** | **14,1%**              | **18,9%**    | **35,8%**        |            |                |


La tasa no baja de corrido hacia el 0. En 0–0,50 hay merchants con muy pocas transacciones (mediana de 5, contra 60 en el resto; el 37% tiene entre 1 y 3), y un fraude les mueve la tasa. En 0,95–0,99, `M46B66AE0E309595B` tiene 256 de los 767 fraudes; sin él la tasa pasa de 0,093% a 0,062%. El score se entrenó con suspensiones, y de los 26 suspendidos solo 1 tenía mucho fraude. Por eso las reglas de la sección 5 no miran solo el puesto: lo cruzan con el volumen del merchant.

---

## 5 · Reglas de pre y post autorización

Fuente: `notebooks/07_reglas.ipynb`. 

Las transacciones (octubre a diciembre) no se usaron para entrenar: son el conjunto independiente para medir el impacto. Las reglas combinan `risk_pct` con el **volumen** del merchant (intentos en los 30 días hasta el 30-sep: sin actividad, baja 1–9, media 10–99, alta 100 o más), porque el score ordena el fraude solo en parte (sección 4). En volumen alto casi no separa: con riesgo ≥ 95% la tasa de fraude es 0,05%–0,07%, contra 0,4%–1,1% en volumen bajo y medio. El cruce está en `docs/reglas/tasa_por_volumen.png`.

**Supuestos que no están en los datos:** margen de Clip sobre el TPV de 1% y retención de pagos de 7 días.

### Las reglas


| Etapa                          | Regla                                                                         | Por qué                                                                                                                                     |
| ------------------------------ | ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| **Pre-autorización: rechazar** | `risk_pct` ≥ 0,90 y volumen **bajo**; o `risk_pct` ≥ 0,99 y volumen **medio** | Donde el score discrimina y el equilibrio supera con holgura el margen (1,56% y 2,34% contra 1%)                                            |
|                                | No se rechaza a volumen **alto**                                              | El score no separa el fraude y rechazar costaría mucho TPV bueno (con corte 0,95: 12% del TPV bueno para evitar 21% del fraude)             |
| **Post-autorización: retener** | **Revisión manual:** volumen alto y `risk_pct` ≥ 0,95                         | 244 merchants y mucho TPV: se retiene y se revisa, sin rechazar                                                                             |
|                                | **Retención automática de 7 días:** merchants sin actividad en 30 días        | El score los pone en el riesgo más bajo (el coeficiente de `sin_actividad_7d` es negativo), pero su tasa de fraude es 2,4 veces el promedio |
|                                | El resto, por el score                                                        | Se paga de inmediato. Las retenciones por tipo de pago y por monto están más abajo                                                          |


**Criterio de los cortes:** una regla de rechazo debe tener un *equilibrio* (fraude evitado en $ / TPV bueno rechazado) de al menos 1,5%, con el margen supuesto de 1%. Los cortes se compararon en `tabla_de_cortes.csv`.

### KPIs de negocio

Sobre lo aprobado (6.615.456 transacciones, TPV $3.000 millones, fraude $7,4 millones = 0,247% del TPV):


|                                | Merchants | Resultado                                                                                                                                                                                                                              |
| ------------------------------ | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Pre: rechazar**              | 1.477     | 19.193 transacciones rechazadas. **Fraude evitado: 165 transacciones, $0,91 millones (12,3% del $).** TPV bueno rechazado: **$53,2 millones (1,78%)**. Equilibrio 1,71%. Beneficio neto de $0,38 millones con el margen supuesto. Aprobación final 93,17% (antes 93,44%) |
| **Post: revisión manual**      | 244       | Retiene el 12,5% del TPV y el **23,6% del $ de fraude**. Demora promedio de 0,88 días                                                                                                                                                  |
| **Post: retención automática** | 8.620     | Retiene el 5,7% del TPV y el 9,7% del $ de fraude. Demora de 0,40 días                                                                                                                                                                 |
| **Post: ambas**                | 8.864     | Retiene el 18,3% del TPV y el **33,3% del $ de fraude**. Demora de 1,28 días                                                                                                                                                           |


Entre rechazo y retención se evita o retiene el **41,5% del $ de fraude**. El **58,5% pasa sin ningún control**.

### Trade-off (`docs/reglas/trade_off.png`)

El gráfico barre el corte de `risk_pct` y muestra cuánto fraude se gana y cuánto TPV se paga.

- **Rechazar, volumen bajo y medio:** las curvas quedan por encima de la línea de equilibrio con margen de 1%. La regla elegida (1,78% del TPV bueno rechazado, 12,3% del fraude evitado) cae entre las líneas de equilibrio de 1% y 2%: rechazar conviene si el margen de Clip es menor que 1,71%.
- **Rechazar, volumen alto:** la curva queda muy por debajo de las líneas de equilibrio y cerca del azar. Por eso no se rechaza.
- **Retener:** las curvas quedan apenas por encima del azar (retener por riesgo el 15,6% del TPV capta el 26,7% del fraude). La mejora de "ambas" (18,3% del TPV, 33,3% del fraude) viene sobre todo de sumar a los merchants sin actividad, que el score no ve. Retener no pierde TPV, solo lo demora, así que no hay un equilibrio en dinero: el costo es la demora y la carga de revisión.

### Segmentos para los emisores (bonus)

Se agrupan las transacciones aprobadas en tres paquetes: **alto** (merchants de las reglas de rechazo o `risk_pct` ≥ 0,99), **medio** (revisión, retención automática o `risk_pct` entre 0,95 y 0,99) y **bajo** (el resto).


| Segmento | Merchants | % transacciones | % TPV | Tasa de fraude |
| -------- | --------- | --------------- | ----- | -------------- |
| Bajo     | 28.360    | 84,0%           | 77,0% | 0,026%         |
| Medio    | 9.582     | 13,6%           | 16,2% | 0,090%         |
| Alto     | 1.411     | 2,4%            | 6,8%  | 0,146%         |


El paquete principal (bajo) tiene una tasa de fraude 31% menor que el promedio de lo aprobado (0,037%). Mandar los otros dos paquetes por separado cuida la reputación frente a los emisores.

### Retención por tipo de pago y por monto

El score es por merchant y no ve el fraude de los merchants chicos que venden sin tarjeta presente (CNP). Estas reglas miran la transacción: el tipo de pago y el monto, que están disponibles al autorizar.

En lo aprobado, el fraude es el 0,55% de las transacciones CNP y el 0,032% de las de tarjeta presente: unas 17 veces. Sobre todas las transacciones, incluidas las rechazadas, la relación es de unas 13 veces (0,41% contra 0,032%). El monto mediano de una transacción CNP es de unos $1.330, contra $144 en tarjeta presente.

**Rechazar por tipo de pago o monto no conviene.** El equilibrio queda entre 0,6% y 0,8%, por debajo del 1,5% del criterio de arriba. El fraude de los montos altos pesa mucho en dólares, pero rechazar esas transacciones cuesta casi lo mismo en TPV bueno.


| Regla de rechazo    | Fraude evitado (% del $) | TPV bueno rechazado | Equilibrio |
| ------------------- | ------------------------ | ------------------- | ---------- |
| Monto ≥ 5.000       | 75,0%                    | 29,7%               | 0,62%      |
| Monto ≥ 20.000      | 32,3%                    | 12,0%               | 0,66%      |
| Todo CNP            | 27,6%                    | 8,5%                | 0,80%      |
| CNP con monto ≥ 500 | 27,5%                    | 8,4%                | 0,81%      |


**Retener sí.** No se pierde TPV, solo se demora. Se retiene, sobre lo que queda aprobado después del rechazo, toda transacción que cumpla alguna de dos condiciones: **CNP con monto de 500 o más**, o **monto de 20.000 o más** (cualquier tipo).


| Retención                            | TPV retenido | Fraude retenido (% del $) | Demora promedio | Transacciones |
| ------------------------------------ | ------------ | ------------------------- | --------------- | ------------- |
| Actual: revisión manual + automática | 18,3%        | 33,3%                     | 1,28 días       | 995.861       |
| Nueva: CNP con monto ≥ 500           | 8,0%         | 23,7%                     | 0,56 días       | 40.297        |
| Nueva: monto ≥ 20.000                | 11,8%        | 34,0%                     | 0,82 días       | 8.767         |
| Nuevas: ambas                        | 16,4%        | 48,3%                     | 1,15 días       | 47.057        |
| **Actual + nuevas**                  | **30,6%**    | **68,6%**                 | **2,13 días**   | 1.031.095     |


Las nuevas suman 12,3 puntos de TPV demorado y 35,3 puntos de fraude retenido: unas 2,9 veces más fraude que TPV. Junto con el rechazo, el fraude evitado o retenido pasa de 41,5% a cerca del 72% (12,3% evitado por el rechazo, más 68,6% de lo que queda). Lo que más aporta es el monto alto: son 8.767 transacciones en tres meses (unas 100 por día), un volumen manejable para revisión manual. La regla CNP son 40.297, y toma el fraude que el score no ve: el 91% de los merchants con CNP no está en el 5% más riesgoso.

**Límites**

- Los cortes (500 y 20.000) se eligieron mirando estas mismas transacciones, igual que el resto de las reglas.
- Las transacciones no tienen fecha, así que no se puede probar estabilidad en el tiempo.
- Entre mitades de merchants el monto alto se sostiene (36,8% y 31,9% del fraude retenido); el CNP varía más (27,3% y 21,0%). Sin el merchant con más fraude (`M46B66AE…`), la retención completa pasa de 68,6% a 65,8%.
- **El fraude CNP está concentrado:** 138 merchants con fraude, y los 10 mayores suman el 58% del $ (el mayor, el 18%). Es la parte más frágil.
- Retener cuesta en operación y en demora: la regla completa demora 2,13 días en promedio el TPV bueno, contra 1,28 de la actual. Depende del supuesto de 7 días de retención, que no está en los datos.
- Cerca del 28% del $ de fraude sigue pasando sin ningún control. Con solo las reglas del score, ese resto es el 58,5%.

### Chequeos y límites

- **Estabilidad por mitades de merchants.** Pre: el fraude evitado es de 11,2% y 13,4%, y el equilibrio de 1,35% y 2,20%, así que la regla de rechazo se sostiene. **Post: el fraude retenido va de 45,0% a 21,5%.** La retención depende de pocos merchants.
- **Sin el merchant con más fraude** (`M46B66AE0E309595B`): el rechazo no cambia (13,3% del $) y la retención baja de 33,3% a 27,4%.
- **La estructura y los cortes se eligieron mirando estas mismas transacciones.** Las mitades muestran estabilidad, pero no son una evaluación independiente: no hay otro período.
- **El equilibrio de 1,71% depende del margen supuesto.** Si el margen real de Clip es mayor, rechazar deja de convenir.
- **Con solo las reglas del score, el 58,5% del $ de fraude pasa sin control:** fraude de merchants de volumen alto o de riesgo bajo que el score no ve. Las retenciones por tipo de pago y por monto bajan ese resto a cerca del 28%.

