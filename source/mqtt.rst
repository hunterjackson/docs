Kafka Connect Mqtt Source
=====================

A Connector to read events from Mqtt and push them to Kafka. The connector subscribes to the specified topics and and
streams the records to Kafka.

Prerequisites
-------------

- Confluent 3.0.1
- Mqtt server
- Java 1.8
- Scala 2.11

Setup
-----

Mqtt Setup
~~~~~~~~~~~~~

For testing we will use a simple application spinning up an mqtt server using Moquette.Download and unzip `this <https://github.com/datamountaineer/mqtt-server/releases/download/v.0.1/mqtt-server-0.1.tgz>`__
Also download the `schema <https://github.com/datamountaineer/mqtt-server/releases/download/v.0.1/temperaturemeasure.avro>`__ required to read the avro records

Once you have unpacked the archiver you should start the server
.. sourcecode:: bash

    ➜  bin/mqtt-server

You should see the following outcome:

.. sourcecode:: bash
    log4j:WARN No appenders could be found for logger (io.moquette.server.Server).
    log4j:WARN Please initialize the log4j system properly.
    log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
    Starting mqtt service on port 11883
    Hit Enter to start publishing messages on topic: /mjson and /mavro


The server has started but no records have been published yet. More on this later once we start the source.

Confluent Setup
~~~~~~~~~~~~~~~

.. sourcecode:: bash

    #make confluent home folder
    ➜  mkdir confluent

    #download confluent
    ➜  wget http://packages.confluent.io/archive/3.0/confluent-3.0.1-2.11.tar.gz

    #extract archive to confluent folder
    ➜  tar -xvf confluent-3.0.1-2.11.tar.gz -C confluent

    #setup variables
    ➜  export CONFLUENT_HOME=~/confluent/confluent-3.0.1

Start the Confluent platform.

.. sourcecode:: bash

    #Start the confluent platform, we need kafka, zookeeper and the schema registry
    ➜  bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    ➜  bin/kafka-server-start etc/kafka/server.properties &
    ➜  bin/schema-registry-start etc/schema-registry/schema-registry.properties &

Build the Connector and CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The prebuilt jars can be taken from `here <https://github.com/datamountaineer/stream-reactor/releases>`__

If you want to build the connector, clone the repo and build the jar.

.. sourcecode:: bash

    ##Build the connectors
    ➜  git clone https://github.com/datamountaineer/stream-reactor
    ➜  cd stream-reactor
    ➜  gradle fatJar



Source Connector QuickStart
-------------------------

Next we will start the connector in distributed mode. Connect has two modes, standalone where the tasks run on only one host
and distributed mode. Usually you'd run in distributed mode to get fault tolerance and better performance. In distributed mode
you start Connect on multiple hosts and they join together to form a cluster. Connectors which are then submitted are
distributed across the cluster.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Source Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a file called ``mqtt-source.json`` with the contents below: (CHANGE PATH_TO to point to the location where you have stored temperaturemeasure.avro!)

.. sourcecode:: bash

    {
      "name": "mqtt-source",
      "config": {
        "name":"mqtt-source",
        "tasks.max":"1",
        "connect.mqtt.connection.clean":"true",
        "connect.mqtt.connection.timeout":"1000",
        "connect.mqtt.source.kcql":"INSERT INTO kjson SELECT * FROM /mjson;INSERT INTO kavro SELECT * FROM /mavro",
        "connect.mqtt.connection.keep.alive":"1000",
        "connect.mqtt.source.converters":"/mjson=com.datamountaineer.streamreactor.connect.converters.source.JsonSimpleConverter;/mavro=com.datamountaineer.streamreactor.connect.converters.source.AvroConverter",
        "connect.source.converter.avro.schemas":"/mavro=$PATH_TO/temperaturemeasure.avro",
        "connect.mqtt.client.id":"dm_source_id",
        "connect.mqtt.converter.throw.on.error":"true",
        "connect.mqtt.hosts":"tcp://127.0.0.1:11883",
        "connect.mqtt.service.quality":"1",
        "connector.class":"com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector"
      }
    }


This configuration defines:

1.  The name of the source.
2.  The name number of tasks.
3.  To clean the mqtt connection.
4.  The Kafka Connect Query statements to read from json and avro topics and insert into Kafka kjson and kavro topics.
5.  Setting the time window to emit keep alive pings
6.  Set the converters for each of the Mqtt topics. If a source doesn't get a converter set it will default to BytesConverter
7.  Set the avro schema for the 'avro' Mqtt topic.
8.  The mqtt client identifier
9.  If a conversion can't happen it will throw an exception
10  The connection to the Mqtt server
11  The quality of service for the messages
12  Set the connector source class

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Connectors can be deployed distributed mode. In this mode one or many connectors are started on the same or different
hosts with the same cluster id. The cluster id can be found in ``etc/schema-registry/connect-avro-distributed.properties.``

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

Now start the connector in distributed mode. We only give it one properties file for the kafka, zookeeper and
schema registry configurations.

First add the connector jar to the CLASSPATH and then start Connect.

.. note::

    You need to add the connector to your classpath or you can create a folder in ``share/java`` of the Confluent
    install location like, kafka-connect-myconnector and the start scripts provided by Confluent will pick it up.
    The start script looks for folders beginning with kafka-connect.

Start the simple mqtt server first. It won't publish messages until you hit Enter


.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-mqtt-0.2.X-cp-3.0.1.all.jar

.. sourcecode:: bash

    ➜  bin/connect-distributed etc/schema-registry/connect-avro-distributed.properties


Once the connector worker has started lets post the start the Mqtt source connector:

.. sourcecode:: bash

    ➜  curl -X POST -H "Content-Type: application/json" --data @mqtt-source.json http://localhost:8083/connectors



If you switch back to the terminal you started the Connector in you should see the Mqtt source being accepted and the
task starting.

.. sourcecode:: bash

    [2016-12-20 16:51:08,058] INFO
     ____        _        __  __                   _        _
    |  _ \  __ _| |_ __ _|  \/  | ___  _   _ _ __ | |_ __ _(_)_ __   ___  ___ _ __
    | | | |/ _` | __/ _` | |\/| |/ _ \| | | | '_ \| __/ _` | | '_ \ / _ \/ _ \ '__|
    | |_| | (_| | || (_| | |  | | (_) | |_| | | | | || (_| | | | | |  __/  __/ |
    |____/_\__,_|\__\__,_|_|__|_|\___/ \__,_|_| |_|\__\__,_|_|_| |_|\___|\___|_|
    |  \/  | __ _| |_| |_  / ___|  ___  _   _ _ __ ___ ___
    | |\/| |/ _` | __| __| \___ \ / _ \| | | | '__/ __/ _ \
    | |  | | (_| | |_| |_   ___) | (_) | |_| | | | (_|  __/
    |_|  |_|\__, |\__|\__| |____/ \___/ \__,_|_|  \___\___| by Stefan Bocutiu
               |_|
     (com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceTask:37)
    [2016-12-20 16:51:08,090] INFO MqttSourceConfig values:
        connect.mqtt.source.kcql = INSERT INTO kjson SELECT * FROM /mjson;INSERT INTO kavro SELECT * FROM /mavro
        connect.mqtt.service.quality = 1
        connect.mqtt.connection.ssl.cert = null
        connect.mqtt.source.converters = /mjson=com.datamountaineer.streamreactor.connect.converters.source.JsonSimpleConverter;/mavro=com.datamountaineer.streamreactor.connect.converters.source.AvroConverter
        connect.mqtt.connection.keep.alive = 1000
        connect.mqtt.hosts = tcp://127.0.0.1:11883
        connect.mqtt.converter.throw.on.error = true
        connect.mqtt.connection.timeout = 1000
        connect.mqtt.user = null
        connect.mqtt.connection.clean = true
        connect.mqtt.connection.ssl.ca.cert = null
        connect.mqtt.connection.ssl.key = null
        connect.mqtt.password = null
        connect.mqtt.client.id = dm_source_id
     (com.datamountaineer.streamreactor.connect.mqtt.config.MqttSourceConfig:178)


Test Records
^^^^^^^^^^^^

Go to the mqtt-server application you downloaded and unzipped and execute: ./bin/mqtt-server
This will put the following records into the avro and json Mqtt topic:

    TemperatureMeasure(1, 31.1, "EMEA", System.currentTimeMillis()),
    TemperatureMeasure(2, 30.91, "EMEA", System.currentTimeMillis()),
    TemperatureMeasure(3, 30.991, "EMEA", System.currentTimeMillis()),
    TemperatureMeasure(4, 31.061, "EMEA", System.currentTimeMillis()),

    TemperatureMeasure(101, 27.001, "AMER", System.currentTimeMillis()),
    TemperatureMeasure(102, 38.001, "AMER", System.currentTimeMillis()),
    TemperatureMeasure(103, 26.991, "AMER", System.currentTimeMillis()),
    TemperatureMeasure(104, 34.17, "AMER", System.currentTimeMillis())

Check for records in Kafka
~~~~~~~~~~~~~~~~~~~~~~~~~~

Check Kafka with the console consumer the topic for kjson (the Mqtt payload was a json and we translated that into a Kafka Connect Struct)

.. sourcecode:: bash

 ➜  bin/kafka-avro-console-consumer --zookeeper localhost:2181 --topic kjson --from-beginning

You should see the following output

.. sourcecode:: bash
    SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
    {"deviceId":1,"value":31.1,"region":"EMEA","timestamp":1482236627236}
    {"deviceId":2,"value":30.91,"region":"EMEA","timestamp":1482236627236}
    {"deviceId":3,"value":30.991,"region":"EMEA","timestamp":1482236627236}
    {"deviceId":4,"value":31.061,"region":"EMEA","timestamp":1482236627236}
    {"deviceId":101,"value":27.001,"region":"AMER","timestamp":1482236627236}
    {"deviceId":102,"value":38.001,"region":"AMER","timestamp":1482236627236}
    {"deviceId":103,"value":26.991,"region":"AMER","timestamp":1482236627236}
    {"deviceId":104,"value":34.17,"region":"AMER","timestamp":1482236627236}

Check Kafka with the console consumer the topic for kavro (the Mqtt payload was a avro and we translated that into a Kafka Connect Struct)

.. sourcecode:: bash

 ➜  bin/kafka-avro-console-consumer --zookeeper localhost:2181 --topic kavro --from-beginning

You should see the following output

.. sourcecode:: bash
    SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
    SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
    {"deviceId":1,"value":31.1,"region":"EMEA","timestamp":1482236627236}
    {"deviceId":2,"value":30.91,"region":"EMEA","timestamp":1482236627236}
    {"deviceId":3,"value":30.991,"region":"EMEA","timestamp":1482236627236}
    {"deviceId":4,"value":31.061,"region":"EMEA","timestamp":1482236627236}
    {"deviceId":101,"value":27.001,"region":"AMER","timestamp":1482236627236}
    {"deviceId":102,"value":38.001,"region":"AMER","timestamp":1482236627236}
    {"deviceId":103,"value":26.991,"region":"AMER","timestamp":1482236627236}
    {"deviceId":104,"value":34.17,"region":"AMER","timestamp":1482236627236}

Features
--------

The Mqtt source allows you to plugin your own converter. Say you receive protobuf data, all you have to do is to write your own
very specific converter that knows how to convert from protobuf to SourceRecord. All you have to do is set the connect.mqtt.source.converters
for the topic containing the protobuf data.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The Mqtt Source supports the following:

.. sourcecode:: bash

    INSERT INTO <target topic> SELECT * FROM <mqtt source topic>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO kafkaTopic1 SELECT * FROM mqttTopicA



Configurations
--------------

``connect.mqtt.source.kcql``

Kafka connect query language expression. Allows for expressive Mqtt topic to Kafka topicrouting. Currently there is no support
for filtering the fields from the incoming payload.

* Data type : string
* Importance: high
* Optional  : no

Examples:

.. sourcecode:: sql

    INSERT INTO KAFKA_TOPIC1 SELECT * FROM MQTT_TOPIC1;INSERT INTO KAFKA_TOPIC2 SELECT * FROM MQTT_TOPIC2

``connect.mqtt.hosts``

Specifies the mqtt connection endpoints.

* Data type : string
* Importance: high
* Optional  : no


Example
~~~~~~~

.. sourcecode:: bash
  tcp://broker.datamountaineer.com:1883


``connect.mqtt.service.quality``

he Quality of Service (QoS) level is an agreement between sender and receiver of a message regarding the guarantees of delivering a message. There are 3 QoS levels in MQTT:
At most once (0); At least once (1); Exactly once (2).

* Data type : int
* Importance: high
* Optional  : yes
* Default:    1


``connect.mqtt.user``
Contains the Mqtt connection user name

* Data type : string
* Importance: medium
* Optional  : yes
* Default:    null


``connect.mqtt.password``
Contains the Mqtt connection password

* Data type : string
* Importance: medium
* Optional  : yes
* Default:     null


``connect.mqtt.client.id``
Provides the client connection identifier. If is not provided the framework will generate one.

* Data type:  string
* Importance: medium
* Optional:   yes
* Default:    generated


``connect.mqtt.connection.timeout``
Sets the timeout to wait for the broker connection to be established

* Data type:  int
* Importance: medium
* Optional:   yes
* Default:    3000 (ms)


``connect.mqtt.connection.clean``
The clean session flag indicates the broker, whether the client wants to establish a persistent session or not.
A persistent session (the flag is false) means, that the broker will store all subscriptions for the client and also all missed messages,
when subscribing with Quality of Service (QoS) 1 or 2. If clean session is set to true, the broker won’t store anything for the client and will
also purge all information from a previous persistent session.

* Data type:  boolean
* Importance: medium
* Optional:   yes
* Default:    true


``connect.mqtt.connection.keep.alive``
The keep alive functionality assures that the connection is still open and both broker and client are connected to one another.
Therefore the client specifies a time interval in seconds and communicates it to the broker during the establishment of the connection.
The interval is the longest possible period of time, which broker and client can endure without sending a message."

* Data type:  int
* Importance: medium
* Optional:   yes
* Default:    5000

``connect.mqtt.connection.ssl.ca.cert``
Provides the path to the CA certificate file to use with the Mqtt connection"

* Data type:  string
* Importance: medium
* Optional:   yes
* Default:    null

``connect.mqtt.connection.ssl.cert``
Provides the path to the certificate file to use with the Mqtt connection

* Data type:  string
* Importance: medium
* Optional:   yes
* Default:    null

``connect.mqtt.connection.ssl.key``
Certificate private key file path

* Data type:  string
* Importance: medium
* Optional:   yes
* Default:    null

``connect.mqtt.source.converters``
Contains a tuple (mqtt source topic and the canonical class name for the converter of a raw Mqtt message bytes to a SourceRecord).
If the source topic is not matched it will default to the BytesConverter.This will send an avro message over Kafka using Schema.BYTES

* Data type:  string
* Importance: medium
* Optional:   yes
* Default:    null

.. sourcecode:: bash
  $mqtt_source1=com.datamountaineer.streamreactor.connect.mqtt.source.converters.AvroConverter;$mqtt_source2=com.datamountaineer.streamreactor.connect.mqtt.source.converters.JsonSimpleConverter""".stripMargin

We provide three converters out of the box.
com.datamountaineer.streamreactor.connect.mqtt.source.converters.AvroConverter - the payload for the Mqtt message is an avro message. In this case you need to provide a path for the avro schema file to be able to decode it.
com.datamountaineer.streamreactor.connect.mqtt.source.converters.JsonSimpleConverter -  the payload for the Mqtt message is a json message. This converter will parse the json and create an avro record for it which will be sent over to Kafka
com.datamountaineer.streamreactor.connect.mqtt.source.converters.BytesConverter = this is the default implementation. The Mqtt payload is taken as is: an array of bytes and sent over Kafka as an avro record with Scheam.BYTES. You don't have to provide a mapping for the source to get this converter!!



``connect.mqtt.converter.throw.on.error``
If set to false the conversion exception will be swallowed and everything carries on BUT the message is lost!!; true will throw the exception.Default is false."

* Data type:  bool
* Importance: medium
* Optional:   yes
* Default:    false

``connect.source.converter.avro.schemas``
If the AvroConverter is used you need to provide an avro Schema to be able to read and translate the raw bytes to an avro record.
The formate is $MQTT_TOPIC=$PATH_TO_AVRO_SCHEMA_FILE

* Data type:  bool
* Importance: medium
* Optional:   yes
* Default:    null



Example
~~~~~~~

.. sourcecode:: bash

    "name":"mqtt-source",
    "tasks.max":"1",
    "connect.mqtt.connection.clean":"true",
    "connect.mqtt.connection.timeout":"1000",
    "connect.mqtt.source.kcql":"INSERT INTO kjson SELECT * FROM /mjson;INSERT INTO kavro SELECT * FROM /mavro",
    "connect.mqtt.connection.keep.alive":"1000",
    "connect.mqtt.source.converters":"/mjson=com.datamountaineer.streamreactor.connect.converters.source.JsonSimpleConverter;/mavro=com.datamountaineer.streamreactor.connect.converters.source.AvroConverter",
    "connect.source.converter.avro.schemas":"/mavro=$PATH_TO/temperaturemeasure.avro",
    "connect.mqtt.client.id":"dm_source_id",
    "connect.mqtt.converter.throw.on.error":"true",
    "connect.mqtt.hosts":"tcp://127.0.0.1:11883",
    "connect.mqtt.service.quality":"1",
    "connector.class":"com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector"

Provide your own Converter
----------------
You can always provide your own logic for converting the raw Mqtt message bytes to your an avro record.
If you have messages coming in Protobuf format you can deserialize the message based on the schema and create the avro record.
All you have to do is create a new project and add our dependency:

Gradle:
compile "com.datamountaineer:kafka-connect-common:0.6.1"

Maven:
<dependency>
    <groupId>com.datamountaineer</groupId>
    <artifactId>kafka-connect-common</artifactId>
    <version>0.6.1</version>
</dependency>

Then all you have to do is implement `com.datamountaineer.streamreactor.connect.converters.source.Converter`.
Here is our BytesConverter class code:

.. sourcecode:: scala

    class BytesConverter extends Converter {
      override def convert(kafkaTopic: String, sourceTopic: String, messageId: Int, bytes: Array[Byte]): SourceRecord = {
        new SourceRecord(Collections.singletonMap(Converter.TopicKey, sourceTopic),
          null,
          kafkaTopic,
          MsgKey.schema,
          MsgKey.getStruct(sourceTopic, messageId),
          Schema.BYTES_SCHEMA,
          bytes)
      }
    }


All our implementation will send a a MsgKey object as the Kafka message key. It contains the Mqtt source topic and the Mqtt message id

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
