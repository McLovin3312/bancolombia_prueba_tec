# Prueba Técnica Bancolombia

# Reporte Técnico (Punto 1) - Modelo Predictivo de Avalúos de Vivienda 

## 1. Entendimiento y Limpieza de Datos
Durante la fase exploratoria del dataset de entrenamiento (12.857 registros), se detectaron y corrigieron los siguientes desafíos críticos:

* **Prevención de Fuga de Información (Data Leakage):** Se identificaron 9 variables exclusivas en el set de entrenamiento que componían directamente la variable objetivo (ej. `valor_area_terreno`, `valor_area_construccion`). Fueron eliminadas para asegurar la viabilidad del modelo en producción.
* **Limpieza del Target:** La variable `valor_total_avaluo` presentaba tipos de dato `object` y valores en 0. Se casteó a formato numérico puro y se eliminaron los valores irreales.
* **Manejo de Valores Atípicos (Outliers):** Se aplicó un recorte estadístico (trimming) eliminando el percentil 1% inferior y el 99% superior. Esto corrigió la distorsión causada por propiedades con valores atípicos extremos (ej. casas tasadas en 3 pesos o en 5.7 billones), dejando una distribución asimétrica positiva lógica para el mercado inmobiliario colombiano.
* **Manejo de Nulos:** Columnas con más del 50% de datos faltantes (como `tipo_subsidio` y `fecha_aprobacion`) fueron descartadas.

## 2. Ingeniería de Características (Feature Engineering)
* **Transformación de Texto a Número:** Variables vitales como `area_privada` y `area_valorada` se encontraban anidadas con texto (ej. "120 m2"). Se utilizaron expresiones regulares para extraer los valores flotantes numéricos.
* **Eliminación de Texto Libre:** Se descartaron variables de observaciones manuales hechas por peritos (`observaciones_generales_inmueble`, `direccion...`) para evitar ruido y la necesidad de NLP avanzado en un modelo tabular.
* **Imputación y Codificación:** Los nulos restantes se imputaron con la mediana (numéricas) y la etiqueta 'Sin_Dato' (categóricas). Finalmente, se descartaron categóricas de alta cardinalidad (>30 únicas) y se aplicó *One-Hot Encoding*, resultando en un dataset final de 565 columnas.

## 3. Entrenamiento y Resultados del Modelo
Se seleccionó un modelo **Random Forest Regressor** por su robustez frente a distribuciones asimétricas y su capacidad para capturar relaciones no lineales en bienes raíces.

**Métricas de Desempeño (Set de Validación 20%):**
* **R2 Score:** `0.8274` (El modelo explica ~83% de la varianza en los precios).
* **MAE (Mean Absolute Error):** `~$45,396,287 COP` (Margen de error altamente aceptable dadas las magnitudes del mercado).

**Explicabilidad (Feature Importance):**
Las 3 variables que más dictan el avalúo según el modelo son:
1. `area_privada` (37.2%)
2. `area_valorada` (15.4%)
3. `estrato_6` (6.3%)
*Nota: Se garantizó la exclusión de variables sin peso matemático (como identificadores de fila `id`) antes de la predicción final.*

## 4. Entregable y Evaluación
Las predicciones finales para los 5.385 registros del set de pruebas han sido generadas cumpliendo estrictamente con el formato oficial requerido por la prueba.

* **Métrica de Evaluación:** El modelo alcanzó un **MAPE (Error Porcentual Absoluto Medio) del 21.79%**, lo que demuestra un rendimiento sólido y un buen nivel de generalización para la estimación de precios en el sector inmobiliario sin incurrir en sobreajuste.
* **Archivo final:** Los resultados han sido exportados exitosamente en formato de texto separado por comas (`data/base_evaluada.csv`), conteniendo únicamente los campos solicitados de `id` y `valor_total_avaluo`.


## 5. Recomendaciones de Negocio y Trabajo Futuro 
**Utilidad del Modelo:**
Este modelo es altamente útil para el área de **Riesgos y Originación de Crédito** de Bancolombia. Permite automatizar el "Retanqueo" y la "Normalización de cartera", dándole al banco una tasación preliminar en milisegundos sin depender inmediatamente de un perito humano, ahorrando costos operativos y acelerando la respuesta al cliente.

**Datos adicionales sugeridos para mejorar el modelo:**
Si el presupuesto y la factibilidad lo permiten, el modelo mejoraría drásticamente incluyendo:
1. **Datos Macroeconómicos:** Tasa de interés del Banco de la República (impacta el poder adquisitivo) y tasa de inflación al momento del avalúo.
2. **Datos Geoespaciales:** Distancia en metros a estaciones del Metro/Transmilenio, centros comerciales y vías principales (usando la API de Google Maps o los Puntosinteres.csv).
3. **Antigüedad real del inmueble:** Factor crítico de depreciación estructural.

