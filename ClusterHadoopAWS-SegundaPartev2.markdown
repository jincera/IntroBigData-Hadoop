# Hadoop y Map Reduce en AWS -segunda parte

**José Incera, Mayo 2019**

## Introducción

En la primera parte de esta práctica desplegamos una instancia de Hadoop en un solo nodo en Amazon EC2.  

Ahora extenderemos nuestro ambiente a un cluster de cuatro nodos.

## 4. Creación de un cluster en AWS EC2

En esta sección extenderemos nuestro ambiente Hadoop a un cluster de cuatro nodos: un *NameNode* y tres *DataNodes*. El primero es la instancia que creamos en la primera parte de esta práctica y fungirá como maestro. Los DataNodes son esclavos de procesamiento: en ellos se ejecutarán las tareas Map y Reduce. 

Antes de iniciar, si no lo ha hecho todavía, conviene detener los procesos de YARN y HDFS:

```bash
$ stop-yarn.sh
stopping yarn daemons
stopping resourcemanager

$ stop-dfs.sh
Stopping namenodes on [localhost]
localhost: stopping namenode
localhost: stopping datanode
```

### 4.1 Creación de una imagenes de la instancia

Como ya hemos hecho buena parte de la configuración al final de la primera parte, conviene que los tres nuevos nodos sean una imagen de la instancia actual.

#### 4.1.1 Reiniciar carpetas para HDFS

Al crear el cluster, trabajaremos en un nuevo sistema de archivos distribuido.  Por ello, debemos borrar los contenidos de las carpetas donde se crea el sistema de archivos HDFS. 

```bash
$ rm -r /usr/local/hadoop/hadoop_data/namenode/*
$ rm -r /usr/local/hadoop/hadoop_data/datanode/*
```

### 4.1.2 Modificación del grupo de seguridad

En el cluster, los nodos intercambiarán información entre sí. Además de la regla de acceso para el protocolo `ssh`, añadiremos una regla que permita el tráfico de entrada de los nodos que están en la misma subred del nodo maestro.  Ese será el caso en nuestro cluster.  En un despliegue en producción, las reglas de control de acceso deben ser más restrictivas.

En la consola de administración EC2 de AWS, seleccione `Security Groups` en el menú de la izquierda y seleccione el grupo de seguridad que utilizó para crear la instancia maestra (ClusterHadoop).

![](https://imgur.com/RuIC0qJ.jpg)

Seleccione la pestaña `Inbound` en el menú inferior, clic en `Edit` y `Add Rule`para añadir la regla para aceptar todo el tráfico de la fuente `172.31.0.0\16`, que es la subred privada de las instancias. Clic en `Save`.

![](https://imgur.com/9ZErzJf.jpg)

#### 4.1.3 Creación de la imagen y lanzamiento de instancias

Bien, tenemos todo listo para crear la imagen.

En la consola EC2 de AWS, seleccione su instancia y en el menú Actions, seleccione `Image/Create Image`.  Cuando se le solicite, asigne un nombre a la imagen, por ejemplo, Hadoop.

![](https://imgur.com/chVeLCP.jpg)

Una vez creada la imagen (puede tomar algún tiempo), podemos lanzar tres instancias de esa imagen. De clic en `Launch Instance`. La imagen se selecciona en la opción `My AMIs` del menú de la izquierda.

![](https://imgur.com/TJ6k0eI.jpg)

Elija nuevamente **t2.medium** para el tipo de instancia, de clic en `Next:Configure Instance Details` y seleccione **3** en la opción `Number of instances`.  

De clic en `Next: Add Storage`, `Next: Add Tags` y `Next:Configure Security Group`

En el sexto paso, seleccione `Select an existing security group`y seleccione el que utilizó para crear la instancia maestra y que modificó en el paso anterior (ClusterHadoop).

![](https://imgur.com/tBLA7fe.jpg)

Clic en `Review and Launch` y en `Launch`. Finalmente, seleccione el key-pair que utilizó al configurar la instancia maestra. 

Conviene cambiar el nombre a las instancias.  Se trata de un nombre local, no de uno asociado al DNS, así podemos cambiarlos sin conflicto.

Al pasar el cursor por el campo `Name`, aparece el ícono de un pequeño lápiz.  De clic en él para poder cambiar el nombre de la instancia.  Asigne los nombres NameNode (la instancia maestra) y DataNode1/2/3, respectivamente, a las nuevas instancias:

![](https://imgur.com/3SOazEs.jpg)

### 4.2.- Acceso a las instancias

**4.2.1.** Cónectese con `ssh` a cada uno de los nuevos nodos (si trabaja en Windows, use PuTTY con la misma llave para SSH/Auth).  

Lo único que cambia es la dirección IP (la cual puede ver en la consola de administración de EC2):

```bash
$ ssh -i <su_archivo_pem> ubuntu@<la_ip_publica_de_la_instancia>
```

Quizás sea conveniente que abra cuatro sesiones con cada uno de los nodos y las despliegue  en mosaico:

![cuatro sesiones](https://imgur.com/olzGcrZ.jpg)

*Observe que hemos cambiado el "prompt" en cada instancia para hacer más claro qué función ejecuta cada nodo (utilizamos el comando `PS1=`). En el resto del tutorial estaremos usando esta representación de prompt.*

**4.2.2.** Anote la dirección **privada** de cada instancia (recuerde, se utiliza el comando `ifconfig`) y modifique el archivo `/etc/hosts` en el nodo maestro como corresponda.  Deberá tener una estructura similar a la siguiente:

```bash
NameNode> sudo vim /etc/hosts
127.0.0.1 localhost
172.31.3.61  master
172.31.6.247 datanode1
172.31.2.99  datanode2
172.31.3.61  datanode3

# The following lines 
...
```

En Hadoop habrá una interacción continua entre el NameNode y los DataNodes. Esta interacción requiere de acceso automático, sin contraseñas, entre la instancia del NameNode y las demás.

Como las instancias de los esclavos se crearon como una imagen del nodo maestro, ya tienen la llave pública que configuramos en la primera parte de este tutorial, en el archivo `~/.ssh/authorized_keys`. De lo contrario, será necesario copiar la llave pública del nodo maestro a ese archivo en cada uno de los esclavos.

**4.2.3.** Verifique que el ingreso automático funciona.
La primera vez le hará una pregunta antes de registrar la computadora como conocida. Responda `yes`.

```bash
NameNode> ssh datanode1
DataNode1> exit
NameNode> ssh datanode2
DataNode2> exit
NameNode> ssh datanode3
DataNode3> exit
NameNode>
```

###4.3.- Reconfiguración del NameNode

El NameNode va a fungir como el nodo maestro en nuestro cluster. Se deben crear algunos archivos y modificar otros para poder distribuir las tareas en el cluster.  

**4.3.1.- hdfs-site.xml**

Se debe cambiar el número de réplicas a 3 (tendremos 3 DataNodes).

```xml
<property>
    <name>dfs.replication</name>
    <value>3</value>
...
```

**4.3.2.- Configuración Maestros y Esclavos**
Cree los archivos `masters` y `slaves` en la carpeta `/usr/local/hadoop/etc/hadoop` con la siguiente información.  Observe que el archivo `slaves` ya existía y que se debe eliminar la línea que hace referencia al `localhost`:

```bash
NameNode> cd $HADOOP_CONF_DIR
NameNode> vi masters
master

NameNode> vi slaves
datanode1
datanode2
datanode3
```

**4.3.3. Configuralción esclavos en los DataNodes**

En los DataNodes, solo el archivo `slaves`  debe configurarse y sólo con su nombre.  El archivo en los cuatro nodos quedará de la siguiente manera:

![](https://imgur.com/QQZlfLu.jpg)

### 4.4 Preparación del ambiente y lanzamiento de los procesos

**4.4.1.** Como en la primera parte, empezamos por formatear el sistema de archivos desde el nodo maestro:

```bash
NameNode> hdfs namenode -format
```

**4.4.2.** Lanzamos las tareas de HDFS.  Veremos que algunos se ejecutan en el nodo maestro y otros en los esclavos:

```bash
NameNode>start-dfs.sh
Starting namenodes on [master]
master: starting namenode, logging to /usr/local/hadoop/logs/hadoop-ubuntu-namenode-ip-172-31-14-251.out
slave1: starting datanode, logging to /usr/local/hadoop/logs/hadoop-ubuntu-datanode-ip-172-31-27-231.out
slave2: starting datanode, logging to /usr/local/hadoop/logs/hadoop-ubuntu-datanode-ip-172-31-29-106.out
Starting secondary namenodes [0.0.0.0]
0.0.0.0: starting secondarynamenode, logging to /usr/local/hadoop/logs/hadoop-ubuntu-secondarynamenode-ip-172-31-14-251.out

NameNode>jps
3728 SecondaryNameNode
3504 NameNode
3843 Jps

DataNode2>jps
2212 Jps
2136 DataNode
```

Efectivamente, en el NameNode se ejecutan los jobs `NameNode` y `SecondaryNameNode` (lo cual es MUY PELIGROSO, pero en este tutorial sólo estamos realizando una prueba de concepto) y en los DataNodes se ejecuta el job `DataNode`.

**4.4.3.** Ahora lanzamos los procesos de YARN y verificamos que todos se estén ejecutando:

```bash
NameNode>start-yarn.sh
NameNode>jps
2466 ResourceManager
2150 SecondaryNameNode
2729 Jps
1933 NameNode

DataNode3>jps
2442 DataNode
2715 Jps
2590 NodeManager
```

Como es de esperar, en el NameNode se ejecuta el `ResourceManager` y en cada un de los esclavos, un `NodeManager`.

Podemos ver qué nodos tiene disponibles el ResourceManager:

```bash
NameNode>yarn node -list
19/05/14 18:19:22 INFO client.RMProxy: Connecting to ResourceManager at master/172.31.39.146:8032
Total Nodes:3
  Node-Id Node-State Node-Http-Address Number-of-Running-Containers
datanode1:35570  RUNNING datanode1:8042    0
datanode2:33267  RUNNING datanode2:8042    0
datanode3:38446  RUNNING datanode3:8042    0
```

### 4.5 Ejecución de tareas

**4.5.1.** Recordemos que antes de invocar la ejecución de tareas MapReduce, debemos tener un directorio para el usuario en HDFS:

```bash
NameNode>hdfs dfs -mkdir -p /user/ubuntu
ubuntu@ip-172-31-14-251:~$ hdfs dfs -ls /user
Found 1 items
drwxr-xr-x   - ubuntu supergroup          0 2019-05-14 17:56 /user/ubuntu
```

**4.5.2.** Empecemos por lanzar nuevamente el ejemplo para calcular PI.  Como ahora ya tenemos un ambiente distribuido, el tiempo de ejecución deberá ser menor que el observado en la primera parte (además de que las instancias son más poderosas).

```bash
NameNode>hadoop jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.7.jar pi 8 5000
Number of Maps  = 8
Samples per Map = 5000
Wrote input for Map #0
Wrote input for Map #1
Wrote input for Map #2
Wrote input for Map #3

...

Job Finished in 26.34 seconds
Estimated value of Pi is 3.14140000000000000000

```

**4.5.3.** Ahora lancemos el proceso para calcular el número de votos con nuestros mappers y reducers.  Como tenemos un nuevo sistema de archivos HDFS, debemos enviar nuevamente el archivo votación.

```bash
NameNode>cd ~/HdpProy/data
NameNode>hdfs dfs -put votacion.csv

NameNode>cd ../code
NameNode>hadoop jar /usr/local/hadoop/share/hadoop/tools/lib/hadoop-streaming-2.7.7.jar -files EjMapper.py,EjReducer.py -input votacion.csv -output OpElec -mapper EjMapper.py -reducer EjReducer.py

NameNode>hdfs dfs -cat OpElec/part-00000
CAND1    10056
CAND2    9884
CAND3    15051
CAND4    40018
CAND5    24991
```

**4.5.4** Cuando haya terminado de experimentar, detenga los procesos y suspenda sus instancias en EC2.

```bash
NameNode>stop-yarn.sh
NameNode>stop-dfs.sh
```

