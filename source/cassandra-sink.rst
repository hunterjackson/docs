Kafka Connect Cassandra Sink
============================

The Cassandra Sink allows you to write events from Kafka to Cassandra. The connector converts the value from the Kafka
Connect SinkRecords to Json and uses Cassandra's JSON insert functionality to insert the rows. The task expects pre-created
tables in Cassandra.

.. note:: The table and keyspace must be created before hand!
.. note:: If the target table has TimeUUID fields the payload string the corresponding field in Kafka must be a UUID.

The Sink supports:

1.  :ref:`The KCQL routing querying <kcql>` - Kafka topic payload field selection is supported, allowing you to have choose selection of fields
    or all fields written to Cassandra.
2.  Topic to table routing via KCQL.
3.  Error policies for handling failures.
4.  Payload support for Schema.Struct and payload Struct, Schema.String and Json payload and Json payload with no schema.
5.  Optional TTL, time to live on inserts. See Cassandras `documentation <https://docs.datastax.com/en/cql/3.3/cql/cql_using/useTTL.html>`__
    for more information.
6.  Delete for records in Cassandra for null payloads.   

The Sink supports three Kafka payloads type:

**Connect entry with Schema.Struct and payload Struct.** If you follow the best practice while producing the events, each
message should carry its schema information. Best option is to send Avro. Your connect configurations should be set to
``value.converter=io.confluent.connect.avro.AvroConverter``.
You can find an example `here <https://github.com/confluentinc/kafka-connect-blog/blob/master/etc/connect-avro-standalone.properties>`__.
To see how easy is to have your producer serialize to Avro have a look at
`this <http://docs.confluent.io/3.0.1/schema-registry/docs/serializer-formatter.html?highlight=kafkaavroserializer>`__.
This requires the SchemaRegistry which is open source thanks to Confluent! Alternatively you can send Json + Schema.
In this case your connect configuration should be set to ``value.converter=org.apache.kafka.connect.json.JsonConverter``. This doesn't
require the SchemaRegistry.

**Connect entry with Schema.String and payload json String.** Sometimes the producer would find it easier, despite sending
Avro to produce a GenericRecord, to just send a message with Schema.String and the json string.

**Connect entry without a schema and the payload json String.** There are many existing systems which are publishing json
over Kafka and bringing them in line with best practices is quite a challenge. Hence we added the support

Prerequisites!!
---------------

-  Cassandra **2.2.4+** if your are on version 2.* or **3.0.1+** if you are on version 3.*
- Confluent 3.3
-  Java 1.8
-  Scala 2.11

.. note::

    You must be using at least Cassandra 3.0.9 to have JSON support!

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

Follow the instructions :ref:`here <install>`.

Sink Connector QuickStart
~~~~~~~~~~~~~~~~~~~~~~~~~

We you start the Confluent Platform, Kafka Connect is started in distributed mode (``confluent start``). 
In this mode a Rest Endpoint on port ``8083`` is exposed to accept connector configurations. 
We developed Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

Test data
^^^^^^^^^

The Sink currently expects precreated tables and keyspaces. So lets create a keyspace and table in Cassandra via the CQL
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

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor and Confluent. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Once the Connect has started we can now use the kafka-connect-tools :ref:`cli <kafka-connect-cli>` to post in our distributed properties file for Cassandra.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/connect-cli create cassandra-sink-orders < conf/cassandra-sink.properties

    #Connector `cassandra-sink-orders`:
    name=cassandra-sink-orders
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.cassandra.kcql=INSERT INTO orders SELECT * FROM orders-topic
    connect.cassandra.port=9042
    connect.cassandra.key.space=demo
    connect.cassandra.contact.points=localhost
    connect.cassandra.username=cassandra
    connect.cassandra.password=cassandra
    #task ids: 0

The ``cassandra-sink.properties`` file defines:

1.  The name of the sink.
2.  The Sink class.
3.  The max number of tasks the connector is allowed to created (1 task only).
4.  The topics to read from.
5.  :ref:`The KCQL routing querying. <kcql>`
6.  The Cassandra host.
7.  The Cassandra port.
8.  The Cassandra Keyspace.
9.  The username.
10. The password.

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

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
     --broker-list localhost:9092 --topic orders-topic \
     --property value.schema='{"type":"record","name":"myrecord","fields":[{"name":"id","type":"int"},{"name":"created","type":"string"},{"name":"product","type":"string"},{"name":"price","type":"double"}, {"name":"qty", "type":"int"}]}'

Now the producer is waiting for input. Paste in the following (each on a line separately):

.. sourcecode:: bash

    {"id": 1, "created": "2016-05-06 13:53:00", "product": "OP-DAX-P-20150201-95.7", "price": 94.2, "qty":100}
    {"id": 2, "created": "2016-05-06 13:54:00", "product": "OP-DAX-C-20150201-100", "price": 99.5, "qty":100}
    {"id": 3, "created": "2016-05-06 13:55:00", "product": "FU-DATAMOUNTAINEER-20150201-100", "price": 10000, "qty":100}
    {"id": 4, "created": "2016-05-06 13:56:00", "product": "FU-KOSPI-C-20150201-100", "price": 150, "qty":100}

Now if we check the logs of the connector we should see 2 records being inserted to Cassandra:

.. sourcecode:: bash

    [2016-05-06 13:55:10,368] INFO Setting newly assigned partitions [orders-topic-0] for group connect-cassandra-sink-1 (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:219)
    [2016-05-06 13:55:16,423] INFO Received 4 records. (com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraJsonWriter:96)
    [2016-05-06 13:55:16,484] INFO Processed 4 records. (com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraJsonWriter:138)

.. sourcecode:: bash

    use demo;
    SELECT * FROM orders;

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

The Sink connector uses Cassandra's `JSON <http://www.datastax.com/dev/blog/whats-new-in-cassandra-2-2-json-support>`__
insert functionality. The SinkRecord from Kafka Connect is converted to JSON and feed into the prepared statements for
inserting into Cassandra.

See Cassandra's `documentation <http://cassandra.apache.org/doc/cql3/CQL-2.2.html#insertJson>`__ for type mapping.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The Cassandra Sink supports the following:

.. sourcecode:: bash

    INSERT INTO <target table> SELECT <fields> FROM <source topic> TTL=<TTL>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB


    #Insert mode, select 3 fields and rename from topicB and write to tableB with TTL
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB TTL=100000


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

The length of time the Sink will retry can be controlled by using the ``connect.cassandra.max.retries`` and the
``connect.cassandra.retry.interval``.

Topic Routing
^^^^^^^^^^^^^

The Sink supports topic routing that allows mapping the messages from topics to a specific table. For example map
a topic called "bloomberg_prices" to a table called "prices". This mapping is set in the
``connect.cassandra.kcql`` option.

Field Selection
^^^^^^^^^^^^^^^

The Sink supports selecting fields from the Source topic or selecting all fields and mapping of these fields to columns
in the target table. For example, map a field called "qty"  in a topic to a column called "quantity" in the target
table.

All fields can be selected by using "*" in the field part of ``connect.cassandra.kcql``.

Leaving the column name empty means trying to map to a column in the target table with the same name as the field in the
source topic.

Deletion in Cassandra
~~~~~~~~~~~~~~~~~~~~~

Compacted topics in Kafka retain the last message per key. Deletion in Kafka occurs by tombestoning. If compaction is enabled on the topic 
and a message is sent with a null payload, Kafka flags this record for delete and is compacted/removed from the topic. For more information
on compaction see `this <http://cloudurable.com/blog/kafka-architecture-log-compaction/index.html>`__.

The use case for this delete functionality would be, for example, when the source topic is a compacted topic, perhaps capturing data changes 
from an upstream source such as a CDC connector. Let's say a record is deleted from the upstream source and that delete operation is propagated 
to the kafka topic, with the key of the kafka message as the PK of the record in the targeted cassandra table - meaning the value of the kafka message 
is now null. This feature allows you to delete these records in Cassandra.

This functionality will be migrated to `KCQL <https://github.com/datamountaineer/kafka-connector-query-language>`__ in future releases.

Deletion in Cassandra is supported based on fields in the ``key`` of messages with a empty/null payload. Deletion is enabled by settings 
the ``connect.delete.enabled`` option. A Cassandra delete statement must be provided, ``connect.delete.statement`` which specifies the Cassandra 
CQL delete statement with parameters to bind field values from the ``key`` to, for example, with the delete statement of:

.. sourcecode:: bash

    DELETE FROM orders WHERE id = ? and product = ?

If a message was received with a empty/null value and key fields ``key.id`` and ``key.product`` the final bound Cassandra statement would be:

.. sourcecode:: bash

    # connect.delete.enabled=true
    # connect.delete.statement=DELETE FROM orders WHERE id = ? and product = ?
    # connect.delete.struct_flds=id,product   

    # "{ "key": { "id" : 999, "product" : "DATAMOUNTAINEER" }, "value" : null }"
    DELETE FROM orders WHERE id = 999 and product = "DATAMOUNTAINEER"

.. note::

    Deletion will only occur if a message with an empty payload is recieved from Kafka.

.. important::

    Ensure your ordinal position of the ``connect.delete.struct_flds`` matches the bind order in the Cassandra delete statement!  

Legacy topics (plain text payload with a json string)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We have found some of the clients have already an infrastructure where they publish pure json on the topic and obviously the jump to follow
the best practice and use schema registry is quite an ask. So we offer support for them as well.

This time we need to start the connect with a different set of settings.

.. sourcecode:: bash

      #create a new configuration for connect
      ➜ cp  etc/schema-registry/connect-avro-distributed.properties etc/schema-registry/connect-distributed-json.properties
      ➜ vi etc/schema-registry/connect-distributed-json.properties

Replace the following 4 entries in the config

.. sourcecode:: bash

      key.converter=io.confluent.connect.avro.AvroConverter
      key.converter.schema.registry.url=http://localhost:8081
      value.converter=io.confluent.connect.avro.AvroConverter
      value.converter.schema.registry.url=http://localhost:8081

      with the following
.. sourcecode:: bash

      key.converter=org.apache.kafka.connect.json.JsonConverter
      key.converter.schemas.enable=false
      value.converter=org.apache.kafka.connect.json.JsonConverter
      value.converter.schemas.enable=false

Now let's restart the connect instance:
.. sourcecode:: bash

      #start a new instance of connect
      ➜   $CONFLUENT_HOME/bin/connect-distributed etc/schema-registry/connect-distributed-json.properties

Configurations
--------------

Configurations common to both Sink and Source are:

``connect.cassandra.contact.points``

Contact points (hosts) in Cassandra cluster. This is a comma separated value.
i.e:host-1,host-2

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

Username to connect to Cassandra with.

* Data type: string
* Optional : yes

``connect.cassandra.password``

Password to connect to Cassandra with.

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


``connect.cassandra.kcql``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming.

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT field1, field2, field3 as renamedField FROM TOPIC2


* Data Type: string
* Optional : no

``connect.cassandra.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.cassandra.max.retries``
option. The ``connect.cassandra.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high
* Default: ``throw``

``connect.cassandra.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.cassandra.error.policy`` is set to ``retry``.

* Type: string
* Importance: high
* Default: 10

``connect.cassandra.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.cassandra.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Default : 60000 (1 minute)

``connect.progress.enabled``

Enables the output for how many records have been processed.

* Type: boolean
* Importance: medium
* Optional: yes
* Default : false

``connect.delete.enabled``

Enables row deletion from cassandra.

* Type: boolean
* Importance: medium
* Optional: yes
* Default : false

``connect.delete.statement``

Delete statement for cassandra. Required if ``connect.delete.enabled`` is set. 

* Type: string
* Importance: medium
* Optional: yes
* Default : 
  
``connect.delete.struct_flds``

Fields in the key struct data type used in there delete statement. Comma-separated in the order they are found in ``connect.delete.statement``

* Type: string
* Importance: medium
* Optional: yes
* Default : 

Example
~~~~~~~

.. sourcecode:: bash

    name=cassandra-sink-orders
    connector.class=com.datamountaineer.streamreactor.connect.cassandra.sink.CassandraSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.cassandra.kcql = INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT field1,
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
can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.

For the Sink connector, if columns are add to the target Cassandra table and not present in the Source topic they will be
set to null by Cassandras Json insert functionality. Columns which are omitted from the JSON value map are treated as a
null insert (which results in an existing value being deleted, if one is present), if a record with the same key is
inserted again.

Future releases will support auto creation of tables and adding columns on changes to the topic schema.

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

