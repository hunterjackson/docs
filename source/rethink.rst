Kafka Connect ReThink
=====================

A Connector and Sink to write events from Kafka to RethinkDb. The connector takes the value from the Kafka Connect
SinkRecords and inserts a new entry to RethinkDb.

Prerequisites
-------------

- Confluent 3.0.0
- RethinkDb 2.3.0
- Java 1.8
- Scala 2.11

Setup
-----

rethink Setup
~~~~~~~~~~~

Download and install RethinkDb. Follow the instruction `here <https://rethinkdb.com/docs/install/>`__ dependent on your
operating system.


Confluent Setup
~~~~~~~~~~~~~~~

.. sourcecode:: bash

    #make confluent home folder
    ➜  mkdir confluent

    #download confluent
    ➜  wget http://packages.confluent.io/archive/2.0/confluent-2.0.1-2.11.7.tar.gz

    #extract archive to confluent folder
    ➜  tar -xvf confluent-2.0.1-2.11.7.tar.gz -C confluent

    #setup variables
    ➜  export CONFLUENT_HOME=~/confluent/confluent-2.0.1

Start the Confluent platform.

.. sourcecode:: bash

    #Start the confluent platform, we need kafka, zookeeper and the schema registry
    ➜  bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    ➜  bin/kafka-server-start etc/kafka/server.properties &
    ➜  bin/schema-registry-start etc/schema-registry/schema-registry.properties &

Build the Connector and CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The prebuilt jars can be taken from here and
`here <https://github.com/datamountaineer/kafka-connect-tools/releases>`__
or from `Maven <http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22kafka-connect-cli%22>`__

If you want to build the connector, clone the repo and build the jar.

.. sourcecode:: bash

    ##Build the connectors
    ➜  git clone https://github.com/datamountaineer/stream-reactor
    ➜  cd stream-reactor
    ➜  gradle fatJar

    ##Build the CLI for interacting with Kafka connectors
    ➜  git clone https://github.com/datamountaineer/kafka-connect-tools
    ➜  cd kafka-connect-tools
    ➜  gradle fatJar

Sink Connector QuickStart
-------------------------

Sink Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Next we start the connector in standalone mode. This useful for testing and one of jobs, usually you'd run in
distributed mode to get fault tolerance and better performance.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Since we are in standalone mode we'll create a file called ``rethinkdb-sink.properties`` with the contents below:

.. sourcecode:: bash

    name=rethink-sink
    connect.rethink.sink.db=localhost
    connect.rethink.sink.port=28015
    connector.class=com.datamountaineer.streamreactor.connect.rethink.sink.ReThinkSinkConnector
    tasks.max=1
    topics=person_rethink
    connect.rethink.export.route.query=INSERT INTO TABLE1 SELECT * FROM person_rethink

This configuration defines:

1.  The name of the sink.
2.  The name of the rethink host to connect to.
3.  The rethink port to connect to.
4.  The sink class.
5.  The max number of tasks the connector is allowed to created. Should not be greater than the number of partitions in
    the source topics otherwise tasks will be idle.
6.  The source kafka topics to take events from.

Starting the Sink Connector (Standalone)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now we are ready to start the rethink sink Connector in standalone mode.

.. note::

    You need to add the connector to your classpath or you can create a folder in ``share/java`` of the Confluent
    install location like, kafka-connect-myconnector and the start scripts provided by Confluent will pick it up.
    The start script looks for folders beginning with kafka-connect.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-rethink-0.1-all.jar
    #Start the connector in standalone mode, passing in two properties files, the first for the schema registry, kafka
    #and zookeeper and the second with the connector properties.
    ➜  bin/connect-standalone etc/schema-registry/connect-avro-standalone.properties rethink-sink.properties

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    ➜ java -jar build/libs/kafka-connect-cli-0.2-all.jar get rethink-sink

    #Connector name=`rethink-sink`
    name=rethink-sink
    connect.rethink.sink.db=localhost
    connect.rethink.sink.port=28015
    connector.class=com.datamountaineer.streamreactor.connect.rethink.sink.ReThinkSinkConnector
    tasks.max=1
    topics=person_rethink
    connect.rethink.export.route.query=INSERT INTO TABLE1 SELECT * FROM person_rethink
    #task ids: 0

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

    bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic person_rethink \
      --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.rethink" \
      ,"fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"salary","type":"double"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}

Check for records in rethink
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this:

.. sourcecode:: bash

    INFO Received record from topic:person_rethink partition:0 and offset:0 (com.datamountaineer.streamreactor.connect.rethink.sink.writer.rethinkDbWriter:48)
    INFO Empty list of records received. (com.datamountaineer.streamreactor.connect.rethink.sink.RethinkSinkTask:75)

Check the rethink.


Now stop the connector.

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Connectors can be deployed distributed mode. In this mode one or many connectors are started on the same or different
hosts with the same cluster id. The cluster id can be found in ``etc/schema-registry/connect-avro-distributed.properties.``

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

For this quick-start we will just use one host.

Now start the connector in distributed mode, this time we only give it one properties file for the kafka, zookeeper and
schema registry configurations.

.. sourcecode:: bash

    ➜  confluent-3.0.0/bin/connect-distributed confluent-3.0.0/etc/schema-registry/connect-avro-distributed.properties

Once the connector has started lets use the kafka-connect-tools cli to
post in our distributed properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.2-all.jar create rethink-sink < rethink-sink.properties

If you switch back to the terminal you started the Connector in you should see the rethink sink being accepted and the
task starting.

Features
--------

The ReThinkDb sink writes records from Kafka to RethinkDb.

The sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to have choose selection of fields
   or all fields written to RethinkDb.
2. Topic to table routing.
3. RowKey selection - Selection of fields to use as the row key, if none specified the topic name, partition and offset is
   used.
4. RethinkDB write modes.
5. Error policies for handling failures.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The ReThink sink supports the following:

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

The sink support two write modes **insert** and **upsert** which map to RethinkDb's conflict policies, **insert** to **ERROR**
and **upsert** to **REPLACE**.

Error Polices
~~~~~~~~~~~~~

The sink has three error policies that determine how failed writes to the target database are handled. The error policies
affect the behaviour of the schema evolution characteristics of the sink. See the schema evolution section for more
information.

**Throw**

Any error on write to the target database will be propagated up and processing is stopped. This is the default
behaviour.

**Noop**

Any error on write to the target database is ignored and processing continues.

.. warning::

    This can lead to missed errors if you don't have adequate monitoring. Data is not lost as it's still in Kafka
    subject to Kafka's retention policy. The sink currently does **not** distinguish between integrity constraint
    violations and or other expections thrown by the driver,

**Retry**

Any error on write to the target database causes the RetryIterable exception to be thrown. This causes the
Kafka connect framework to pause and replay the message. Offsets are not committed. For example, if the table is offline
it will cause a write failure, the message can be replayed. With the Retry policy the issue can be fixed without stopping
the sink.

The length of time the sink will retry can be controlled by using the ``connect.rethink.sink.max.retries`` and the
``connect.rethink.sink.retry.interval``.

Topic Routing
~~~~~~~~~~~~~

The sink supports topic routing that allows mapping the messages from topics to a specific table. For example, map a
topic called "bloomberg_prices" to a table called "prices". This mapping is set in the ``connect.rethink.export.route.query``
option.

Example:

.. sourcecode:: sql

    //Select all
    INSERT INTO table1 SELECT * FROM topic1; INSERT INTO tableA SELECT * FROM topicC

.. tip::

    Explicit mapping of topics to tables is required. If not present the sink will not start and fail validation checks.
    Use AUTOCREATE to have the sink create tables for you based on the topic schema.

Field Selection
~~~~~~~~~~~~~~~

The ReThink sink supports field selection and mapping. This mapping is set in the ``connect.rethink.export.route.query`` option.


Examples:

.. sourcecode:: sql

    //Rename or map columns
    INSERT INTO table1 SELECT lst_price AS price, qty AS quantity FROM topicA

    //Select all
    INSERT INTO table1 SELECT * FROM topic1

.. tip:: Check you mappings to ensure the target columns exist.

Auto Create Tables
~~~~~~~~~~~~~~~~~~

The sink supports auto creation of tables for each topic. This mapping is set in the ``connect.rethink.export.route.query`` option.

A user specified primary can be set in the ``PK`` clause for the ``connect.rethink.export.route.query`` option. Only one
key is supported. If more than one is set only the first is used. If no primary keys are set the default primary key
called ``id`` is used. The value for the default key is the topic name, partition and offset of the records.

.. sourcecode:: sql

    #AutoCreate the target table
    INSERT INTO table1 SELECT * FROM topic AUTOCREATE PK field1

..	note::

    The fields specified as the primary keys must be in the SELECT clause or all fields must be selected

The sink will try and create the table at start up if a schema for the topic is found in the Schema Registry. If no
schema is found the table is created when the first record is received for the topic.

Configurations
--------------

``connect.rethink.sink.host``

Specifies the rethink server.

* Data type : string
* Optional  : no

``connect.rethink.sink.port``

Specifies the rethink server port number.

* Data type : int
* Optional  : yes

``connect.rethink.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.rethink.sink.max.retries``
option. The ``connect.rethink.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high

``connect.rethink.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.rethink.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: high
* Default: 10


``connect.rethink.sink.retry.interval``

The interval, in milliseconds between retries if the sink is using ``connect.rethink.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Default : 60000 (1 minute)

``connect.rethink.sink.batch.size``

Specifies how many records to insert together at one time. If the connect framework provides less records when it is
calling the sink it won't wait to fulfill this value but rather execute it.

* Type : int
* Importance : medium
* Defaults : 3000

``connect.rethink.export.route.query``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming. Fields
to be used as the row key can be set by specifing the ``PK``. The below example uses field1 as the primary key.

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT * FROM TOPIC2 PK field1


Example
~~~~~~~

.. sourcecode:: bash

    name=rethink-sink
    connect.rethink.sink.db=localhost
    connect.rethink.sink.port=28015
    connector.class=com.datamountaineer.streamreactor.connect.rethink.sink.ReThinkSinkConnector
    tasks.max=1
    topics=person_rethink
    connect.rethink.export.route.query=INSERT INTO TABLE1 SELECT * FROM person_rethink

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/2.0.1/schema-registry/docs/api.html#compatibility>`_.

The rethink sink will automatically write and update the rethink table if new fields are added to the source topic,
if fields are removed the Kafka Connect framework will return the default value for this field, dependent of the
compatibility settings of the Schema registry.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
