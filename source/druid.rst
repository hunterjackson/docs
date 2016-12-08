Kafka Connect Druid
===================

A Connector and Sink to write events from Kafka to Druid using the Tranquility client.

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>` - Topic to index mapping and Field selection.

Prerequisites
-------------

- Confluent 3.0.1
- Druid 0.9.2
- Tranquility 0.7.4
- Java 1.8
- Scala 2.11

Setup
-----

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Druid Setup
~~~~~~~~~~~

.. tip::

    Start Confluent first. Druid requires Zookeeper.

.. sourcecode:: bash

    curl -O http://static.druid.io/artifacts/releases/druid-0.9.2-bin.tar.gz
    tar -xzf druid-0.9.2-bin.tar.gz
    cd druid-0.9.2/

    #initialise install
    bin/init

    #start
    java `cat conf-quickstart/druid/historical/jvm.config | xargs` -cp "conf-quickstart/druid/_common:conf-quickstart/druid/historical:lib/*" io.druid.cli.Main server historical
    java `cat conf-quickstart/druid/broker/jvm.config | xargs` -cp "conf-quickstart/druid/_common:conf-quickstart/druid/broker:lib/*" io.druid.cli.Main server broker
    java `cat conf-quickstart/druid/coordinator/jvm.config | xargs` -cp "conf-quickstart/druid/_common:conf-quickstart/druid/coordinator:lib/*" io.druid.cli.Main server coordinator
    java `cat conf-quickstart/druid/overlord/jvm.config | xargs` -cp "conf-quickstart/druid/_common:conf-quickstart/druid/overlord:lib/*" io.druid.cli.Main server overlord
    java `cat conf-quickstart/druid/middleManager/jvm.config | xargs` -cp "conf-quickstart/druid/_common:conf-quickstart/druid/middleManager:lib/*" io.druid.cli.Main server middleManager


Sink Connector QuickStart
-------------------------

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and updating the configuration. We have developed
a Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.


Starting the Connector
~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running the ``start-connect.sh`` script in the ``bin`` folder.

.. sourcecode:: bash

    ➜ bin/start-connect.sh

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for Elastic.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create druid-sink < conf/quickstarts/druid-sink.properties

    #Connector name=`druid-sink`
    #task ids: 0

The ``druid-sink.properties`` file defines:

1. The name of the connector.
2. The class containing the connector.
3. The druid config file.
4. The max number of task allowed for this connector.
5. The Source topic to get records from.
6. :ref:`The KCQL routing querying. <kcql>`

If you switch back to the terminal you started the Connector in you should see the Elastic Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    druid-sink

.. sourcecode:: bash



Test Records
^^^^^^^^^^^^

.. hint::

    If your input topic doesn't match the target use Kafka Streams to transform in realtime the input. Also checkout the
    `Plumber <https://github.com/rollulus/kafka-streams-plumber>`__, which allows you to inject a Lua script into
    `Kafka Streams <http://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple>`__ to do this,
    no Java or Scala required!

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``id`` field of type int
and a ``random_field`` of type string.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
     --broker-list localhost:9092 --topic TOPIC1 \
     --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"id","type":"int"},
    {"name":"random_field","type": "string"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"id": 999, "random_field": "foo"}
    {"id": 888, "random_field": "bar"}


Check for records in Druid
^^^^^^^^^^^^^^^^^^^^^^^^^^

Now if we check the logs of the connector we should see 2 records being inserted to Druid:

.. sourcecode:: bash


If we query Druid:

.. sourcecode:: bash



Features
--------

1. Auto mapping of the Kafka topic schema to the index.
2. Field selection

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L**, :ref:`KCQL <kcql>` allows for routing and mapping using a SQL like syntax,
consolidating typically features in to one configuration option.

The Druid Sink supports the following:

.. sourcecode:: bash

    INSERT INTO <datasource> SELECT <fields> FROM <source topic>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to indexA
    INSERT INTO indexA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to indexB
    INSERT INTO indexB SELECT x AS a, y AS b and z AS c FROM topicB

This is set in the ``connect.druid.sink.kcql`` option.

Configurations
--------------

``connect.druid.sink.config.file``

The path to the configuration file.

* Data type : string
* Importance: high
* Optional  : no

``connnect.druid.sink.write.timeout``

Specifies the number of seconds to wait for the write to Druid to happen

* Data type : int
* Importance: low
* Optional  : yes
* Default   : 6000 milliseconds

``connect.druid.sink.kcql``

Kafka connect query language expression. Allows for expressive table to topic routing, field selection and renaming.

Examples:

.. sourcecode:: sql

    INSERT INTO dataSource SELECT field1, field2 FROM TOPIC1

* Data type : string
* Importance: high
* Optional  : no

Example
~~~~~~~

.. sourcecode:: bash



Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.


Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
