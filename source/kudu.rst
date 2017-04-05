Kafka Connect Kudu
===================

A Connector and Sink to write events from Kafka to kudu.The connector takes the value from the Kafka Connect SinkRecords
and inserts a new entry to Kudu.

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>` - Kafka topic payload field selection is supported, allowing you to select fields written to Kudu.
2. Topic to table routing via KCQL.
3. Auto table create with DISTRIBUTE BY partition strategy via KCQL.
4. Auto evolution of tables via KCQL.
5. Error policies for handling failures.

Prerequisites
-------------

- Confluent 3.2
- Kudu 0.8
- Java 1.8
- Scala 2.11

Setup
-----

Kudu Setup
~~~~~~~~~~~

Download and check Kudu QuickStart VM starts up.

.. sourcecode:: bash

    curl -s https://raw.githubusercontent.com/cloudera/kudu-examples/master/demo-vm-setup/bootstrap.sh | bash

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Sink Connector QuickStart
-------------------------

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and updating the configuration. We have developed
a Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.


Kudu Table
~~~~~~~~~~

Lets create a table in Kudu via Impala. The Sink does support auto creation of tables but they are not sync'd yet with Impala.

.. sourcecode:: bash

    #demo/demo
    ssh demo@quickstart -t impala-shell

    CREATE TABLE default.kudu_test (id INT,random_field STRING  )
    TBLPROPERTIES ('kudu.master_addresses'='127.0.0.1', 'kudu.key_columns'='id',
    'kudu.table_name'='kudu_test', 'transient_lastDdlTime'='1456744118',
    'storage_handler'='com.cloudera.kudu.hive.KuduStorageHandler')
    exit;

.. note:: The Sink will fail to start if the tables matching the topics do not already exist and the Sink is not in auto create mode.


Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running the ``start-connect.sh`` script in the ``bin`` folder.

.. sourcecode:: bash

    ➜ bin/start-connect.sh

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for Kudu.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create kudu-sink < conf/kudu-sink.properties
    #Connector name=kudu-sink
    connector.class=com.datamountaineer.streamreactor.connect.kudu.KuduSinkConnector
    tasks.max=1
    connect.kudu.master=quickstart
    connect.kudu.sink.kcql = INSERT INTO kudu_test SELECT * FROM kudu-test
    topics=kudu_test
    #task ids: 0


The ``kudu-sink.properties`` file defines:

1.  The name of the sink.
2.  The Sink class.
3.  The max number of tasks the connector is allowed to created. Should not be greater than the number of partitions in
    the Source topics otherwise tasks will be idle.
4.  The Kudu master host.
5.  :ref:`The KCQL routing querying. <kcql>`
6.  The Source kafka topics to take events from.

If you switch back to the terminal you started the Connector in you should see the Kudu Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    kudu-sink

.. sourcecode:: bash

    [2016-05-08 22:00:20,823] INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      /  / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
           __ __          __      _____ _       __
          / //_/_  ______/ /_  __/ ___/(_)___  / /__
         / ,< / / / / __  / / / /\__ \/ / __ \/ //_/
        / /| / /_/ / /_/ / /_/ /___/ / / / / / ,<
       /_/ |_\__,_/\__,_/\__,_//____/_/_/ /_/_/|_|


    by Andrew Stevenson
           (com.datamountaineer.streamreactor.connect.kudu.KuduSinkTask:37)
    [2016-05-08 22:00:20,823] INFO KuduSinkConfig values:
        connect.kudu.master = quickstart
     (com.datamountaineer.streamreactor.connect.kudu.KuduSinkConfig:165)
    [2016-05-08 22:00:20,824] INFO Connecting to Kudu Master at quickstart (com.datamountaineer.streamreactor.connect.kudu.KuduWriter$:33)
    [2016-05-08 22:00:20,875] INFO Initialising Kudu writer (com.datamountaineer.streamreactor.connect.kudu.KuduWriter:40)
    [2016-05-08 22:00:20,892] INFO Assigned topics  (com.datamountaineer.streamreactor.connect.kudu.KuduWriter:42)
    [2016-05-08 22:00:20,904] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@b60ba7b finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)

Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``id`` field of type int
and a ``random_field`` of type string.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
     --broker-list localhost:9092 --topic kudu-test \
     --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"id","type":"int"},
    {"name":"random_field", "type": "string"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"id": 999, "random_field": "foo"}
    {"id": 888, "random_field": "bar"}

Check for records in Kudu
~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this:

.. sourcecode:: bash

    [2016-05-08 22:09:22,065] INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
           __ __          __      _____ _       __
          / //_/_  ______/ /_  __/ ___/(_)___  / /__
         / ,< / / / / __  / / / /\__ \/ / __ \/ //_/
        / /| / /_/ / /_/ / /_/ /___/ / / / / / ,<
       /_/ |_\__,_/\__,_/\__,_//____/_/_/ /_/_/|_|


    by Andrew Stevenson
           (com.datamountaineer.streamreactor.connect.kudu.KuduSinkTask:37)
    [2016-05-08 22:09:22,065] INFO KuduSinkConfig values:
        connect.kudu.master = quickstart
     (com.datamountaineer.streamreactor.connect.kudu.KuduSinkConfig:165)
    [2016-05-08 22:09:22,066] INFO Connecting to Kudu Master at quickstart (com.datamountaineer.streamreactor.connect.kudu.KuduWriter$:33)
    [2016-05-08 22:09:22,116] INFO Initialising Kudu writer (com.datamountaineer.streamreactor.connect.kudu.KuduWriter:40)
    [2016-05-08 22:09:22,134] INFO Assigned topics kudu_test (com.datamountaineer.streamreactor.connect.kudu.KuduWriter:42)
    [2016-05-08 22:09:22,148] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@68496440 finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)
    [2016-05-08 22:09:22,476] INFO Written 2 for kudu_test (com.datamountaineer.streamreactor.connect.kudu.KuduWriter:90)


In Kudu:

.. sourcecode:: bash

    #demo/demo
    ssh demo@quickstart -t impala-shell

    SELECT * FROM kudu_test;

    Query: select * FROM kudu_test
    +-----+--------------+
    | id  | random_field |
    +-----+--------------+
    | 888 | bar          |
    | 999 | foo          |
    +-----+--------------+
    Fetched 2 row(s) in 0.14s

Now stop the connector.

Features
--------

The Kudu Sink writes records from Kafka to Kudu.

The Sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to select fields written to Kudu.
2. Topic to table routing.
3. Auto table create with DISTRIBUTE BY partition strategy.
4. Auto evolution of tables.
5. Error policies for handling failures.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The Kudu Sink supports the following:

.. sourcecode:: bash

    <write mode> INTO <target table> SELECT <fields> FROM <source topic> <AUTOCREATE> <AUTOEVOLVE> <DISTRIBUTEBY> <PK_FIELDS> INTO <NBR_OF_BUCKETS> BUCKETS

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB

    #Upsert mode, select all fields from topicC, auto create tableC and auto evolve, use field1 and field2 as the primary keys
    UPSERT INTO tableC SELECT * FROM topicC AUTOCREATE  DISTRIBUTEBY field1, field2 INTO 10 BUCKETS AUTOEVOLVE

Error Polices
~~~~~~~~~~~~~

The Sink has three error policies that determine how failed writes to the target database are handled. The error policies
affect the behaviour of the schema evolution characteristics of the Sink. See the schema evolution section for more
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

The length of time the Sink will retry can be controlled by using the ``connect.kudu.sink.max.retries`` and the
``connect.kudu.sink.retry.interval``.

Auto conversion of Connect records to Kudu
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The Sink automatically converts incoming Connect records to Kudu inserts or upserts.

Topic Routing
~~~~~~~~~~~~~

The Sink supports topic routing that allows mapping the messages from topics to a specific table. For example, map a
topic called "bloomberg_prices" to a table called "prices". This mapping is set in the ``connect.kudu.sink.kcql``
option.

Example:

.. sourcecode:: sql

    //Select all
    INSERT INTO table1 SELECT * FROM topic1; INSERT INTO tableA SELECT * FROM topicC

Field Selection
~~~~~~~~~~~~~~~

The Kudu Sink supports field selection and mapping. This mapping is set in the ``connect.kudu.sink.kcql`` option.

Examples:

.. sourcecode:: sql

    //Rename or map columns
    INSERT INTO table1 SELECT lst_price AS price, qty AS quantity FROM topicA

    //Select all
    INSERT INTO table1 SELECT * FROM topic1

.. tip:: Check you mappings to ensure the target columns exist.

.. warning::

    Field selection disables evolving the target table if the upstream schema in the Kafka topic changes. By specifying
    field mappings it is assumed the user is not interested in new upstream fields. For example they may be tapping into a
    pipeline for a Kafka stream job and not be intended as the final recipient of the stream.

    If you chose field selection you must include the primary key fields otherwise the insert will fail.

Write Modes
~~~~~~~~~~~

The Sink supports both **insert** and **upsert** modes.  This mapping is set in the ``connect.kudu.sink.export.mappings`` option.

**Insert**

Insert is the default write mode of the sink.

**Insert Idempotency**

Kafka currently provides at least once delivery semantics. Therefore, this mode may produce errors if unique constraints
have been implemented on the target tables. If the error policy has been set to NOOP then the Sink will discard the error
and continue to process, however, it currently makes no attempt to distinguish violation of integrity constraints from other
exceptions such as casting issues.

**Upsert**

The Sink support Kudu upserts which replaces the existing row if a match is found on the primary keys.

**Upsert Idempotency**

Kafka currently provides at least once delivery semantics and order is a guaranteed within partitions.

This mode will, if the same record is delivered twice to the sink, result in an idempotent write. The existing record
will be updated with the values of the second which are the same.

If records are delivered with the same field or group of fields that are used as the primary key on the target table,
but different values, the existing record in the target table will be updated.

Since records are delivered in the order they were written per partition the write is idempotent on failure or restart.
Redelivery produces the same result.

Auto Create Tables
~~~~~~~~~~~~~~~~~~

The Sink supports auto creation of tables for each topic. This mapping is set in the ``connect.kudu.sink.kcql`` option.

Primary keys are set in the ``DISTRIBUTEBY`` clause of the ``connect.kudu.sink.kcql``.

Tables are created with the Kudu hash partition strategy. The number of buckets must be specified in the ``kcql``
statement.

.. sourcecode:: sql

    #AutoCreate the target table
    INSERT INTO table1 SELECT * FROM topic AUTOCREATE DISTRIBUTEBY field1, field2 INTO 10 BUCKETS

..	note::

    The fields specified as the primary keys (distributeby) must be in the SELECT clause or all fields must be selected

The Sink will try and create the table at start up if a schema for the topic is found in the Schema Registry. If no
schema is found the table is created when the first record is received for the topic.

Auto Evolve Tables
~~~~~~~~~~~~~~~~~~

The Sink supports auto evolution of tables for each topic. This mapping is set in the ``connect.kudu.sink.kcql`` option.
When set the Sink will identify new schemas for each topic based on the schema version from the Schema registry. New columns
will be identified and an alter table DDL statement issued against Kudu.

Schema evolution can occur upstream, for example any new fields or change in data type in the schema of the topic, or
downstream DDLs on the database.

Upstream changes must follow the schema evolution rules laid out in the Schema Registry. This Sink only supports BACKWARD
and FULLY compatible schemas. If new fields are added the Sink will attempt to perform a ALTER table DDL statement against
the target table to add columns. All columns added to the target table are set as nullable.

Fields cannot be deleted upstream. Fields should be of Avro union type [null, <dataType>] with a default set. This allows
the Sink to either retrieve the default value or null. The Sink is not aware that the field has been deleted
as a value is always supplied to it.

.. warning::

    If a upstream field is removed and the topic is not following the Schema Registry's evolution rules, i.e. not full
    or backwards compatible, any errors will default to the error policy.

Downstream changes are handled by the sink. If columns are removed, the mapped fields from the topic are ignored. If
columns are added, we attempt to find a matching field by name in the topic.

Data Type Mappings
~~~~~~~~~~~~~~~~~~

+------------------+------------------+
| Connect Type     | Kudu Data Type   |
+==================+==================+
| INT8             | INT8             |
+------------------+------------------+
| INT16            | INT16            |
+------------------+------------------+
| INT32            | INT32            |
+------------------+------------------+
| INT64            | INT64            |
+------------------+------------------+
| BOOLEAN          | BOOLEAN          |
+------------------+------------------+
| FLOAT32          | FLOAT            |
+------------------+------------------+
| FLOAT64          | FLOAT            |
+------------------+------------------+
| BYTES            | BINARY           |
+------------------+------------------+

Configurations
--------------

``connect.kudu.master``

Specifies a Kudu server.

* Data type : string
* Importance: high
* Optional  : no

``connect.kudu.sink.kcql``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming.

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT field1, field2, field3 as renamedField FROM TOPIC2


* Data Type: string
* Importance: high
* Optional : no

``connect.kudu.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.kudu.sink.max.retries``
option. The ``connect.kudu.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high
* Optional : yes
* Default: RETRY

``connect.kudu.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.kudu.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: medium
* Optional : yes
* Default: 10


``connect.kudu.sink.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.kudu.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Optional : yes
* Default : 60000 (1 minute)

``connect.kudu.sink.schema.registry.url``

The url for the Schema registry. This is used to retrieve the latest schema for table creation.

* Type : string
* Importance : high
* Optional : yes
* Default : http://localhost:8081

``connect.kudu.sink.batch.size``

Specifies how many records to insert together at one time. If the connect framework provides less records when it is
calling the Sink it won't wait to fulfill this value but rather execute it.

* Type : int
* Importance : medium
* Optional : yes
* Defaults : 3000

Example
~~~~~~~

.. sourcecode:: bash

    name=kudu-sink
    connector.class=com.datamountaineer.streamreactor.connect.kudu.KuduSinkConnector
    tasks.max=1
    connect.kudu.master=quickstart
    connect.kudu.sink.kcql=INSERT INTO kudu_test SELECT * FROM kudu_test AUTOCREATE DISTRIBUTEBY id INTO 5 BUCKETS
    topics=kudu_test
    connect.kudu.sink.schema.registry.url=http://myhost:8081

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
