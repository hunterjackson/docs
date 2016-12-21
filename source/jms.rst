Kafka Connect JMS
=======================

The JMS Sink connector allows you to extract entries from a Kafka topic with the CQL driver and pass them to a JMS topic/queue.
The connector allows you to specify the payload type sent to the JMS target:

1. JSON
2. AVRO
3. MAP
4. OBJECT

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>`. Kafka topic payload field selection is supported, allowing you to select fields written to the queue or topic in JMS.
2. Topic to topic routing via KCQL.
3. Payload format selection via KCQL.
4. Error policies for handling failures.

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

Follow the instructions :ref:`here <install>`.

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

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for JMS.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/cli.sh create jms-sink < conf/jms-sink.properties
    #Connector name=`jms-sink`
    name=jms-sink
    connector.class=com.datamountaineer.streamreactor.connect.jms.sink.JMSSinkConnector
    tasks.max=1
    topics=person_jms
    connect.jms.sink.url=tcp://somehost:61616
    connect.jms.sink.connection.factory=org.apache.activemq.ActiveMQConnectionFactory
    connect.jms.sink.export.route.query=INSERT INTO topic_1 SELECT * FROM person_jms
    connect.jms.sink.message.type=AVRO
    connect.jms.error.policy=THROW
    connect.jms.sink.export.route.topics=topic_1
    #task ids: 0

The ``jms-sink.properties`` file defines:

1.  The name of the sink.
2.  The Sink class.
3.  The max number of tasks the connector is allowed to created.
4.  The Kafka topics to take events from.
5.  The JMS url.
6.  The factory class for the JSM endpoint.
7.  :ref:`The KCQL routing querying. <kcql>`
8.  The message type storage format.
9.  The error policy.
10. The list of target topics (must match the targets set in ``connect.jms.sink.export.route.query``

If you switch back to the terminal you started the Connector in you should see the JMS Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
    jms-sink


Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``id`` field of type int
and a ``random_field`` of type string.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
     --broker-list localhost:9092 --topic jms_test \
     --property value.schema='{"type":"record","name":"User","namespace":"com.datamountaineer.streamreactor.connect.jms",
    "fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"salary","type":"double"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}
    {"firstName": "Anna", "lastName": "Jones", "age":28, "salary": 5430}

Now check for records in ActiveMQ.

Now stop the connector.


Features
--------

The Sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to select fields written to the queue or topic in JMS.
2. Topic to topic routing.
3. Payload format selection.
4. Error policies for handling failures.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option.

The JMS Sink supports the following:

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

1.  JSON -   Send a TextMessage;
2.  AVRO -   Send a BytesMessage;
3.  MAP -    Send a MapMessage;
4.  OBJECT - Send an ObjectMessage

Topic Routing
~~~~~~~~~~~~~

The Sink supports topic routing that allows mapping the messages from topics to a specific jms target. For example, map a
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
* Importance: high
* Optional : no

``connect.jms.sink.user``

Provides the user for the JMS connection.

* Data Type: string
* Importance: high
* Optional : no

``connect.jms.sink.password``

Provides the password for the JMS connection.

* Data Type: string
* Importance: high
* Optional : no

``connect.jms.sink.connection.factory``

Provides the full class name for the ConnectionFactory implementation to use.

* Data Type: string
* Importance: high
* Optional : no

``connect.jms.sink.export.route.query``

KCQL expression describing field selection and routes.

* Data Type: string
* Importance: high
* Optional : no

``connect.jms.sink.export.route.topics``

Lists all the jms target topics.

* Data Type: list (comma separated strings)
* Importance: medium
* Optional : yes

``connect.jms.sink.export.route.queue``

Lists all the jms target queues.

* Data Type: list (comma separated strings)
* Importance: medium
* Optional : yes

``connect.jms.sink.message.type``

Specifies the JMS payload. If JSON is chosen it will send a TextMessage.

* Data Type: string
* Importance: medium
* Optional : yes
* Default : AVRO

``connect.jms.sink.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.jms.sink.max.retries``
option. The ``connect.jms.sink.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: medium
* Optional: yes
* Default: RETRY

``connect.jms.sink.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.jms.sink.error.policy`` is set to ``retry``.

* Type: string
* Importance: medium
* Optional: yes
* Default: 10

``connect.jms.sink.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.jms.sink.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
* Optional: yes
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
