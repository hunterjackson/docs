Kafka Connect VoltDB
=======================

A Connector and Sink to write events from Kafka to VoltDB. The connector used the built in stored procedures
for inserts and upserts but requires the tables to be pre-created.

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>` - Kafka topic payload field selection is supported, allowing you to select fields written to VoltDB.
2. Topic to table routing via KCQL.
3. Voltdb write modes, upsert and insert via KCQL.
4. Error policies for handling failures.

Prerequisites
-------------

- Confluent 3.0.1
- VoltDB 6.4
- Java 1.8
- Scala 2.11

Setup
-----

VoltDB Setup
~~~~~~~~~~~~~~~

Download VoltDB from `here <http://learn.voltdb.com/DLSoftwareDownload.html/>`__

Unzip the archive

.. sourcecode:: bash

    tar -xzf voltdb-ent-*.tar.gz

Start VoltDB:

.. sourcecode:: bash

    ➜  cd voltdb-ent-*
    ➜  bin/voltdb create

    Build: 6.5 voltdb-6.5-0-gd1fe3fa-local Enterprise Edition
    Initializing VoltDB...

     _    __      ____  ____  ____
    | |  / /___  / / /_/ __ \/ __ )
    | | / / __ \/ / __/ / / / __  |
    | |/ / /_/ / / /_/ /_/ / /_/ /
    |___/\____/_/\__/_____/_____/

    --------------------------------

    Connecting to VoltDB cluster as the leader...
    Host id of this node is: 0
    Starting VoltDB with trial license. License expires on Sep 11, 2016.
    Initializing the database and command logs. This may take a moment...
    WARN: This is not a highly available cluster. K-Safety is set to 0.

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Sink Connector QuickStart
-------------------------

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and updating the configuration. We have developed
a Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

Create Voltdb Table
~~~~~~~~~~~~~~~~~~~

At present the Sink doesn't support auto creation of tables so we need to login to VoltDb to create one. In the directory
you extracted Voltdb start the ``sqlcmd`` shell and enter the following DDL statement. This creates a table called person.

.. sourcecode:: sql

   create table person(firstname varchar(128), lastname varchar(128), age int, salary float, primary key (firstname, lastname));

.. sourcecode:: bash

    ➜  bin ./sqlcmd
    SQL Command :: localhost:21212
    1> create table person(firstname varchar(128), lastname varchar(128), age int, salary float, primary key (firstname, lastname));
    Command succeeded.
    2>

Starting the Connector
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running the ``start-connect.sh`` script in the ``bin`` folder.

.. sourcecode:: bash

    ➜ bin/start-connect.sh

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for VoltDB.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create voltdb-sink < conf/quickstarts/voltdb-sink.properties

    #Connector `voltdb-sink`:
    name=voltdb-sink
    connector.class=com.datamountaineer.streamreactor.connect.voltdb.VoltSinkConnector
    max.tasks=1
    topics=sink-test
    connect.volt.connection.servers=localhost:21212
    connect.volt.sink.kcql=INSERT INTO person SELECT * FROM sink-test
    connect.volt.connection.password=
    connect.volt.connection.user=
    #task ids:

The ``voltdb-sink.properties`` file defines:

1.  The name of the sink.
2.  The Sink class.
3.  The max number of tasks the connector is allowed to created.
4.  The topics to read from (Required by framework)
5.  The name of the voltdb host to connect to.
6.  Username to connect as.
7.  The password for the username.
8.  :ref:`The KCQL routing querying. <kcql>`

If you switch back to the terminal you started the Connector in you should see the VoltDb Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    voltdb-sink

.. sourcecode:: bash

    [2016-08-21 20:31:36,398] INFO Finished starting connectors and tasks (org.apache.kafka.connect.runtime.distributed.DistributedHerder:769)
    [2016-08-21 20:31:36,406] INFO
     _____                                                    _
    (____ \       _                                 _        (_)
     _   \ \ ____| |_  ____ ____   ___  _   _ ____ | |_  ____ _ ____   ____ ____  ____
    | |   | / _  |  _)/ _  |    \ / _ \| | | |  _ \|  _)/ _  | |  _ \ / _  ) _  )/ ___)
    | |__/ ( ( | | |_( ( | | | | | |_| | |_| | | | | |_( ( | | | | | ( (/ ( (/ /| |
    |_____/ \_||_|\___)_||_|_|_|_|\___/ \____|_| |_|\___)_||_|_|_| |_|\____)____)_|
                                        by Stefan Bocutiu
     _    _     _      _____   _           _    _       _
    | |  | |   | |_   (____ \ | |         | |  (_)     | |
    | |  | |__ | | |_  _   \ \| | _        \ \  _ ____ | |  _
     \ \/ / _ \| |  _)| |   | | || \        \ \| |  _ \| | / )
      \  / |_| | | |__| |__/ /| |_) )   _____) ) | | | | |< (
    \/ \___/|_|\___)_____/ |____/   (______/|_|_| |_|_| \_)
      (com.datamountaineer.streamreactor.connect.voltdb.VoltSinkTask:44)
    [2016-08-21 20:31:36,407] INFO VoltSinkConfig values:
        connect.volt.error.policy = THROW
        connect.volt.retry.interval = 60000
        connect.volt.sink.kcql = INSERT INTO person SELECT * FROM sink-test
        connect.volt.max.retires = 20
        connect.volt.connection.servers = localhost:21212
        connect.volt.connection.user =
        connect.volt.connection.password =
     (com.datamountaineer.streamreactor.connect.voltdb.config.VoltSinkConfig:178)
    [2016-08-21 20:31:36,501] INFO Settings:com.datamountaineer.streamreactor.connect.voltdb.config.VoltSettings$@34c34c3e (com.datamountaineer.streamreactor.connect.voltdb.VoltSinkTask:71)
    [2016-08-21 20:31:36,565] INFO Connecting to VoltDB... (com.datamountaineer.streamreactor.connect.voltdb.writers.VoltConnectionConnectFn$:28)
    [2016-08-21 20:31:36,636] INFO Connected to VoltDB node at: localhost:21212 (com.datamountaineer.streamreactor.connect.voltdb.writers.VoltConnectionConnectFn$:46)


Test Records
^^^^^^^^^^^^

.. hint::

    If your input topic doesn't match the target use Kafka Streams to transform in realtime the input. Also checkout the
    `Plumber <https://github.com/rollulus/kafka-streams-plumber>`__, which allows you to inject a Lua script into
    `Kafka Streams <http://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple>`__ to do this,
    no Java or Scala required!

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``firstname`` field of type
string a ``lastname`` field of type string, an ``age`` field of type int and a ``salary`` field of type double.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic sink-test \
      --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.voltdb"
      ,"fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"salary","type":"double"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}

Check for records in VoltDb
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this:

.. sourcecode:: bash

    [2016-08-21 20:41:25,361] INFO Writing complete (com.datamountaineer.streamreactor.connect.voltdb.writers.VoltDbWriter:61)
    [2016-08-21 20:41:25,362] INFO Records handled (com.datamountaineer.streamreactor.connect.voltdb.VoltSinkTask:86)

In Voltdb sqlcmd terminal

.. sourcecode:: sql

    SELECT * FROM PERSON;

    FIRSTNAME  LASTNAME  AGE  SALARY
    ---------- --------- ---- -------
    John       Smith       30  4830.0

    (Returned 1 rows in 0.01s)

Now stop the connector.


Features
--------

The Sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to select fields written to VoltDB.
2. Topic to table routing.
3. Voltdb write modes, upsert and insert.
4. Error policies for handling failures.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L**, :ref:`KCQL <kcql>` allows for routing and mapping using a SQL like syntax,
consolidating typically features in to one configuration option.


The Voltdb Sink supports the following:

.. sourcecode:: bash

    INSERT INTO <table> SELECT <fields> FROM <source topic>
    UPSERT INTO <table> SELECT <fields> FROM <source topic>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB

    #Upsert mode, select 3 fields and rename from topicB and write to tableB
    UPSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB

This is set in the ``connect.volt.sink.kcql`` option.

Error Polices
~~~~~~~~~~~~~

The Sink has three error policies that determine how failed writes to the target database are handled. The error policies
affect the behaviour of the schema evolution characteristics of the sink. See the schema evolution section for more
information.

**Throw**

Any error on write to the target database will be propagated up and processing is stopped. This is the default behaviour.

**Noop**

Any error on write to the target database is ignored and processing continues.

.. warning::

    This can lead to missed errors if you don't have adequate monitoring. Data is not lost as it's still in Kafka
    subject to Kafka's retention policy. The Sink currently does **not** distinguish between integrity constraint
    violations and or other expections thrown by drivers..

**Retry**

Any error on write to the target database causes the RetryIterable exception to be thrown. This causes the
Kafka connect framework to pause and replay the message. Offsets are not committed. For example, if the table is offline
it will cause a write failure, the message can be replayed. With the Retry policy the issue can be fixed without stopping
the sink.

The length of time the Sink will retry can be controlled by using the ``connect.hazelcast.sink.max.retries`` and the
``connect.hazelcast.sink.retry.interval``.

Topic Routing
~~~~~~~~~~~~~

The Sink supports topic routing that allows mapping the messages from topics to a specific table. For example, map a
topic called "bloomberg_prices" to a table called "prices". This mapping is set in the ``connect.volt.sink.kcql``
option.

Example:

.. sourcecode:: sql

    //Select all
    INSERT INTO table1 SELECT * FROM topic1; INSERT INTO tableA SELECT * FROM topicC

Write Modes
~~~~~~~~~~~

The Sink supports both **insert** and **upsert** modes.  This mapping is set in the ``connect.volt.sink.export.mappings`` option.

**Insert**

Insert is the default write mode of the sink.

**Insert Idempotency**

Kafka currently provides at least once delivery semantics. Therefore, this mode may produce errors if unique constraints
have been implemented on the target tables. If the error policy has been set to NOOP then the Sink will discard the error
and continue to process, however, it currently makes no attempt to distinguish violation of integrity constraints from other
exceptions such as casting issues.

**Upsert**

The Sink support VoltDB upserts which replaces the existing row if a match is found on the primary keys.

**Upsert Idempotency**

Kafka currently provides at least once delivery semantics and order is a guaranteed within partitions.

This mode will, if the same record is delivered twice to the sink, result in an idempotent write. The existing record
will be updated with the values of the second which are the same.

If records are delivered with the same field or group of fields that are used as the primary key on the target table,
but different values, the existing record in the target table will be updated.

Since records are delivered in the order they were written per partition the write is idempotent on failure or restart.
Redelivery produces the same result.

Configurations
--------------

``connect.volt.sink.kcql``

KCQL expression describing field selection and routes.

* Data type : string
* Importance : high
* Optional  : no

``connect.volt.connection.servers``

Comma separated server[:port].

* Type : string
* Importance : high
* Optional  : no

``connect.volt.connection.user``

The user to connect to the volt database.

* Type : string
* Importance : high
* Optional  : no

``connect.volt.connection.password``

The password for the voltdb user.

* Type : string
* Importance : high
* Optional  : no

``connect.volt.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.volt.sink.max.retries``
option. The ``connect.volt.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high
* Default: ``throw``

``connect.volt.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.volt.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: medium
* Optional: yes
* Default: 10


``connect.volt.sink.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.volt.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Optional: yes
* Default : 60000 (1 minute)

``connect.volt.sink.batch.size``

Specifies how many records to insert together at one time. If the connect framework provides less records when it is
calling the Sink it won't wait to fulfill this value but rather execute it.

* Type : int
* Importance : medium
* Optional: yes
* Defaults : 1000

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.

No schema evolution is handled by the Sink yet on changes in the upstream topics.


Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO