Kafka Connect HBase
===================

A Connector and Sink to write events from Kafka to HBase. The connector takes the value from the Kafka Connect SinkRecords
and inserts a new entry to HBase.

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>` - Kafka topic payload field selection is supported, allowing you to select fields written to HBase.
2. Topic to table routing via KCQL.
3. RowKey selection - Selection of fields to use as the row key, if none specified the topic name, partition and offset are
   used via KCQL.
4. Error policies.

Prerequisites
-------------

- Confluent 3.0.1
- HBase 1.2.0
- Java 1.8
- Scala 2.11

Setup
-----

HBase Setup
~~~~~~~~~~~

Download and extract HBase:

.. sourcecode:: bash

    wget https://www.apache.org/dist/hbase/1.2.1/hbase-1.2.1-bin.tar.gz
    tar -xvf hbase-1.2.1-bin.tar.gz -C hbase


Edit ``conf/quickstarts/hbase-site.xml`` and add the following content:

.. sourcecode:: html

    <?xml version="1.0"?>
    <?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
    <configuration>
     <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
      </property>
      <property>
        <name>hbase.rootdir</name>
        <value>file:///tmp/hbase</value>
      </property>
      <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/tmp/zookeeper</value>
      </property>
    </configuration>

The ``hbase.cluster.distributed`` is required since when you start hbase it will try and start it's own Zookeeper, but in
this case we want to use Confluents.

Now start HBase and check the logs to ensure it's up:

.. sourcecode:: bash

    bin/start-hbase.sh

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Sink Connector QuickStart
-------------------------

We will start the connector in distributed mode. Each connector exposes a rest endpoint for stopping, starting and updating the configuration. We have developed
a Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

HBase Table
~~~~~~~~~~~

The Sink expects a precreated table in HBase. In the HBase shell create the test table, go to your HBase install location.

.. sourcecode:: bash

    bin/hbase shell
    hbase(main):001:0> create 'person_hbase',{NAME=>'d', VERSIONS=>1}

    hbase(main):001:0> list
    person
    1 row(s) in 0.9530 seconds

    hbase(main):002:0> describe 'person'
    DESCRIPTION
     'person', {NAME => 'd', BLOOMFILTER => 'ROW', VERSIONS => '1', IN_MEMORY => 'false', KEEP_DELETED_CELLS => 'false', DATA_BLOCK_ENCOD true
     ING => 'NONE', TTL => 'FOREVER', COMPRESSION => 'NONE', MIN_VERSIONS => '0', BLOCKCACHE => 'true', BLOCKSIZE => '65536', REPLICATION
     _SCOPE => '0'}
    1 row(s) in 0.0810 seconds

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Start Kafka Connect in distributed more by running the ``start-connect.sh`` script in the ``bin`` folder.

.. sourcecode:: bash

    ➜ bin/start-connect.sh

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for HBase.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create hbase-sink < conf/quickstarts/hbase-sink.properties

    #Connector name=`hbase-sink`
    name=person-hbase-test
    connector.class=com.datamountaineer.streamreactor.connect.hbase.HbaseSinkConnector
    tasks.max=1
    topics=TOPIC1
    connect.hbase.sink.column.family=d
    connect.hbase.sink.kcql=INSERT INTO person_hbase SELECT * FROM TOPIC1
    #task ids: 0

This ``hbase-sink.properties`` configuration defines:

1.  The name of the sink.
2.  The Sink class.
3.  The max number of tasks the connector is allowed to created. Should not be greater than the number of partitions in the Source topics
    otherwise tasks will be idle.
4.  The Source kafka topics to take events from.
5.  The HBase column family to write to.
6.  :ref:`The KCQL routing querying. <kcql>`

If you switch back to the terminal you started the Connector in you should see the HBase Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    hbase-sink

.. sourcecode:: bash

    INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\\_,\\\\\\\__,_/_/  /_/\___\\\\\,\/_/ /_/\\_/\__,_/_/_/ /_/\___/\___/_/
          / / / / __ )____ _________ / ___/(_)___  / /__
         / /_/ / __  / __ `/ ___/ _ \\__ \/ / __ \/ //_/
        / __  / /_/ / /_/ (__  )  __/__/ / / / / / ,<
       /_/ /_/_____/\__,_/____/\___/____/_/_/ /_/_/|_|

    By Stefan Bocutiu (com.datamountaineer.streamreactor.connect.hbase.HbaseSinkTask:44)


Test Records
^^^^^^^^^^^^

.. hint::

    If your input topic doesn't match the target use Kafka Streams to transform in realtime the input. Also checkout the
    `Plumber <https://github.com/rollulus/kafka-streams-plumber>`__, which allows you to inject a Lua script into
    `Kafka Streams <http://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple>`__ to do this,
    no Java or Scala required!

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``firstname`` field of type string
a ``lastname`` field of type string, an ``age`` field of type int and a ``salary`` field of type double.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic TOPIC1 \
      --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.hbase"
      "fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},
      {"name":"salary","type":"double"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}
    {"firstName": "Anna", "lastName": "Jones", "age":28, "salary": 5430}

Check for records in HBase
~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this

.. sourcecode:: bash

    INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@48ffb4dc finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)
    INFO Writing 2 rows to Hbase... (com.datamountaineer.streamreactor.connect.hbase.writers.HbaseWriter:83)

In HBase:

.. sourcecode:: bash

    hbase(main):004:0* scan 'person_hbase'
    ROW                                                  COLUMN+CELL
     Anna\x0AJones                                       column=d:age, timestamp=1463056888641, value=\x00\x00\x00\x1C
     Anna\x0AJones                                       column=d:firstName, timestamp=1463056888641, value=Anna
     Anna\x0AJones                                       column=d:income, timestamp=1463056888641, value=@\xB56\x00\x00\x00\x00\x00
     Anna\x0AJones                                       column=d:lastName, timestamp=1463056888641, value=Jones
     John\x0ASmith                                       column=d:age, timestamp=1463056693877, value=\x00\x00\x00\x1E
     John\x0ASmith                                       column=d:firstName, timestamp=1463056693877, value=John
     John\x0ASmith                                       column=d:income, timestamp=1463056693877, value=@\xB2\xDE\x00\x00\x00\x00\x00
     John\x0ASmith                                       column=d:lastName, timestamp=1463056693877, value=Smith
    2 row(s) in 0.0260 seconds

Now stop the connector.

Features
--------

The HBase Sink writes records from Kafka to HBase.

The Sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to select fields written to HBase.
2. Topic to table routing.
3. RowKey selection - Selection of fields to use as the row key, if none specified the topic name, partition and offset are
   used.
4. Error policies.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L**, :ref:`KCQL <kcql>` allows for routing and mapping using a SQL like syntax,
consolidating typically features in to one configuration option.

The HBase Sink supports the following:

.. sourcecode:: bash

    INSERT INTO <table> SELECT <fields> FROM <source topic> <PK> primary_key_cols

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA and use the default rowkey (topic name, partition, offset)
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB, use field y from the topic as the row key
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB PK y

This is set in the ``connect.hbase.sink.kcql`` option.

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

The length of time the Sink will retry can be controlled by using the ``connect.hbase.sink.max.retries`` and the
``connect.hbase.sink.retry.interval``.


Configurations
--------------

``connect.hbase.sink.column.family``

The hbase column family.

* Type: string
* Importance: high
* Optional: no

``connect.hbase.sink.kcql``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming. Fields
to be used as the row key can be set by specifing the ``PK``. The below example uses field1 and field2 are the row key.

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT * FROM TOPIC2 PK field1, field2

If no primary keys are specified the topic name, partition and offset converted to bytes are used as the HBase rowkey.

* Type: string
* Importance: high
* Optional: no

``connect.hbase.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.hbase.sink.max.retries``
option. The ``connect.hbase.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: medium
* Optional: yes
* Default: RETRY

``connect.hbase.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.habse.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: medium
* Optional: yes
* Default: 10

``connect.hbase.sink.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.hbase.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Optional: yes
* Default : 60000 (1 minute)


Example
~~~~~~~

.. sourcecode:: bash

    connect.hbase.sink.column.family=d
    connect.hbase.sink.kcql=INSERT INTO person_hbase SELECT * FROM TOPIC1
    connector.class=com.datamountaineer.streamreactor.connect.hbase.HbaseSinkConnector
    tasks.max=1
    topics=TOPIC1
    name=hbase-test

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.

The HBase Sink will automatically write and update the HBase table if new fields are added to the Source topic,
if fields are removed the Kafka Connect framework will return the default value for this field, dependent of the
compatibility settings of the Schema registry. This value will be put into the HBase column family cell based on the
``connect.hbase.sink.fields`` mappings.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
