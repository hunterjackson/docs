Kafka Connect JMS
=======================

Kafka Connect JMS is a Sink Connector for reading data from
a Kafka topic and writing the payload to a JMS queue/topic.

Prerequisites
-------------
-  Confluent 3.0.0
-  Java 1.8
-  Scala 2.11
-  A JMS framework (ActiveMQ for example)

Setup
-----

Before we can do anything, including the QuickStart we need to install the Confluent platform.
For ActiveMQ follow http://activemq.apache.org/getting-started.html for the instruction of setting
it up.


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

The prebuilt jars can be taken from here and
`here <https://github.com/datamountaineer/kafka-connect-tools/releases>`__
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

Sink Connector
----------------

The JMS sink connector allows you to extract entries from a Kafka topic with the CQL driver and pass them to a JMS topic/queue.
The connector allows you to specify the payload type sent to the JMS target:

1. JSON
2. AVRO
3. MAP
4. OBJECT

Sink Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Next we start the connector in standalone mode. This is useful for testing
one of jobs, usually you'd run in distributed mode to get fault tolerance and better performance.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Since we are in standalone mode we'll create a file called ``jms-sink.properties`` with the contents below:

.. sourcecode:: bash

    connector.class=com.datamountaineer.streamreactor.connect.jms.sink.JMSSinkConnector
    tasks.max=1
    topics=person_jms
    name=person-jms-test

    connect.jms.sink.url=tcp://somehost:61616
    connect.jms.sink.connection.factory=org.apache.activemq.ActiveMQConnectionFactory
    connect.jms.sink.export.route.query=INSERT INTO topic_1 SELECT * FROM person_jms
    connect.jms.sink.message.type=AVRO
    connect.jms.sink.export.route.topics=person_jms
    connect.jms.sink.export.route.queues=
    connect.jms.error.policy=THROW

This configuration defines:

1.  The name of the sink.
2.  The sink class.
3.  The max number of tasks the connector is allowed to created. Should not be greater than the number of partitions in
    the source topics otherwise tasks will be idle.
4.  The source kafka topics to take events from.


Starting the Sink Connector (Standalone)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now we are ready to start the JMS sink Connector in standalone mode.

.. note::

    You need to add the connector to your classpath or you can create a folder in ``share/java`` of the Confluent
    install location like, kafka-connect-myconnector and the start scripts provided by Confluent will pick it up.
    The start script looks for folders beginning with kafka-connect.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-jms-0.1-all.jar
    #Start the connector in standalone mode, passing in two properties files, the first for the schema registry, kafka
    #and zookeeper and the second with the connector properties.
    ➜  bin/connect-standalone etc/schema-registry/connect-avro-standalone.properties jms-sink.properties

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    ➜ java -jar build/libs/kafka-connect-cli-0.2-all.jar get jms-sink


Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``id`` field of type int
and a ``random_field`` of type string.

.. sourcecode:: bash

    bin/kafka-avro-console-producer \
    > --broker-list localhost:9092 --topic jms_test \
    > --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.jms","fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"salary","type":"double"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}
    {"firstName": "Anna", "lastName": "Jones", "age":28, "salary": 5430}

Now check for records in ActiveMQ.

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

Once the connector has started lets use the kafka-connect-tools cli to post in our distributed properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.2-all.jar create jms-sink < jms-sink.properties

If you switch back to the terminal you started the Connector in you should see the JMS sink being accepted and the task
starting.

Insert the records as before to have them written to JMS.

Features
--------

The JMS sink writes records from Kafka to a JMS queue.

The sink supports:

The sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to have choose selection of fields
   or all fields written to Kudu.
2. Topic to topic routing.
3. Payload format selection.
4. Error policies for handling failures.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The JMS sink supports the following:

.. sourcecode:: bash

    INSERT INTO <jms target> SELECT <fields> FROM <source topic>

Example:

.. sourcecode:: sql

    #select all fields from topicA and write to jmsA
    INSERT INTO jmsA SELECT * FROM topicA

    #select 3 fields and rename from topicB and write to jmsB
    INSERT INTO jmsB SELECT x AS a, y AS b and z AS c FROM topicB


JMS Payload
~~~~~~~~~~~

When a message is sent to a JMS target it can be one of the following:

1.JSON - it will send a TextMessage;
2.AVRO -send a BytesMessage;
3.MAP - it will send a MapMessage;
4.OBJECT - it will send an ObjectMessage

Topic Routing
~~~~~~~~~~~~~

The sink supports topic routing that allows mapping the messages from topics to a specific jms target. For example, map a
topic called "bloomberg_prices" to a jms target named "prices". This mapping is set in the ``connect.jms.sink.export.route.query``
option.

Example:

.. sourcecode:: sql

    //Select all
    INSERT INTO jms1 SELECT * FROM topic1; INSERT INTO jms3 SELECT * FROM topicCConfigurations

Configurations
--------------

``connect.jms.sink.url``

Provides the JMS broker url

* Data Type: string
* Optional : no

``connect.jms.sink.user``

Provides the user for the JMS connection.

* Data Type: string
* Optional : no

``connect.jms.sink.password``

Provides the password for the JMS connection.

* Data Type: string
* Optional : no

``connect.jms.sink.connection.factory``

Provides the full class name for the ConnectionFactory implementation to use.

* Data Type: string
* Optional : no

``connect.jms.sink.export.route.query``

KCQL expression describing field selection and routes.

* Data Type: string
* Optional : no

``connect.jms.sink.export.route.topics``

Lists all the jms target topics.

* Data Type: list (comma separated strings)
* Optional : yes

``connect.jms.sink.export.route.queue``

Lists all the jms target queues.

* Data Type: list (comma separated strings)
* Optional : yes

``connect.jms.sink.message.type``

Specifies the JMS payload. If JSON is chosen it will send a TextMessage.
- AVRO is chosen it will send a BytesMessage.
- MAP is chosen it will send a MapMessage.
- OBJECT is chosen it will send an ObjectMessage.

* Data Type: string
* Optional : yes
* Default : AVRO


``connect.jms.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.jms.sink.max.retries``
option. The ``connect.jms.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high

``connect.jms.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.jms.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: high
* Default: 10


``connect.jms.sink.retry.interval``

The interval, in milliseconds between retries if the sink is using ``connect.jms.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Default : 60000 (1 minute)


Schema Evolution
----------------

Not applicable.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO
