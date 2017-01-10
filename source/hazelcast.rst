Kafka Connect HazelCast
=======================

A Connector and Sink to write events from Kafka to HazelCast. The connector takes the value from the Kafka Connect
SinkRecords and inserts a new entry to a HazelCast reliable topic. The Sink only supports writing to reliable topics.

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>` - Kafka topic payload field selection is supported, allowing you to have choose selection of fields
   or all fields written to Hazelcast.
2. Topic to table routing via KCQL.
3. Error policies for handling failures.
4. Storing as JSON or Avro in Hazelcast via KCQL.

Prerequisites
-------------

- Confluent 3.0.1
- Hazelcast 3.6.4
- Java 1.8
- Scala 2.11

Setup
-----

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

HazelCast Setup
~~~~~~~~~~~~~~~

Download and install HazelCast from `here <https://hazelcast.org/staging-dl/>`__

When you download and extract the Hazelcast ZIP or TAR.GZ package, you will see 3 scripts under the ``/bin`` folder which
provide basic functionality for member and cluster management.

The following are the names and descriptions of each script:

- start.sh  - Starts a Hazelcast member with default configuration in the working directory.
- stop.sh   - Stops the Hazelcast member that was started in the current working directory.

Start HazelCast:

.. sourcecode:: bash

    ➜  bin/start.sh

    INFO: [10.128.137.102]:5701 [dev] [3.6.4] Address[10.128.137.102]:5701 is STARTING
    Aug 16, 2016 2:43:04 PM com.hazelcast.nio.tcp.nonblocking.NonBlockingIOThreadingModel
    INFO: [10.128.137.102]:5701 [dev] [3.6.4] TcpIpConnectionManager configured with Non Blocking IO-threading model: 3 input threads and 3 output threads
    Aug 16, 2016 2:43:07 PM com.hazelcast.cluster.impl.MulticastJoiner
    INFO: [10.128.137.102]:5701 [dev] [3.6.4]


    Members [1] {
        Member [10.128.137.102]:5701 this
    }

    Aug 16, 2016 2:43:07 PM com.hazelcast.core.LifecycleService
    INFO: [10.128.137.102]:5701 [dev] [3.6.4] Address[10.128.137.102]:5701 is STARTED

This will start Hazelcast with a default group called *dev* and password *dev-pass*


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

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for HazelCast.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create hazelcast-sink < conf/hazelcast-sink.properties

    #Connector name=`hazelcast-sink`
    name=hazelcast-sink
    connector.class=com.datamountaineer.streamreactor.connect.hazelcast.sink.HazelCastSinkConnector
    max.tasks=1
    topics = hazelcast-topic
    connect.hazelcast.sink.cluster.members=locallhost
    connect.hazelcast.sink.group.name=dev
    connect.hazelcast.sink.group.password=dev-pass
    connect.hazelcast.sink.kcql=INSERT INTO sink-test SELECT * FROM hazelcast-topic WITHFORMAT JSON BATCH 100
    #task ids: 0

The ``hazelcast-sink.properties`` configuration defines:

1.  The name of the sink.
2.  The Sink class.
3.  The max number of tasks the connector is allowed to created.
4.  The topics to read from (Required by framework)
5.  The name of the HazelCast host to connect to.
6.  The name of the group to connect to.
7.  The password for the group.
8.  :ref:`The KCQL routing querying. <kcql>`

If you switch back to the terminal you started the Connector in you should see the Hazelcast Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    hazelcast-sink


.. sourcecode:: bash


    (org.apache.kafka.clients.consumer.ConsumerConfig:178)
    [2016-08-20 16:45:39,518] INFO Kafka version : 0.10.0.0 (org.apache.kafka.common.utils.AppInfoParser:83)
    [2016-08-20 16:45:39,518] INFO Kafka commitId : b8642491e78c5a13 (org.apache.kafka.common.utils.AppInfoParser:84)
    [2016-08-20 16:45:39,520] INFO Created connector hazelcast-sink (org.apache.kafka.connect.cli.ConnectStandalone:91)
    [2016-08-20 16:45:39,520] INFO

        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
        __  __                 ________           __  _____ _       __
       / / / /___ _____  ___  / / ____/___ ______/ /_/ ___/(_)___  / /__
      / /_/ / __ `/_  / / _ \/ / /   / __ `/ ___/ __/\__ \/ / __ \/ //_/
     / __  / /_/ / / /_/  __/ / /___/ /_/ (__  ) /_ ___/ / / / / / ,<
    /_/ /_/\__,_/ /___/\___/_/\____/\__,_/____/\__//____/_/_/ /_/_/|_|


    by Andrew Stevenson
           (com.datamountaineer.streamreactor.connect.hazelcast.sink.HazelCastSinkTask:41)
    [2016-08-20 16:45:39,521] INFO HazelCastSinkConfig values:
        connect.hazelcast.connection.buffer.size = 32
        connect.hazelcast.connection.keep.alive = true
        connect.hazelcast.connection.tcp.no.delay = true
        connect.hazelcast.sink.group.password = [hidden]
        connect.hazelcast.connection.retries = 2
        connect.hazelcast.connection.linger.seconds = 3
        connect.hazelcast.sink.retry.interval = 60000
        connect.hazelcast.max.retires = 20
        connect.hazelcast.sink.batch.size = 1000
        connect.hazelcast.connection.reuse.address = true
        connect.hazelcast.sink.group.name = dev
        connect.hazelcast.sink.cluster.members = [192.168.99.100]
        connect.hazelcast.sink.error.policy = THROW
        connect.hazelcast.sink.kcql = INSERT INTO sink-test SELECT * FROM hazelcast-topic WITHFORMAT JSON BATCH 100
        connect.hazelcast.connection.timeout = 5000
     (com.datamountaineer.streamreactor.connect.hazelcast.config.HazelCastSinkConfig:178)
    Aug 20, 2016 4:45:39 PM com.hazelcast.core.LifecycleService
    INFO: HazelcastClient[dev-kafka-connect-05e64989-41d9-433e-ad21-b54894486384][3.6.4] is STARTING
    Aug 20, 2016 4:45:39 PM com.hazelcast.core.LifecycleService
    INFO: HazelcastClient[dev-kafka-connect-05e64989-41d9-433e-ad21-b54894486384][3.6.4] is STARTED
    Aug 20, 2016 4:45:39 PM com.hazelcast.client.spi.impl.ClientMembershipListener
    INFO:

    Members [1] {
        Member [172.17.0.2]:5701
    }

    Aug 20, 2016 4:45:39 PM com.hazelcast.core.LifecycleService
    INFO: HazelcastClient[dev-kafka-connect-05e64989-41d9-433e-ad21-b54894486384][3.6.4] is CLIENT_CONNECTED

Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``firstname`` field of type
string a ``lastname`` field of type string, an ``age`` field of type int and a ``salary`` field of type double.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic hazelcast-topic \
      --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.HazelCast"
      ,"fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"salary","type":"double"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}

Check for records in HazelCast
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this:

.. sourcecode:: bash

    [2016-08-20 16:53:58,608] INFO Received 1 records. (com.datamountaineer.streamreactor.connect.hazelcast.sink.HazelCastWriter:62)
    [2016-08-20 16:53:58,644] INFO Written 1 (com.datamountaineer.streamreactor.connect.hazelcast.sink.HazelCastWriter:71)

Now stop the connector.

Features
--------

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The HazelCast Sink supports the following:

.. sourcecode:: bash

    INSERT INTO <reliable topic> SELECT <fields> FROM <source topic> WITHFORMAT <JSON|AVRO> STOREAS <RELIABLE_TOPIC|RING_BUFFER> BATCH BATCH_SIZE

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO tableA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB, store as serialized avro encoded byte arrays, write in batches of 100
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB WITHFORMAT avro STOREAS RING_BUFFER BATCH 100

This is set in the ``connect.hazelcast.sink.kcql`` option.

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

The length of time the Sink will retry can be controlled by using the ``connect.hazelcast.sink.max.retries`` and the
``connect.hazelcast.sink.retry.interval``.

With Format
~~~~~~~~~~~

Hazelcast requires that data stored in collections and topics is serializable. The Sink offers two modes to store data.

*Avro* In this mode the Sink converts the SinkRecords from Kafka to Avro encoded byte arrays.
*Json* In this mode the Sink converts the SinkRecords from Kafka to Json strings and stores the resulting bytes.

This behaviour is controlled by the KCQL statement in the ``connect.hazelcast.sink.kcql`` option. The default
is JSON.

Configurations
--------------

``connect.hazelcast.sink.kcql``

KCQL expression describing field selection and routes.

* Data type : string
* Importance: high
* Optional  : no

``connect.hazelcast.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.hazelcast.sink.max.retries``
option. The ``connect.hazelcast.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high
* Optional: yes
* Default: ``throw``

``connect.hazelcast.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.hazelcast.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: medium
* Optional: yes
* Default: 10

``connect.hazelcast.sink.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.hazelcast.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Optional: yes
* Default : 60000 (1 minute)

``connect.hazelcast.sink.batch.size``

Specifies how many records to insert together at one time. If the connect framework provides less records when it is
calling the Sink it won't wait to fulfill this value but rather execute it.

* Type : int
* Importance : medium
* Optional: yes
* Defaults : 1000

``connect.hazelcast.sink.cluster.members``

Address List is the initial list of cluster addresses to which the client will connect. The client uses this list to
find an alive node. Although it may be enough to give only oneaddress of a node in the cluster (since all nodes
communicate with each other),it is recommended that you give the addresses for all the nodes.

* Data type : string
* Importance : high
* Optional: no
* Default: localhost

``connect.hazelcast.sink.group.name``

The group name of the connector in the target Hazelcast cluster.

* Data type : string
* Importance : high
* Optional: no
* Default: dev

``connect.hazelcast.sink.group.password``

The password for the group name.

* Data type : string
* Importance : high
* Optional  : yes
* Default	: dev-pass

``connect.hazelcast.connection.timeout``

Connection timeout is the timeout value in milliseconds for nodes to accept client connection requests.

* Data type : int
* Importance : low
* Optional  : yes
* Default	: 5000

``connect.hazelcast.connection.retries``

Number of times a client will retry the connection at startup.

* Data type : int
* Importance : low
* Optional  : yes
* Default	: 2

``connect.hazelcast.connection.keep.alive``

Enables/disables the SO_KEEPALIVE socket option. The default value is true.

* Data type : boolean
* Importance : low
* Optional  : yes
* Default	: true

``connect.hazelcast.connection.tcp.no.delay``

Enables/disables the SO_REUSEADDR socket option. The default value is true.

* Data type : boolean
* Importance : low
* Optional  : yes
* Default	: true

``connect.hazelcast.connection.linger.seconds``

Enables/disables SO_LINGER with the specified linger time in seconds. The default value is 3.

* Data type : int
* Importance : low
* Optional  : yes
* Default	: 3

``connect.hazelcast.connection.buffer.size``

Sets the SO_SNDBUF and SO_RCVBUF options to the specified value in KB for this Socket. The default value is 32.

* Data type : int
* Importance : low
* Optional  : yes
* Default	: 32

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.

The Sink serializes either an Avro or Json representation of the Sink record to the target reliable topic in Hazelcaset.
Hazelcast is agnostic to the schema.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO