Kafka Connect Cassandra Sink
============================

The Cassandra Sink allows you to write events from Kafka to Cassandra. The connector converts the value from the Kafka
Connect SinkRecords to Json and uses Cassandra's JSON insert functionality to insert the rows. The task expects pre-created
tables in Cassandra.

.. note:: The table and keyspace must be created before hand!
.. note:: If the target table has TimeUUID fields the payload string the corresponding field in Kafka must be a UUID.

Prerequisites
-------------

-  Cassandra 2.2.4
-  Confluent 3.0.0
-  Java 1.8
-  Scala 2.11

Setup
-----

Before we can do anything, including the QuickStart we need to install
Cassandra and the Confluent platform.

Cassandra Setup
~~~~~~~~~~~~~~~

First download and install Cassandra if you don't have a compatible
cluster available.

.. sourcecode:: bash

    #make a folder for cassandra
    mkdir cassandra

    #Download Cassandra
    wget http://apache.cs.uu.nl/cassandra/3.5/apache-cassandra-3.5-bin.tar.gz

    #extract archive to cassandra folder
    tar -xvf apache-cassandra-3.5-bin.tar.gz -C cassandra

    #Set up environment variables
    export CASSANDRA_HOME=~/cassandra/apache-cassandra-3.5-bin
    export PATH=$PATH:$CASSANDRA_HOME/bin

    #Start Cassandra
    sudo sh ~/cassandra/bin/cassandra


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
    bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    bin/kafka-server-start etc/kafka/server.properties &
    bin/schema-registry-start etc/schema-registry/schema-registry.properties &

Build the Connector and CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The prebuilt CLI jar can be taken from `here <https://github.com/datamountaineer/kafka-connect-tools/releases>`__
or from `Maven <http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22kafka-connect-cli%22>`__

If you want to build the connector, clone the repo and build the jar.

.. sourcecode:: bash

    ##Build the connectors
    git clone https://github.com/datamountaineer/stream-reactor
    cd stream-reactor
    gradle fatJar

    ##Build the CLI for interacting with Kafka connectors
    git clone https://github.com/datamountaineer/kafka-connect-tools
    cd kafka-connect-tools
    gradle fatJar


Sink Connector QuickStart
~~~~~~~~~~~~~~~~~~~~~~~~~

Next we will start the connector in distributed mode. Connect has two modes, standalone where the tasks run on only one host
and distributed mode. Usually you'd run in distributed mode to get fault tolerance and better performance. In distributed mode
you start Connect on multiple hosts and they join together to form a cluster. Connectors which are then submitted are
distributed across the cluster.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Test data
^^^^^^^^^

The sink currently expects precreated tables and keyspaces. So lets create a keyspace and table in Cassandra via the CQL
shell first.

Once you have installed and started Cassandra create a table to write records to. This snippet creates a table called
orders to hold fictional orders on a trading platform.

Start the Cassandra cql shell

.. sourcecode:: bash

    ➜  bin ./cqlsh
    Connected to Test Cluster at 127.0.0.1:9042.
    [cqlsh 5.0.1 | Cassandra 3.0.2 | CQL spec 3.3.1 | Native protocol v4]
    Use HELP for help.
    cqlsh>

Execute the following to create the keyspace and table:

.. sourcecode:: sql

    CREATE KEYSPACE demo WITH REPLICATION = {'class' : 'SimpleStrategy', 'replication_factor' : 3};
    use demo;

    create table orders (id int, created varchar, product varchar, qty int, price float, PRIMARY KEY (id, created))
    WITH CLUSTERING ORDER BY (created asc);

Sink Connector Configuration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Create a file called cassandra-sink-distributed-orders.properties with contents below.

.. sourcecode:: bash

    name=cassandra-sink-orders
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.cassandra.export.route.query=INSERT INTO orders SELECT * FROM orders-topic
    connect.cassandra.contact.points=localhost
    connect.cassandra.port=9042
    connect.cassandra.key.space=demo
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra

.. note:: All tables must be in the same keyspace.

Starting the Sink Connector (Distributed)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We will start Kafka Connect in distributed mode.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-cassandra-0.1-cp-3.0.0-all.jar

.. sourcecode:: bash

    ➜  confluent-3.0.0/bin/connect-distributed etc/schema-registry/connect-avro-distributed.properties

Once the connector has started lets use the kafka-connect-tools cli to post in our distributed properties file. You can
download the CLI from `here <http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22kafka-connect-cli%22>`__

.. sourcecode:: bash

    ➜  java -jar kafka-connect-cli-0.5-all.jar create cassandra-sink-orders < cassandra-sink-distributed-orders.properties

    #Connector `cassandra-sink-orders`:
    name=cassandra-sink-orders
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.cassandra.export.route.query=INSERT INTO orders SELECT * FROM orders-topic
    connect.cassandra.contact.points=localhost
    connect.cassandra.port=9042
    connect.cassandra.key.space=demo
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra
    #task ids: 0

If you switch back to the terminal you started the Connector in you should see the Cassandra sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ java -jar build/libs/kafka-connect-cli-0.5-all.jar ps
    cassandra-sink


.. sourcecode:: bash

    [2016-05-06 13:52:28,178] INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
           ______                                __           _____ _       __
          / ____/___ _______________ _____  ____/ /________ _/ ___/(_)___  / /__
         / /   / __ `/ ___/ ___/ __ `/ __ \/ __  / ___/ __ `/\__ \/ / __ \/ //_/
        / /___/ /_/ (__  |__  ) /_/ / / / / /_/ / /  / /_/ /___/ / / / / / ,<
        \____/\__,_/____/____/\__,_/_/ /_/\__,_/_/   \__,_//____/_/_/ /_/_/|_|

     By Andrew Stevenson. (com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraSinkTask:50)
    [2016-05-06 13:52:28,179] INFO Attempting to connect to Cassandra cluster at localhost and create keyspace demo. (com.datamountaineer.streamreactor.connect.cassandra.CassandraConnection$:49)
    [2016-05-06 13:52:28,187] WARN You listed localhost/0:0:0:0:0:0:0:1:9042 in your contact points, but it wasn't found in the control host's system.peers at startup (com.datastax.driver.core.Cluster:2105)
    [2016-05-06 13:52:28,211] INFO Using data-center name 'datacenter1' for DCAwareRoundRobinPolicy (if this is incorrect, please provide the correct datacenter name with DCAwareRoundRobinPolicy constructor) (com.datastax.driver.core.policies.DCAwareRoundRobinPolicy:95)
    [2016-05-06 13:52:28,211] INFO New Cassandra host localhost/127.0.0.1:9042 added (com.datastax.driver.core.Cluster:1475)
    [2016-05-06 13:52:28,290] INFO Initialising Cassandra writer. (com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraJsonWriter:40)
    [2016-05-06 13:52:28,295] INFO Preparing statements for orders-topic. (com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraJsonWriter:62)
    [2016-05-06 13:52:28,305] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@37e65d57 finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)
    [2016-05-06 13:52:28,331] INFO Source task Thread[WorkerSourceTask-cassandra-source-orders-0,5,main] finished initialization and start (org.apache.kafka.connect.runtime.WorkerSourceTask:342)


Test Records
^^^^^^^^^^^^

Now we need to put some records it to the orders-topic. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema matches the table created earlier.

.. hint::

    If your input topic doesn't match the target use Kafka Streams to transform in realtime the input. Also checkout the
    `Plumber <https://github.com/rollulus/kafka-streams-plumber>`__, which allows you to inject a Lua script into
    `Kafka Streams <http://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple>`__ to do this,
    no Java or Scala required!

.. sourcecode:: bash

    bin/kafka-avro-console-producer \
     --broker-list localhost:9092 --topic orders-topic \
     --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"id","type":"int"},
    {"name":"created", "type": "string"}, {"name":"product", "type": "string"}, {"name":"price", "type": "double"}]}'

Now the producer is waiting for input. Paste in the following (each on a line separately):

.. sourcecode:: bash

    {"id": 1, "created": "2016-05-06 13:53:00", "product": "OP-DAX-P-20150201-95.7", "price": 94.2}
    {"id": 2, "created": "2016-05-06 13:54:00", "product": "OP-DAX-C-20150201-100", "price": 99.5}
    {"id": 3, "created": "2016-05-06 13:55:00", "product": "FU-DATAMOUNTAINEER-20150201-100", "price": 10000}
    {"id": 4, "created": "2016-05-06 13:56:00", "product": "FU-KOSPI-C-20150201-100", "price": 150}

Now if we check the logs of the connector we should see 2 records being inserted to Elastic Search:

.. sourcecode:: bash

    [2016-05-06 13:55:10,368] INFO Setting newly assigned partitions [orders-topic-0] for group connect-cassandra-sink-1 (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:219)
    [2016-05-06 13:55:16,423] INFO Received 4 records. (com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraJsonWriter:96)
    [2016-05-06 13:55:16,484] INFO Processed 4 records. (com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraJsonWriter:138)

.. sourcecode:: bash

    use demo;
    SELECT * FROM orders_write_back;

     id | created             | price | product                           | qty
    ----+---------------------+-------+-----------------------------------+-----
      1 | 2016-05-06 13:53:00 |  94.2 |            OP-DAX-P-20150201-95.7 | 100
      2 | 2016-05-06 13:54:00 |  99.5 |             OP-DAX-C-20150201-100 | 100
      3 | 2016-05-06 13:55:00 | 10000 |   FU-DATAMOUNTAINEER-20150201-100 | 500
      4 | 2016-05-06 13:56:00 |   150 |           FU-KOSPI-C-20150201-100 | 200

    (4 rows)

Bingo, our 4 rows!

Features
--------

The sink connector uses Cassandra's `JSON <http://www.datastax.com/dev/blog/whats-new-in-cassandra-2-2-json-support>`__
insert functionality. The SinkRecord from Kafka Connect is converted to JSON and feed into the prepared statements for
inserting into Cassandra.

See Cassandra's `documentation <http://cassandra.apache.org/doc/cql3/CQL-2.2.html#insertJson>`__ for type mapping.

The sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to have choose selection of fields
   or all fields written to Cassandra.
2. Topic to table routing.
3. Error policies for handling failures.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The Cassandra sink supports the following:

.. sourcecode:: bash

    INSERT INTO <target table> SELECT <fields> FROM <source topic>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB


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
    violations and or other expections thrown by drivers..

**Retry**

Any error on write to the target database causes the RetryIterable exception to be thrown. This causes the
Kafka connect framework to pause and replay the message. Offsets are not committed. For example, if the table is offline
it will cause a write failure, the message can be replayed. With the Retry policy the issue can be fixed without stopping
the sink.

The length of time the sink will retry can be controlled by using the ``connect.cassandra.sink.max.retries`` and the
``connect.cassandra.sink.retry.interval``.

Topic Routing
^^^^^^^^^^^^^

The sink supports topic routing that allows mapping the messages from topics to a specific table. For example map
a topic called "bloomberg_prices" to a table called "prices". This mapping is set in the
``connect.cassandra.export.route.query`` option.

Field Selection
^^^^^^^^^^^^^^^

The sink supports selecting fields from the source topic or selecting all fields and mapping of these fields to columns
in the target table. For example, map a field called "qty"  in a topic to a column called "quantity" in the target
table.

All fields can be selected by using "*" in the field part of ``connect.cassandra.export.route.query``.

Leaving the column name empty means trying to map to a column in the target table with the same name as the field in the
source topic.

Configurations
--------------

Configurations common to both sink and source are:

``connect.cassandra.contact.points``

Contact points (hosts) in Cassandra cluster.

* Data type: string
* Optional : no

``connect.cassandra.key.space``

Key space the tables to write belong to.

* Data type: string
* Optional : no

``connect.cassandra.port``

Port for the native Java driver.

* Data type: int
* Optional : yes
* Default : 9042


``connect.cassandra.username``

Username to connect to Cassandra with if ``connect.cassandra.authentication.mode`` is set to *username_password*.

* Data type: string
* Optional : yes

``connect.cassandra.password``

Password to connect to Cassandra with if ``connect.cassandra.authentication.mode`` is set to *username_password*.

* Data type: string
* Optional : yes

``connect.cassandra.ssl.enabled``

Enables SSL communication against SSL enable Cassandra cluster.

* Data type: boolean
* Optional : yes
* Default : false

``connect.cassandra.trust.store.password``

Password for truststore.

* Data type: string
* Optional : yes

``connect.cassandra.key.store.path``

Path to truststore.

* Data type: string
* Optional : yes

``connect.cassandra.key.store.password``

Password for key store.

* Data type: string
* Optional : yes

``connect.cassandra.ssl.client.cert.auth``

Path to keystore.

* Data type: string
* Optional : yes


``connect.cassandra.export.route.query``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming.

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT field1, field2, field3 as renamedField FROM TOPIC2


* Data Type: string
* Optional : no

``connect.cassandra.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.cassandra.sink.max.retries``
option. The ``connect.cassandra.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high
* Default: ``throw``

``connect.cassandra.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.cassandra.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: high
* Default: 10

``connect.cassandra.sink.retry.interval``

The interval, in milliseconds between retries if the sink is using ``connect.cassandra.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Default : 60000 (1 minute)

Example
~~~~~~~

.. sourcecode:: bash

    name=cassandra-sink-orders
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.cassandra.export.route.query = INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT field1,
    field2, field3 as renamedField FROM TOPIC2
    connect.cassandra.contact.points=localhost
    connect.cassandra.port=9042
    connect.cassandra.key.space=demo
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal or fields,
data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules. More information
can be found `here <http://docs.confluent.io/2.0.1/schema-registry/docs/api.html#compatibility>`_.

For the Sink connector, if columns are add to the target Cassandra table and not present in the source topic they will be
set to null by Cassandras Json insert functionality. Columns which are omitted from the JSON value map are treated as a
null insert (which results in an existing value being deleted, if one is present), if a record with the same key is
inserted again.

Future releases will support auto creation of tables and adding columns on changes to the topic schema.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
