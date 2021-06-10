# Datos en streaming con Kafka y Spark

En la siguiente documentación, vamos a realizar una conexión ( de forma local ) entre las herramientas **Kafka** y **Spark** de forma que, simularemos un flujo de datos constante a través de la lectura de ficheros que entrarán en kafka con un producer en pyspark y que más adelante serán procesados por un consumer también en pyspark. ¿ Por qué esas dos herramientas y este experimento en concreto ? Porque es un núcleo sólido por así decirlo, en el sentido de que, si quisieramos realizar un streaming de datos de fuentes externas y que terminen siendo procesados y finalmente almacenados en una base de datos, el núcleo seguiría siendo el mismo con los añadidos de las herramientas externas para la ingesta y almacenamiento de datos. 

#### - Requisitos

Para poder ejecutar código pyspark en nuestra máquina, es importante que tengamos instalado python3 y pip3. Por supuesto, Kafka y Zookeeper deberán estar también funcionando en nuestro equipo para poder echar a andar todo esto. Es probable que más adelante escalemos este ejemplo, por lo que es conveniente tener instalado también pandas para nuestro pyspark. Imprescindible tener la librería de conexión entre python y kafka, ya que esta es menos frecuente tenerla instala, podremos hacerlo con la orden:

```bash
$ sudo pip3 install kafka-python
```

### Montando la infraestructura con Kafka y Spark

Para simular el streaming de datos, vamos a utilizar 3 ficheros csv con una serie de datos que harán las veces de ingesta o input de información en nuestro sistema. Podemos descargarlos desde el directorio **Data_soruce**

A continuación, crearemos un tópico a propósito que servirá como puente entre nuestros scripts y el kafka. Lo podemos crear con la orden:

```bash
$ kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic unique-topic-name
```

Nos aseguramos que lo hemos creado correctamente listándolo con:

```bash
$ bin/kafka-topics.sh --list --zookeeper localhost:2181
```

Y debería aparecer el tópico que hemos creado anteriormente. En principio no es necesario tener más de un broker de Kafka disponible para lanzar ambos procesos ya que ambos se van a suscribir al mismo tópico.

Como estamos realizando la conexión entre los elementos en local, no es necesario revisar la conectividad entre máquinas ni entrar en complicaciones varias.

Vamos a crear nuestro listener.py que será el encargado de recibir los datos que simularemos en la ingesta del "streaming" y podemos encontrarlo en la carpeta **scripts**. El código de nuestro listener es el siguiente:

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import explode
from pyspark.sql.functions import split

spark = SparkSession.builder.appName("Listener").getOrCreate()
spark.sparkContext.setLogLevel("WARN")

lines = spark.readStream.format("kafka").option("kafka.bootstrap.servers","localhost:9092").option("subscribe","testTopic").load().selectExpr("CAST(value AS STRING)")


words = lines.select( explode( split(lines.value, ':')).alias('word') )

wordCounts = words.groupBy('word').count()

query = wordCounts.writeStream.outputMode('complete').format('console').start()

query.awaitTermination()
```

Donde "testTopic" es el nombre del tópico creado en Kafka, y le damos un procesamiento básico a los datos, en este caso una contabilidad de palabras. Ahora hablaremos del producer que también podemos encontrar en la carpeta de **scripts**.

Y el código para nuestro producer será:

```python
import pandas as pd
import time
from kafka import KafkaProducer

kafka_broker_hostname='127.0.0.1'
kafka_broker_portno='9092'
kafka_broker=kafka_broker_hostname + ':' + kafka_broker_portno
kafka_topic='testTopic'

data_send_interval=5

if __name__ == "__main__":

    producer = KafkaProducer(bootstrap_servers=kafka_broker)


    iot_data_id10 = pd.read_csv('/home/usuario/Descargas/kafka-spark-demo-pyconsg19/iot_demo/data/iot_data_id10.csv')
    iot_data_id11 = pd.read_csv('/home/usuario/Descargas/kafka-spark-demo-pyconsg19/iot_demo/data/iot_data_id11.csv')
    iot_data_id12 = pd.read_csv('/home/usuario/Descargas/kafka-spark-demo-pyconsg19/iot_demo/data/iot_data_id12.csv')



    for _index in range(0, len(iot_data_id10)):
        json_iot_id10 = iot_data_id10[iot_data_id10.index==_index].to_json(orient='records')
        producer.send(kafka_topic, bytes(json_iot_id10, 'utf-8'))
        print(json_iot_id10)
        json_iot_id11 = iot_data_id11[iot_data_id11.index==_index].to_json(orient='records')
        producer.send(kafka_topic, bytes(json_iot_id11, 'utf-8'))
        print(json_iot_id11)
        json_iot_id12 = iot_data_id12[iot_data_id12.index==_index].to_json(orient='records')
        producer.send(kafka_topic, bytes(json_iot_id12, 'utf-8'))
        print(json_iot_id12)
        time.sleep(data_send_interval)
```

Donde la ruta de los csv será la que tengamos nosotros cuando hemos descargado los archivos de simulación de ingesta de datos.

### Ejecutando el sistema completo

Para realizar la simulación y comprobar que todo el sistema funciona correctamente, pondremos en funcionamiento el listener primer con la siguiente orden:

```bash
$ sudo ./spark-submit --packages org.apache.spark:spark-sql-kafka-0-10_2.12:3.1.2 /home/usuario/Escritorio/listener_0.py
```

Y cuando veamos que aparece el primer volcado ( Batch 0 ), echaremos a andar el producer de la misma forma en otra terminal distinta. Nos daremos cuenta de que el Batch se va actualizando y va creciendo, además de que el script como tal no tiene mucho sentido pero nos sirve para comprobar que el núcleo está funcionando y está disponible.

# Data streaming con Kafka y Kafka streaming

Para más referencias : [Confluent.io](https://docs.confluent.io/platform/current/streams/quickstart.html)

Partimos de que tenemos una instalación de Kafka completamente funcional ya realizada y que hemos realizado las pertenentes pruebas locales. 

La prueba que vamos a plantear es la siguiente: crearemos una tubería de paso de información a través de dos tópicos en kafka, uno a modo de listener ( input de datos ) y otro a modo de producer, que devolverá los datos recibidos realizando una operación de conteo de palabras encima de ellos. En este caso es un contado de palabras por realizar alguna operación en concreto, pero más adelante probaremos operaciones más complejas e incluso volcado de datos.

### Iniciando el cluster de Kafka

Para realizar los pasos detallados a continuación, repetimos, suponemos que ya hay una instalación de Java 8 previa y una instalación de Kafka.

Lo primero será iniciar el servidor de Zookeeper incluido por defecto en Kafka. Podemos hacerlo de dos formas distintas. Primero, a través del propio script de arranque del Kafka que trae por defecto con:

```bash
$   ./bin/zookeeper-server-start ./etc/kafka/zookeeper.properties
```

O bien, a través del systemctl de nuestro sistema:

```bash
$ systemctl start zookeeper
```

Usemos el arranque que usemos, es conveniente revisar el estado del zookeeper y asegurarnos que se ha iniciado con éxito y sin problemas con la orden:

```bash
$ systemctl status zookeeper
```

A continuación, realizamos lo propio con el servicio de Kafka. De nuevo, tenemos dos opciones para iniciar los servicios de Kafka así que podemos usar la que más nos convenga o nos apetezca. La primera a través del script por defecto que otorga Kafka:

```bash
$  ./bin/kafka-server-start ./etc/kafka/server.properties
```

O bien, a través del systemctl de nuestro sistema:

```bash
$ systemctl start kafka
```

Usemos el arranque que usemos, es conveniente revisar el estado del zookeeper y asegurarnos que se ha iniciado con éxito y sin problemas con la orden:

```bash
$ systemctl status kafka
```

### Preparando los tópicos de Kafka y la ingesta de datos en streaming

Para crear los tópicos, utilizaremos el script propio de Kafka "kafka-topics", tanto para el de entrada como el de salida, ambos con las mismas características, en el mismo puerto pero con distinto nombre. Para el tópico de entrada, usaremos:

```bash
$ ./bin/kafka-topics --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic streams-plaintext-input
```

Le damos el nombre "streams-plaintext-input" concretamente porque luego el script de Java que vamos a correr utiliza ese nombre y el que le vamos a dar al siguiente topic por defecto para realizar el streaming de datos. Para el topic de salida:

```bash
$ ./bin/kafka-topics --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic streams-wordcount-output
```

Con la misma explicación para el nombre del topic de salida.

Ahora, vamos a generar o simular una entrada de datos en tiempo real, para ello volcaremos una frase sencilla sobre la cual aplicaremos ciertas operaciones con nuestro kafka streaming más adelante. Para dicho volcado, utilizaremos la orden:

```bash
$ echo -e "all streams lead to kafka\nhello kafka streams\njoin kafka summit" > /tmp/file-input.txt
```

De forma que el contenido del fichero file-input.txt quedaría algo así como:

- - **all 	streams lead to kafka**

  - **hello 	kafka streams**

  - **join 	kafka summit**

Por último, dejamos lanzado ese volumen de datos sobre la entrada de nuestro tópico en kafka con la siguiente orden a través del script producer de kafka:

```bash
$ cat /tmp/file-input.txt | ./bin/kafka-console-producer --broker-list localhost:9092 --topic streams-plaintext-input
```

Esto hará que se vuelque el contenido del fichero linea por linea y se publique en el tópico de kafka cada línea como mensajes independientes. Es importante anotar aquí, que este script podremos ejecutarlo tantas veces como querramos, ya que por como funcionan los datos en streaming, podremos enviarle el volumen de datos que nos de la gana a que a fin de cuentas estamos simulando una generación de datos en tiempo real.

### Procesando la entrada de datos con Kafka Streams

Ahora que ya tenemos toda la infraestructura preparada y estamos recibiendo datos en ella, podemos lanzar nuestro script que procese esos datos que recibe. En este caso y como hemos comentado antes, el script es uno de los que trae kafka a modo de ejemplos que utiliza las librerias de Kafka Streaming disponibles para Java 8 y realiza un sencillo conteo de palabras sobre el input que recibe. Para dejar dicho script corriendo y a la espera de recibir datos a través del topic, usaremos:

```bash
$   ./bin/kafka-run-class org.apache.kafka.streams.examples.wordcount.WordCountDemo
```

Con este comando lo único que hemos hecho ha sido realizar la operación y dejar ejecutando el script, para ver si realmente ha surgido efecto, tendremos que consultar el contenido del tópico de salida que establecimos anteriormente.

### Inspeccionando el output de data

ara visualizar el contenido en el topic de Kafka de forma que sea legible para nosotros y podamos iterar sobre dicho contenido con cada ejecución de esta orden, usaremos:

```bash
$ ./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic streams-wordcount-output --from-beginning --formatter kafka.tools.DefaultMessageFormatter --property print.key=true --property key.deserializer=org.apache.kafka.common.serialization.StringDeserializer –property value.deserializer=org.apache.kafka.common.serialization.LongDeserializer
```

Y debería aparecernos por pantalla un output similar al siguiente:

![image-20210610121711215](C:\Users\EnriqueSolaGayoso\Pictures\image-20210610121711215.png)

Ahora recomendamos dejar a libre elección y albedrío el ejecutar varias veces el script para lanzar datos al topic de input de kafka y luego volver a ejecutar la orden anterior, de forma que observemos como va cambiando el contenido de la lista y sus valores.



# Posibles inputs de Kafka Streaming y tecnologías con las que trabaja

Por lo que sabemos, Kafka puede trabajar de primera mano y directamente con tecnologías como Flume/Flafka, Spark Streaming, Storm, HBase, Flink y Spark para ingesta, análisis y procesamiento de datos en tiempo real.

Y por supuesto, su total cooperación con toda la suite de Hadoop completa.

En cuanto al input que puede manejar, las posibilidades son ilimitadas. Por ejemplo el código que hemos ejecutado en este documento, no deja de ser un script en java que utiliza dos topics y las librerias de Kafka streaming, pero un lenguaje como Pyhton tambien cuenta con librerias para lectura en streaming y librerias para conectarse con kafka como le de la gana, por lo que no es demasiado complicado echar a andar un script en python que realice un 'scrapping' de twitter y luego por ejemplo vuelque esa información en un tópico de kafka que luego en paralelo procese esa información de alguna forma y lo devuelva por un tópico de salida como es el caso del ejemplo planteado. 