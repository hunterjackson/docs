Kafka Connect Cassandra Source
==============================

Kafka Connect Cassandra is a Source Connector for reading data from Cassandra and writing to Kafka.

The Source supports:

1. :ref:`The KCQL routing querying <kcql>` - Allows for table to topic routing.
2. Incremental mode
3. Bulk mode
4. Error policies for handling failures.

Prerequisites
-------------

-  Cassandra 2.2.4
-  Confluent 3.0.0
-  Java 1.8
-  Scala 2.11

Setup
-----

Before we can do anything, including the QuickStart we need to install Cassandra and the Confluent platform.

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

Follow the instructions :ref:`here <install>`.

Source Connector
----------------

The Cassandra Source connector allows you to extract entries from Cassandra with the CQL driver and write them into a
Kafka topic.

Each table specified in the configuration is polled periodically and each record from the result is converted to a Kafka
Connect record. These records are then written to Kafka by the Kafka Connect framework.

The Source connector operates in two modes:

1. Bulk - Each table is selected in full each time it is polled.
2. Incremental - Each table is querying with lower and upper bounds to
   extract deltas.

In incremental mode the column used to identify new or delta rows has to be provided. This column must be of CQL Type
Timestamp. Due to Cassandra's and CQL restrictions this should be a primary key or part of a composite primary keys.
ALLOW\_FILTERING can also be supplied as an configuration.

.. note::

    TimeUUIDs are converted to strings. Use the `UUIDs <https://docs.datastax.com/en/drivers/java/2.0/com/datastax/driver/core/utils/UUIDs.html>`__
    helpers to convert to Dates.

Source Connector QuickStart
~~~~~~~~~~~~~~~~~~~~~~~~~~~

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and updating the configuration. We have developed
a Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

Test data
^^^^^^^^^

Once you have installed and started Cassandra create a table to extract records from. This snippet creates a table called
orders and inserts 3 rows representing fictional orders or some options and futures on a trading platform.

Start the Cassandra cql shell

.. sourcecode:: bash

    ➜  bin ./cqlsh
    Connected to Test Cluster at 127.0.0.1:9042.
    [cqlsh 5.0.1 | Cassandra 3.0.2 | CQL spec 3.3.1 | Native protocol v4]
    Use HELP for help.
    cqlsh>

Execute the following:

.. sourcecode:: sql

    CREATE KEYSPACE demo WITH REPLICATION = {'class' : 'SimpleStrategy', 'replication_factor' : 3};
    use demo;

    create table orders (id int, created timeuuid, product text, qty int, price float, PRIMARY KEY (id, created))
    WITH CLUSTERING ORDER BY (created asc);

    INSERT INTO orders (id, created, product, qty, price) VALUES (1, now(), 'OP-DAX-P-20150201-95.7', 100, 94.2);
    INSERT INTO orders (id, created, product, qty, price) VALUES (2, now(), 'OP-DAX-C-20150201-100', 100, 99.5);
    INSERT INTO orders (id, created, product, qty, price) VALUES (3, now(), 'FU-KOSPI-C-20150201-100', 200, 150);

    SELECT * FROM orders;

     id | created                              | price | product                 | qty
    ----+--------------------------------------+-------+-------------------------+-----
      1 | 17fa1050-137e-11e6-ab60-c9fbe0223a8f |  94.2 |  OP-DAX-P-20150201-95.7 | 100
      2 | 17fb6fe0-137e-11e6-ab60-c9fbe0223a8f |  99.5 |   OP-DAX-C-20150201-100 | 100
      3 | 17fbbe00-137e-11e6-ab60-c9fbe0223a8f |   150 | FU-KOSPI-C-20150201-100 | 200

    (3 rows)

    (3 rows)

Starting the Connector (Distributed)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running the ``start-connect.sh`` script in the ``bin`` folder.

.. sourcecode:: bash

    ➜ bin/start-connect.sh

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for Cassandra.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create cassandra-source-orders < cassandra-source-incr-orders.properties

    #Connector `cassandra-source-orders`:
    name=cassandra-source-orders
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector
    connect.cassandra.key.space=demo
    connect.cassandra.source.kcql=INSERT INTO orders-topic SELECT * FROM orders PK created
    connect.cassandra.import.mode=incremental
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra
    #task ids: 0

The ``cassandra-source.properties`` file defines:

1. The name of the connector, must be unique.
2. The name of the connector class.
3. The keyspace (demo) we are connecting to.
4. The table to topic import map. This allows you to route tables to different topics. Each mapping is comma separated
   and for each mapping the table and topic are separated by a colon, if no topic is provided the records from the table
   will be routed to a topic matching the table name. In this example the orders table records are routed to the topic
   orders-topic. This property sets the tables to import!
5. The import mode, either incremental or bulk.
6. The ip or host name of the nodes in the Cassandra cluster to connect to.
7. Username and password, ignored unless you have set Cassandra to use the PasswordAuthenticator.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    cassandra-sink

.. sourcecode:: bash

    [2016-05-06 13:52:28,178] INFO
         ____        __        __  ___                  __        _
        / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
       / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
      / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
     /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
        ______                                __           _____
       / ____/___ _______________ _____  ____/ /________ _/ ___/____  __  _______________
      / /   / __ `/ ___/ ___/ __ `/ __ \/ __  / ___/ __ `/\__ \/ __ \/ / / / ___/ ___/ _ \
     / /___/ /_/ (__  |__  ) /_/ / / / / /_/ / /  / /_/ /___/ / /_/ / /_/ / /  / /__/  __/
     \____/\__,_/____/____/\__,_/_/ /_/\__,_/_/   \__,_//____/\____/\__,_/_/   \___/\___/

    By Andrew Stevenson. (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceTask:64)
    [2016-05-06 13:34:41,193] INFO Attempting to connect to Cassandra cluster at localhost and create keyspace demo. (com.datamountaineer.streamreactor.connect.cassandra.CassandraConnection$:49)
    [2016-05-06 13:34:41,263] INFO Using username_password. (com.datamountaineer.streamreactor.connect.cassandra.CassandraConnection$:83)
    [2016-05-06 13:34:41,459] INFO Did not find Netty's native epoll transport in the classpath, defaulting to NIO. (com.datastax.driver.core.NettyUtil:83)
    [2016-05-06 13:34:41,823] INFO Using data-center name 'datacenter1' for DCAwareRoundRobinPolicy (if this is incorrect, please provide the correct datacenter name with DCAwareRoundRobinPolicy constructor) (com.datastax.driver.core.policies.DCAwareRoundRobinPolicy:95)
    [2016-05-06 13:34:41,824] INFO New Cassandra host localhost/127.0.0.1:9042 added (com.datastax.driver.core.Cluster:1475)
    [2016-05-06 13:34:41,868] INFO Connection to Cassandra established. (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceTask:87)
    ....


If you switch back to the terminal you started the Connector in you should see the Cassandra Source being accepted and
the task starting and processing the 3 existing rows.

.. sourcecode:: bash

    [2016-05-06 13:44:33,132] INFO Source task Thread[WorkerSourceTask-cassandra-source-orders-0,5,main] finished initialization and start (org.apache.kafka.connect.runtime.WorkerSourceTask:342)
    [2016-05-06 13:44:33,137] INFO Query SELECT * FROM demo.orders WHERE created > maxTimeuuid(?) AND created <= minTimeuuid(?)  ALLOW FILTERING executing with bindings (2016-05-06 09:23:28+0200, 2016-05-06 13:44:33+0200). (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:156)
    [2016-05-06 13:44:33,151] INFO Querying returning results for demo.orders. (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:185)
    [2016-05-06 13:44:33,160] INFO Processed 3 rows for table orders-topic.orders (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:206)
    [2016-05-06 13:44:33,160] INFO Found 3. Draining entries to batchSize 100. (com.datamountaineer.streamreactor.connect.queues.QueueHelpers$:45)
    [2016-05-06 13:44:33,197] WARN Error while fetching metadata with correlation id 0 : {orders-topic=LEADER_NOT_AVAILABLE} (org.apache.kafka.clients.NetworkClient:582)
    [2016-05-06 13:44:33,406] INFO Found 0. Draining entries to batchSize 100. (com.datamountaineer.streamreactor.connect.queues.QueueHelpers$:45)

Check Kafka, 3 rows as before.

.. sourcecode:: bash

    ➜  confluent-3.0.1/bin/kafka-avro-console-consumer \
    --zookeeper localhost:2181 \
    --topic orders-topic \
    --from-beginning
    {"id":{"int":1},"created":{"string":"Thu May 05 13:24:22 CEST 2016"},"price":{"float":94.2},"product":{"string":"DAX-P-20150201-95.7"},"qty":{"int":100}}
    {"id":{"int":2},"created":{"string":"Thu May 05 13:26:21 CEST 2016"},"price":{"float":99.5},"product":{"string":"OP-DAX-C-20150201-100"},"qty":{"int":100}}
    {"id":{"int":3},"created":{"string":"Thu May 05 13:26:44 CEST 2016"},"price":{"float":150.0},"product":{"string":"FU-KOSPI-C-20150201-100"},"qty":{"int":200}}

The Source tasks will continue to poll but not pick up any new rows yet.

.. code-block::bash

    INFO Query SELECT * FROM demo.orders WHERE created > ? AND created <= ?  ALLOW FILTERING executing with bindings (Thu May 05 13:26:44 CEST 2016, Thu May 05 21:19:38 CEST 2016). (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:152)
    INFO Querying returning results for demo.orders. (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:181)
    INFO Processed 0 rows for table orders-topic.orders (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:202)

Inserting new data
''''''''''''''''''

Now lets insert a row into the Cassandra table. Start the CQL shell.

.. code-block:: bash

    ➜  bin ./cqlsh
    Connected to Test Cluster at 127.0.0.1:9042.
    [cqlsh 5.0.1 | Cassandra 3.0.2 | CQL spec 3.3.1 | Native protocol v4]
    Use HELP for help.

Execute the following:

.. code-block:: sql

    use demo;

    INSERT INTO orders (id, created, product, qty, price) VALUES (4, now(), 'FU-DATAMOUNTAINEER-C-20150201-100', 500, 10000);

    SELECT * FROM orders;

     id | created                              | price | product                           | qty
    ----+--------------------------------------+-------+-----------------------------------+-----
      1 | 17fa1050-137e-11e6-ab60-c9fbe0223a8f |  94.2 |            OP-DAX-P-20150201-95.7 | 100
      2 | 17fb6fe0-137e-11e6-ab60-c9fbe0223a8f |  99.5 |             OP-DAX-C-20150201-100 | 100
      4 | 02acf5d0-1380-11e6-ab60-c9fbe0223a8f | 10000 | FU-DATAMOUNTAINEER-C-20150201-100 | 500
      3 | 17fbbe00-137e-11e6-ab60-c9fbe0223a8f |   150 |           FU-KOSPI-C-20150201-100 | 200

    (4 rows)
    cqlsh:demo>

Check the logs.

.. sourcecode:: bash

    [2016-05-06 13:45:33,134] INFO Query SELECT * FROM demo.orders WHERE created > maxTimeuuid(?) AND created <= minTimeuuid(?)  ALLOW FILTERING executing with bindings (2016-05-06 13:31:37+0200, 2016-05-06 13:45:33+0200). (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:156)
    [2016-05-06 13:45:33,137] INFO Querying returning results for demo.orders. (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:185)
    [2016-05-06 13:45:33,138] INFO Processed 1 rows for table orders-topic.orders (com.datamountaineer.streamreactor.connect.cassandra.source.CassandraTableReader:206)
    [2016-05-06 13:45:33,138] INFO Found 0. Draining entries to batchSize 100. (com.datamountaineer.streamreactor.connect.queues.QueueHelpers$:45)

Check Kafka.

.. sourcecode:: bash

    ➜  confluent confluent-3.0.1/bin/kafka-avro-console-consumer \
    --zookeeper localhost:2181 \
    --topic orders-topic \
    --from-beginning

    {"id":{"int":1},"created":{"string":"17fa1050-137e-11e6-ab60-c9fbe0223a8f"},"price":{"float":94.2},"product":{"string":"OP-DAX-P-20150201-95.7"},"qty":{"int":100}}
    {"id":{"int":2},"created":{"string":"17fb6fe0-137e-11e6-ab60-c9fbe0223a8f"},"price":{"float":99.5},"product":{"string":"OP-DAX-C-20150201-100"},"qty":{"int":100}}
    {"id":{"int":3},"created":{"string":"17fbbe00-137e-11e6-ab60-c9fbe0223a8f"},"price":{"float":150.0},"product":{"string":"FU-KOSPI-C-20150201-100"},"qty":{"int":200}}
    {"id":{"int":4},"created":{"string":"02acf5d0-1380-11e6-ab60-c9fbe0223a8f"},"price":{"float":10000.0},"product":{"string":"FU-DATAMOUNTAINEER-C-20150201-100"},"qty":{"int":500}}

Bingo, we have our extra row.


Features
--------

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L**, :ref:`KCQL <kcql>` allows for routing and mapping using a SQL like syntax,
consolidating typically features in to one configuration option.

Data Types
^^^^^^^^^^

The Source connector supports copying tables in bulk and incrementally to Kafka.

The following CQL data types are supported:

+-------------+---------------------+
| CQL Type    | Connect Data Type   |
+=============+=====================+
| TimeUUID    | Optional String     |
+-------------+---------------------+
| UUID        | Optional String     |
+-------------+---------------------+
| Inet        | Optional String     |
+-------------+---------------------+
| Ascii       | Optional String     |
+-------------+---------------------+
| Text        | Optional String     |
+-------------+---------------------+
| Timestamp   | Optional String     |
+-------------+---------------------+
| Date        | Optional String     |
+-------------+---------------------+
| Tuple       | Optional String     |
+-------------+---------------------+
| UDT         | Optional String     |
+-------------+---------------------+
| Boolean     | Optional Boolean    |
+-------------+---------------------+
| TinyInt     | Optional Int8       |
+-------------+---------------------+
| SmallInt    | Optional Int16      |
+-------------+---------------------+
| Int         | Optional Int32      |
+-------------+---------------------+
| Decimal     | Optional String     |
+-------------+---------------------+
| Float       | Optional Float32    |
+-------------+---------------------+
| Counter     | Optional Int64      |
+-------------+---------------------+
| BigInt      | Optional Int64      |
+-------------+---------------------+
| VarInt      | Optional Int64      |
+-------------+---------------------+
| Double      | Optional Int64      |
+-------------+---------------------+
| Time        | Optional Int64      |
+-------------+---------------------+
| Blob        | Optional Bytes      |
+-------------+---------------------+
| Map         | Optional String     |
+-------------+---------------------+
| List        | Optional String     |
+-------------+---------------------+
| Set         | Optional String     |
+-------------+---------------------+

.. note:: For Map, List and Set the value is extracted from the Cassandra Row and inserted as a JSON string representation.

Modes
^^^^^

The Source connector runs in both bulk and incremental mode.

Each mode has a polling interval. This interval determines how often the readers execute queries against the Cassandra
tables. It applies to both incremental and bulk modes. The ``cassandra.import.mode`` setting controls the import behaviour.

Incremental
'''''''''''

In ``incremental`` mode the connector supports querying based on a column in the tables with CQL data type of TimeUUID.

Kafka Connect tracks the latest record it retrieved from each table, so it can start at the correct location on the next
iteration (or in case of a crash). In this case the maximum value of the records returned by the result-set is tracked
and stored in Kafka by the framework. If no offset is found for the table at startup a default timestamp of 1900-01-01
is used. This is then passed to a prepared statement containing a range query. For example:

.. sourcecode:: sql

    SELECT * FROM demo.orders WHERE created > maxTimeuuid(?) AND created <= minTimeuuid(?)

.. warning::::

    If the column used for tracking timestamps is a compound key, ALLOW FILTERING is appended to the query.
    This can have a detrimental performance impact of Cassandra as it is effectively issuing a full scan.

Bulk
''''

In ``bulk`` mode the connector extracts the full table, no where clause is attached to the query.

.. warning::

    Watch out with the poll interval. After each interval the bulk query will be executed again.

Topic Routing
^^^^^^^^^^^^^

The Sink supports topic routing that allows mapping the messages from topics to a specific table. For example map
a topic called "bloomberg_prices" to a table called "prices". This mapping is set in the
``connect.cassandra.source.kcql`` option.

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
    violations and or other expections thrown by drivers..

**Retry**

Any error on write to the target database causes the RetryIterable exception to be thrown. This causes the
Kafka connect framework to pause and replay the message. Offsets are not committed. For example, if the table is offline
it will cause a write failure, the message can be replayed. With the Retry policy the issue can be fixed without stopping
the sink.

The length of time the Sink will retry can be controlled by using the ``connect.cassandra.source.max.retries`` and the
``connect.cassandra.source.retry.interval``.

Configurations
--------------

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


``connect.cassandra.import.poll.interval``


The polling interval between queries against tables for bulk mode in milliseconds.
Default is 1 minute.

* Data type: int
* Optional : yes
* Default  : 10

.. warning::

    WATCH OUT WITH BULK MODE AS MAY REPEATEDLY PULL IN THE SAME DATE.

``connect.cassandra.import.mode``

Either bulk or incremental.

* Data type : string
* Optional  : no


``connect.cassandra.source.kcql``

Kafka connect query language expression. Allows for expressive table to topic routing, field selection and renaming.
In incremental mode the timestampColumn can be specified by ``PK colName``.

Examples:

.. sourcecode:: sql

    INSERT INTO TOPIC1 SELECT * FROM TOPIC1 PK myTimeUUICol

* Data type : string
* Optional  : no

.. warning::

    The timestamp column must be of CQL Type TimeUUID.

``connect.cassandra.import.fetch.size``

The fetch size for the Cassandra driver to read.

* Data type : int
* Optional  : yes
* Default   : 1000

``connect.cassandra.source.task.buffer.size``

The size of the queue for buffering resultset records before write to Kafka.

* Data type : int
* Optional  : yes
* Default   : 10000


``connect.cassandra.source.task.batch.size``

The number of records the Source  task should drain from the reader queue.

* Data type : int
* Optional  : yes
* Default   : 1000

``connect.cassandra.source.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.cassandra.source.max.retries``
option. The ``connect.cassandra.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high
* Default: ``throw``

``connect.cassandra.source.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.cassandra.source.error.policy`` is set to ``retry``.

* Type: string
* Importance: high
* Default: 10

``connect.cassandra.source.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.cassandra.source.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Default : 60000 (1 minute)



Bulk Example
~~~~~~~~~~~~

.. sourcecode:: bash

    name=cassandra-source-orders-bulk
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector
    connect.cassandra.key.space=demo
    connect.cassandra.source.kcql=INSERT INTO TABLE_X SELECT * FROM TOPIC_Y
    connect.cassandra.import.mode=bulk
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra

Incremental Example
~~~~~~~~~~~~~~~~~~~

.. sourcecode:: bash

    name=cassandra-source-orders-incremental
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.source.CassandraSourceConnector
    connect.cassandra.key.space=demo
    connect.cassandra.source.kcql=INSERT INTO TABLE_X SELECT * FROM TOPIC_Y PK created
    connect.cassandra.import.mode=incremental
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra


Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal or fields,
data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules. More information
can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.

For the Source connector, at present no column selection is handled, every column from the table is queried to column
additions and deletions are handled in accordance with the compatibility mode of the Schema Registry.

Future releases will support auto creation of tables and adding columns on changes to the topic schema.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
