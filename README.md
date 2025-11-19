# Proyecto de Procesamiento de Datos Spotify: Arquitectura, Pipeline y Desarrollo

## 1. Introducción General del Proyecto

Este proyecto aborda la construcción de un pipeline moderno de datos utilizando un conjunto de tecnologías y enfoques de la ingeniería de datos actual.  
Partiendo de un dataset estilo "Spotify" en formato Parquet (con tablas de hechos y dimensiones), se buscó replicar un flujo profesional que abarca:

- extracción e ingesta,
- procesamiento por capas (Bronze → Silver → Gold),
- modelado de datos orientado a analytics,
- y generación dinámica de consultas para exploración avanzada.

El objetivo principal fue comprender, diseñar y poner en práctica la arquitectura Lakehouse en el entorno de Azure.

---

## 2. Configuración Inicial en Azure

### 2.1 Creación de la Cuenta Azure
El proyecto comenzó en Microsoft Azure mediante:

- la creación de una suscripción temporal, free trial.
- habilitación de acceso a servicios administrados,
-  y configuración inicial para trabajar con orquestación y bases de datos.

### 2.2 Recursos Implementados (SQL Database, Data Factory)

Se desplegaron dos servicios clave:

- **Azure SQL Database**, pensado como repositorio estructurado o staging transaccional.
- **Azure Data Factory**, utilizado para:
  - iterar sobre archivos,
  - extraerlos desde la fuente,
  - transformarlos a .parquet y
  - cargarlos en un Data Lake.

### 2.3 Pipeline Inicial en ADF y Limitaciones con Unity Catalog

El flujo planteado requería escribir datos dentro de un entorno Databricks gobernado por **Unity Catalog**, que exige:

- permisos elevados,
- almacenamiento en Volumes administrados,
- y un workspace con plan Premium/Enterprise.

Dado que el entorno no cumplía esos requisitos, no fue posible integrar ADF con Unity Catalog. Esto obligó a replantear la arquitectura.
Es de notar que por región o flag no se me otorgó acceso a clusteres en Azure Databricks, impidiendo incluso trabajar con el servicio normal del free.

---

## 3. Migración a Databricks Community Edition

### 3.1 Descarga de Archivos Parquet desde la Fuente

Como alternativa práctica, se descargaron manualmente los archivos Parquet correspondientes a:

- `FactStream`,
- `DimUser`,
- `DimTrack`,
- `DimArtist`,
- `DimDate`.

### 3.2 Carga Manual de Datos al Entorno Databricks

Los archivos se subieron directamente al almacenamiento de Databricks (DBFS / Volumes), permitiendo reconstruir desde allí todo el pipeline sin depender de integración Azure → UC.

### 3.3 Justificación del Cambio de Arquitectura

La migración respondió a tres razones principales:

- evitar las restricciones de Unity Catalog para ADF,
- mantener un entorno gratuito y plenamente funcional,
- continuar el proyecto con foco en la arquitectura Lakehouse y no en las limitaciones de permisos.

---

## 4. Implementación del Modelo Lakehouse

El proyecto siguió el patrón de madallon:

### 4.1 Capa Bronze: Ingesta y Persistencia de Datos Crudos

- Se almacenaron los Parquet tal como se recibieron.
- Se implementó ingesta **incremental simulada** mediante `readStream`.
- Se agregaron metadatos como `ingestion_ts` y `source_file`.
- Se utilizó Delta Lake en modo append.

### 4.2 Capa Silver: Limpieza, Estandarización y Modelado

Las tablas fueron estandarizadas:

- tipos de datos,
- nombres de columnas,
- relaciones entre dimensiones y hechos,
- calidad básica de datos,
- conformidad entre claves.

Silver representa un modelo **limpio y analítico**, listo para explotar.

### 4.3 Capa Gold: Definición de KPIs y Modelos Analíticos

La capa Gold se centra en métricas de negocio como:

- total de streams,
- horas escuchadas,
- rankings por artista o track,

Estas tablas son consumibles por dashboard de Databricks o cualquier otro servicio tal Power Bi o Tableu.

---

## 5. Procesos y Flujos Desarrollados

### 5.1 Ingesta Incremental (Streaming Simulado)

Aunque la ingesta original venía como batch (Parquet), se emuló un flujo continuo con:

- `readStream`
- checkpoints
- Delta Lake en **modo append**
- triggers `availableNow`

### 5.2 Limpieza y Conformación de Dimensiones

Las dimensiones fueron normalizadas y preparadas para análisis:

- `DimUser`
- `DimTrack`
- `DimArtist`
- `DimDate`

Esto habilita un modelo tipo **star schema**.

### 5.3 Enriquecimiento del Fact Stream

`FactStream` fue limpiado, tipado y relacionado con sus dimensiones, dejando el modelo listo para:

- joins en Silver,
- agregados en Gold.

### 5.4 Preparación de Tablas para Análisis y Dashboards

Se diseñaron consultas y transformaciones que combinan:

- hechos + dimensiones
- vistas para KPI
- agregados diarios
- datasets para dashboards

--------------------------------------------------

# Append A: Jinja2 para Generación Dinámica de SQL

### A.1 ¿Qué es Jinja2?

Es un motor de plantillas en Python que permite generar texto dinámicamente.  
En ingeniería de datos se utiliza para crear:

- consultas SQL automáticas,
- scripts repetitivos,
- configuraciones parametrizadas.

### A.2 Uso de Jinja2 en el Proyecto

Se utilizó para construir un **SQL dinámico** sobre las tablas Silver:

- seleccionando columnas según dimensión,
- generando joins automáticamente,
- permitiendo exploración flexible sin reescribir SQL.

### A.3 Beneficios

- Menos errores humanos  
- Reutilización de joins  
- Flexibilidad ante cambios de columnas o tablas  
- Base para automatización avanzada  



# Append B: Delta Live Tables (DLT)

### B.1 ¿Qué es DLT?

Delta Live Tables es un framework declarativo de Databricks que permite:

- definir pipelines con funciones,
- mantener calidad del dato,
- gestionar dependencias,
- operar en batch o streaming de forma automática.

### B.2 Por qué no se utilizó

DLT **requiere Unity Catalog habilitado y un plan pago**, algo no disponible en Databricks Community Edition.

### B.3 Ventajas Conceptuales

- lineage automático  
- monitoreo avanzado  
- manejo de calidad del dato  
- menos código imperativo  
- pipelines reproducibles y escalables  

---

# Append C: Slowly Changing Dimensions (SCD)

### C.1 Tipos utilizados

- **SCD 0**: el dato no cambia  
- **SCD 1**: se sobrescribe el dato (sin historial)  
- **SCD 2**: se genera una nueva versión con historial  

### C.2 Cuándo aplicarlos

- cambios de perfil del usuario → SCD2  
- correcciones menores → SCD1  
- datos inmutables (categorías) → SCD0  

### C.3 Ejemplos

- `DimUser`: historial de cambios en perfil → SCD2  
- `DimArtist`: corrección de nombre → SCD1  
- `DimTrack`: valores estables → SCD0  



