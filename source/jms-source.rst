Kafka Connect JMS Source
========================

The JMS Source connector allows subscribe to messages on JMS queues and topics.

The Source supports:

1.  Pluggable converters of JMS payloads. If no converters are specified a Avro message is created representing the JMS Message,
    the payload from the message is stored as a byte array in the `payload` field of the Avro.
2.  :ref:`Out of the box converters for Json/Avro and Binary <jms_converters>`
3.  :ref:`The KCQL routing querying <kcql>` - JMS Destination to Kafka topic mapping.

Prerequisites
-------------
- Confluent 3.2
- Java 1.8
- Scala 2.11
- A JMS framework (ActiveMQ for example)

Setup
-----

Before we can do anything, including the QuickStart we need to install the Confluent platform.
For ActiveMQ follow http://activemq.apache.org/getting-started.html for the instruction of setting
it up.


Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Source Connector QuickStart
---------------------------

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and updating the configuration. We have developed
a Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running the ``start-connect.sh`` script in the ``bin`` folder.

.. sourcecode:: bash

    ➜ bin/start-connect.sh

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for JMS.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create jms-sink < conf/jms-source.properties


The ``jms-source.properties`` file defines:

name=jms-source
connector.class=com.datamountaineer.streamreactor.connect.jms.source.JMSSourceConnector
tasks.max=1
connect.jms.kcql=INSERT INTO topic SELECT * FROM jms-queue
connect.jms.queues=jms-queue
connect.jms.initial.context.factory=org.apache.activemq.jndi.ActiveMQInitialContextFactory
connect.jms.url=tcp://localhost:61616
connect.jms.connection.factory=ConnectionFactory

1.  The source connector name.
2.  The JMS Source Connector class name.
3.  The number of tasks to start.
4.  :ref:`The KCQL routing querying. <kcql>`
5.  A comma separated list of queues destination types on the target JMS, must match the `from` element in KCQL.
6.  The JMS initial context factory.
7.  The url of the JMS broker.
8.  The JMS connection factory.


If you switch back to the terminal you started the Connector in you should see the JMS Source being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    jms-source

    #Connector `jms-source`:
    name=jms-source
    connect.jms.kcql=INSERT INTO topic SELECT * FROM jms-queue
    tasks.max=1
    connector.class=com.datamountaineer.streamreactor.connect.jms.source.JMSSourceConnector
    connect.jms.queues=jms-queue
    connect.jms.initial.context.factory=org.apache.activemq.jndi.ActiveMQInitialContextFactory
    connect.jms.url=tcp://localhost:61616
    connect.jms.connection.factory=ConnectionFactory
    #task ids: 0

.. sourcecode:: bash

    INFO Kafka version : 0.10.2.0-cp1 (org.apache.kafka.common.utils.AppInfoParser:83)
    INFO Kafka commitId : 64c9b42f3319cdc9 (org.apache.kafka.common.utils.AppInfoParser:84)
    INFO     ____        __        __  ___                  __        _
            / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
           / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
          / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
         /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
                 ____  _____________
                / /  |/  / ___/ ___/____  __  _______________
           __  / / /|_/ /\__ \\__ \/ __ \/ / / / ___/ ___/ _ \  By Andrew Stevenson
          / /_/ / /  / /___/ /__/ / /_/ / /_/ / /  / /__/  __/
          \____/_/  /_//____/____/\____/\__,_/_/   \___/\___/
     (com.datamountaineer.streamreactor.connect.jms.source.JMSSourceTask:22)
    INFO JMSConfig values:
        connect.jms.batch.size = 100
        connect.jms.connection.factory = ConnectionFactory
        connect.jms.converter.throw.on.error = false
        connect.jms.destination.selector = CDI
        connect.jms.error.policy = THROW
        connect.jms.initial.context.extra.params = []
        connect.jms.initial.context.factory = org.apache.activemq.jndi.ActiveMQInitialContextFactory
        connect.jms.kcql = INSERT INTO topic SELECT * FROM jms-queue
        connect.jms.max.retries = 20
        connect.jms.password = null
        connect.jms.queues = [jms-queue]
        connect.jms.retry.interval = 60000
        connect.jms.source.converters =
        connect.jms.topics = []
        connect.jms.url = tcp://localhost:61616
        connect.jms.user = null
     (com.datamountaineer.streamreactor.connect.jms.config.JMSConfig:180)
    INFO Instantiated connector jms-source with version null of type class com.datamountaineer.streamreactor.connect.jms.source.JMSSourceConnector (org.apache.kafka.connect.runtime.Worker:181)
    INFO Finished creating connector jms-source (org.apache.kafka.connect.runtime.Worker:194)
    INFO SourceConnectorConfig values:
        connector.class = com.datamountaineer.streamreactor.connect.jms.source.JMSSourceConnector
        key.converter = null
        name = jms-source
        tasks.max = 1
        transforms = null
        value.converter = null
     (org.apache.kafka.connect.runtime.SourceConnectorConfig:180)

Test Records
^^^^^^^^^^^^

Now we need to send some records into the ActiveMQ broker for the Source Connector to pick up. We can do this with the
ActiveMQ command line producer. In the bin folder of the Active MQ location run the following to insert 1000 messages into
a queue called `jms-queue`.

.. sourcecode:: bash

    activemq producer --destination queue://jms-queue --message "hello DataMountaineer"


We should immediately see the records coming through the sink and into our Kafka topic:

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-consumer \
    --zookeeper localhost:2181 \
    --topic topic \
    --from-beginning

.. sourcecode:: json

    {"message_timestamp":{"long":1490799748984},"correlation_id":null,"redelivered":{"boolean":false},"reply_to":null,"destination":{"string":"queue://jms-queue"},"message_id":{"string":"ID:Andrews-MacBook-Pro.local-49870-1490799747943-1:1:1:1:997"},"mode":{"int":2},"type":null,"priority":{"int":4},"bytes_payload":{"bytes":"hello"},"properties":null}
    {"message_timestamp":{"long":1490799748985},"correlation_id":null,"redelivered":{"boolean":false},"reply_to":null,"destination":{"string":"queue://jms-queue"},"message_id":{"string":"ID:Andrews-MacBook-Pro.local-49870-1490799747943-1:1:1:1:998"},"mode":{"int":2},"type":null,"priority":{"int":4},"bytes_payload":{"bytes":"hello"},"properties":null}
    {"message_timestamp":{"long":1490799748986},"correlation_id":null,"redelivered":{"boolean":false},"reply_to":null,"destination":{"string":"queue://jms-queue"},"message_id":{"string":"ID:Andrews-MacBook-Pro.local-49870-1490799747943-1:1:1:1:999"},"mode":{"int":2},"type":null,"priority":{"int":4},"bytes_payload":{"bytes":"hello"},"properties":null}
    {"message_timestamp":{"long":1490799748987},"correlation_id":null,"redelivered":{"boolean":false},"reply_to":null,"destination":{"string":"queue://jms-queue"},"message_id":{"string":"ID:Andrews-MacBook-Pro.local-49870-1490799747943-1:1:1:1:1000"},"mode":{"int":2},"type":null,"priority":{"int":4},"bytes_payload":{"bytes":"hello"},"properties":null}


Features
--------

The Source supports:

1.  KCQL routing of JMS destination messages to Kafka topics.
2.  Pluggable converters.
3.  Default conversion of JMS Messages to Avro with the payload as a Byte array.
4.   Extra connection properties for specialized connections such as SOLACE_VPN.

.. _jms_converters:

Converters
~~~~~~~~~~

We provide four converters out of the box but you can plug your own. See an example :ref:`here. <jms_converter_example>`

**AvroConverter**


``com.datamountaineer.streamreactor.connect.source.converters.AvroConverter``

The payload of the JMS message is an Avro message. In this case you need to provide a path for the Avro schema file to
be able to decode it.

**JsonSimpleConverter**

``com.datamountaineer.streamreactor.connect.source.converters.JsonSimpleConverter``

The payload for the JMS message is a Json message. This converter will parse the json and create an Avro record for it which
will be sent over to Kafka.

**JsonConverterWithSchemaEvolution**

An experimental converter for converting Json messages to Avro. The resulting  Avro schema is fully compatible as new fields are
added as the JMS json payload evolves.

**BytesConverter**

``com.datamountaineer.streamreactor.connect.source.converters.BytesConverter``

This is the default implementation. The JMS payload is taken as is: an array of bytes and sent over Kafka as an avro
record with ``Schema.BYTES``. You don't have to provide a mapping for the source to get this converter!!


Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The JMS Source supports the following:

.. sourcecode:: bash

    INSERT INTO <kafka target> SELECT * FROM <jms destination>

Example:

.. sourcecode:: sql

    #select from a JMS queue and write to a kafka topic
    INSERT INTO topicA SELECT * FROM jms_queue

Configurations
--------------

``connect.jms.url``

Provides the JMS broker url

* Data Type: string
* Importance: high
* Optional : no

``connect.jms.user``

Provides the user for the JMS connection.

* Data Type: string
* Importance: high
* Optional : no

``connect.jms.password``

Provides the password for the JMS connection.

* Data Type: string
* Importance: high
* Optional : no

``connect.jms.initial.context.factory``

* Data Type: string
* Importance: high
* Optional: no

Initial Context Factory, e.g: org.apache.activemq.jndi.ActiveMQInitialContextFactory.

``connect.jms.connection.factory``

The ConnectionFactory implementation to use.

* Data Type: string
* Importance: high
* Optional : no

``connect.jms.destination.selector``

* Data Type: String
* Importance: high
* Optional: no
* Default: CDI

Selector to use for destination lookup. Either CDI or JNDI.

``connect.jms.initial.context.extra.params``

* Data Type: String
* Importance: high
* Optional: yes

List (comma separated) of extra properties as key/value pairs with a colon delimiter to supply to the initial context e.g. SOLACE_JMS_VPN:my_solace_vp.

``connect.jms.kcql``

KCQL expression describing field selection and routes.

* Data Type: string
* Importance: high
* Optional : no

``connect.jms.topics``

Comma separated list of all the jms target topics.

* Data Type: list
* Importance: medium
* Optional : yes

``connect.jms.queues``

Comma separated list of all the jms target queues.

* Data Type: list
* Importance: medium
* Optional : yes


``connect.jms.source.converters``

Contains a tuple (jms source topic and the canonical class name for the converter to convert the JMS message to a SourceRecord).
If the source topic is not matched it will default to the BytesConverter. This will send an avro message over Kafka using Schema.BYTES

* Data type:  string
* Importance: medium
* Optional:   yes
* Default:    null

.. sourcecode:: bash

  jms_source1=com.datamountaineer.streamreactor.connect.source.converters.AvroConverter;jms_source2=com.datamountaineer.streamreactor.connect.source.converters.JsonSimpleConverter


``connect.source.converter.avro.schemas``

If the AvroConverter is used you need to provide an avro Schema to be able to read and translate the raw bytes to an avro record.
The format is $JMS_TOPIC=$PATH_TO_AVRO_SCHEMA_FILE

* Data type:  bool
* Importance: medium
* Optional:   yes
* Default:    null

`connect.jms.batch.size`

* Type: int
* Importance: medium
* Optional: yes
* Default: 100

The batch size to take from the JMS destination on each poll of Kafka Connect.

.. _jms_converter_example:

Provide your own Converter
--------------------------

You can always provide your own logic for converting the JMS message to your an avro record.
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


Schema Evolution
----------------

Not applicable.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
