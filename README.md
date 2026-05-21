# 🎯 BigData - Modelo de Predicción de Churn

## 📋 Descripción del Proyecto

Proyecto de Big Data para predecir el riesgo de retiro (churn) de asociados en una cooperativa financiera, utilizando **arquitectura Medallion (Bronze-Silver-Gold)** en Databricks con PySpark y Delta Lake.

El proyecto implementa un **pipeline end-to-end** que procesa datos de múltiples fuentes, genera features de Machine Learning y prepara el dataset para modelos de predicción de churn.

---

## 🏗️ Arquitectura Medallion

### 🥉 Bronze Layer - Ingesta Cruda
**Propósito**: Ingesta de datos crudos desde archivos CSV sin transformaciones

**Tablas creadas** (5):
- `workspace.churn_bronze.raw_demo_asociados` - 248,859 registros
- `workspace.churn_bronze.raw_retiro_aportes` - 66,218 registros  
- `workspace.churn_bronze.raw_plano_ahorros` - 50,000 registros
- `workspace.churn_bronze.raw_detallado_operaciones` - 50,000 registros
- `workspace.churn_bronze.raw_publiturno` - 50,000 registros

**Total**: 465,077 registros

**Características técnicas**:
- Formato: Delta Lake
- Particionamiento: Por `load_date`
- Metadata: `timestamp_ingestion`, `source_file`, `load_date`
- Optimización: OPTIMIZE + ZORDER por `Nit`

### 🥈 Silver Layer - Transformación y Limpieza
**Propósito**: Limpieza, validación, deduplicación y consolidación de datos

**Tablas creadas** (6):
- `workspace.churn_silver.asociados_demographics` - 248,515 clientes
- `workspace.churn_silver.retiros_aportes` - 66,186 registros
- `workspace.churn_silver.productos_ahorros` - 49,984 registros
- `workspace.churn_silver.operaciones_transaccionales` - 49,984 registros
- `workspace.churn_silver.interacciones_canal` - 49,984 registros
- **`workspace.churn_silver.clientes_consolidado`** - 248,515 clientes, 31 columnas (vista 360°)

### 🥇 Gold Layer - Features para Machine Learning
**Propósito**: Dataset analítico final con features engineered para modelos de churn

**Tabla creada**:
- **`workspace.churn_gold.churn_prediction_features`** - 248,515 clientes, **40 features**

---

## 📊 Features Generados (40 total)

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

### 7. Churn Risk Score - 2 features (🎯 Target Proxy)
- **`churn_risk_score`** (0-100): Score de riesgo de churn
- **`churn_risk_segment`**: Alto_Riesgo / Riesgo_Moderado / Riesgo_Bajo / Sin_Riesgo

### 8. Demográficos - 4 features
- `cliente_id`, `sexo`, `edad`, `nivel_ingresos`

### 9. Contexto Adicional - 4 features
- `participacion_social`, `uso_creditos`, `retiro_motivo`, `retiro_tendencia_saldo`

### 10. Metadata - 2 features
- `timestamp_ingestion`, `load_date`


---

## 📂 Estructura del Proyecto

```
BigData---Modelo-churn/
│
├── README.md                          # Este archivo
├── data/
│   └── raw/                          # Datos fuente (CSVs)
│       ├── demo_asociados.csv        # 248,859 registros
│       ├── Retiro_aportes_2022.csv   # 66,218 registros
│       ├── Plano_ahorros.csv         # 50,000 registros
│       ├── detalladoperaciones.csv   # 50,000 registros
│       └── publiturno.csv            # 50,000 registros
│
└── notebooks/
    ├── 00_config                      # Configuración centralizada
    ├── 01_bronze_ingestion            # Ingesta a Bronze
    ├── 02_silver_transformation       # Transformación a Silver
    └── 03_gold_feature_engineering    # Feature engineering a Gold
```

---

## 📓 Notebooks Implementados

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

**Estado**: ✅ Ejecutado exitosamente

### 2. 01_bronze_ingestion (ID: 1878657110422116)
**Ingesta de CSVs a tablas Bronze en Delta Lake**

**Proceso**:
1. Importa configuración vía `%run ./00_config`
2. Función genérica: `ingest_csv_to_bronze(source_config)`
3. Ingesta 5 archivos CSV con metadata automática
4. Particiona por `load_date`, optimiza con ZORDER por `Nit`

**Resultados**: 465,077 registros en 5 tablas Bronze

**Estado**: ✅ Ejecutado exitosamente

### 3. 02_silver_transformation (ID: 1878657110422117)
**Limpieza, transformación y consolidación**

**Funciones utilitarias**:
- `clean_string_column()`, `deduplicate_by_key()`, `validate_not_null()`

**Proceso**:
1. Transforma 5 tablas individuales con validaciones específicas
2. Consolida mediante LEFT JOIN desde `demographics` como base
3. Deduplica por `Nit` (mantiene registro más reciente vía Window functions)
4. Optimiza con ZORDER

**Resultados**: 
- 464,653 registros individuales
- 248,515 clientes únicos consolidados (vista 360°)

**Estado**: ✅ Ejecutado exitosamente

### 4. 03_gold_feature_engineering (ID: 1878657110422118)
**Generación de 40 features para ML**

**Proceso**:
1. Lee `clientes_consolidado` de Silver
2. Genera features en 10 categorías (RFM, tenure, diversificación, etc.)
3. Calcula scores compuestos: `engagement_score`, `churn_risk_score`
4. Selecciona 40 features finales
5. Escribe tabla Gold optimizada

**Correcciones aplicadas**:
- ✅ Celda 4: Agregado `from pyspark.sql.functions import col`
- ✅ Celda 5: Agregado `from pyspark.sql.functions import col, count, avg, desc`
- ✅ Celda 6: Agregado `from pyspark.sql.functions import col`

**Estado**: ✅ Ejecutado exitosamente (7 celdas)

---

## 🔧 Problemas Técnicos Resueltos

### Problema 1: ZORDER en columna de partición
**Error**: `DELTA_ZORDERING_ON_PARTITION_COLUMN`
- **Causa**: ZORDER intentaba ordenar por `load_date` (columna de partición)
- **Solución**: Removido `load_date`, ZORDER solo por `[Nit]` en Bronze

### Problema 2: ZORDER en columna sin estadísticas
**Error**: `DELTA_ZORDERING_ON_COLUMN_WITHOUT_STATS`
- **Causa**: `churn_risk_score` es calculado, Delta no colecta stats automáticamente
- **Solución**: ZORDER solo por `cliente_id` en Gold

### Problema 3: Importaciones faltantes
**Errores**: `NameError: name 'col' is not defined`, `name 'desc' is not defined`
- **Causa**: Funciones PySpark usadas sin importar en múltiples celdas
- **Solución**: Agregadas importaciones necesarias en cada celda que las requiere

---

## 📊 Resultados del Análisis Exploratorio

### Distribución de Riesgo de Churn
- **Alto Riesgo**: 26,537 clientes (10.7%) → 🎯 Target prioritario
- **Riesgo Moderado**: 221,978 clientes (89.3%)

### Top 5 Factores de Riesgo (Prevalencia)
1. **Inactivo transaccional**: 233,629 clientes (94.01%) 🔴
2. **Sin productos de ahorro**: 204,017 clientes (82.09%)
3. **Ha solicitado retiro**: 50,020 clientes (20.13%)
4. **Variación saldo negativa**: 22,370 clientes (9.0%)
5. **Tiene castigo cartera**: 17,099 clientes (6.88%)

### Distribución de Engagement
- **Muy Bajo**: 200,283 clientes (80.6%)
- **Bajo**: 22,606 clientes (9.1%)
- **Medio**: 23,263 clientes (9.4%)
- **Alto**: 2,363 clientes (0.9%)

### 💡 Insights Clave
- **94% inactividad transaccional** → Principal factor de riesgo, requiere estrategia de activación
- **82% sin productos ahorro** → Baja diversificación, oportunidad de cross-selling
- **20% solicitó retiro** → Señal fuerte de churn inmediato
- **Solo 0.9% engagement alto** → Gran oportunidad de mejora en experiencia del cliente


---

## 🤖 Workflow / Job Configurado

**Nombre**: Churn Prediction - Medallion Pipeline  
**Job ID**: 115016573045398

**Configuración actual**:
- **Schedule**: Diario a las 2:00 AM (timezone: America/Bogota)
- **Cron**: `0 0 2 * * ? *`
- **Task**: `bronze_ingestion` (notebook: `01_bronze_ingestion`)

**💡 Mejoras sugeridas**:
- Agregar tareas para Silver (`02_silver_transformation`) y Gold (`03_gold_feature_engineering`)
- Configurar dependencias secuenciales: Bronze → Silver → Gold
- Agregar alertas por email/Slack en caso de fallo
- Configurar retry policy

---

## 🎯 Próximos Pasos Recomendados

### 1. 🤖 Entrenamiento de Modelo ML

#### Opción A: Databricks AutoML (⭐ Recomendado)
```python
import databricks.automl

summary = databricks.automl.classify(
    df=spark.table("workspace.churn_gold.churn_prediction_features"),
    target_col="churn_risk_segment",
    timeout_minutes=30
)
```
- Prueba automáticamente múltiples algoritmos
- Genera notebooks con código reproducible
- Registra experimentos en MLflow

#### Opción B: MLflow Manual
```python
import mlflow
from pyspark.ml import Pipeline
from pyspark.ml.feature import VectorAssembler, StringIndexer
from pyspark.ml.classification import RandomForestClassifier

with mlflow.start_run():
    # Feature engineering + training
    # ... código de ML
    mlflow.log_metric("accuracy", accuracy)
    mlflow.spark.log_model(model, "churn_model")
```

### 2. 📦 Feature Store (Opcional)
```python
from databricks.feature_store import FeatureStoreClient

fs = FeatureStoreClient()
fs.create_table(
    name="workspace.churn_gold.churn_features",
    primary_keys=["cliente_id"],
    df=spark.table("workspace.churn_gold.churn_prediction_features"),
    description="Features de churn prediction - RFM + Engagement + Risk"
)
```

### 3. 📊 Dashboard de Monitoreo (Lakeview)
Crear dashboard con:
- Distribución de riesgo de churn por segmento
- KPIs: % Alto Riesgo, Engagement Score promedio
- Tendencias temporales (evolución mensual)
- Top 20 clientes en riesgo
- Heatmap de factores de riesgo

### 4. 🔄 Completar Workflow
- Agregar tareas Silver y Gold con dependencias
- Configurar alertas (email cuando > 15% Alto Riesgo)
- Implementar data quality checks (Great Expectations)

### 5. 🚀 Modelo en Producción
- Deploy en Model Serving (endpoint REST)
- Scoring en tiempo real o batch
- Monitoreo de drift del modelo
- Re-entrenamiento automático (mensual)

---

## 📚 Tecnologías Utilizadas

| Tecnología | Uso |
|------------|-----|
| **Databricks** | Plataforma de procesamiento distribuido |
| **PySpark** | Procesamiento de Big Data |
| **Delta Lake** | Storage ACID con time travel |
| **Unity Catalog** | Gobernanza y lineage de datos |
| **Databricks Workflows** | Orquestación y scheduling |
| **MLflow** | (próximo) Tracking de experimentos |
| **Lakeview** | (próximo) Dashboards y visualización |

---

## 👥 Autor

**Rafael** - rafael7cor7@gmail.com

---

## 📅 Historial de Cambios

### 2026-05-17 - Implementación Completa ✅
- ✅ Arquitectura Medallion completa (Bronze-Silver-Gold)
- ✅ 12 tablas Delta creadas (5 Bronze + 6 Silver + 1 Gold)
- ✅ 40 features ML-ready con feature engineering avanzado
- ✅ Corrección de 3 problemas técnicos (ZORDER e importaciones)
- ✅ 4 notebooks funcionando end-to-end
- ✅ Workflow diario configurado
- ✅ Análisis exploratorio completo
- ✅ Documentación exhaustiva

---

## 📝 Notas Técnicas

### Performance & Optimización
- **ZORDER**: Aplicado en columnas de alta cardinalidad usadas en filtros (`Nit`, `cliente_id`)
- **Particionamiento**: `load_date` permite queries temporales eficientes y time-travel
- **Auto-optimization**: Habilitado en Gold (`optimizeWrite`, `autoCompact`)

### Calidad de Datos
- **Deduplicación**: Window functions con `row_number()` por `Nit` ordenado por `timestamp_ingestion DESC`
- **Validaciones**: NOT NULL en campos críticos, validación de rangos
- **Consolidación**: LEFT JOIN desde demographics preserva todos los clientes

### Escalabilidad
- Pipeline diseñado para **carga incremental** (append a Bronze, merge en Silver/Gold)
- Delta Lake soporta **MERGE UPSERT** para actualizaciones eficientes
- Particionamiento por fecha permite **data pruning** automático

---

## 🎓 Conceptos Aplicados

- **Medallion Architecture**: Bronze (raw) → Silver (curated) → Gold (aggregated/features)
- **RFM Analysis**: Recency, Frequency, Monetary para segmentación de clientes
- **Feature Engineering**: Transformación de datos crudos en features predictivos
- **Delta Lake**: Versionamiento, time travel, ACID transactions
- **Unity Catalog**: 3-level namespace, lineage tracking

---

## 📖 Referencias

- [Databricks Medallion Architecture](https://docs.databricks.com/lakehouse/medallion.html)
- [Delta Lake Documentation](https://docs.delta.io/)
- [PySpark ML Guide](https://spark.apache.org/docs/latest/ml-guide.html)
- [Unity Catalog Best Practices](https://docs.databricks.com/unity-catalog/best-practices.html)

---

**🎯 Estado del Proyecto: READY FOR ML TRAINING**

Dataset analítico completo con **248,515 clientes** y **40 features** listos para modelos de churn prediction.

**Próximo paso**: Entrenar modelo con AutoML o MLflow y deployar en producción. 🚀
