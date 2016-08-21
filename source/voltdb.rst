Kafka Connect VoltDB
=======================

A Connector and Sink to write events from Kafka to VoltDB.

WIP!!!

Prerequisites
-------------

- Confluent 3.0.0
- VoltDB 6.4
- Java 1.8
- Scala 2.11

Setup
-----

HazelCast Setup
~~~~~~~~~~~~~~~

Download VoltDB from `here <http://learn.voltdb.com/DLSoftwareDownload.html/>`__

Unzip the archive

.. sourcecode:: bash

    tar -xzf voltdb-ent-*.tar.gz

Start VoltDB:

.. sourcecode:: bash

    ➜  cd voltdb-ent-*
    ➜  bin/voltdb create

Confluent Setup
~~~~~~~~~~~~~~~

.. sourcecode:: bash

    #make confluent home folder
    ➜  mkdir confluent

    #download confluent
    ➜  wget http://packages.confluent.io/archive/3.0/confluent-3.0.0-2.11.tar.gz

    #extract archive to confluent folder
    ➜  tar -xvf confluent-3.0.0-2.11.tar.gz -C confluent

    #setup variables
    ➜  export CONFLUENT_HOME=~/confluent/confluent-3.0.0

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

Next we will start the connector in distributed mode. Connect has two modes, standalone where the tasks run on only one host
and distributed mode. Usually you'd run in distributed mode to get fault tolerance and better performance. In distributed mode
you start Connect on multiple hosts and they join together to form a cluster. Connectors which are then submitted are
distributed across the cluster.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Sink Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a file called ``voltdb-sink.properties`` with the contents below:

.. sourcecode:: bash

    name=voltdb-sink
    connector.class=com.datamountaineer.streamreactor.connect.voltdb.VoltSinkConnector
    max.tasks=1
    topics = sink-test
    connect.volt.connection.servers=localhost:9999
    connect.volt.connection.user=
    connect.volt.connection.password=
    connect.volt.export.route.query=INSERT INTO sink-test FROM sink-test

This configuration defines:

1.  The name of the sink.
2.  The sink class.
3.  The max number of tasks the connector is allowed to created.
4.  The topics to read from (Required by framework)
5.  The name of the voltdb host to connect to.
6.  Username to connect as.
7.  The password for the username.
8.  The KCQL statement for topic routing and field selection.


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

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-voltdb-0.1-cp-3.0.all.jar

.. sourcecode:: bash

    ➜  confluent-3.0.0/bin/connect-distributed confluent-3.0.0/etc/schema-registry/connect-avro-distributed.properties

Once the connector has started lets use the kafka-connect-tools cli to post in our distributed properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.2-all.jar create voltdb-sink < voltdb-sink.properties

If you switch back to the terminal you started the Connector in you should see the VoltDb sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    ➜ java -jar build/libs/kafka-connect-cli-0.2-all.jar get voltdb-sink





    #check for running connectors with the CLI
    ➜ java -jar build/libs/kafka-connect-cli-0.2-all.jar ps
    rethink-sink

.. sourcecode:: bash



Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``firstname`` field of type
string a ``lastname`` field of type string, an ``age`` field of type int and a ``salary`` field of type double.

.. sourcecode:: bash

    bin/kafka-avro-console-producer \
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


Now stop the connector.


Features
--------

The sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to select fields written to VoltDB.
2. Topic to table routing.
3. Voltdb write modes, upsert and insert.
4. Error policies for handling failures.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The Voltdb sink supports the following:

.. sourcecode:: bash

    INSERT INTO <table> SELECT <fields> FROM <source topic>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB

    #Upsert mode, select 3 fields and rename from topicB and write to tableB
    UPSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB

This is set in the ``connect.volt.export.route.query`` option.

Error Polices
~~~~~~~~~~~~~

The sink has three error policies that determine how failed writes to the target database are handled. The error policies
affect the behaviour of the schema evolution characteristics of the sink. See the schema evolution section for more
information.

**Throw**

Any error on write to the target database will be propagated up and processing is stopped. This is the default behaviour.

**Noop**

Any error on write to the target database is ignored and processing continues.

.. warning::

    This can lead to missed errors if you don't have adequate monitoring. Data is not lost as it's still in Kafka
    subject to Kafka's retention policy. The sink currently does **not** distinguish between integrity constraint
    violations and or other expections thrown by drivers..

**Retry**

Any error on write to the target database causes the RetryIterable exception to be thrown. This causes the
Kafka connect framework to pause and replay the message. Offsets are not committed. For example, if the table is offline
it will cause a write failure, the message can be replayed. With the Retry policy the issue can be fixed without stopping
the sink.

The length of time the sink will retry can be controlled by using the ``connect.hazelcast.sink.max.retries`` and the
``connect.hazelcast.sink.retry.interval``.

Topic Routing
~~~~~~~~~~~~~

The sink supports topic routing that allows mapping the messages from topics to a specific table. For example, map a
topic called "bloomberg_prices" to a table called "prices". This mapping is set in the ``connect.volt.export.route.query``
option.

Example:

.. sourcecode:: sql

    //Select all
    INSERT INTO table1 SELECT * FROM topic1; INSERT INTO tableA SELECT * FROM topicC

Write Modes
~~~~~~~~~~~

The sink supports both **insert** and **upsert** modes.  This mapping is set in the ``connect.volt.sink.export.mappings`` option.

**Insert**

Insert is the default write mode of the sink.

**Insert Idempotency**

Kafka currently provides at least once delivery semantics. Therefore, this mode may produce errors if unique constraints
have been implemented on the target tables. If the error policy has been set to NOOP then the sink will discard the error
and continue to process, however, it currently makes no attempt to distinguish violation of integrity constraints from other
exceptions such as casting issues.

**Upsert**

The sink support VoltDB upserts which replaces the existing row if a match is found on the primary keys.

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

``connect.volt.export.route.query``

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

The interval, in milliseconds between retries if the sink is using ``connect.volt.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Optional: yes
* Default : 60000 (1 minute)

``connect.volt.sink.batch.size``

Specifies how many records to insert together at one time. If the connect framework provides less records when it is
calling the sink it won't wait to fulfill this value but rather execute it.

* Type : int
* Importance : medium
* Optional: yes
* Defaults : 1000

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/2.0.1/schema-registry/docs/api.html#compatibility>`_.

No schema evolution is handled by the sink yet on changes in the upstream topics. If fields are missing in the
topic which are expected in the store procedures a null value is inserted.


Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO