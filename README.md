## Dashboard del modelo Churn - DataBricks

https://dbc-9bfeaa20-a128.cloud.databricks.com/dashboardsv3/01f1556773bc175ab4419cd42ea42ae0/published?o=7474658389438863

#  BigData - Modelo de Predicción de Churn

##  Descripción del Proyecto

Proyecto de Big Data para predecir el riesgo de retiro (churn) de asociados en una cooperativa financiera, utilizando **arquitectura Medallion (Bronze-Silver-Gold)** en Databricks con PySpark y Delta Lake para desarrollar estrategias de fidelización y retención que permitan disminuir la cantidad de retiros mensual.

El proyecto implementa un **pipeline end-to-end** que procesa datos de múltiples fuentes (sociodemograficas, transaccionales, comportamentales y financieras), genera features de Machine Learning y prepara el dataset para modelos de predicción de churn.

---

##  Arquitectura Medallion

###  Bronze Layer - Ingesta Cruda
**Propósito**: Ingesta de datos crudos desde archivos CSV sin transformaciones

**Tablas creadas** (5):
- `workspace.churn_bronze.raw_demo_asociados` - 248,859 registros
- `workspace.churn_bronze.raw_retiro_aportes` - 66,218 registros  
- `workspace.churn_bronze.raw_plano_ahorros` - 50,000 registros
- `workspace.churn_bronze.raw_detallado_operaciones` - 50,000 registros
- `workspace.churn_bronze.raw_publiturno` - 50,000 registros

**Total**: 465,077 registros de personas únicas.

**Características técnicas**:
- Formato: Delta Lake
- Particionamiento: Por `load_date`
- Metadata: `timestamp_ingestion`, `source_file`, `load_date`
- Optimización: OPTIMIZE + ZORDER por `Nit`

### Silver Layer - Transformación y Limpieza
**Propósito**: Limpieza, validación, deduplicación y consolidación de datos para el entrenamiento del modelo.

**Tablas creadas** (6):
- `workspace.churn_silver.asociados_demographics` - 248,515 clientes
- `workspace.churn_silver.retiros_aportes` - 66,186 registros
- `workspace.churn_silver.productos_ahorros` - 49,984 registros
- `workspace.churn_silver.operaciones_transaccionales` - 49,984 registros
- `workspace.churn_silver.interacciones_canal` - 49,984 registros
- **`workspace.churn_silver.clientes_consolidado`** - 248,515 clientes, 31 columnas (vista 360°)

### Gold Layer - Features para Machine Learning
**Propósito**: Dataset analítico final con features engineered para modelos de churn

**Tabla creada**:
- **`workspace.churn_gold.churn_prediction_features`** - 248,515 clientes, **40 features**

---

## Features Generados (40 total)

### 1. RFM (Recency, Frequency, Monetary) - 10 features
- **Recencia**: `recencia_dias_min`, `recencia_dias_max`, `recencia_segment`
- **Frecuencia**: `frecuencia_tx_total`, `frecuencia_tx_mensual_avg`, `actividad_transaccional`
- **Monetario**: `saldo_total`, `retiro_valor_total`, `segmento_valor`, `variacion_saldo_6m`

### 2. Tenure & Lifecycle - 4 features
- `antiguedad_total_meses`, `antiguedad_anos`, `permanencia_meses`, `segmento_antiguedad`

### 3. Diversificación de Productos - 3 features
- `num_productos`, `nivel_diversificacion_num`, `segmento_diversificacion`

### 4. Comportamiento y Canales - 4 features
- `usa_canal_digital`, `visita_fisica_agencia`, `es_multicanal`, `canal_preferido`

### 5. Indicadores de Riesgo - 5 features (binarios)
- `ha_solicitado_retiro`, `tiene_castigo`, `variacion_saldo_negativa`
- `inactivo_transaccional`, `sin_productos_ahorro`

### 6. Engagement Score - 2 features
- **`engagement_score`** (0-100): Score compuesto ponderado
  * Frecuencia transaccional (25 pts)
  * Diversificación de productos (25 pts)
  * Recencia de actividad (25 pts)
  * Uso de canales (25 pts)
- `segmento_engagement`: Alto / Medio / Bajo / Muy_Bajo

### 7. Churn Risk Score - 2 features
- **`churn_risk_score`** (0-100): Score de riesgo de churn
- **`churn_risk_segment`**: Alto_Riesgo / Riesgo_Moderado / Riesgo_Bajo / Sin_Riesgo

### 8. Demográficos - 4 features
- `cliente_id`, `sexo`, `edad`, `nivel_ingresos`

### 9. Contexto Adicional - 4 features
- `participacion_social`, `uso_creditos`, `retiro_motivo`, `retiro_tendencia_saldo`

### 10. Metadata - 2 features
- `timestamp_ingestion`, `load_date`


---

## Notebooks Implementados

### 1. 00_config (ID: 1878657110422119)
**Configuración centralizada y funciones utilitarias**

**Contenido clave**:
- Schemas Unity Catalog: `churn_bronze`, `churn_silver`, `churn_gold`
- Diccionario `SOURCE_TABLES_CONFIG` mapeando 5 fuentes CSV
- Funciones utilitarias:
  - `add_ingestion_metadata()`, `validate_schema()`, `log_ingestion_stats()`
  - `optimize_delta_table()`, `vacuum_delta_table()`
  - `get_execution_params()` (Databricks widgets)
- **Configuración ZORDER**:
  - Bronze: `[Nit]` (sin `load_date` porque es columna de partición)
  - Silver: `[Nit]`
  - Gold: `[cliente_id]`

### 2. 01_bronze_ingestion (ID: 1878657110422116)
**Ingesta de CSVs a tablas Bronze en Delta Lake**

## Resultados del Análisis Exploratorio

### Distribución de Riesgo de Churn
- **Alto Riesgo**: 26,537 clientes (10.7%) → prioritario
- **Riesgo Moderado**: 221,978 clientes (89.3%)

### Top 5 Factores de Riesgo (Prevalencia)
1. **Inactivo transaccional**: 233,629 clientes (94.01%)
2. **Sin productos de ahorro**: 204,017 clientes (82.09%)
3. **Ha solicitado retiro**: 50,020 clientes (20.13%)
4. **Variación saldo negativa**: 22,370 clientes (9.0%)
5. **Tiene castigo cartera**: 17,099 clientes (6.88%)

### Distribución de Engagement
- **Muy Bajo**: 200,283 clientes (80.6%)
- **Bajo**: 22,606 clientes (9.1%)
- **Medio**: 23,263 clientes (9.4%)
- **Alto**: 2,363 clientes (0.9%)

### Insights Clave
- **94% inactividad transaccional** → Principal factor de riesgo, requiere estrategia de activación
- **82% sin productos ahorro** → Baja diversificación, oportunidad de cross-selling
- **20% solicitó retiro** → Señal fuerte de churn inmediato
- **Solo 0.9% engagement alto** → Gran oportunidad de mejora en experiencia del cliente

** MODELO CHURN

Para ver el codigo usado en el desarrollo del modelo enytrar al sigueinte enlace: [04_modelo_prediccion_churn.ipynb](https://github.com/user-attachments/files/28164411/04_modelo_prediccion_churn.ipynb)

Este script hace:

1. Carga el dataset Gold exportado a CSV.
2. Crea la variable objetivo churn.
3. Separa train/test antes de cualquier balanceo.
4. Audita fuga de información usando SOLO train.
5. Detecta fuga en:
   - nombres sospechosos,
   - variables directas,
   - variables numéricas con AUC univariado alto,
   - variables categóricas con tasas de churn casi perfectas,
   - variables numéricas de baja cardinalidad tratadas como categóricas.
6. Hace balanceo de clases SOLO en train.
7. Hace una poda iterativa de variables que siguen generando AUC sospechosamente alto.
8. Entrena varios modelos.
9. Evalúa sobre test original, sin balancear.
10. Genera métricas clásicas, ranking, matrices, predicciones e importancia de variables.

En resumen, lo que se hizo fue comparar diferentes modelos que pudieran predecir de una manera mas precisa y con los resultados adecuados de acuerdo a las metricas que evaluan cada uno de los modelos.

## Autores

**Rafael** - rafael7cor7@gmail.com

**Carolina** - carolina.jimenez3376@unaula.edu.co

**Dulce** - dulcedanielavargas@hotmail.com
