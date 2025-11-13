# Data Mining - Deber 4

## Arquitectura

El proyecto usa Docker Compose con 4 servicios:

- **postgres**: Base de datos con esquemas `raw` y `analytics`.
- **pgadmin**: UI web para administrar Postgres (era opcional).
- **pyspark-notebook**: Jupyter con Spark para ingesta de datos.
- **obt-builder**: Servicio que ejecuta el script CLI para construir la OBT.

## Setup

1. **Clonar el repositorio**

2. **Configurar variables de ambiente**

Copiar `.env.example` a `.env` y ajustar valores según tu entorno:
```bash
cp .env.example .env
```

### Variables de Ambiente

Editar el archivo `.env` con las siguientes variables:

| Variable | Descripción |
|----------|-------------|
| **PostgreSQL** | |
| `POSTGRES_HOST` | Nombre del servicio en Docker |
| `POSTGRES_PORT` | Puerto de PostgreSQL |
| `POSTGRES_DB` | Nombre de la base de datos |
| `POSTGRES_USER` | Usuario de PostgreSQL |
| `POSTGRES_PASSWORD` | Contraseña de PostgreSQL |
| `PG_SCHEMA_RAW` | Esquema para datos crudos |
| `PG_SCHEMA_ANALYTICS` | Esquema para OBT |
| **pgAdmin** | |
| `PGADMIN_EMAIL` | Email para login en pgAdmin |
| `PGADMIN_PASSWORD` | Contraseña para pgAdmin |
| **Spark** | |
| `SPARK_DRIVER_MEMORY` | Memoria para Spark driver |
| `SPARK_EXECUTOR_MEMORY` | Memoria para Spark executor |
| **Ingesta** | |
| `SOURCE_BASE` | URL base de datos NYC TLC |
| `TAXI_ZONE_URL` | URL del archivo taxi_zone_lookup.csv |
| `START_YEAR` | Año inicial para ingesta |
| `END_YEAR` | Año final para ingesta |
| `SERVICES` | Servicios a ingestar: `yellow , green` |
| `RUN_ID` | Identificador único de la ejecución |
| `BATCH_SIZE` | Meses a procesar por batch |
| **OBT Builder** | |
| `OBT_MODE` | Modo: `full` o `by-partition` |
| `OBT_YEAR_START` | Año inicial para construir OBT |
| `OBT_YEAR_END` | Año final para construir OBT |
| `OBT_OVERWRITE` | `true` para sobrescribir, `false` para solo nuevas |

3. **Levantar servicios base**
```bash
docker compose up -d postgres pgadmin pyspark-notebook
```

Esto levanta:
- Postgres en `localhost:5432`
- pgAdmin en `localhost:8080`
- Jupyter en `localhost:8888`

## Ingesta de Datos (RAW)

1. **Acceder a Jupyter**

Ir a `http://localhost:8888` (sin token requerido)

2. **Ejecutar notebook de ingesta**

Abrir y ejecutar `01_ingesta_parquet_raw.ipynb`

Este notebook:
- Descarga Parquet files de Yellow y Green taxis (2015-2025)
- Procesa y estandariza columnas
- Carga datos a `raw.yellow_taxi_trip` y `raw.green_taxi_trip`
- Carga `raw.taxi_zone_lookup`
- Agrega metadatos de ingesta (run_id, source_year, source_month, ingested_at_utc)

El proceso puede tomar varias horas dependiendo de la conexión.

## Construcción de OBT

La One Big Table (`analytics.obt_trips`) se construye desde las tablas RAW usando el servicio `obt-builder`.

### Construcción (3 años)
```bash
docker compose build obt-builder
docker compose run --rm obt-builder --overwrite
```

### Opciones del script

El script `build_obt.py` acepta los siguientes argumentos:
```bash
--mode {full, by-partition}     # Modo de construcción (default: full)
--year-start YYYY                # Año inicial (default: 2020)
--year-end YYYY                  # Año final (default: 2022)
--overwrite                      # Sobrescribir particiones existentes
```

Ejemplos:
```bash
# Construir solo años específicos
docker compose run --rm obt-builder --year-start 2020 --year-end 2021

# Modo by-partition (solo particiones faltantes)
docker compose run --rm obt-builder --mode by-partition
```

### Características de la OBT

La tabla `analytics.obt_trips` incluye:

- **Tiempo**: pickup_datetime, dropoff_datetime, pickup_hour, pickup_dow, month, year
- **Ubicación**: pu_location_id, pu_zone, pu_borough, do_location_id, do_zone, do_borough
- **Servicio**: service_type, vendor_id, rate_code_id, payment_type, trip_type
- **Montos**: fare_amount, tip_amount, tolls_amount, total_amount, etc.
- **Derivadas**: trip_duration_min, avg_speed_mph, tip_pct
- **Metadatos**: run_id, source_year, source_month, ingested_at_utc

El script usa **COPY** en lugar de INSERT para máxima velocidad y crea índices automáticamente al finalizar.

## Machine Learning

### Ejecutar notebook de ML

Abrir y ejecutar `ml_total_amount_regression.ipynb`

El notebook implementa:

1. **Modelos from-scratch (NumPy)**:
   - SGD Regression
   - Ridge Regression (L2)
   - Lasso Regression (L1)
   - Elastic Net (L1 + L2)

2. **Modelos scikit-learn equivalentes**:
   - SGDRegressor
   - Ridge
   - Lasso
   - ElasticNet

Todos los modelos usan:
- Mismo preprocesamiento (scaling, OHE)
- Polynomial Features (degree=2)
- Mismo split temporal (Train/Val/Test)
- Mismas seeds para reproducibilidad

### Target y Features

**Target**: `total_amount`

**Features disponibles en pickup** (sin leakage):
- Numéricas: trip_distance, passenger_count, pickup_hour, pickup_dow, month, year
- Categóricas: service_type, pu_borough, pu_zone, vendor_id, rate_code_id
- Flags derivados: is_rush_hour, is_weekend

### Métricas

- RMSE (primaria)
- MAE
- R²

Comparación completa entre implementaciones propias y sklearn en validación y test.

## Validación de Datos

- Usar pgAdmin en `http://localhost:8080` para explorar las tablas visualmente creadas en RAW y ANALYTICS.
- Se dispone de un archivo PDF que contiene capturas de pantalla como evidencia del proceso, recopiladas durante el desarrollo del proyecto.

## Checklist de Aceptación

- [x] **RAW** en Postgres: `raw.yellow_taxi_trip`, `raw.green_taxi_trip`, `raw.taxi_zone_lookup` (2015–2025).
- [x] **OBT** `analytics.obt_trips` creada por **obt-builder** (comando reproducible, logs).
- [x] **ML**: 4 modelos **from-scratch** + 4 **sklearn** (mismo preprocesamiento y split).
- [x] **Comparativa**: tabla RMSE/MAE/R² (validación y test) + tiempos.
- [x] **Diagnóstico**: residuales y errores por buckets.
- [x] **README**: comandos de ingesta, **creación OBT** (comando que yo ejecutaré), ejecución notebook, variables `.env`.
- [x] Seeds fijas; resultados reproducibles.

## Autor

- Anahí Andrade (00323313)
- Data Mining - CMP 5002
- Universidad San Francisco de Quito
