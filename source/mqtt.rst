Kafka Connect Mqtt Source
=========================

A Connector to read events from Mqtt and push them to Kafka. The connector subscribes to the specified topics and and
streams the records to Kafka.

The Source supports:

1.  Pluggable converters of MQTT payloads.
2.  :ref:`Out of the box converters for Json/Avro and Binary <mqtt_converters>`
3.  :ref:`The KCQL routing querying <kcql>` - Topic to Topic mapping and Field selection.

Prerequisites
-------------

- Confluent 3.3
- Mqtt server
- Java 1.8
- Scala 2.11

Setup
-----

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Mqtt Setup
~~~~~~~~~~

For testing we will use a simple application spinning up an mqtt server using Moquette. Download and unzip `this <https://github.com/datamountaineer/mqtt-server/releases/download/v.0.1/mqtt-server-0.1.tgz>`__.

Once you have unpacked the archiver you should start the server.

.. sourcecode:: bash

    ➜  bin/mqtt-server

You should see the following outcome:

.. sourcecode:: bash

    log4j:WARN No appenders could be found for logger (io.moquette.server.Server).
    log4j:WARN Please initialize the log4j system properly.
    log4j:WARN See http://logging.apache.org/log4j/1.2/faq.html#noconfig for more info.
    Starting mqtt service on port 11883
    Hit Enter to start publishing messages on topic: /mjson and /mavro.

The server has started but no records have been published yet. More on this later once we start the source.

Source Connector QuickStart
---------------------------

We you start the Confluent Platform, Kafka Connect is started in distributed mode (``confluent start``). 
In this mode a Rest Endpoint on port ``8083`` is exposed to accept connector configurations. 
We developed Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

Starting the Connector
~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor and Confluent. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Once the Connect has started we can now use the kafka-connect-tools :ref:`cli <kafka-connect-cli>` to post in our distributed properties file for MQTT.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/connect-cli create mqtt-source < conf/source.kcql/mqtt-source.properties

    #Connector name=`mqtt-source`
    name=mqtt-source
    tasks.max=1
    connect.mqtt.connection.clean=true
    connect.mqtt.connection.timeout=1000
    connect.mqtt.kcql=INSERT INTO kjson SELECT * FROM /mjson WITHCONVERTER=com.datamountaineer.streamreactor.connect.converters.source.JsonSimpleConverter
    connect.mqtt.connection.keep.alive=1000
    connect.mqtt.client.id=dm_source_id,
    connect.mqtt.converter.throw.on.error=true
    connect.mqtt.hosts=tcp://127.0.0.1:11883
    connect.mqtt.service.quality=1
    connector.class=com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector
    #task ids: 0

The ``mqtt-source.properties`` file defines:

1.  The name of the source.
2.  The name number of tasks.
3.  Clean the mqtt connection.
4.  The Kafka Connect Query statements to read from json and avro topics and insert into Kafka kjson and kavro topics.
5.  Setting the time window to emit keep alive pings
6.  The mqtt client identifier.
7.  If a conversion can't happen it will throw an exception.
8.  The connection to the Mqtt server.
9. The quality of service for the messages.
10. Set the connector source class.

Use the Confluent CLI to view Connects logs.

.. sourcecode:: bash

    # Get the logs from Connect
    confluent log connect

    # Follow logs from Connect
    confluent log connect -f

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/connect-cli ps
    mqtt-source

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
        connect.mqtt.kcql = INSERT INTO kjson SELECT * FROM /mjson WITHCONVERTER=com.datamountaineer.streamreactor.connect.converters.source.JsonSimpleConverter
        connect.mqtt.service.quality = 1
        connect.mqtt.connection.ssl.cert = null
        connect.mqtt.connection.keep.alive = 1000
        connect.mqtt.hosts = tcp://127.0.0.1:11883
        connect.mqtt.converter.throw.on.error = true
        connect.mqtt.connection.timeout = 1000
        connect.mqtt.username = null
        connect.mqtt.connection.clean = true
        connect.mqtt.connection.ssl.ca.cert = null
        connect.mqtt.connection.ssl.key = null
        connect.mqtt.password = null
        connect.mqtt.client.id = dm_source_id
     (com.datamountaineer.streamreactor.connect.mqtt.config.MqttSourceConfig:178)


Test Records
^^^^^^^^^^^^

Go to the mqtt-server application you downloaded and unzipped and execute:

.. sourcecode:: bash

    ./bin/mqtt-server

This will put the following records into the json Mqtt topic:


.. sourcecode:: scala

    TemperatureMeasure(1, 31.1, "EMEA", System.currentTimeMillis())
    TemperatureMeasure(2, 30.91, "EMEA", System.currentTimeMillis())
    TemperatureMeasure(3, 30.991, "EMEA", System.currentTimeMillis())
    TemperatureMeasure(4, 31.061, "EMEA", System.currentTimeMillis())
    TemperatureMeasure(101, 27.001, "AMER", System.currentTimeMillis())
    TemperatureMeasure(102, 38.001, "AMER", System.currentTimeMillis())
    TemperatureMeasure(103, 26.991, "AMER", System.currentTimeMillis())
    TemperatureMeasure(104, 34.17, "AMER", System.currentTimeMillis())

Check for records in Kafka
~~~~~~~~~~~~~~~~~~~~~~~~~~

Check for records in Kafka with the console consumer. the topic for kjson (the Mqtt payload was a json and we translated that into a Kafka Connect Struct)

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

Check for records in Kafka with the console consumer. the topic for kavro (the Mqtt payload was a avro and we translated that into a Kafka Connect Struct)

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
very specific converter that knows how to convert from protobuf to SourceRecord. All you have to do is set the ``WITHCONVERTER=`` 
in the KCQL statement.

.. _mqtt_converters:

Converters
~~~~~~~~~~

We provide four converters out of the box but you can plug your own. See an example :ref:`here. <mqtt_converter_example>`

**AvroConverter**

``com.datamountaineer.streamreactor.connect.converters.source.AvroConverter``

The payload for the Mqtt message is an Avro message. In this case you need to provide a path for the Avro schema file to
be able to decode it.

**JsonSimpleConverter**

``com.datamountaineer.streamreactor.connect.converters.source.JsonSimpleConverter``

The payload for the Mqtt message is a Json message. This converter will parse the json and create an Avro record for it which
will be sent over to Kafka.

**JsonConverterWithSchemaEvolution**

An experimental converter for converting Json messages to Avro. The resulting  Avro schema is fully compatible as new fields are
added as the MQTT json payload evolves.

**BytesConverter**

``com.datamountaineer.streamreactor.connect.converters.source.BytesConverter``

This is the default implementation. The Mqtt payload is taken as is: an array of bytes and sent over Kafka as an avro
record with ``Schema.BYTES``. You don't have to provide a mapping for the source to get this converter!!

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The Mqtt Source supports the following:

.. sourcecode:: bash

    INSERT INTO <target topic> SELECT * FROM <mqtt source topic> [WITHCONVERTER=myclass]

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to topic kafkaTopic1 with converter myclass
    INSERT INTO kafkaTopic1 SELECT * FROM mqttTopicA [WITHCONVERTER=myclass]

    #wildcard
    INSERT INTO kafkaTopic1 SELECT * FROM mqttTopicA/+/sensors [WITHCONVERTER=myclass]

.. note::

    Wildcard MQTT subscriptions are supported but require the same converter to be used for all.

Configurations
--------------

``connect.mqtt.kcql``

Kafka connect query language expression. Allows for expressive Mqtt topic to Kafka topic routing. Currently there is no support
for filtering the fields from the incoming payload.

* Data type : string
* Importance: high
* Optional  : no

``connect.mqtt.hosts``

Specifies the mqtt connection endpoints.

* Data type : string
* Importance: high
* Optional  : no

Example:

.. sourcecode:: bash

  tcp://broker.datamountaineer.com:1883

``connect.mqtt.service.quality``

The Quality of Service (QoS) level is an agreement between sender and receiver of a message regarding the guarantees of delivering a message. There are 3 QoS levels in MQTT:
At most once (0); At least once (1); Exactly once (2).

* Data type : int
* Importance: high
* Optional  : yes
* Default:    1

``connect.mqtt.username``

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
The interval is the longest possible period of time, which broker and client can endure without sending a message.

* Data type:  int
* Importance: medium
* Optional:   yes
* Default:    5000

``connect.mqtt.connection.ssl.ca.cert``

Provides the path to the CA certificate file to use with the Mqtt connection

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

Certificate private key file path.

* Data type:  string
* Importance: medium
* Optional:   yes
* Default:    null

``connect.mqtt.converter.throw.on.error``

If set to false the conversion exception will be swallowed and everything carries on BUT the message is lost!!; true will throw the exception.Default is false."

* Data type:  bool
* Importance: medium
* Optional:   yes
* Default:    false

``connect.converter.avro.schemas``

If the AvroConverter is used you need to provide an avro Schema to be able to read and translate the raw bytes to an avro record.
The format is $MQTT_TOPIC=$PATH_TO_AVRO_SCHEMA_FILE

* Data type:  bool
* Importance: medium
* Optional:   yes
* Default:    null

``connect.progress.enabled``

Enables the output for how many records have been processed.

* Type: boolean
* Importance: medium
* Optional: yes
* Default : false

Example
~~~~~~~

.. sourcecode:: bash

    name=mqtt-source
    tasks.max=1
    connect.mqtt.connection.clean=true
    connect.mqtt.connection.timeout=1000
    connect.mqtt.kcql=INSERT INTO kjson SELECT * FROM /mjson WITHCONVERTER=myclass;INSERT INTO kavro SELECT * FROM /mavro WITHCONVERTER=myclass
    connect.mqtt.connection.keep.alive=1000
    connect.converter.avro.schemas=/mavro=$PATH_TO/temperaturemeasure.avro
    connect.mqtt.client.id=dm_source_id,
    connect.mqtt.converter.throw.on.error=true
    connect.mqtt.hosts=tcp://127.0.0.1:11883
    connect.mqtt.service.quality=1
    connector.class=com.datamountaineer.streamreactor.connect.mqtt.source.MqttSourceConnector

.. _mqtt_converter_example:

Provide your own Converter
--------------------------

You can always provide your own logic for converting the raw Mqtt message bytes to your an avro record.
If you have messages coming in Protobuf format you can deserialize the message based on the schema and create the avro record.
All you have to do is create a new project and add our dependency:

Gradle:

.. sourcecode:: groovy

    compile "com.datamountaineer:kafka-connect-common:0.7.1"

Maven:

.. sourcecode:: xml

    <dependency>
        <groupId>com.datamountaineer</groupId>
        <artifactId>kafka-connect-common</artifactId>
        <version>0.7.1</version>
    </dependency>

Then all you have to do is implement ``com.datamountaineer.streamreactor.connect.converters.source.Converter``.

Here is our BytesConverter class code:

.. sourcecode:: scala

    class BytesConverter extends Converter {
      override def convert(kafkaTopic: String, sourceTopic: String, messageId: String, bytes: Array[Byte]): SourceRecord = {
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

Distributed Mode
~~~~~~~~~~~~~~~~

Connect, in production should be run in distributed mode. 

1.  Install the Confluent Platform on each server that will form your Connect Cluster.
2.  Create a folder on the server called ``plugins/streamreactor/libs``.
3.  Copy into the folder created in step 2 the required connector jars from the stream reactor download.
4.  Edit ``connect-avro-distributed.properties`` in the ``etc/schema-registry`` folder where you installed Confluent
    and uncomment the ``plugin.path`` option. Set it to the path you deployed the stream reactor connector jars
    in step 2.
5.  Start Connect, ``bin/connect-distributed etc/schema-registry/connect-avro-distributed.properties``

Connect Workers are long running processes so set an ``init.d`` or ``systemctl`` service accordingly.

Connector configurations can then be push to any of the workers in the Cluster via the CLI or curl, if using the CLI 
remember to set the location of the Connect worker you are pushing to as it defaults to localhost.

.. sourcecode:: bash

    export KAFKA_CONNECT_REST="http://myserver:myport"

Kubernetes
~~~~~~~~~~

Helm Charts are provided at our `repo <https://datamountaineer.github.io/helm-charts/>`__, add the repo to your Helm instance and install. We recommend using the Landscaper
to manage Helm Values since typically each Connector instance has it's own deployment.

Add the Helm charts to your Helm instance:

.. sourcecode:: bash

    helm repo add datamountaineer https://datamountaineer.github.io/helm-charts/


TroubleShooting
---------------

Please review the :ref:`FAQs <faq>` and join our `slack channel <https://slackpass.io/datamountaineers>`_.

