# Uso de IA

El análisis lo planteó, lo guió y lo controló Leandro Garcia de principio a fin. Usó Claude Code como asistente de trabajo: él definía qué probar, revisaba cada resultado, pedía correcciones y tomaba la decisión final. La IA escribió el código, ejecutó las pruebas y redactó lo que él le pidió.

## Cómo se trabajó

En cada etapa el flujo fue el mismo:

1. Leandro planteaba la pregunta o la hipótesis.
2. La IA la ejecutaba sobre los datos y devolvía los números.
3. Leandro revisaba el resultado, lo cuestionaba y pedía cambios o pruebas adicionales.
4. Leandro decidía, y la decisión quedó registrada con su evidencia en [`decisions.md`](decisions.md).

## Por etapa

| Etapa | Cómo la guió y la decidió Leandro | Qué ejecutó la IA |
|---|---|---|
| Planteo del modelo | Preguntó por alternativas al clasificador y pidió incluir en el roadmap la prueba de un detector de anomalías. Fijó cada versión del target. | Comparó las opciones y construyó los datasets. |
| Target | Pidió mirar cada suspensión día por día, cuestionó qué fila representaba a cada positivo y decidió qué casos quedaban fuera del análisis. Consultó a la empresa qué día resumía cada fila. | Generó los gráficos por merchant y armó el dataset. |
| Variables y modelo | Pidió entrenar con distintos diseños, decidió qué variables usaba cada modelo y qué capturaba la regla frente a la logística. Decidió quedarse con el modelo más simple. | Escribió los notebooks, la validación repetida y las métricas. |
| Score y join | Pidió unir el score a las transacciones y verificar la cobertura. | Implementó la unión y los chequeos. |
| Reglas | Fijó los supuestos de negocio (margen y retención), eligió los cortes y pidió sumar las reglas de CNP y monto alto. | Calculó los KPIs de cada alternativa. |
| Documentación y presentación | Definió la estructura, el estilo y que cada decisión mostrara en qué se basa. | Redactó `decisions.md`, el README y el HTML. |

## Control de resultados

Leandro revisó los resultados y pidió corregirlos cuando algo no cerraba. Entre otros:

- Las métricas del modelo se recalcularon después de detectar que el orden de las filas favorecía a los positivos.
- El criterio de qué fila representa a cada positivo se ajustó tras la respuesta de la empresa sobre el día que resume cada fila.
- Se unificaron notebooks y se corrigieron inconsistencias de `decisions.md`.

## Qué no se delegó

El criterio de negocio, los supuestos, la elección entre alternativas y la lectura de cada resultado son de Leandro.
