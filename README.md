# Infraestructura de Análisis Ambiental — Ayuntamiento de Madrid

## Arquitectura implementada

La infraestructura sigue un modelo de **Data Lake por zonas** complementado con un **Data Warehouse** para análisis, organizado en tres capas de procesamiento:

```
Fuentes externas
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│                    ZONA DE INGESTA (N8N)                    │
│  Open-Meteo API ──┐                                         │
│                   ├──► raw-zone (MinIO)  ──► RabbitMQ       │
│  datos.madrid.es ─┘                                         │
└─────────────────────────────────────────────────────────────┘
      │ (mensajes RabbitMQ)
      ▼
┌─────────────────────────────────────────────────────────────┐
│                   ZONA DE LIMPIEZA (N8N)                    │
│  RabbitMQ ──► Transformación ──► clean-zone (MinIO)         │
└─────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│                  ZONA CURADA (N8N)                          │
│  clean-zone ──► Fusión y normalización ──► curated-zone     │
│                                            (MinIO CSV)      │
└─────────────────────────────────────────────────────────────┘
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│              DATA WAREHOUSE (PostgreSQL + esquema DW)       │
│  stg_curated_csv ──► dim_tiempo / dim_fuente /              │
│                      dim_estacion / dim_magnitud ──►        │
│                      fact_ambiental_horaria                 │
└─────────────────────────────────────────────────────────────┘
      │                          │
      ▼                          ▼
  Adminer                  Apache Superset
(analistas SQL)          (dashboards políticos)
```

Los datos se recuperan diariamente de forma automática correspondiendo al día anterior a la ejecución. RabbitMQ actúa como bus de mensajes entre la ingesta y la limpieza, desacoplando ambos procesos.

---

## Componentes utilizados

| Componente | Imagen Docker | Función |
|---|---|---|
| **MinIO** | `minio/minio:latest` | Data Lake con tres zonas: `raw-zone`, `clean-zone`, `curated-zone` |
| **N8N** | `n8nio/n8n:latest` | Orquestador de workflows de ingesta, limpieza y curación |
| **RabbitMQ** | `rabbitmq:3-management` | Cola de mensajes entre ingesta y limpieza |
| **PostgreSQL (N8N)** | `postgres:15-alpine` | Base de datos interna de N8N |
| **PostgreSQL (DW)** | `postgres:15-alpine` | Data Warehouse con esquema dimensional |
| **Adminer** | `adminer:latest` | Interfaz web SQL para analistas de datos |
| **Apache Superset** | `apache/superset:latest` (imagen personalizada) | Plataforma de visualización y dashboards |

### Estructura del Data Warehouse

El esquema `dw` implementa un modelo dimensional (estrella):

- `dim_tiempo` — dimensión temporal con granularidad horaria
- `dim_fuente` — origen del dato (Open-Meteo, datos.madrid.es, curated)
- `dim_estacion` — estaciones de medición
- `dim_magnitud` — tipo de magnitud medida
- `fact_ambiental_horaria` — tabla de hechos con métricas ambientales horarias

Además se crean cinco vistas analíticas: `v_dw_resumen_diario`, `v_dw_resumen_mensual`, `v_dw_resumen_anual`, `v_dw_franjas_horarias` y `v_dw_fuente_diaria`.

---

## Configuración y puesta en marcha

### Requisitos previos

- Docker y Docker Compose instalados
- Acceso a internet para descargar las imágenes y los datos de las APIs

### Paso 1 — Crear el archivo `.env`

Copiar el archivo de ejemplo `env.example` y ajustar los valores según el entorno:

El archivo `.env` debe contener las siguientes variables:

```env
# MinIO – Data Lake
MINIO_ROOT_USER=minioadmin
MINIO_ROOT_PASSWORD=minioadmin123
MINIO_API_PORT=9000
MINIO_CONSOLE_PORT=9001

# PostgreSQL – Base de datos de N8N
POSTGRES_USER=n8n
POSTGRES_PASSWORD=n8n123
POSTGRES_DB=n8n

# N8N – Orquestador de workflows
N8N_PORT=5678
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=admin123

# PostgreSQL – Data Warehouse
WAREHOUSE_POSTGRES_USER=madrid
WAREHOUSE_POSTGRES_PASSWORD=madrid123
WAREHOUSE_POSTGRES_DB=madrid_analytics
WAREHOUSE_POSTGRES_PORT=5433
```

### Paso 2 — Levantar la infraestructura

```bash
docker compose up --build
```

Docker Compose levantará todos los servicios en el orden correcto. Los servicios de inicialización (`minio-init`, `rabbitmq-init`) se ejecutarán automáticamente y crearán los buckets y las colas necesarias.

### Paso 3 — Importar los workflows en N8N

1. Acceder a N8N en `http://localhost:5678` con las credenciales definidas en `.env` (`N8N_BASIC_AUTH_USER` / `N8N_BASIC_AUTH_PASSWORD`).
2. Desde el menú lateral, ir a **Workflows → Import from file**.
3. Importar los cuatro archivos JSON en el siguiente orden recomendado:
   - `n8n/workflows/INGEST.json`
   - `n8n/workflows/CLEAN.json`
   - `n8n/workflows/CURATED.json`
   - `n8n/workflows/CURATED-POSTGRES.json`

### Paso 4 — Configurar credenciales en N8N

Tras importar los workflows, es necesario configurar las credenciales en los nodos que las requieren. Se deben crear las siguientes:

**Credencial S3 (para los nodos de MinIO):**

| Campo | Valor |
|---|---|
| Tipo | S3 |
| Access Key ID | valor de `MINIO_ROOT_USER` |
| Secret Access Key | valor de `MINIO_ROOT_PASSWORD` |
| Region | `us-east-1` (o cualquier valor) |
| Endpoint | `http://minio:9000` |
| Force Path Style | activado |

**Credencial RabbitMQ:**

| Campo | Valor |
|---|---|
| Tipo | RabbitMQ |
| Hostname | `rabbitmq` |
| Port | `5672` |
| Username | `guest` |
| Password | `guest` |

**Credencial PostgreSQL (Data Warehouse):**

| Campo | Valor |
|---|---|
| Tipo | PostgreSQL |
| Host | `warehouse-postgres` |
| Port | `5432` |
| Database | valor de `WAREHOUSE_POSTGRES_DB` |
| User | valor de `WAREHOUSE_POSTGRES_USER` |
| Password | valor de `WAREHOUSE_POSTGRES_PASSWORD` |

Una vez creadas las credenciales, asignarlas a cada nodo que lo requiera dentro de cada workflow (nodos S3, RabbitMQ y PostgreSQL).

### Paso 5 — Activar los workflows

- Activar el workflow **CLEAN** para que quede escuchando las colas de RabbitMQ de forma continua.
- El workflow **INGEST** tiene un `Schedule Trigger` configurado para ejecutarse a la 1:00 AM diariamente. Activarlo para que funcione de forma automática.
- Los workflows **CURATED** y **CURATED-POSTGRES** pueden ejecutarse manualmente tras la ingesta o encadenarse según lo deseado.

---

## Flujos de trabajo N8N

### INGEST

Recupera datos del día anterior desde las dos fuentes externas, los almacena en `raw-zone` de MinIO y publica un mensaje en RabbitMQ para notificar al workflow de limpieza.

- **Trigger:** programado a las 01:00 AM (o manual)
- **Fuentes:** Open-Meteo API (meteorología) y Portal de Datos Abiertos de Madrid (calidad del aire)
- **Salida:** archivos JSON en `raw-zone/openmeteo/` y `raw-zone/madrid/`
- **Colas:** `openmeteo-raw` y `madrid-raw`

### CLEAN

Escucha las colas de RabbitMQ, transforma los datos crudos a un formato tabular normalizado y los almacena en `clean-zone`.

- **Trigger:** RabbitMQ Trigger (escucha continua en `openmeteo-raw` y `madrid-raw`)
- **Transformaciones:** aplanado de arrays horarios (Open-Meteo), parseo de CSV con separador `;` (Madrid)
- **Salida:** archivos JSON en `clean-zone/openmeteo/` y `clean-zone/madrid/`

### CURATED

Lee los archivos limpios de ambas fuentes desde `clean-zone`, los fusiona por hora y genera un CSV normalizado con las métricas ambientales combinadas.

- **Salida:** archivo CSV en `curated-zone/YYYY/MM/DD/datos.csv`

### CURATED-POSTGRES

Lee el CSV curado desde MinIO y ejecuta el proceso ETL completo hacia el Data Warehouse insertando en las dimensiones y en la tabla de hechos.

- **Proceso:** `dim_tiempo` → `dim_estacion` → `dim_magnitud` → `fact_ambiental_horaria`

---

## Acceso a los servicios

| Servicio | URL | Credenciales |
|---|---|---|
| **N8N** | `http://localhost:5678` | `N8N_BASIC_AUTH_USER` / `N8N_BASIC_AUTH_PASSWORD` |
| **MinIO Console** | `http://localhost:9001` | `MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD` |
| **RabbitMQ Management** | `http://localhost:15672` | `guest` / `guest` |
| **Adminer** | `http://localhost:8080` | Ver tabla inferior |
| **Apache Superset** | `http://localhost:8088` | `admin` / `admin` |

### Conexión a Adminer

Al acceder a Adminer en `http://localhost:8080`, introducir los siguientes datos de conexión:

| Campo | Valor |
|---|---|
| Sistema | PostgreSQL |
| Servidor | `warehouse-postgres` |
| Usuario | valor de `WAREHOUSE_POSTGRES_USER` |
| Contraseña | valor de `WAREHOUSE_POSTGRES_PASSWORD` |
| Base de datos | valor de `WAREHOUSE_POSTGRES_DB` |

### Conexión a Apache Superset

1. Acceder a `http://localhost:8088` con usuario `admin` y contraseña `admin`.
2. Ir a **Settings → Database Connections → + Database**.
3. Seleccionar **PostgreSQL** e introducir la cadena de conexión:
   ```
   postgresql://madrid:madrid123@warehouse-postgres:5432/madrid_analytics
   ```
   (ajustar usuario y contraseña según los valores del `.env`)
4. Crear los datasets apuntando a las vistas del esquema `dw` para construir los dashboards.