# Análisis de Ventas 2020–2023

Proyecto de análisis de datos end-to-end sobre un dataset de ventas de 4 años. Cubre desde la limpieza y modelado de datos hasta la segmentación de clientes, un modelo predictivo de rentabilidad e insights estratégicos accionables.

**Autor:** Sebastian Lesmes  
**Fecha:** Abril 2026

---

## Problema de negocio

El 19% de las ventas registradas generaban pérdidas (~$145,000 en el periodo), principalmente por una política de descuentos sin criterio de rentabilidad. El objetivo del proyecto fue:

1. Limpiar y modelar los datos en una arquitectura reproducible.
2. Entender el comportamiento de ventas, clientes y productos.
3. Segmentar la base de clientes para estrategias diferenciadas.
4. Construir un modelo que prediga si una venta generará pérdida **antes** de aprobarla.
5. Traducir los hallazgos en recomendaciones concretas de negocio.

---

## Estructura del proyecto

```
Analisis_ventas/
│
├── data/
│   ├── raw/                    # Datos originales (Excel sin modificar)
│   │   └── datos.xlsx
│   └── processed/              # Tabla plana por cliente para ML
│       └── flat_table_model.csv
│
├── notebooks/
│   ├── 01_limpieza_datos.ipynb     # Fase 1: diagnóstico, limpieza y carga a MariaDB
│   ├── 02_EDA.ipynb                # Fase 2: análisis exploratorio
│   ├── 03_segmentacion.ipynb       # Fases 3 y 4: feature engineering + K-Means
│   ├── 04_modelo_ml.ipynb          # Fase 5: modelo ML predictor de rentabilidad
│   └── 05_insights.ipynb           # Fase 6: insights estratégicos accionables
│
├── models/
│   ├── profit_predictor_gb.pkl     # Modelo Gradient Boosting (principal)
│   ├── profit_predictor_rf.pkl     # Modelo Random Forest (alternativo)
│   └── feature_names.pkl           # Lista de features del modelo
│
├── outputs/                    # Gráficas generadas por los notebooks
│
├── sql/
│   ├── 01_create_schema.sql        # DDL del star schema en MariaDB
│   ├── 03_feature_engineering.sql  # Queries de feature engineering
│   └── 04_powerbi_views.sql        # Vistas para Power BI
│
├── presentacion/
│   └── sales_hallazgos.pbix        # Dashboard Power BI
│
├── .env                        # Credenciales de BD (no versionado)
├── enviroment.yml              # Entorno Conda reproducible
└── README.md
```

---

## Arquitectura de datos

El pipeline carga los datos en MariaDB en dos bases de datos separadas:

```
Excel (raw) ──► sales_raw   (tablas originales — trazabilidad/auditoría)
            └──► sales_db    (star schema limpio — análisis y ML)
```

**Star schema en `sales_db`:**

```
                 dim_customer
                      │
dim_product ── fact_sales ── dim_regional_manager
                      │
                 dim_returns
```

---

## Flujo de los notebooks

### `01_limpieza_datos.ipynb` — Limpieza y carga
Diagnóstico exhaustivo de calidad y limpieza documentada tabla por tabla.

| Tabla | Antes | Después | Principales acciones |
|---|---|---|---|
| Sales | 11,742 | 10,187 | Eliminación de 1,548 duplicados, 3 valores absurdos, 1 Discount > 1, corrección de 4 fechas en año 2031/2033 |
| Customer | 1,608 | 804 | Eliminación de duplicados exactos, corrección de edades imposibles (260→26, 115→15) |
| Returns | 800 | 296 | Deduplicación por Order_ID |
| Regional Manager | 4 | 4 | Corrección de typos en nombres de región ("Wes t", "cenntral") |
| Product | 1,894 | 1,894 | Sin cambios necesarios |

Integridad referencial post-carga: **0 huérfanos** en Customer_ID ni Product_ID.

---

### `02_EDA.ipynb` — Análisis exploratorio
Análisis univariado, bivariado y temporal sobre 10,529 transacciones.

Hallazgos principales:
- **Descuentos > 20%** tienen correlación negativa fuerte con el profit (−0.22).
- El **15% de transacciones premium** (Sales > $389) generan el **70.6% del revenue**.
- **Tables y Bookcases** operan con profit total negativo.
- Pico estacional en Q4 con la paradoja: mayor volumen, menor margen.
- **Región West** lidera en ventas; **Technology** en revenue por categoría.

---

### `03_segmentacion.ipynb` — Feature Engineering y K-Means
Construcción de 15 features por cliente (RFM + conductuales + socioeconómicas) y clustering con K-Means.

| Segmento | Clientes | Recencia media | Monetary medio | Desc. prom | Profit/orden |
|---|---|---|---|---|---|
| Habituales | 349 (43%) | 73 días | $3,010 | 16% | $33.8 |
| Poco Activos | 236 (29%) | 139 días | $1,369 | 12% | $47.8 |
| En Riesgo | 112 (14%) | 428 días | $1,735 | 24% | −$46.7 |
| VIP / Champions | 107 (13%) | 111 días | $7,698 | 12% | $221.5 |

Selección de k: método del codo + Silhouette Score. k=4 elegido por interpretabilidad de negocio.

---

### `04_modelo_ml.ipynb` — Predictor de rentabilidad
Clasificador binario (`is_profitable`) entrenado con SMOTE para balancear el desbalance de clases (80/20).

| Modelo | AUC-ROC | Recall pérdidas | Precisión pérdidas |
|---|---|---|---|
| Logistic Regression | 0.974 | 82% | 83% |
| Random Forest | 0.977 | 79% | 90% |
| **Gradient Boosting** ✓ | **0.984** | **83%** | **89%** |

**ROI estimado del sistema de alertas:**
- Ahorro potencial: **$133,481/año**
- Costo de implementación: $30,000
- Payback: **2.7 meses** — ROI primer año: **345%**

El sistema recibe los parámetros de una venta propuesta y devuelve la predicción de rentabilidad junto con el descuento máximo viable.

---

### `05_insights.ipynb` — Insights estratégicos

**Insight 1 — Política de descuentos:**

| Categoría | Límite recomendado | Punto de quiebre |
|---|---|---|
| Furniture | 10% | Pérdidas desde 20–30% |
| Technology | 20% | Pérdidas desde 30–40% |
| Office Supplies | 15% | Pérdidas desde 20–30% |

**Insight 2 — Matriz BCG:**  
Tables y Bookcases son sub-categorías "Dog" con profit total negativo — descuento promedio >29% y más del 55% de sus ventas generan pérdida.

**Insight 3 — Estacionalidad:**  
Limitar descuentos en Q4 (Nov–Dic) a 15% recupera margen sin sacrificar el volumen estacional.

---

## Configuración del entorno

### 1. Clonar y crear el entorno Conda

```bash
git clone <url-del-repo>
cd Analisis_ventas
conda env create -f enviroment.yml
conda activate sales_env
```

### 2. Configurar credenciales de base de datos

Crear un archivo `.env` en la raíz del proyecto:

```
DB_USER=tu_usuario
DB_PASS=tu_contraseña
DB_HOST=localhost
DB_PORT=3306
DB_NAME=sales_db
```

> El archivo `.env` está en `.gitignore` y nunca debe subirse al repositorio.

### 3. Ejecutar los notebooks en orden

```
01_limpieza_datos.ipynb  →  Crea las BDs en MariaDB y carga los datos
02_EDA.ipynb             →  Requiere que el paso anterior esté completo
03_segmentacion.ipynb    →  Genera outputs/segmentos_clientes.csv
04_modelo_ml.ipynb       →  Genera models/profit_predictor_gb.pkl
05_insights.ipynb        →  Requiere fact_sales y dim_product en MariaDB
```

---

## Dependencias principales

| Librería | Uso |
|---|---|
| `pandas` / `numpy` | Manipulación de datos |
| `sqlalchemy` / `pymysql` | Conexión a MariaDB |
| `scikit-learn` | K-Means, Gradient Boosting, métricas |
| `imbalanced-learn` | SMOTE para balanceo de clases |
| `matplotlib` / `seaborn` | Visualizaciones |
| `python-dotenv` | Gestión segura de credenciales |
| `joblib` | Serialización del modelo |

---

## Resultados clave

- **10,187 transacciones limpias** cargadas en un star schema en MariaDB.
- **4 segmentos de clientes** identificados con perfiles de negocio diferenciados.
- **Modelo con AUC-ROC 0.984** capaz de detectar el 83% de las ventas que generarán pérdida.
- **3 insights accionables** con simulaciones de impacto económico cuantificado.
- Dashboard interactivo en Power BI (`presentacion/sales_hallazgos.pbix`).
