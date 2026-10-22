# Merchant Risk Scoring · Clip DS Challenge 2026

Un score de riesgo por merchant para decidir qué transacciones se aprueban y a qué merchants se les paga de inmediato.

Todas las decisiones, con sus pruebas y límites, están en [`docs/decisions.md`](docs/decisions.md).

## El problema

Clip agrupa merchants en afiliaciones que los emisores evalúan en conjunto: unos pocos merchants riesgosos bajan la aprobación de todos. El challenge pide (1) entrenar un modelo, (2) puntuar a cada merchant y unir el score a las transacciones, y (3) proponer reglas de pre y post autorización, midiendo por separado la calidad del modelo y el impacto de negocio.

## Cómo se resolvió

**Target.** `is_suspended` es un estado diario, no un evento. Cada merchant con inicio de suspensión observado aporta **una fila positiva: su primer día suspendido**. Esa fila resume la actividad hasta el día anterior (confirmado por la empresa), así que no usa información posterior. Los negativos son merchants nunca suspendidos en abril–septiembre, un día activo sorteado con la misma mezcla de fechas que los positivos. Quedan 12 positivos y unos 33.500 negativos. Se excluyen los suspendidos que no muestran actividad sospechosa antes del inicio (probable suspensión por documentación o cumplimiento): el recall reportado es sobre las suspensiones visibles en la actividad.

**Modelo.** Regresión logística con regularización L2, 4 variables de la foto del merchant: aprobación reciente (la peor entre 7 y 30 días), aprobación histórica, indicador de "sin actividad a 7 días" y el peor día de rechazos en 30 días. Validación por merchant: 5 folds estratificados repetidos con 10 semillas. KPIs del modelo: PR-AUC y positivos capturados en el 5% de mayor score (10,6 de 12).

**Score y join.** Cada merchant se puntúa con su fila del 1 de octubre y el score se une por `merchant_id` a las 7,08 millones de transacciones de octubre a diciembre (cobertura 100%). Las transacciones no se usaron para entrenar: son el conjunto independiente para medir el impacto.

**Reglas.** Los cortes se definen por percentil de score (la probabilidad no está calibrada) y se combinan con el volumen del merchant:
- *Pre-autorización:* rechazar volumen bajo con riesgo ≥ 90% y volumen medio con riesgo ≥ 99%.
- *Post-autorización:* revisión manual para volumen alto con riesgo ≥ 95%, y retención de 7 días para merchants sin actividad.
- *Bonus:* tres paquetes de transacciones (bajo, medio, alto riesgo) para los emisores.

Con los supuestos de margen de Clip de 1% y retención de 7 días, el rechazo evita el 12,3% del $ de fraude con 1,78% del TPV bueno rechazado; rechazo más retención cubre el 41,5% del $ de fraude. El 58,5% restante pasa sin control. Detalle, trade-offs y chequeos en la sección 5 de `decisions.md`.

## Limitaciones

- Con 12 positivos, cualquier métrica es ruidosa; un caso mueve 8 puntos de recall.
- La regla, las variables y los cortes se eligieron mirando los mismos datos. No hay un período independiente adicional.
- El score se entrena con suspensión, no con fraude: ordena bien el 5% más riesgoso y casi nada dentro del resto (el 5% más riesgoso concentra el 36% del fraude).
- El margen de Clip y la retención de 7 días son supuestos que no están en los datos.

## Cómo reproducir

Requiere [uv](https://docs.astral.sh/uv/) y Python 3.12 o superior.

```bash
uv sync
```

Los datos que entregó la empresa no se versionan. Hay que dejarlos en `data/`:

```text
data/merchant_rating_featureset_sample.parquet
data/txn_sample.parquet
```

Los notebooks se ejecutan en orden y desde la carpeta `notebooks/`, porque resuelven las rutas desde ahí:

```bash
cd notebooks
uv run --with jupyter jupyter lab
```

| Notebook | Qué hace | Salida |
|---|---|---|
| `01_EDA` | Exploración de los datos | — |
| `02_suspensiones` | Cada suspensión día por día | `docs/suspensiones*`, `docs/variables/` |
| `03_dataset` | Arma el dataset: una fila por merchant | `data/processed/dataset.parquet` |
| `04_features` | Arma las features | `data/processed/features.parquet` |
| `05_logistica` | Entrena y compara la regla y la logística; puntúa octubre | `docs/modelo/`, `data/processed/score_octubre.parquet` |
| `06_transacciones_con_score` | Une el score a las transacciones | `data/processed/transacciones_con_score.parquet` |
| `07_reglas` | Reglas de pre y post autorización y KPIs de negocio | `docs/reglas/` |

`requirements.txt` es una copia exportada de `uv.lock` para quien no use uv: `pip install -r requirements.txt`.

## Estructura

```text
data/            datos de la empresa y derivados (no se versionan)
docs/            decisiones, gráficos y tablas de resultados
guideline/       enunciado del challenge
notebooks/       el pipeline, en orden
presentacion/    presentación
```
