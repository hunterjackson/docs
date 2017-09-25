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

- Confluent 3.3
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


Edit ``conf/hbase-site.xml`` and add the following content:

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

We you start the Confluent Platform, Kafka Connect is started in distributed mode (``confluent start``). 
In this mode a Rest Endpoint on port ``8083`` is exposed to accept connector configurations. 
We developed Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

HBase Table
~~~~~~~~~~~

The Sink expects a precreated table in HBase. In the HBase shell create the test table, go to your HBase install location.

.. sourcecode:: bash

    bin/hbase shell
    hbase(main):001:0> create 'person',{NAME=>'d', VERSIONS=>1}

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

Download, unpack and install the Stream Reactor and Confluent. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Once the Connect has started we can now use the kafka-connect-tools :ref:`cli <kafka-connect-cli>` to post in our distributed properties file for HBase.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/connect-cli create hbase-sink < conf/hbase-sink.properties

    #Connector name=`hbase-sink`
    name=person-hbase-test
    connector.class=com.datamountaineer.streamreactor.connect.hbase.HbaseSinkConnector
    tasks.max=1
    topics=hbase-topic
    connect.hbase.column.family=d
    connect.hbase.kcql=INSERT INTO person SELECT * FROM hbase-topic PK firstName, lastName
    #task ids: 0

This ``hbase-sink.properties`` configuration defines:

1.  The name of the sink.
2.  The Sink class.
3.  The max number of tasks the connector is allowed to created. Should not be greater than the number of partitions in the Source topics
    otherwise tasks will be idle.
4.  The Source kafka topics to take events from.
5.  The HBase column family to write to.
6.  :ref:`The KCQL routing querying. <kcql>`

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

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``firstname`` field of type string,
a ``lastname`` field of type string, an ``age`` field of type int and a ``salary`` field of type double.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic hbase-topic \
      --property value.schema='{"type":"record","name":"User",
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

    hbase(main):004:0* scan 'person'
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

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The HBase Sink supports the following:

.. sourcecode:: bash

    INSERT INTO <table> SELECT <fields> FROM <source topic> <PK> primary_key_cols

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA and use the default rowkey (topic name, partition, offset)
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB, use field y from the topic as the row key
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB PK y

This is set in the ``connect.hbase.kcql`` option.

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

The length of time the Sink will retry can be controlled by using the ``connect.hbase.max.retries`` and the
``connect.hbase.retry.interval``.


Configurations
--------------

``connect.hbase.column.family``

The hbase column family.

* Type: string
* Importance: high
* Optional: no

``connect.hbase.kcql``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming. Fields
to be used as the row key can be set by specifing the ``PK``. The below example uses field1 and field2 are the row key.

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT * FROM TOPIC2 PK field1, field2

If no primary keys are specified the topic name, partition and offset converted to bytes are used as the HBase rowkey.

* Type: string
* Importance: high
* Optional: no

``connect.hbase.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.hbase.max.retries``
option. The ``connect.hbase.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: medium
* Optional: yes
* Default: RETRY

``connect.hbase.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.hbase.error.policy`` is set to ``retry``.

* Type: string
* Importance: medium
* Optional: yes
* Default: 10

``connect.hbase.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.hbase.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Optional: yes
* Default : 60000 (1 minute)

``connect.progress.enabled``

Enables the output for how many records have been processed.

* Type: boolean
* Importance: medium
* Optional: yes
* Default : false

Example
~~~~~~~

.. sourcecode:: bash

    connect.hbase.column.family=d
    connect.hbase.kcql=INSERT INTO person SELECT * FROM TOPIC1
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
``connect.hbase.kcql`` value.

Deployment Guidelines
---------------------

Ensure the ``hbase-site.xml`` is on the the classpath of the connector.

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
