Kafka Connect Redis
===================

A Connector and Sink to write events from Kafka to Redis. The connector takes the value from the Kafka Connect
SinkRecords and inserts a new entry to Redis.

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>` - Kafka topic payload field selection is supported, allowing you to select fields written to Redis.
2. Topic to table routing via KCQL.
3. RowKey selection - Selection of fields to use as the row key, if none specified the topic name, partition and offset are
   used via KCQL.
4. Error policies for handling failures.
5. Storing as one or more Stored Sets.

Prerequisites
-------------

- Confluent 3.2
- Jedis 2.8.1
- Java 1.8
- Scala 2.11

Setup
-----

Redis Setup
~~~~~~~~~~~

Download and install Redis.

.. sourcecode:: bash

    ➜  wget http://download.redis.io/redis-stable.tar.gz
    ➜  tar xvzf redis-stable.tar.gz
    ➜  cd redis-stable
    ➜  sudo make install


Start Redis

.. sourcecode:: bash

    ➜  bin/redis-server

Check Redis is running:

.. sourcecode:: bash

    ➜  redis-cli ping
        PONG
    ➜  sudo service redis-server status


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

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor and Confluent. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Once the Connect has started we can now use the kafka-connect-tools :ref:`cli <kafka-connect-cli>` to post in our distributed properties file for Redis.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/connect-cli create redis-sink < conf/redis-sink.properties
    #Connector name=`redis-sink`
    connect.redis.host=localhost
    connect.redis.port=6379
    connector.class=com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkConnector
    tasks.max=1
    topics=redis-topic
    connect.redis.kcql=INSERT INTO TABLE1 SELECT * FROM redis-topic
    #task ids: 0

The ``redis-sink.properties`` file defines:

1.  The name of the sink.
2.  The name of the redis host to connect to.
3.  The redis port to connect to.
4.  The Sink class.
5.  The max number of tasks the connector is allowed to created. Should not be greater than the number of partitions in
    the Source topics otherwise tasks will be idle.
6.  The Source kafka topics to take events from.
7.  :ref:`The KCQL routing querying. <kcql>`


.. warning::

    If your redis server is requiring the connection to be authenticated you will need to provide an extra setting:

    .. sourcecode:: bash

        connect.redis.connection.password=$REDIS_PASSWORD

    Don't set the value to empty if no password is required.

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
    redis-sink

.. sourcecode:: bash

    [2016-05-08 22:37:05,616] INFO
        ____        __        __  ___                  __        _
       / __ \____ _/ /_____ _/  |/  /___  __  ______  / /_____ _(_)___  ___  ___  _____
      / / / / __ `/ __/ __ `/ /|_/ / __ \/ / / / __ \/ __/ __ `/ / __ \/ _ \/ _ \/ ___/
     / /_/ / /_/ / /_/ /_/ / /  / / /_/ / /_/ / / / / /_/ /_/ / / / / /  __/  __/ /
    /_____/\__,_/\__/\__,_/_/  /_/\____/\__,_/_/ /_/\__/\__,_/_/_/ /_/\___/\___/_/
        ____           ___      _____ _       __
       / __ \___  ____/ (_)____/ ___/(_)___  / /__
      / /_/ / _ \/ __  / / ___/\__ \/ / __ \/ //_/
     / _, _/  __/ /_/ / (__  )___/ / / / / / ,<
    /_/ |_|\___/\__,_/_/____//____/_/_/ /_/_/|_|


     (com.datamountaineer.streamreactor.connect.redis.sink.config.RedisSinkConfig:165)
    [2016-05-08 22:37:05,641] INFO Settings:
    RedisSinkSettings(RedisConnectionInfo(localhost,6379,None),RedisKey(FIELDS,WrappedArray(firstName, lastName)),PayloadFields(false,Map(firstName -> firstName, lastName -> lastName, age -> age, salary -> income)))
           (com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkTask:65)
    [2016-05-08 22:37:05,687] INFO Sink task org.apache.kafka.connect.runtime.WorkerSinkTask@44b24eaa finished initialization and start (org.apache.kafka.connect.runtime.WorkerSinkTask:155)


Test Records
^^^^^^^^^^^^

Now we need to put some records it to the test_table topics. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema has a ``firstname`` field of type
string, a ``lastname`` field of type string, an ``age`` field of type int and a ``salary`` field of type double.

.. sourcecode:: bash

    ${CONFLUENT_HOME}/bin/kafka-avro-console-producer \
      --broker-list localhost:9092 --topic redis-topic \
      --property value.schema='{"type":"record","name":"User",
      "fields":[{"name":"firstName","type":"string"},{"name":"lastName","type":"string"},{"name":"age","type":"int"},{"name":"salary","type":"double"}]}'

Now the producer is waiting for input. Paste in the following:

.. sourcecode:: bash

    {"firstName": "John", "lastName": "Smith", "age":30, "salary": 4830}

Check for records in Redis
~~~~~~~~~~~~~~~~~~~~~~~~~~

Now check the logs of the connector you should see this:

.. sourcecode:: bash

    INFO Received record from topic:redis-topic partition:0 and offset:0 (com.datamountaineer.streamreactor.connect.redis.sink.writer.RedisDbWriter:48)
    INFO Empty list of records received. (com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkTask:75)

Check in Redis.

.. sourcecode:: bash

    redis-cli

    127.0.0.1:6379> keys *
    1) "John.Smith"
    2) "11"
    3) "10"
    127.0.0.1:6379>
    127.0.0.1:6379> get "John.Smith"
    "{\"firstName\":\"John\",\"lastName\":\"Smith\",\"age\":30,\"income\":4830.0}"


Now stop the connector.

Features
--------

The Redis Sink writes records from Kafka to Redis.

The Sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to select fields written to Redis.
2. Topic to table routing.
3. RowKey selection - Selection of fields to use as the row key, if none specified the topic name, partition and offset are
   used.
4. Error policies for handling failures.
5. Sorted sets/Cache modes

Cache Mode
~~~~~~~~~~

The purpose of this mode is to **cache** in Redis [*Key-Value*] pairs. Imagine a Kafka topic with currency foreign exchange rate messages:

.. sourcecode:: bash

    { "symbol": "USDGBP" , "price": 0.7943 }
    { "symbol": "EURGBP" , "price": 0.8597 }

You may want to store in Redis: the **symbol** as the `Key` and the **price** as the `Value`. This will effectively make Redis a **caching** system, 
which multiple other application can access to get the *(latest)* value. To achieve that using this particular Kafka Redis Sink Connector, you need 
to specify the **KCQL** as:

.. sourcecode:: sql

    SELECT price from yahoo-fx PK symbol

This will update the keys `USDGBP` , `EURGBP` with the relevant price using the (default) Json format:

.. sourcecode:: bash

    Key=EURGBP  Value={ "price": 0.7943 }

We can prefix the name of the `Key` using the INSERT statement:

.. sourcecode:: sql

    INSERT INTO FX- SELECT price from yahoo-fx PK symbol

This will create key with names `FX-USDGBP` , `FX-EURGBP` etc.

Sorted Sets
~~~~~~~~~~~

To **insert** messages from a Kafka topic into 1 Sorted Set (SS) use the following **KCQL** syntax:

.. sourcecode:: sql

    INSERT INTO cpu_stats SELECT * from cpuTopic STOREAS SortedSet(score=timestamp)

This will create and add entries into the (sorted set) named **cpu_stats**. The entries will be ordered in the Redis set based on the `score` 
that we define it to be the value of the `timestamp` field of the Avro message from Kafka. In the above example we are selecting and storing all 
the fields of the Kafka message.

Multiple Sorted Sets
~~~~~~~~~~~~~~~~~~~~

You can create multiple sorted sets by promoting each value of **one field** from the Kafka message into one Sorted Set (SS) and selecting 
which values to store into the sorted-sets. You can achieve that by using the KCQL syntax and defining with the filed using **PK** (primary key)

 .. sourcecode:: sql

    SELECT temperature, humidity FROM sensorsTopic PK sensorID STOREAS SortedSet(score=timestamp)

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L** anguage found here `GitHub repo <https://github.com/datamountaineer/kafka-connector-query-language>`_
allows for routing and mapping using a SQL like syntax, consolidating typically features in to one configuration option. This is set in the ``connect.redis.kcql`` option.

The Redis Sink supports the following:

.. sourcecode:: bash

    INSERT INTO cache|sortedSet SELECT * FROM TOPIC [PK FIELD] [STOREDAS SortedSet(key=FIELD)]

.. sourcecode:: bash

    #insert messages from a Kafka topic into 1 Sorted Set (SS) named cpuTopic and ordered by score with the value of the timestamp field in the message
    INSERT INTO cpu_stats SELECT * from cpuTopic STOREAS SortedSet(score=timestamp)

    #insert into multiple sorted sets by setting the PK key word to select a field from the message as a primary key
    INSERT INTO cpu_stats SELECT temperature, humidity FROM sensorsTopic PK sensorID STOREAS SortedSet(score=timestamp)


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

The length of time the Sink will retry can be controlled by using the ``connect.redis.max.retries`` and the
``connect.redis.retry.interval``.

Configurations
--------------

``connect.redis.kcql``

Kafka connect query language expression. Allows for expressive topic to table routing, field selection and renaming. Fields
to be used as the row key can be set by specifing the ``PK``. The below example uses field1 as the primary key.

* Data type : string
* Importance: high
* Optional  : no

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT * FROM TOPIC2 PK field1

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT * FROM TOPIC2 PK field1, field2

``connect.redis.host``

Specifies the Redis server.

* Data type : string
* Importance: high
* Optional  : no

``connect.redis.port``

Specifies the Redis server port number.

* Data type : int
* Importance: high
* Optional  : no

``connect.redis.password``

Specifies the authorization password.

* Data type : string
* Importance: high
* Optional  : yes
* Description: If you don't have a password set up on the redis server don't provide the value or you will see this error: "ERR Client sent AUTH, but no password is set"

``connect.redis.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **noop**, the error is swallowed, **throw**, the error is allowed to propagate and retry.
For **retry** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.redis.max.retries``
option. The ``connect.redis.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: medium
* Optional: yes
* Default: RETRY


``connect.redis.max.retries``

The maximum number of times a message is retried. Only valid when the ``connect.redis.error.policy`` is set to ``retry``.

* Type: string
* Importance: medium
* Optional: yes
* Default: 10


``connect.redis.retry.interval``

The interval, in milliseconds between retries if the Sink is using ``connect.redis.error.policy`` set to **RETRY**.

* Type: int
* Importance: high
* Optional: no
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

    name=redis-sink
    connect.redis.host=localhost
    connect.redis.port=6379
    connector.class=com.datamountaineer.streamreactor.connect.redis.sink.RedisSinkConnector
    tasks.max=1
    topics=redis-topic
    connect.redis.kcql=INSERT INTO TABLE1 SELECT * FROM redis-topic

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal
or fields, data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules.
More information can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.

The Redis Sink will automatically write and update the Redis table if new fields are added to the Source topic,
if fields are removed the Kafka Connect framework will return the default value for this field, dependent of the
compatibility settings of the Schema registry. This value will be put into the Redis column family cell based on the
``connect.redis.kcql`` mappings.

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

Please review the :ref:`FAQs <faq>` and join our `slack channel <https://slackpass.io/datamountaineers>`_.

