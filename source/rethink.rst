Kafka Connect ReThink
=====================

A Connector and Sink to write events from Kafka to RethinkDb. The connector takes the value from the Kafka Connect
SinkRecords and inserts a new entry to RethinkDb.

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>` - Kafka topic payload field selection is supported, allowing you to select fields written to RethinkDb.
2. Topic to table routing via KCQL.
3. RowKey selection - Selection of fields to use as the row key, if none specified the topic name, partition and offset are
   used via KCQL.
4. RethinkDB write modes via KCQL.
5. Error policies for handling failures.
6. Schema.Struct and payload Struct, Schema.String and Json payload and Json payload with no schema

The Sink supports three Kafka payloads type:

**Connect entry with Schema.Struct and payload Struct.** If you follow the best practice while producing the events, each
message should carry its schema information. Best option is to send Avro. Your connect configurations should be set to
``value.converter=io.confluent.connect.avro.AvroConverter``.
You can fnd an example `here <https://github.com/confluentinc/kafka-connect-blog/blob/master/etc/connect-avro-standalone.properties>`__.
To see how easy is to have your producer serialize to Avro have a look at
`this <http://docs.confluent.io/3.0.1/schema-registry/docs/serializer-formatter.html?highlight=kafkaavroserializer>`__.
This requires SchemaRegistry which is open source thanks to Confluent! Alternatively you can send Json + Schema.
In this case your connect configuration should read ``value.converter=org.apache.kafka.connect.json.JsonConverter``.
The difference would be to point your serialization to ``org.apache.kafka.connect.json.JsonSerializer``. This doesn't
require the SchemaRegistry.

**Connect entry with Schema.String and payload json String.** Sometimes the producer would find it easier, despite sending
Avro to produce a GenericRecord, to just send a message with Schema.String and the json string.

**Connect entry without a schema and the payload json String.** There are many existing systems which are publishing json
over Kafka and bringing them in line with best practices is quite a challenge. Hence we added the support

Prerequisites
-------------

- Confluent 3.1.1
- RethinkDb 2.3.3
- Java 1.8
- Scala 2.11

Setup
-----

Rethink Setup
~~~~~~~~~~~~~

Download and install RethinkDb. Follow the instruction `here <https://rethinkdb.com/docs/install/>`__ dependent on your
operating system.


Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Sink Connector QuickStart
-------------------------

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

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for ReThinkDB.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create rethink-sink < rethink-sink.properties
    #Connector name=`rethink-sink`
    name=rethink-sink
    connect.rethink.sink.db=localhost
    connect.rethink.sink.port=28015
    connector.class=com.datamountaineer.streamreactor.connect.rethink.sink.ReThinkSinkConnector
    tasks.max=1
    topics=rethink-topic
    connect.rethink.sink.kcql=INSERT INTO TABLE1 SELECT * FROM rethink-topic
    #task ids: 0

The ``rethink-sink.properties`` file defines:

1.  The name of the sink.
2.  The name of the rethink host to connect to.
3.  The rethink port to connect to.
4.  The Sink class.
5.  The max number of tasks the connector is allowed to created. Should not be greater than the number of partitions in
    the Source topics otherwise tasks will be idle.
6.  The Source kafka topics to take events from.
7.  :ref:`The KCQL routing querying. <kcql>`

If you switch back to the terminal you started the Connector in you should see the ReThinkDB Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    rethink-sink

.. sourcecode:: bash

    [2016-05-08 22:37:05,616] INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
        ____     ________    _       __   ____  ____
       / __ \___/_  __/ /_  (_)___  / /__/ __ \/ __ )
      / /_/ / _ \/ / / __ \/ / __ \/ //_/ / / / __  |
     / _, _/  __/ / / / / / / / / / ,< / /_/ / /_/ /
    /_/ |_|\___/_/ /_/ /_/_/_/ /_/_/|_/_____/_____/

     (com.datamountaineer.streamreactor.connect.rethink.sink.config.RethinkSinkConfig)


Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``firstname`` field of type
string a ``lastname`` field of type string, an ``age`` field of type int and a ``salary`` field of type double.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic rethink-topic \
      --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.rethink"
      ,"fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"salary","type":"double"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}

Check for records in Rethink
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this:

.. sourcecode:: bash

    INFO Received record from topic:person_rethink partition:0 and offset:0 (com.datamountaineer.streamreactor.connect.rethink.sink.writer.rethinkDbWriter:48)
    INFO Empty list of records received. (com.datamountaineer.streamreactor.connect.rethink.sink.RethinkSinkTask:75)

Check for records in Rethink

Now stop the connector.

Features
--------

The ReThinkDb Sink writes records from Kafka to RethinkDb.

The Sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to select fields written to RethinkDb.
2. Topic to table routing.
3. RowKey selection - Selection of fields to use as the row key, if none specified the topic name, partition and offset are
   used.
4. RethinkDB write modes.
5. Error policies for handling failures.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The ReThink Sink supports the following:

.. sourcecode:: bash

    <write mode> INTO <target table> SELECT <fields> FROM <source topic> <AUTOCREATE> <PK_FIELD>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB

    #Upsert mode, select all fields from topicC, auto create tableC and auto evolve, use field1 as the primary key
    UPSERT INTO tableC SELECT * FROM topicC AUTOCREATE PK field1

Write Modes
~~~~~~~~~~~

The Sink support two write modes **insert** and **upsert** which map to RethinkDb's conflict policies, **insert** to **ERROR**
and **upsert** to **REPLACE**.

Error Polices
~~~~~~~~~~~~~

The Sink has three error policies that determine how failed writes to the target database are handled. The error policies
affect the behaviour of the schema evolution characteristics of the sink. See the schema evolution section for more
information.

**Throw**

Any error on write to the target database will be propagated up and processing is stopped. This is the default
behaviour.

**Noop**

Any error on write to the target database is ignored and processing continues.

.. warning::

    This can lead to missed errors if you don't have adequate monitoring. Data is not lost as it's still in Kafka
    subject to Kafka's retention policy. The Sink currently does **not** distinguish between integrity constraint
    violations and or other expections thrown by drivers.

**Retry**

Any error on write to the target database causes the RetryIterable exception to be thrown. This causes the
Kafka connect framework to pause and replay the message. Offsets are not committed. For example, if the table is offline
it will cause a write failure, the message can be replayed. With the Retry policy the issue can be fixed without stopping
the sink.

The length of time the Sink will retry can be controlled by using the ``connect.rethink.sink.max.retries`` and the
``connect.rethink.sink.retry.interval``.

Topic Routing
~~~~~~~~~~~~~

The Sink supports topic routing that allows mapping the messages from topics to a specific table. For example, map a
topic called "bloomberg_prices" to a table called "prices". This mapping is set in the ``connect.rethink.sink.kcql``
option.

Example:

.. sourcecode:: sql

    //Select all
    INSERT INTO table1 SELECT * FROM topic1; INSERT INTO tableA SELECT * FROM topicC

Field Selection
~~~~~~~~~~~~~~~

The ReThink Sink supports field selection and mapping. This mapping is set in the ``connect.rethink.sink.kcql`` option.


Examples:

.. sourcecode:: sql

    //Rename or map columns
    INSERT INTO table1 SELECT lst_price AS price, qty AS quantity FROM topicA

    //Select all
    INSERT INTO table1 SELECT * FROM topic1

.. tip:: Check you mappings to ensure the target columns exist.

Auto Create Tables
~~~~~~~~~~~~~~~~~~

The Sink supports auto creation of tables for each topic. This mapping is set in the ``connect.rethink.sink.kcql`` option.

A user specified primary can be set in the ``PK`` clause for the ``connect.rethink.sink.kcql`` option. Only one
key is supported. If more than one is set only the first is used. If no primary keys are set the default primary key
called ``id`` is used. The value for the default key is the topic name, partition and offset of the records.

.. sourcecode:: sql

    #AutoCreate the target table
    INSERT INTO table1 SELECT * FROM topic AUTOCREATE PK field1

..	note::

    The fields specified as the primary keys must be in the SELECT clause or all fields must be selected

The Sink will try and create the table at start up if a schema for the topic is found in the Schema Registry. If no
schema is found the table is created when the first record is received for the topic.

Configurations
--------------

``connect.rethink.sink.kcql``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming. Fields
to be used as the row key can be set by specifing the ``PK``. The below example uses field1 as the primary key.

* Data type : string
* Importance: high
* Optional  : no

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT * FROM TOPIC2 PK field1

``connect.rethink.sink.host``

Specifies the rethink server.

* Data type : string
* Importance: high
* Optional  : no

``connect.rethink.sink.port``

Specifies the rethink server port number.

* Data type : int
* Importance: high
* Optional  : yes

``connect.rethink.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.rethink.sink.max.retries``
option. The ``connect.rethink.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: medium
* Optional: yes
* Default: RETRY

``connect.rethink.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.rethink.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: high
* Optional: yes
* Default: 10


``connect.rethink.sink.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.rethink.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Optional: yes
* Default : 60000 (1 minute)

``connect.rethink.sink.batch.size``

Specifies how many records to insert together at one time. If the connect framework provides less records when it is
calling the Sink it won't wait to fulfill this value but rather execute it.

* Type : int
* Importance : medium
* Optional: yes
* Defaults : 3000


Example
~~~~~~~

.. sourcecode:: bash

    name=rethink-sink
    connect.rethink.sink.db=localhost
    connect.rethink.sink.port=28015
    connector.class=com.datamountaineer.streamreactor.connect.rethink.sink.ReThinkSinkConnector
    tasks.max=1
    topics=person_rethink
    connect.rethink.sink.kcql=INSERT INTO TABLE1 SELECT * FROM person_rethink

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.

The rethink Sink will automatically write and update the rethink table if new fields are added to the Source topic,
if fields are removed the Kafka Connect framework will return the default value for this field, dependent of the
compatibility settings of the Schema registry.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
