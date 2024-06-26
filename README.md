# datos-no-estructurados
1. [Creación del stage y el fichero bruto con datos no estructurados (JSON)](#schema1)
2. [Parseo y análisis del fichero no estructurado JSON](#schema2)
3. [Manejo de datos nested y arrays](#schema3)
4. [Jerarquías y flatten del fichero no estructurado JSON](#schema4)
5. [Inserción de datos no estructurados JSON en la tabla objetivo (estructurados)](schema5)


<hr>

<a name="schema1"></a>

## 1. Creación del stage y el fichero bruto con datos no estructurados (JSON)


- Creación de esquema y internal_stage
```SQL
CREATE OR REPLACE SCHEMA MANAGE_DB.INTERNAL_STAGES
CREATE OR REPLACE stage MANAGE_DB.INTERNAL_STAGES.JSONSTAGE
```
- Podemos realizar carga desde un fichero local al internal stage creado o bien crear un external stage a la URL del fichero JSON
```SQL
CREATE OR REPLACE stage MANAGE_DB.EXTERNAL_STAGES.JSONSTAGE
     url='s3://bucketsnowflake-jsondemo';

// Definimos el formato tipo JSON
CREATE OR REPLACE file format MANAGE_DB.FILE_FORMATS.JSONFORMAT
    TYPE = JSON;
```
- Creamos la fila con el JSON

```SQL
CREATE OR REPLACE table PRIMERABBDD.PRIMERESQUEMA.JSON_RAW (
    raw_file variant);
    
COPY INTO PRIMERABBDD.PRIMERESQUEMA.JSON_RAW
    FROM @MANAGE_DB.INTERNAL_STAGES.JSONSTAGE
    file_format= MANAGE_DB.FILE_FORMATS.JSONFORMAT
    files = ('HR_data.json');
    
   
SELECT * FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;
```

![](./img/raw_file.png)


<hr>

<a name="schema2"></a>

## 2. Parseo y análisis del fichero no estructurado JSON

- Seleccionar atributo/columna

```SQL
SELECT RAW_FILE:city FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW
```
![](./img/city.png)

```sql
SELECT $1:first_name FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW
```
![](./img/first_name.png)



- Seleccionar atributo/columna - formateo y conversión
```sql
SELECT RAW_FILE:first_name::string as first_name  FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;
```
![](./img/first_name_parse.png)

```sql
SELECT RAW_FILE:id::int as id  FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;
```
![](./img/id_parse.png)


```sql
SELECT 
    RAW_FILE:id::int as id,  
    RAW_FILE:first_name::STRING as first_name,
    RAW_FILE:last_name::STRING as last_name,
    RAW_FILE:gender::STRING as gender

FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;
```
![](./img/tabla.png)


<hr>

<a name="schema3"></a>

## 3. Manejo de datos nested y arrays

- Manejo de datos nested
```sql   
SELECT RAW_FILE:job as job  FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;
```
![](./img/job.png)

- Usando el `.` accedemos al nivel que queremos

```sql
SELECT 
      RAW_FILE:job.salary::INT as salary
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;

```
![](./img/salario.png)


```sql
SELECT 
    RAW_FILE:first_name::STRING as first_name,
    RAW_FILE:job.salary::INT as salary,
    RAW_FILE:job.title::STRING as title
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;
```
![](./img/tabla_2.png)

- Manejo de arrays
```sql
SELECT
    RAW_FILE:prev_company as prev_company
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;
```
![](./img/array.png)
- Acceder al elemento del array con `[posición]`
```sql
SELECT
    RAW_FILE:prev_company[1]::STRING as prev_company
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;
```
![](./img/1_elemento.png)

- contar el total de elemento que hay en ese array
```sql
SELECT
    ARRAY_SIZE(RAW_FILE:prev_company) as prev_company
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;
```
![](./img/contar.png)


<hr>

<a name="schema4"></a>

## 4. Jerarquías y flatten del fichero no estructurado JSON

```sql
SELECT 
    RAW_FILE:spoken_languages as spoken_languages
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;
```
![](./img/jerarquia.png)


- Contamos el número de elementos del array
```sql
SELECT 
     array_size(RAW_FILE:spoken_languages) as spoken_languages
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW
```
![](./img/jerarquia_contar.png)


- Creamos tabla
```sql
SELECT 
     RAW_FILE:first_name::STRING as first_name,
     array_size(RAW_FILE:spoken_languages) as spoken_languages
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW
```
![](./img/jerarquia_tabla.png)

- Acceder a cada elemento
```sql
SELECT 
    RAW_FILE:spoken_languages[0] as First_language
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW;
```

![](./img/jerarquia_1_elemento.png)

- Acceder a los dos atributos
```sql
SELECT 
    RAW_FILE:first_name::STRING as First_name,
    RAW_FILE:spoken_languages[0].language::STRING as First_language,
    RAW_FILE:spoken_languages[0].level::STRING as Level_spoken
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW
```
![](./img/jerarquia_tabla_2.png)


- Unimos los elementos 1 y 2 (con índices 0 y 1 respectivamente)
```sql
SELECT 
    RAW_FILE:id::int as id,
    RAW_FILE:first_name::STRING as First_name,
    RAW_FILE:spoken_languages[0].language::STRING as First_language,
    RAW_FILE:spoken_languages[0].level::STRING as Level_spoken
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW
UNION ALL 
SELECT 
    RAW_FILE:id::int as id,
    RAW_FILE:first_name::STRING as First_name,
    RAW_FILE:spoken_languages[1].language::STRING as First_language,
    RAW_FILE:spoken_languages[1].level::STRING as Level_spoken
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW
UNION ALL 
SELECT 
    RAW_FILE:id::int as id,
    RAW_FILE:first_name::STRING as First_name,
    RAW_FILE:spoken_languages[2].language::STRING as First_language,
    RAW_FILE:spoken_languages[2].level::STRING as Level_spoken
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW
ORDER BY ID
```
![](./img/jerarquia_tabla_3.png)

- Utilizamos el método flatten (más óptimo y eficiente)
`FLATTEN` es una función de Snowflake que aplanará (desanidará) un array o un objeto JSON anidado.
```sql
SELECT
    RAW_FILE:first_name::STRING AS First_name,
    f.value:language::STRING AS First_language,
    f.value:level::STRING AS Level_spoken
FROM PRIMERABBDD.PRIMERESQUEMA.JSON_RAW,
    TABLE(FLATTEN(RAW_FILE:spoken_languages)) f;
```
![](./img/jerarquia_flatten.png)

<hr>

<a name="schema5"></a>

## 5. Inserción de datos no estructurados JSON en la tabla objetivo (estructurados)

- Opción 1: Crear tabla "AS"
```sql
CREATE OR REPLACE TABLE Languages AS
select
   RAW_FILE:first_name::STRING as First_name,
   f.value:language::STRING as First_language,
   f.value:level::STRING as Level_spoken
from PRIMERABBDD.PRIMERESQUEMA.JSON_RAW, table(flatten(RAW_FILE:spoken_languages)) f;

SELECT * FROM Languages;
```
![](./img/opcion_1.png)



- Opción 2: Utilizar INSERT INTO (previa creación de la tabla)
```sql
CREATE OR REPLACE TABLE Languages AS
INSERT INTO Languages
select
   RAW_FILE:first_name::STRING as First_name,
   f.value:language::STRING as First_language,
   f.value:level::STRING as Level_spoken
from PRIMERABBDD.PRIMERESQUEMA.JSON_RAW, table(flatten(RAW_FILE:spoken_languages)) f;


SELECT * FROM Languages;
```
![](./img/opcion_2.png)