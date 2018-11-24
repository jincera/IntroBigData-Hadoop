# Tutorial HIVE

#### José Incera Noviembre 2018

Este tutorial está basado en material disponible libremente en la web.  Entre las fuentes más consultadas están: 

- [Tutorial Hive](https://cwiki.apache.org/confluence/display/Hive/Tutorial)
- [Hive installation](https://www.tutorialspoint.com/es/hive/hive_installation.htm)
- [Bigdata university](https://cognitiveclass.ai/)

### Entregables

La fecha de entrega es el 14 de diciembre a las 23:00.  Se entrega un documento md (o rmd) enviada a jincera@itam.mx con:

+ Las respuestas a las preguntas en el desarrollo mostrando el script utilizado.

Se permite trabajar en equipos de dos personas

### Interfaz de comandos

Estaremos trabajando a través de la interfaz de comandos (CLI), que es la forma más común de interactuar con Hive durante el desarrollo de proyectos.

Desde el shell podemos lanzar queries, DML y DDL.  Podemos ver y manipular la metadata de las tablas, ver planes de ejecución de consultas, etcétera.

**0. Preparación**

* Entre a su máquina virtual en AWS y cámbiese al usuario *hadoop*.  Si no recuerda cómo, revise la sesión práctica del módulo anterior o consulte al instructor.
* Estaremos trabajando con el archivo `votacion.csv` que utilizamos en el módulo anterior. Asegúrese de que lo tiene (`ll $HOME/Prac1/data`); de lo contrario, cárguelo desde el repositorio `jincera/Test-Repo` en `github`.
* También estaremos utilizando el archivo *Encuestas.csv*, por lo que deberá cargarlo:

```bash
hdp> cd /data
hdp> wget https://raw.githubusercontent.com/jincera/Test-Repo/master/Encuestas.csv
hdp> ls
Encuestas.csv  votacion.csv  vottst.csv
```

Este archivo contiene una serie de datos sobre el grado de conocimiento (como porcentaje de la población encuestada) de los candidatos.  El grado de conocimiento se ha resumido en cinco valores: 10%, 30%, 50% 70% y 90% como valor medio de los cinco rangos en que se agruparon las encuestas.

Cada entrada en el archivo tiene tres campos: el candidato, la casa encuestadora y el nivel de conocimiento.



**1. Crear una base de datos**

```bash 
hdp>hive
hive>CREATE DATABASE miPrimerBD;

hive>CREATE DATABASE testBD LOCATION '/hiveDirs/folder1'
> WITH DBPROPERTIES ('Creada Por' = 'jincera', 'date' = '2017-06-11');

hive>SHOW DATABASES;
...
miprimerbd
testbd
...
hive>DESCRIBE DATABASE EXTENDED testbd;
OK
testbd   hdfs://localhost:9000/hiveDirs/folder1      hadoop    USER
{date=2017-06-11, Creada Por=jincera}
...
```

Hive permite introducir casi todos los comandos para interactuar directamente con HDFS.
Los usaremos para verificar que las bases de datos han sido creadas:

```bash
hive>dfs -ls /user/hive/warehouse;
...
drwxrwxrwx   - hadoop  supergroup  0 2017-06-11 09:35 /user/hive/warehouse/miprimebd.db
...

hive>dfs -ls /hiveDirs;
...
drwxr-xr-x   - instructor  hdfs    0 2017-06-11 09:35 /hiveDirs/folder1
...

```
Observe que Hive convirtió los nombres de las bases de datos a minúsculas.  También observe que se pueden agregar propiedades (metadatos) a la tabla.

**Verifique que se crearon las bases de datos y que están vacías.**

Para borrar una BD, usamos el comando `DROP`

```bash
hive>DROP DATABASE IF EXISTS miprimerbd;
```
Se puede añadir al final la cláusula `CASCADE` para borrar las tablas asociadas a esa base de datos, si existen.

**2. Crear una tabla**

```bash
hive>create TABLE IF NOT EXISTS prueba
>(userId INT, Nombre String, Edad int, Depto STRING )
> COMMENT 'Tabla de empleados'
>ROW FORMAT DELIMITED
>   FIELDS TERMINATED BY '\t'
>   LINES TERMINATED BY '\n'
>STORED AS TEXTFILE;
```

Podemos observar varias cosas:

- Hive no es sensible a mayúsculas en palabras reservadas
- Se pueden incluir comentarios a nivel tabla y a nivel columna (no mostrado).  Éstos forman parte de la metadata que se almacena en el metastore.
- La cláusula IF NOT EXISTS es opcional.  Si la tabla existe, se ignora su creación.

Una vez creada la tabla, podemos usar SHOW, DESCRIBE, DESCRIBE EXTENDED para mostrar qué tablas se han creado y cuáles son sus características.

```bash
hive>SHOW TABLES;
hive>DESCRIBE prueba;
hive>DESCRIBE EXTENDED prueba;

OK
userid                  int
nombre                  string
edad                    int
depto                   string

Detailed Table Information  Table(tableName:prueba, dbName:default, owner:hadoop, createTime:1542990453, lastAccessTime:0, retention:0, sd:StorageDescriptor(cols:[FieldSchema(name:userid, type:int, comment:null), FieldSchema(name:nombre, type:string, comment:null), FieldSchema(name:edad, type:int, comment:null), FieldSchema(name:depto, type:string, comment:null)], location:hdfs://localhost:9000/user/hive/warehouse/prueba, ...
```
Con el último comando podrá notar que la base de datos **default** es la que se usa si se introducen comandos sin haber especificado antes con qué BD trabajar. Para especificar con cuál deseamos trabajar, se usa el comando USE: `hive USE [nombre_BD]` o la notación punto: `[nombre_BD].[nombre_tabla]`

Vamos a crear una tabla de votos asociada a la base de datos `testbd`.  Podemos crear la tabla con el nombre `testbd.votos` o utilizando previamente el comando `USE`.

```bash
hive>USE testbd;
hive>create TABLE IF NOT EXISTS votos  
>(hora Int, gen String, dist Int, cand String )
> COMMENT 'Tabla con preferencias electorales'
>ROW FORMAT DELIMITED
>   FIELDS TERMINATED BY ','
>   LINES TERMINATED BY '\n'
>STORED AS TEXTFILE;
```

Podemos confirmar desde Hive que la tabla está donde esperamos:

```bash
hive> dfs -ls /hiveDirs/folder1/;
Found 1 items
drwxrwxrwx   - root hdfs  0 2017-06-10 10:47 /hiveDirs/folder1/votos
hive>
```

**3. Carga de datos**

Para cargar datos a la BD, podemos usar el comando `LOAD`.  Si se usa la cláusula LOCAL, se asume que la fuente está en el sistema de archivos local, en cuyo caso se **copian** los datos a HDFS.  Si no está la cláusula, se asume que la trayectoria es de HDFS y los datos se **mueven** a la carpeta correspondiente.

No se hace ningún tipo de transformación al cargar los datos a las tablas.

Vamos a cargar los datos del archivo `votacion.csv` que hemos usado en las prácticas anteriores. 

```bash
hive>LOAD DATA LOCAL INPATH '/home/hadoop/Prac1/data/votacion.csv'
    OVERWRITE INTO TABLE votos;
```
Hemos especificado solamente un archivo, pero podemos copiar directorios completos en una tabla.

La cláusula `OVERWRITE` indica que si existe información en la tabla destino, ésta se pierde y es remplazada por los datos que se están cargando.  Sin la cláusula, los archivos se agregan a la tabla, pero si hay conflicto con los nombres de archivos, éstos también se reescriben.

Para terminar, hagamos una consulta muy simple (tipo SQL) sobre la tabla de votos... aunque este query es tan sencillo, que no se utilizará el ambiente MapReduce para ejecutarse.

```bash
hive> SELECT * from votos LIMIT 5;
OK
12	M	1048	CAND5
15	H	7932	CAND1
13	H	7373	CAND4
13	H	1232	CAND1
 9	H	5208	CAND4
...
```

**4. Tablas externas**

Las tablas externas pueden ser compartidas fuera de Hive.  Supongamos que es de interés para otras áreas analizar la información relativa a el grado de conocimiento de los candidatos, según lo reportado por las casas encuestadoras.  En ese caso, conviene que esta tabla se defina como externa.

Lo que hacemos es crear una copia del archivo `Encuestas.csv` en HDFS y definimos nueva tabla que apunte a los datos en el sistema de archivos.

Observe que podemos usar los mismos comandos de HDFS dentro de Hive.

```bash
hive> dfs -mkdir datosCompartidosHive;
hive> dfs -ls;
...
drwxr-xr-x   - hadoop      supergroup          0 201
drwxr-xr-x   - hadoop      supergroup          0 2017-06-10 12:09 datosCompartidosHive
...
# Modifiquemos permisos para lectura, escritura y ejecución en ese directorio
hive> dfs -chmod 777 datosCompartidosHive;

hive> dfs -put /home/hadoop/Prac1/data/Encuestas.csv datosCompartidosHive/Encuestas.csv;
hive> dfs -ls datosCompartidosHive;
Found 1 items
-rw-r--r-- 1 hadoop supergroup 2561 2017-06-10 12:13 datosCompartidosHive/Encuestas.csv
hive>
```
Ahora definimos la tabla como EXTERNAL especificando su ubicación. **OBSERVE QUE** especificamos un directorio. Si hubiera muchos archivos ahí (lo cual es común), Hive los usaría todos.

```bash
CREATE EXTERNAL TABLE IF NOT EXISTS testbd.encuestas
(
    cand        STRING,
 	casa        STRING,
  	conocimiento INT 
)
 COMMENT 'Encuestas de conocimiento por casa encuestadora'
 ROW FORMAT DELIMITED
   	 	FIELDS TERMINATED BY ','
	 	LINES TERMINATED BY '\n'
 LOCATION '/user/hadoop/datosCompartidosHive';
```

Para terminar, veamos que están todas las tablas en la base de datos, pero no todas en el warehouse:

```bash
hive> show tables in testbd;
OK
...
encuestas
votos
...
Time taken: 0.374 seconds, Fetched: 4 row(s)

hive> dfs -ls /hiveDirs/folder1;
Found 3 items
drwxrwxrwx   - hadoop supergroup      0 2017-06-10 10:47 /hiveDirs/folder1/votos
...
```

**5. Crear una tabla a partir de una consulta**

También se pueden cargar datos a una tabla como resultado de un query a otras tablas. Esto puede hacerse, entre otras, con las cláusulas `CREATE` e `INSERT`:

```bash
hive>CREATE TABLE VotosPorDist AS
   >SELECT v.dist AS dist, count(*) AS numVotos
   >  FROM votos v 
   >  GROUP BY v.dist;

Query ID = hadoop_20181124053713_4a710d41-5aa3-4878-9f31-32cdd80951ab
Total jobs = 1
Launching Job 1 out of 1
Number of reduce tasks not specified. Estimated from input data size: 1
In order to change the average load for a reducer (in bytes):
  set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
  set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
  set mapreduce.job.reduces=<number>
Job running in-process (local Hadoop)
2018-11-24 05:37:16,875 Stage-1 map = 0%,  reduce = 0%
2018-11-24 05:37:17,884 Stage-1 map = 100%,  reduce = 100%
Ended Job = job_local678844993_0001
Moving data to directory hdfs://localhost:9000/hiveDirs/folder1/votospordist
MapReduce Jobs Launched:
Stage-Stage-1:  HDFS Read: 3190192 HDFS Write: 3183790 SUCCESS

hive>DESCRIBE VotosPorDist;
Ok
dist       int
numVotos   bigint
...
```

**¿Cuántos votos se obtuvieron en los distritos 1048, 5572 y 8999?** (Sugerencia: utilice nuevamente SELECT * ).

**Haga una tabla votosPorGenero que contenga cuántos votos se obtuvieron por hombres y por mujeres**

**6. Exportar datos**

Si los archivos de las tablas ya están en el formato deseado (por ejemplo, porque se crearon con la cláusula `STORED AS TEXTFILE;` simplemente se pueden copiar de HDFS al sistema de archivos local.

Asimismo, los resultados de los queries pueden apuntar a un archivo HDFS o al sistema de archivos local si se usa la cláusula `LOCAL`.  

En el siguiente ejemplo utilizaremos la cláusula `INSERT` para generar un archivo que contenga sólo los votos del candidato CAND5. *Sustituya `<usuario>` por su nombre de usuario.* La cláusula también puede utilizarse para generar una tabla.

```bash
hive>INSERT OVERWRITE LOCAL DIRECTORY '/home/hadoop/Prac1/data/votosCAND5'
>SELECT hora, dist, gen
>FROM votos
>WHERE cand='CAND5';
Query ID = ....
Total jobs = 1
...
Moving data to local directory /tmp/votosCAND5
```
Vemos que el comando anterior sólo requirió de un Mapper.

La cláusula anterior es una buena forma de extraer grandes cantidades de datos de Hive.  Hive puede escribir a directorios HDFS en paralelo dentro de un job MapReduce.

Por default, los datos que se escriben  en el filesystem se serializan como texto en columnas separadas por `^A` y líneas por `\n`, aunque los delimitadores se pueden configurar. Columnas que no tienen tipos primitivos (por ejemplo son STRUCT o MAP) se serializan en formato JSON.

*MUCHO CUIDADO* porque el directorio se pierde y se reescribe.  Hay que asegurarse de que no se estará borrando accidentalmente, por ejemplo, el directorio HOME.

Para verificar que el archivo se generó en el sistema de archivos local, podemos invocar comandos de shell antecediéndolos del signo de admiración:

```bash
hive>!ls /home/hadoop/;
```

**Confirme que el resultado generó un archivo en el sistema de archivos local.  ¿Qué nombre tiene el archivo?  ¿Cuáles son las últimas tres líneas?**

**7. Tabla con particiones**

Las particiones son un concepto sumamente importante en Hive para mejorar el desempeño de las consultas a la base de datos. Aunque en esta sesión no tendremos oportunidad de validar el incremento en desempeño, nos daremos una idea general de cómo se consigue.

Se pueden hacer particiones sobre una o más *llaves*. Conceptualmente, son campos (columnas) de una tabla, pero para el sistema, especifican cómo se almacenarán los datos.  Es decir, los "campos" de partición no se almacenan con los datos en la base sino que indican cómo debería *particionarse* la tabla al almacenarse en HDFS.  Si los datos están distribuidos, las consultas serán mucho más eficientes porque se harán solamente sobre la partición relevante.

Vamos a crear una segunda tabla especificando que esté particionada por candidato:

```bash
hive>create TABLE IF NOT EXISTS votosPart
>(hora int, gen string, dist int)
>PARTITIONED BY (cand string)
>ROW FORMAT DELIMITED
>   FIELDS TERMINATED BY ','
>   LINES TERMINATED BY '\n'
>STORED AS TEXTFILE;

hive> describe votosPart;
OK
hora   int
gen    string
dist   int
cand   string

# Partition Information
# col_name              data_type               comment

cand                     string
...
```
Vamos a tratar de cargar datos *particionados* en esta tabla a partir del archivo fuente `votación.csv`

```bash
hive> LOAD DATA LOCAL INPATH '/home/hadoop/Prac1/data/votacion.csv'
    OVERWRITE INTO TABLE votosPart
    partition (cand='CAND4');
Loading data to table testbd.votospart partition (cand=CAND4)
    
hive>dfs -ls /hiveDirs/folder1/votospart/;
Found  1 items
drwxr-xr-x    - hadoop supergroup      0 2017-06-14 14:06 /hiveDirs/folder1/votospart/cand=CAND4

hive>dfs -ls /hiveDirs/folder1/votospart/cand=CAND4/;
Found 1 items
drwxr-xr-x    - hadoop supergroup      0 2017-06-14 14:06 /hiveDirs/folder1/votospart/votacion.csv
```
Aparentemente todo está bien: Tenemos una carpeta `cand=CAND4`, aunque dentro de esta carpeta encontramos un archivo `votacion.csv`.  Hagamos una consulta:

```bash
hive>select * from votosPart limit 5;
Ok
12	M	1048	CAND4
15	H	7932	CAND4
13	H	7373	CAND4
13	H	1232	CAND4
 9	H	5208	CAND4
...
```
Pues sí.  Todo indica que estamos seleccionando los registros del candidato CAND4.  ¿Será correcto?  Verifiquémoslo:

```bash
hive>SELECT * FROM votos LIMIT 5
Ok
12	M	1048	CAND4
15	H	7932	CAND4
13	H	7373	CAND4
13	H	1232	CAND4
 9	H	5208	CAND4
...

hive>! head -5 /home/hadoop/Prac1/data/votacion.csv;
12; M, 1048, CAND4
15; H, 7932, CAND4
13, H, 7373, CAND4
13, H, 1232, CAND4
 9, H, 5208, CAND4
```
¡Uff!  Esto no es lo que esperábamos. En realidad, no podemos hacer particiones con la cláusula `LOAD` pues ésta no analiza registros, simplemente los lee del archivo fuente.

Vamos a inentarlo a partir de una consulta a la tabla votos con la cláusula `INSERT`.  Además, utilizaremos **particiones dinámicas**, es decir, dinámicamente se irán creando distintos archivos con base en los distintos valores del campo `cand`.  Para ello, necesitamos indicarle a Hive que tolere el que no se especifique una partición estática.

```bash
hive> set hive.exec.dynamic.partition.mode=nonstrict;
hive> insert overwrite table votosPart partition (cand)
    > select * from votos;
Query ID = hadoop_20181124060604_a8943063-6f57-416b-bb8a-a9d30eb12c42
Total jobs = 3
Launching Job 1 out of 3
Number of reduce tasks not specified. Estimated from input data size: 1
In order to change the average load for a reducer (in bytes):

hive>
```
¡Esto es más prometedor!  Veamos cómo están nuestras carpetas:

```bash
hive> dfs -ls /hiveDirs/folder1/votospart;
Found 5 items
drwxr-xr-x   - hadoop supergroup          0 2017-06-29 23:53 /hiveDirs/folder1/votospart/cand=CAND1
drwxr-xr-x   - hadoop supergroup          0 2017-06-29 23:53 /hiveDirs/folder1/votospart/cand=CAND2
drwxr-xr-x   - hadoop supergroup          0 2017-06-29 23:53 /hiveDirs/folder1/votospart/cand=CAND3
drwxr-xr-x   - hadoop supergroup          0 2017-06-29 23:53 /hiveDirs/folder1/votospart/cand=CAND4
drwxr-xr-x   - hadoop supergroup          0 2017-06-29 23:53 /hiveDirs/folder1/votospart/cand=CAND5

hive> dfs -ls /hiveDirs/folder1/votospart/cand=CAND4;
Found 1 items
-rwxr-xr-x   1 hadoop supergroup     149204 2017-06-29 23:53 /hiveDirs/folder1/votospart/cand=CAND4/000000_0
hive>
```
¿Qué contiene el archivo 000000_0? ¿Ahora sí es correcta la partición? 

```bash
hive>dfs -cat /hiveDirs/folder1/votospart/cand=CAND4/000000_0;
...
9,H,6002
14,M,8425
13,H,6592
16,M,5432

hive>SELECT * FROM votos WHERE cand='CAND4';
...
17      H       5432    CAND3
8       M       5572    CAND3
11      H       8999    CAND3
12      M       9600    CAND3
Time taken: 0.139 seconds, Fetched: 15051 row(s)
hive>
```
¡Ahora sí parece que tenemos los resultados esperados!

**¿Qué ocurre si consultamos nuevamente votosPart filtrando o no los campos con la cláusula `WHERE`?**


**8. Join**

Hive soporta JOINS con la sintaxis de ANSI y sólo soporta *equi-joins*.
Como es de esperar, se pueden hacer Inner Joins, Left Outer, right Outer, Full Outer, Left Semi-joins, aunque no serán cubiertos en este tutorial.

Vamos a crear una tabla con los campos candidato, distrito, casa encuestadora y porcentaje de conocimiento uniendo las tablas votos y encuestas:

```bash
hive>CREATE TABLE IF NOT EXISTS candcasa AS
>SELECT v.cand as cand, v.dist as dist, e.casa as casa, e.conocimiento as conoc
>FROM votos v JOIN encuestas e
>ON (v.cand=e.cand);
...
>describe candcasa;
Ok
cand    string
dist    int
casa    string
k       int
...
hive>select * from candcasa limit 3;
OK
CAND5   1048   CASA1   50
CAND5   1048   CASA10  30
CAND5   1048   CASA9   30
```

**9. Funciones de agregación**

Hive cuenta con una amplia gama de operadores y funciones aritméticas. 
Sólo como demostración, veamos cuál es la percepción promedio de los candidatos según las casas encuestadoras:

```bash
hive>SELECT cand, AVG(conoc)
>FROM candcasa
>GROUP BY cand;

Query ID = ...
Total jobs = 1
...
OK
CAND1 30.0
CAND2 16.0
CAND3 50.0
CAND4 68.0
CAND5 30.0
```
**Encuentre el promedio de percepción de conocimiento de los candidatos según las casas encuestadoras 1, 3 y 6** (Sugerencia: Utilice la cláusula `WHERE` con el operador relacional `OR`).



##### Anexo Comandos para configuración de Hive

```bash
> hdfs dfs -mkdir -p /user/hive/warehouse
> hdfs dfs -chmod 765 /user/hive/warehouse
> schematool -initSchema -dbType derby
```

