# Punto 1 - Estimación de avalúos de vivienda

Prueba de conocimiento analítico DiCAGI 2020. El objetivo es predecir `valor_total_avaluo` (precio total del inmueble, en COP) para cada registro de `testeo_precios_vivienda.xlsx`. La métrica de evaluación es el **MAPE**.

## Estructura del proyecto

```
bancolombia_prueba_tec/
├── punto-1-avaluos/
│   ├── data/
│   │   ├── raw/
│   │   │   ├── entrenamiento_precios_vivienda.xlsx
│   │   │   ├── testeo_precios_vivienda.xlsx
│   │   │   └── PuntosInteres.csv            (opcional)
│   │   └── base_evaluada.csv                (entregable generado)
│   ├── notebooks/
│   │   └── 01_exploracion.ipynb             (pipeline completo, ver modelo_avaluos.py)
│   └── src/
├── punto-2-agentes/
├── .gitignore
└── README.md
```

El código está en `modelo_avaluos.py`, escrito en celdas `# %%`. Se puede abrir como notebook interactivo en VS Code o pegar celda por celda en `01_exploracion.ipynb`. Las rutas son relativas a la carpeta `notebooks/`.

## Requisitos

- Python 3.10 o superior (probado con Python 3.13, pandas 3 y scikit-learn 1.9).
- Dependencias obligatorias:

```bash
pip install pandas numpy scikit-learn matplotlib seaborn openpyxl
```

- Opcional: `pip install lightgbm`. Si está instalado, se agrega al ensamble; si no, el código funciona igual.

## Cómo ejecutarlo

1. Colocar los archivos Excel en `punto-1-avaluos/data/raw/` (y `PuntosInteres.csv` si se quiere usar).
2. Ejecutar las celdas en orden desde la carpeta `notebooks/`.
3. Al terminar se genera `punto-1-avaluos/data/base_evaluada.csv` con las columnas `id` y `valor_total_avaluo`.

Todas las semillas están fijadas (`SEMILLA = 42`), por lo que los resultados son reproducibles.

## Descripción del proceso

| Sección | Qué hace | Por qué |
|---|---|---|
| 1. Carga | Lee entrenamiento y testeo | Verifica dimensiones |
| 2. Fuga de datos | Elimina las columnas que existen solo en entrenamiento (`valor_area_*`, `valor_uvr`, `valor_avaluo_en_uvr`) | Se derivan del avalúo y no están disponibles en testeo |
| 3. Target | Parser numérico propio (formatos `1.234.567,89`, `1,234,567.89`, `$`, etc.). Solo descarta filas con target nulo o <= 0 | `pd.to_numeric(errors='coerce')` descartaba muchas filas válidas por formato |
| 4. Funciones | Limpieza de coordenadas, números escondidos en texto, fechas a número, longitud de textos libres, distancias a puntos de interés | La misma transformación se aplica a entrenamiento y testeo |
| 5. Plan de variables | Numéricas no constantes y categóricas con 2 a 100 niveles (codificación ordinal). Se descartan `id`, matrículas, direcciones y textos libres | Todo se aprende solo con entrenamiento para evitar fuga |
| 6. Atípicos | Marca avalúos extremos (percentiles 0,5 y 99,5) y precios por m² imposibles (percentiles 1 y 99). No se usan para entrenar | Son errores de captura que ensucian el aprendizaje |
| 7. Modelos | `log(avalúo)` con HistGradientBoosting, RandomForest y (opcional) LightGBM; ensamble; validación cruzada de 5 folds | El MAPE es un error relativo y los precios tienen cola larga |
| 8. Final | Calibración de un factor de escala que minimiza el MAPE, reentrenamiento con todas las filas normales y predicción | Mejora el MAPE con un solo parámetro |
| 9. Entregable | Valida filas, ids, nulos, infinitos y positivos antes de guardar | Cumple el formato exigido por el enunciado |

## Evaluación del modelo

El notebook imprime, para cada modelo y para el ensamble:

- **MAPE sin atípicos:** criterio con el que se elige el mejor modelo.
- **MAPE todo:** incluye las filas atípicas. Puede ser enorme si hay avalúos de unos pocos pesos (un solo registro así dispara el promedio).
- **Mediana del error porcentual absoluto:** no se ve afectada por esos extremos.

Todas las métricas se calculan **fuera de muestra** (predicciones de validación cruzada). Los resultados con los datos reales se registran aquí tras la ejecución:

| Modelo | MAPE sin atípicos | MAPE todo | Mediana APE |
|---|---|---|---|
| (completar) | | | |

## Supuestos y limitaciones

- Los avalúos con errores de captura (precios imposibles) no se pueden predecir. Si el testeo contiene filas así, el MAPE oficial las incluirá.
- No hay fecha del avalúo confiable ni estado de conservación, por lo que parte de la variación de precio queda sin explicar.
- Las categóricas se codifican de forma ordinal. Las categorías nuevas en testeo se tratan como faltantes.
- Las predicciones se acotan al rango de avalúos normales del entrenamiento.
- Si no se incluye `PuntosInteres.csv`, no se generan variables de cercanía a puntos de interés.

## Utilidad del análisis

| Usuario | Uso | Beneficio |
|---|---|---|
| Originación de crédito hipotecario | Valor comercial preliminar del inmueble a financiar | Reduce tiempos y costos frente al avalúo presencial |
| Riesgos / monitoreo de garantías | Revalorizar el portafolio de colaterales | Recalcula el LTV de toda la cartera de forma masiva |
| Cobranza / normalización de cartera | Evaluar refinanciaciones y daciones en pago | Valor de referencia objetivo para negociar |
| Retanqueo | Actualizar el valor de la garantía existente | Decide nuevos cupos sin un nuevo avalúo |
| Auditoría de peritos | Detectar avalúos que se alejan del modelo | Control de calidad y prevención de fraude |

## Datos adicionales sugeridos

| Dato | Factibilidad | Costo | Impacto |
|---|---|---|---|
| Fecha del avalúo e índice de precios de vivienda (DANE, Banco de la República) | Alta | Bajo | Alto |
| Antigüedad, estado de conservación, piso, ascensor | Media | Bajo | Alto |
| Ofertas y transacciones de la zona | Media | Medio | Alto |
| Datos catastrales y uso del suelo (POT) | Media | Bajo-Medio | Medio |
| Seguridad, ruido, transporte | Media | Medio | Medio |

## Diseño teórico del sistema (no implementado)

Flujo propuesto para servir el modelo a otros servicios:

1. **Ingesta:** ETL batch (por ejemplo Airflow) desde el core bancario, avalúos de peritos, catastro, DANE y puntos de interés hacia un data lake / DW, con controles de calidad.
2. **Feature store:** transformaciones versionadas, idénticas en entrenamiento e inferencia.
3. **Entrenamiento:** pipeline MLOps periódico o disparado por deriva, con validación (MAPE global y por zona) y registro de versiones del modelo (por ejemplo MLflow).
4. **Servicio en línea:** API REST (por ejemplo FastAPI en contenedores con autoescalado). `POST /avaluo` devuelve valor, intervalo de predicción y versión del modelo.
5. **Servicio batch:** scoring nocturno del portafolio para recalcular el LTV y alimentar tableros de BI.
6. **Consumo:** apps de originación, web y móvil a través de un API Gateway con autenticación (OAuth2), límites de uso y trazabilidad.
7. **Monitoreo:** deriva de datos (PSI), MAPE contra avalúos reales que llegan después, latencia y errores, alertas y reentrenamiento. Revisión humana cuando el intervalo de predicción sea amplio.

Consideraciones: protección de datos personales (Ley 1581), explicabilidad para auditoría (importancia de variables o SHAP) y uso del modelo como apoyo al perito, no como reemplazo.

## Posibles mejoras

- Codificación por objetivo (target encoding) de `barrio` y `sector`, con validación cruzada.
- Clústeres geográficos y precio medio por zona.
- Extracción de datos de las observaciones (piso, remodelación, tipo de acabados).
- Búsqueda de hiperparámetros (Optuna) y regresión cuantílica para intervalos de predicción.