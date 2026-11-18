# INSTALACIÓN DE APACHE HIVE 4.2.1 + METASTORE EN POSTGRESQL 18

Apache Hive es un sistema de data warehouse construido sobre Apache Hadoop, diseñado para facilitar la consulta, análisis y procesamiento de grandes volúmenes de datos almacenados en sistemas distribuidos (como HDFS, Amazon S3, Azure Data Lake, etc.).

- Utiliza un lenguaje similar a SQL llamado HiveQL (o HQL).
- Está pensado para procesamiento por lotes (batch) y análisis de datos (OLAP), no para transacciones en tiempo real (OLTP).
- Traduce internamente las consultas HiveQL a trabajos de MapReduce, Tez o Apache Spark, según la configuración.

- _✅ Ideal para analistas de datos, ingenieros de datos y herramientas de BI, sin necesidad de programar en Java/Scala._

## Componentes

- **Metastore**. Almacena _metadatos_: nombres de tablas, columnas, tipos de datos, particiones. Suele usarse una base de datos relacional (MySQL, PostgreSQL, Derby).
- **Driver (Controlador)**. Coordina la ejecución de la consulta: análisis, compilación, optimización y ejecución.
- **Compiler (Compilador)**. Convierte HiveQL en un plan de ejecución (DAG: _Directed Acyclic Graph_).
- **Execution Engine**. Ejecuta el plan usando motores como **Tez** (por defecto), **MapReduce** (legado) o **Spark**.
- **SerDe** (_Serializer/Deserializer_). Define cómo leer/escribir los datos (ej.: CSV, JSON, Parquet, ORC).

## Arquitectura Básica

```text
Usuario (Beeline, CLI, JDBC/ODBC, herramientas BI)
       ↓
Hive → Driver → Parser → Compilador → Optimizador → Motor de Ejecución
       ↓
Metastore (BD relacional)
       ↓
Almacenamiento: HDFS / S3 / ADLS / GCS
```

## Formatos de Almacenamiento Recomendados

- **Parquet**. Columnar, compresión eficiente, _predicate pushdown_. ✅ Analítica de alto rendimiento
- **ORC**. Columnar, soporte ACID (desde Hive 3), índices ligeros. ✅ Grandes volúmenes, actualizaciones
- **Avro**. Soporta evolución de esquema, orientado a filas. ✅ Pipelines ETL, streaming
- **TextFile (CSV)**. Legible por humanos. ✅ Solo para desarrollo o depuración

# Requisitos

1. **Sistema Operativo Ubuntu** 22.04 / 24.04 / 25.10 / 26.04 (Linux, macOS, Windows con WSL)
2. **JDK 8**

- Hive 4.x (2025–2026) --> JDK 21 (Opcional: JDK 25 LTS)

3. **Hadoop (HDFS + YARN)** [Instalar Apache Hadoop 3.5.0](https://github.com/jllanosb/install-apache-hadoop)

- Hadoop --> 3.5.x o superior (Hive 4.x requiere Hadoop ≥ 3.5)
- HDFS --> Debe estar en ejecución y accesible (hdfs dfs -ls /)
- YARN --> Necesario para ejecutar trabajos (MapReduce/Tez/Spark)

4. **Base de Datos para el Metastore**. El Metastore de Hive almacena metadatos (esquemas, tablas, particiones). Requiere una BD relacional:

- **Derby**. Solo modo embedded (1 usuario, desarrollo) ❌ No para producción. ✅ Para pruebas rápidas
- **MySQL / MariaDB**. Producción (usado históricamente). ✔️ Requiere conector JDBC (mysql-connector-java-8.0.x.jar)
- **PostgreSQL**. Alternativa robusta y open-source. ✔️ Muy estable; usa postgresql-42.x.x.jar
- **AWS RDS / Azure SQL**. En la nube. ✔️ Ideal para entornos gestionados

5. **Apache Hive** (Binarios)
6. **Variables de Entorno** (Críticas)

# Iniciar Servicios Hadoop y Yarn

Opcion A. Autenticarse usuario Hadoop en mismo servidor:

```bash
sudo su - hadoop
```

Opcion B. Autenticarse usuario Hadoop con `SSH` e IP:

```bash
ssh hadoop@172.29.96.93
```

Una vez autenticado ya sea en el servidor o via SSH, Iniciar servicios:

```bash
start-dfs.sh
start-yarn.sh
```

# 1. Instalar PostgreSQL (Ubuntu 24.04)

Actualizar Ubuntu e Instalar PosgreSQL

```bash
sudo apt update && sudo apt upgrade -y
```

Instala dependencias necesarias

```bash
sudo apt install wget ca-certificates gnupg
```

Agrega la clave GPG del repositorio de PostgreSQL

```bash
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | \
  sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg
```

Agrega el repositorio oficial de PostgreSQL

```bash
echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" | \
  sudo tee /etc/apt/sources.list.d/pgdg.list
```

Actualiza e instala PostgreSQL 18

```bash
sudo apt update
sudo apt install postgresql-18 postgresql-client-18
```

Verifica la instalación

```bash
sudo systemctl status postgresql
```

verificar la versión

```bash
psql --version
```

### Crear usuario para hive

Crear usuario y BD para metastore Hive:

```bash
sudo -u postgres psql
```

Crear base de datos dentro de postgres:

```sql
CREATE DATABASE hive_metastore;
CREATE USER hiveuser WITH PASSWORD 'HivePassword123';
GRANT ALL PRIVILEGES ON DATABASE hive_metastore TO hiveuser;
\q
```

## Agregar Permisos al Schema Public

### 🧭 Paso 1: Conéctate a PostgreSQL

Inicia sesión como el usuario administrador (`postgres`):

```bash
sudo -u postgres psql
```

Luego conéctate a la base de datos del metastore de Hive:

```sql
\c hive_metastore;
```

### 🧩 Paso 2: Otorga permisos al usuario `hiveuser`

Ejecuta los siguientes comandos dentro de `psql`:

```sql
ALTER SCHEMA public OWNER TO hiveuser;
GRANT ALL PRIVILEGES ON SCHEMA public TO hiveuser;
GRANT ALL PRIVILEGES ON DATABASE hive_metastore TO hiveuser;
```

👉 Si no quieres cambiar el propietario del esquema, también puedes otorgar solo los permisos necesarios:

```sql
GRANT USAGE ON SCHEMA public TO hiveuser;
GRANT CREATE ON SCHEMA public TO hiveuser;
```

### 🧰 Paso 3: Verifica los permisos

Para comprobar quién es el dueño del esquema `public`, ejecuta:

```sql
\dn+
```

Debería mostrar algo así:

```
 public | hiveuser | ...
```

### 🔁 Paso 4: Vuelve a ejecutar la inicialización del esquema

Salir de `psql`:

```sql
\q
```

# 2. Descargar Apache Hive 4.2.1

Hive 4.2.1 (estable para enterprise productivo real con Java 21). [Consulta nuevas versiones](https://hive.apache.org/general/downloads/)

```bash
cd /tmp
sudo wget https://archive.apache.org/dist/hive/hive-4.2.1/apache-hive-4.2.1-bin.tar.gz
sudo mkdir -p /opt/hive
sudo tar -xzf apache-hive-4.2.1-bin.tar.gz -C /opt/hive --strip-components=1
sudo chown -R hadoop:hadoop /opt/hive
```

# 3. Variables de entorno Hive

Abrir

```bash
sudo -u hadoop nano ~/.bashrc
```

Agregar al final:

```bash

# Apache Hive
export HIVE_HOME=/opt/hive
export PATH=$PATH:$HIVE_HOME/bin
```

Actualizar

```bash
source ~/.bashrc
```

Verificar version:

```
hive --version
```

Resultado de la version:

```txt
Hive 4.2.1
Git git://MacBook-Pro.fios-router.home/Users/ngangam/commit/hive -r 4df4d75bf1e16fe0af75aad0b4179c34c07fc975
Compiled by ngangam on Sun Apr 3 16:58:16 EDT 2022
From source with checksum 5da234766db5dfbe3e92926c9bbab2af
```

# 4. Descargar Driver JDBC PostgreSQL

Descargar version mas compatible.

```bash
cd /opt/hive/lib
wget https://jdbc.postgresql.org/download/postgresql-42.7.14.jar
sudo chown hadoop:hadoop postgresql-42.7.14.jar
```

# 5. Configuración Hive Metastore

Configurar `hive-site.xml`

```
sudo -u hadoop nano /opt/hive/conf/hive-site.xml
```

Agregar las Lineas:

```xml
<configuration>

  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:postgresql://localhost:5432/hive_metastore</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>org.postgresql.Driver</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hiveuser</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>HivePassword123</value>
  </property>

  <property>
    <name>datanucleus.autoCreateSchema</name>
    <value>false</value>
  </property>

  <property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
  </property>

</configuration>
```

# 6. Habilitar Java 21 (Solo si no esta instalado)

`Nota`: Debe haber instalado al configurar Hadoop 3.5.0

## 🔹 Paso 1: Instalar OpenJDK 21

En Ubuntu/Debian:

```bash
sudo apt update
sudo apt install openjdk-21-jdk -y
```

## 🔹Paso 2: Configurar Hive y Hadoop para usar Java 21

Verifica qué versiones de Java tienes:

```bash
sudo update-alternatives --config java
```

Selecciona la opción que apunte a Java 17 (normalmente `/usr/lib/jvm/java-21-openjdk-amd64/jre/bin/java`).

También puedes exportar la ruta manualmente en tu entorno:

```bash
sudo nano ~/.bashrc
```

Para hacerlo permanente, agrega esas líneas a tu archivo:

```bash

# Java 21
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
```

y luego recarga el entorno:

```bash
source ~/.bashrc
```

## 🔹Paso 3: Verifica la versión de Java usada por Hive

Ejecuta:

```bash
java -version
```

Debe mostrar algo como:

```
openjdk version "21.0.21.0.3" 2026-10-24
OpenJDK Runtime Environment (build 21.0.21.0.3+1-1-26.04.1-Ubuntu)
OpenJDK 64-Bit Server VM (build 21.0.21.0.3+1-1-26.04.1-Ubuntu, mixed mode, sharing)
```

# 7. Inicializar BD Metastore

```bash
cd /opt/hive
schematool -dbType postgres -initSchema
```

si → Executed successfully

```
Initialization script completed
schemaTool completed
```

Hive está ok.

# 8. Crear warehouse en HDFS

Creando Warehouse para hive:

```bash
hdfs dfs -mkdir -p /user/hive/warehouse
hdfs dfs -chmod -R 777 /user/hive/warehouse

hdfs dfs -mkdir -p /tmp/hive
hdfs dfs -chmod -R 777 /tmp/hive
```

Asignando Permisos:

```bash
hdfs dfs -chown -R hive:hadoop /user/hive/warehouse
hdfs dfs -chown -R hadoop:hadoop /user/hive/warehouse
hdfs dfs -chmod -R 777 /user/hive/warehouse
```

# 9. Probar Inicia Hive

Iniciar Servicio

```bash
hive --service hiveserver2 &
```

# 10. Configurar Hive con Beeline (CLI)

🔹1️⃣ Edita el archivo `core-site.xml`

```bash
sudo nano /opt/hadoop/etc/hadoop/core-site.xml
```

Abre en tu nodo maestro (donde está Hadoop) y agregar las lineas dentro de `<configuration>`:

```xml

  <property>
    <name>hadoop.proxyuser.hadoop.hosts</name>
    <value>*</value>
  </property>

  <property>
    <name>hadoop.proxyuser.hadoop.groups</name>
    <value>*</value>
  </property>

  <!-- (Opcional) ubicación del directorio temporal -->
  <property>
    <name>hadoop.tmp.dir</name>
    <value>/opt/hadoop/tmp</value>
  </property>

```

# 11. Configurar Beeline para conexion externa

Configurar si y solo si sale

```
Error: Could not open client transport with JDBC Uri: jdbc:hive2://localhost:10000: java.net.ConnectException: Connection refused: connect (state=08S01,code=0)
Beeline version 2.3.9 by Apache Hive
beeline> show databases;
No current connection
```

```bash
sudo nano $HIVE_HOME/conf/hive-site.xml
```

Para que acepte conexiones externas

```xml

<property>
  <name>hive.server2.thrift.port</name>
  <value>10000</value>
</property>

<property>
  <name>hive.server2.thrift.bind.host</name>
  <!--value>localhost</value-->
  <value>0.0.0.0</value>
  <description>Bind HiveServer2 to all interfaces</description>
</property>

```

Luego reinicia HiveServer2:

```bash
pkill -f HiveServer2
hive --service hiveserver2 &
```

# 12. Agregar librerias faltantes

🔹Paso 1: Descargar commons-collections v3.2.2

Ejecuta en tu servidor (hadoop-master):

```bash
cd /opt/hive/lib
sudo wget https://repo1.maven.org/maven2/commons-collections/commons-collections/3.2.2/commons-collections-3.2.2.jar
```

✅ Esta es la versión compatible con la mayoría de distribuciones de Hadoop/Hive.

🔹Paso 2: Verifica que no haya conflicto con commons-collections4

Asegúrate de que también tengas la versión 4 (usada por Hive 3):

```bash
ls /opt/hive/lib/commons-collections4-*.jar
```

Si no está, instálala también (aunque Hive normalmente la incluye):
bash

```bash
sudo wget https://repo1.maven.org/maven2/org/apache/commons/commons-collections4/4.6.0/commons-collections4-4.6.0.jar
```

✅ Tener ambas versiones (commons-collections y commons-collections4) es normal y necesario en Hive 4.x.

🔹🔁 Paso 3: Vuelve a iniciar el Metastore

Luego inicia el Metastore:

```bash
hive --service metastore &
```

Ahora debería iniciar sin el error de `ClassNotFoundException`.
🔹🛠️ Verificación adicional
Puedes confirmar que el JAR está en el classpath listando:

```bash
ls -l /opt/hive/lib/commons-collections*.jar
```

Deberías ver algo como:

```bash
commons-collections-3.2.2.jar
commons-collections4-4.4.jar
commons-collections4-4.6.0.jar
```

## 2️⃣ REINICIAR SERVICIOS DE `HADOOP` Y `HIVE`

Detener servicios en el servidor:

```bash
stop-yarn.sh
stop-dfs.sh

# Si HiveServer2 está en ejecución:
pkill -f HiveServer2
pkill -f HiveMetaStore
```

Luego reinicia todo:

```bash
start-dfs.sh
start-yarn.sh
hive --service metastore &
sleep 10
hive --service hiveserver2 &
```

# 13. Acceder a Hive `Abrir una Nueva Ventana` de Terminal para realizar la conexión

Acceso por Defecto:

```bash
beeline -u "jdbc:hive2://172.29.96.93:10000/default" -n hadoop
```

Acceso Localhost:

```bash
beeline -u jdbc:hive2://localhost:10000 -n hadoop -p --verbose=true
```

Acceso WSL:

```bash
beeline -u jdbc:hive2://172.29.96.93:10000 -n hadoop -p --verbose=true
```

Acceso IP_PUBLICA

```bash
beeline -u jdbc:hive2://IP_PUBLICA:10000 -n hadoop -p --verbose=true
```

`Nota`: Salir de beeline `!q`

# 14. Probando Hive usando los Comandos Básicos SQL

Revisar `usuario` activo y Version de `Hive` corriendo::

```sql
SELECT current_user();
SELECT version();
```

Listar bases de datos:

```sql
SHOW DATABASES;
```

Crear y Activar uso de base de datos `test_db` :

```sql
CREATE DATABASE IF NOT EXISTS test_db
LOCATION '/user/hive/warehouse/test_db';
USE test_db;
```

Crear una tabla en la base de datos tipo `TextFile`:

```sql
CREATE TABLE test_db.persona (id INT, name STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '|'
LINES TERMINATED BY '\n'
STORED AS TEXTFILE;
```

Verificar la tabla creada

```sql
SHOW TABLES IN test_db;
```

Descripción de las columnas y sus tipos de datos.

```sql
DESC test_db.persona;
```

Descripción más detallada de la tabla

```sql
DESC FORMATTED test_db.persona;
```

Insertar un registro de modo tradicional (Hive trabaja formatos ORC, PARQUET, Avro en produccion):

```sql
INSERT INTO test_db.persona VALUES (1, 'Jaime');
-- Generará errores
```

Consultar registros:

```sql
SELECT * FROM test_db.persona;
```

Eliminar base de datos y todas las tablas:

```sql
DROP DATABASE test_db CASCADE;
```

# 15. Tabla en produccion formato `ORC`

Crear base de datos, poner uso, crear tabla particionada:

```sql
CREATE DATABASE IF NOT EXISTS bigdata
LOCATION '/user/hive/warehouse/bigdata';

USE bigdata;

CREATE TABLE IF NOT EXISTS bigdata.empleados (
    id_empleado        INT,
    nombre             STRING,
    departamento       STRING,
    salario            DECIMAL(10,2),
    fecha_ingreso      DATE
)
PARTITIONED BY (
    anio INT,
    mes  INT
)
STORED AS ORC
TBLPROPERTIES (
    'orc.compress'='SNAPPY',
    'transactional'='false'
);
```

```bash
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;
# Por si hubiera errores de memoria
SET hive.exec.max.dynamic.partitions=1000;
SET hive.exec.max.dynamic.partitions.pernode=100;
```

Habilitar `Permisos en HDFS de acceso ala base de datos para el usuario Hive`

Asinar y verificar permisos en HDFS:

```bash
hdfs dfs -chown -R hadoop:hadoop /user/hive/warehouse/bigdata.db
hdfs dfs -chown -R hive:hadoop /user/hive/warehouse/bigdata.db
hdfs dfs -chmod -R 777 /user/hive/warehouse/bigdata.db
hdfs dfs -ls /user/hive/warehouse/bigdata.db
```

Reiniciar servicios:

```bash
# Detener
pkill -f HiveMetaStore
pkill -f HiveServer2

# Iniciar nuevamente servicios
hive --service metastore &
sleep 10
hive --service hiveserver2 &
```

Cargar Datos:

```sql
INSERT INTO TABLE bigdata.empleados
PARTITION (anio=2026, mes=1)
VALUES (1, 'Jaime', 'Sistemas', 3113.15, '2026-01-08');
```

```sql
INSERT INTO TABLE bigdata.empleados
PARTITION (anio=2026, mes=1)
VALUES (2, 'Diana', 'Administradora', 2013.15, '2026-01-09');
```

```sql
INSERT INTO TABLE bigdata.empleados
PARTITION (anio=2026, mes=1)
VALUES (3, 'Susana', 'Cajera', 2013.15, '2026-01-10');
```

```sql
INSERT INTO TABLE bigdata.empleados
PARTITION (anio=2026, mes=1)
VALUES (4, 'Iris', 'Secretaria', 2013.15, '2026-01-07');
```

Consultar registros:

```sql
SELECT * FROM bigdata.empleados;
```

# 16. Creando Tabla CARRERA en formato `PARQUET`

1. Crear tabla carrera en formato Parquet

```sql
-- Usar la base de datos BIGDATA
USE bigdata;

-- Crear tabla carrera en formato Parquet
CREATE TABLE IF NOT EXISTS bigdata.carrera (
    id_carrera INT,
    nombre_carrera STRING,
    facultad STRING,
    duracion_anios INT,
    creditos_totales INT,
    fecha_creacion DATE,
    activa BOOLEAN
)
PARTITIONED BY (
    anio INT,
    mes INT
)
STORED AS PARQUET
TBLPROPERTIES (
    'parquet.compression'='SNAPPY',  -- Compresión
    'parquet.block.size'='134217728' -- Tamaño de bloque 128MB
);
```

2. Verificar la tabla creada

```sql
-- Ver estructura
DESCRIBE FORMATTED bigdata.carrera;

-- Ver tablas en la base de datos
SHOW TABLES;
```

3. Métodos para insertar datos

Método 1: INSERT con VALUES (directo)

```sql
INSERT INTO TABLE bigdata.carrera
PARTITION (anio=2026, mes=1)
VALUES (
    1,
    'Ingeniería de Sistemas',
    'Facultad de Ingeniería',
    5,
    220,
    '2020-01-15',
    TRUE
);
```

Método 2: INSERT múltiple con VALUES

```sql
INSERT INTO TABLE bigdata.carrera
PARTITION (anio=2026, mes=1)
VALUES
    (1, 'Ingeniería de Sistemas', 'Facultad de Ingeniería', 5, 220, '2020-01-15', TRUE),
    (2, 'Medicina', 'Facultad de Medicina', 7, 350, '2018-08-01', TRUE),
    (3, 'Administración', 'Facultad de Ciencias Económicas', 5, 200, '2019-03-10', TRUE);
```

Método 3: INSERT con SELECT (recomendado)

```sql
INSERT INTO TABLE bigdata.carrera
PARTITION (anio=2026, mes=1)
SELECT
    1 as id_carrera,
    'Ingeniería de Sistemas' as nombre_carrera,
    'Facultad de Ingeniería' as facultad,
    5 as duracion_anios,
    220 as creditos_totales,
    '2020-01-15' as fecha_creacion,
    TRUE as activa
FROM (SELECT 1) tmp;  -- Tabla dummy para generar una fila
```

Método 4: INSERT dinámico con particiones variables

```sql
-- Habilitar particiones dinámicas
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;

INSERT INTO TABLE bigdata.carrera
PARTITION (anio, mes)
SELECT
    1, 'Ingeniería de Sistemas', 'Facultad de Ingeniería', 5, 220, '2020-01-15', TRUE,
    2026, 1
FROM (SELECT 1) tmp;
```

5. Consultar datos insertados

```sql
-- Ver todos los datos
SELECT * FROM bigdata.carrera;

-- Ver por partición específica
SELECT * FROM bigdata.carrera WHERE anio=2026 AND mes=1;
```

# 17. Creando Tabla ALUMNO en formato `PARQUET`

1. Crear tabla carrera en formato Parquet

```sql
-- Usar la base de datos BIGDATA
USE bigdata;

-- Opción 1: Tabla básica con particiones
CREATE TABLE IF NOT EXISTS bigdata.alumno (
    id_alumno INT,
    nombre STRING,
    apellido STRING,
    dni STRING,
    fecha_nacimiento DATE,
    correo STRING,
    telefono STRING,
    direccion STRING,
    id_carrera INT,
    semestre_actual INT,
    promedio DECIMAL(4,2),
    estado STRING  -- 'ACTIVO', 'INACTIVO', 'EGRESADO', etc.
)
PARTITIONED BY (
    anio_ingreso INT,
    semestre_ingreso INT
)
STORED AS PARQUET
TBLPROPERTIES (
    'parquet.compression'='SNAPPY',
    'comment'='Tabla de alumnos en formato Parquet'
);
```

Opción 2: Con más restricciones y optimizaciones

```sql
CREATE TABLE IF NOT EXISTS bigdata.alumno (
    id_alumno INT COMMENT 'Identificador único del alumno',
    nombre STRING COMMENT 'Nombre del alumno',
    apellido STRING COMMENT 'Apellido del alumno',
    dni STRING COMMENT 'Documento Nacional de Identidad',
    fecha_nacimiento DATE COMMENT 'Fecha de nacimiento',
    correo STRING COMMENT 'Correo electrónico',
    telefono STRING COMMENT 'Número de teléfono',
    direccion STRING COMMENT 'Dirección de residencia',
    id_carrera INT COMMENT 'ID de la carrera que estudia',
    semestre_actual INT COMMENT 'Semestre actual que cursa',
    promedio DECIMAL(4,2) COMMENT 'Promedio académico',
    estado STRING COMMENT 'Estado: ACTIVO, INACTIVO, EGRESADO'
)
PARTITIONED BY (
    anio_ingreso INT COMMENT 'Año de ingreso a la universidad',
    semestre_ingreso INT COMMENT 'Semestre de ingreso (1 o 2)'
)
CLUSTERED BY (id_carrera) INTO 4 BUCKETS  -- Opcional: bucketing por carrera
STORED AS PARQUET
TBLPROPERTIES (
    'parquet.compression'='SNAPPY',
    'parquet.block.size'='268435456',  -- 256MB
    'parquet.page.size'='1048576',     -- 1MB
    'transactional'='false'
);
```

Opción 3: Tabla sin particiones (más simple)

```sql
CREATE TABLE IF NOT EXISTS bigdata.alumno_simple (
    id_alumno INT,
    nombre STRING,
    apellido STRING,
    dni STRING,
    fecha_nacimiento DATE,
    id_carrera INT,
    anio_ingreso INT,
    semestre_ingreso INT
)
STORED AS PARQUET;
```

3. Verificar la creación

```sql
-- Ver estructura detallada
DESCRIBE FORMATTED alumno;

-- Ver solo columnas
DESCRIBE alumno;

-- Ver tablas en la base de datos
SHOW TABLES LIKE 'alumno*';
```

4. Insertar datos - Diferentes métodos:

Método 1: INSERT simple con VALUES

```sql
-- Configurar para mejor rendimiento
SET hive.exec.dynamic.partition=true;
SET hive.exec.dynamic.partition.mode=nonstrict;

INSERT INTO TABLE bigdata.alumno
PARTITION (anio_ingreso=2026, semestre_ingreso=1)
VALUES (
    1,
    'Juan',
    'Pérez',
    '12345678',
    '2000-05-15',
    'juan.perez@email.com',
    '999888777',
    'Av. Siempre Viva 123',
    1,  -- id_carrera (Ing. Sistemas)
    3,  -- semestre_actual
    15.5,
    'ACTIVO'
);
```

Método 2: INSERT múltiple con VALUES

```sql
INSERT INTO TABLE bigdata.alumno
PARTITION (anio_ingreso=2026, semestre_ingreso=1)
VALUES
    (1, 'Juan', 'Pérez', '12345678', '2000-05-15', 'juan@email.com', '999888777', 'Av. 123', 1, 3, 15.5, 'ACTIVO'),
    (2, 'María', 'Gómez', '87654321', '2001-03-20', 'maria@email.com', '999777666', 'Calle 456', 2, 5, 16.8, 'ACTIVO'),
    (3, 'Carlos', 'López', '11223344', '1999-11-10', 'carlos@email.com', '999666555', 'Jr. 789', 1, 7, 14.2, 'ACTIVO');
```

Método 3: INSERT con SELECT (recomendado)

```sql
INSERT INTO TABLE bigdata.alumno
PARTITION (anio_ingreso, semestre_ingreso)
SELECT
    1 as id_alumno,
    'Ana' as nombre,
    'Torres' as apellido,
    '55667788' as dni,
    '2002-08-25' as fecha_nacimiento,
    'ana@email.com' as correo,
    '999555444' as telefono,
    'Av. Universitaria 456' as direccion,
    3 as id_carrera,  -- Administración
    2 as semestre_actual,
    17.3 as promedio,
    'ACTIVO' as estado,
    2025 as anio_ingreso,  -- Esta columna va a la partición
    2 as semestre_ingreso  -- Esta columna va a la partición
FROM (SELECT 1) tmp;  -- Tabla dummy para generar una fila
```

5. Consultar datos

```sql
-- Ver todos los alumnos
SELECT * FROM bigdata.alumno LIMIT 10;
```

# 18. Soluciones de Permisos de Escritura (Test)

`Nota`: Salir de jdbc:hive2://ip:10000/default> digita `!q`

Solucion individual:

```bash
hdfs dfs -chown -R hive:hadoop /user/hive/warehouse/basededatoscreada.db
hdfs dfs -chown -R hadoop:hadoop /user/hive/warehouse/basededatoscreada.db
hdfs dfs -chmod -R 777 /user/hive/warehouse/basededatoscreada.db
```

Verificar:

```bash
hdfs dfs -ls /user/hive/warehouse/basededatoscreada.db
```

Se visualiza asi `hive hadoop ...`:

```
drwxrwxr-x hive hadoop alumno
drwxrwxr-x hive hadoop carrera
```

Reiniciar servicios:

```bash
# Detener
pkill -f HiveMetaStore
pkill -f HiveServer2

# Iniciar nuevamente servicios
hive --service metastore &
sleep 10
hive --service hiveserver2 &
```

`Nota`: En Produccion se usa el permiso `775`

© 2026 Jaime Llanos Bardales.

Este trabajo está bajo una licencia [Creative Commons Attribution 4.0 Internacional](LICENSE).
