# INSTALACIÓN DE APACHE HIVE + METASTORE EN POSTGRESQL

Apache Hive es un sistema de data warehouse construido sobre Apache Hadoop, diseñado para facilitar la consulta, análisis y procesamiento de grandes volúmenes de datos almacenados en sistemas distribuidos (como HDFS, Amazon S3, Azure Data Lake, etc.).

- Utiliza un lenguaje similar a SQL llamado HiveQL (o HQL).
- Está pensado para procesamiento por lotes (batch) y análisis de datos (OLAP), no para transacciones en tiempo real (OLTP).
- Traduce internamente las consultas HiveQL a trabajos de MapReduce, Tez o Apache Spark, según la configuración.

- _✅ Ideal para analistas de datos, ingenieros de datos y herramientas de BI, sin necesidad de programar en Java/Scala._

## Componentes
- **Metastore**. Almacena *metadatos*: nombres de tablas, columnas, tipos de datos, particiones. Suele usarse una base de datos relacional (MySQL, PostgreSQL, Derby).
- **Driver (Controlador)**. Coordina la ejecución de la consulta: análisis, compilación, optimización y ejecución.
- **Compiler (Compilador)**. Convierte HiveQL en un plan de ejecución (DAG: *Directed Acyclic Graph*).
- **Execution Engine**. Ejecuta el plan usando motores como **Tez** (por defecto), **MapReduce** (legado) o **Spark**.
- **SerDe** (*Serializer/Deserializer*). Define cómo leer/escribir los datos (ej.: CSV, JSON, Parquet, ORC).

## Arquitectura Básica
Usuario (Beeline, CLI, JDBC/ODBC, herramientas BI)
       ↓
Hive → Driver → Parser → Compilador → Optimizador → Motor de Ejecución
       ↓
Metastore (BD relacional)
       ↓
Almacenamiento: HDFS / S3 / ADLS / GCS

## Formatos de Almacenamiento Recomendados
- **Parquet**. Columnar, compresión eficiente, *predicate pushdown*. ✅ Analítica de alto rendimiento
- **ORC**. Columnar, soporte ACID (desde Hive 3), índices ligeros. ✅ Grandes volúmenes, actualizaciones
- **Avro**. Soporta evolución de esquema, orientado a filas. ✅ Pipelines ETL, streaming
- **TextFile (CSV)**. Legible por humanos. ✅ Solo para desarrollo o depuración

# Requisitos
1. Sistema Operativo Ubuntu 20.04 / 22.04 / 24.04 / 25.10 (Linux, macOS, Windows con WSL)
2. JDK 8
- Hive 2.x --> JDK 8
- Hive 3.x --> JDK 8 o 11
- Hive 4.x (2024–2025) --> JDK 11 o 17 (recomendado: JDK 17 LTS)
3. Hadoop (HDFS + YARN)
- Hadoop --> 3.3.x o superior (Hive 4.x requiere Hadoop ≥ 3.3)
- HDFS --> Debe estar en ejecución y accesible (hdfs dfs -ls /)
- YARN --> Necesario para ejecutar trabajos (MapReduce/Tez/Spark)
4. Base de Datos para el Metastore 
El Metastore de Hive almacena metadatos (esquemas, tablas, particiones). Requiere una BD relacional:
- Derby. Solo modo embedded (1 usuario, desarrollo) ❌ No para producción. ✅ Para pruebas rápidas
- MySQL / MariaDB Producción (usado históricamente). ✔️ Requiere conector JDBC (mysql-connector-java-8.0.x.jar)
- PostgreSQL Alternativa robusta y open-source. ✔️ Muy estable; usa postgresql-42.x.x.jar
- AWS RDS / Azure SQL. En la nube. ✔️ Ideal para entornos gestionados
5. Apache Hive (Binarios)
6. Variables de Entorno (Críticas)