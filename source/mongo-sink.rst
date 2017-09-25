Kafka Connect Mongo Sink
========================

The Mongo Sink allows you to write events from Kafka to your MongoDB instance. The connector converts the value from the Kafka
Connect SinkRecords to MongoDB Document and will do an insert or upsert depending on the configuration you chose. It is expected the
database is created upfront; the targeted MongoDB collections will be created if they don't exist

.. note:: The database needs to be created upfront!

The Sink supports:

1. :ref:`The KCQL routing querying <kcql>` - Topic to measure mapping and Field selection.
2. Schema registry support for Connect/Avro with a schema.
3. Schema registry support for Connect and no schema (schema set to Schema.String)
4. Json payload support, no Schema Registry.
5. Error policies.
6. Payload support for Schema.Struct and payload Struct, Schema.String and Json payload and Json payload with no schema

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
over Kafka and bringing them in line with best practices is quite a challenge. Hence we added the support.

Prerequisites
-------------

-  MongoDB 3.2.10
- Confluent 3.3
-  Java 1.8
-  Scala 2.11

Setup
-----

Before we can do anything, including the QuickStart we need to install MongoDb and the Confluent platform.

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

MongoDb Setup
~~~~~~~~~~~~~

If you already have an instance of Mongo running you can skip this step.
First download and install MongoDb Community edition. This is the manual approach for installing on Ubuntu. You can
follow the details https://docs.mongodb.com/v3.2/administration/install-community/ for your OS.

.. sourcecode:: bash

    #go to home folder
    ➜  cd ~
    #make a folder for mongo
    ➜  mkdir mongodb

    #Download Mongo
    ➜  wget wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-ubuntu1604-3.2.10.tgz

    #extract the archive
    ➜  tar xvf mongodb-linux-x86_64-ubuntu1604-3.2.10.tgz -C mongodb
    ➜  cd mongodb
    ➜  mv mongodb-linux-x86_64-ubuntu1604-3.2.10/* .

    #create the data folder
    ➜  mkdir data
    ➜  mkdir data/db

    #Start MongoDb
    ➜  bin/mongod --dbpath data/db

Sink Connector QuickStart
~~~~~~~~~~~~~~~~~~~~~~~~~

We you start the Confluent Platform, Kafka Connect is started in distributed mode (``confluent start``). 
In this mode a Rest Endpoint on port ``8083`` is exposed to accept connector configurations. 
We developed Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

The important configuration for Connect is related to the key and value deserializer. In the first example we default to the
best practice where the source sends Avro messages to a Kafka topic. It is not enough to just be Avro messages but also the producer
must work with the Schema Registry to create the schema if it doesn't exist and set the schema id in the message.
Every message sent will have a magic byte followed by the Avro schema id and then the actual Avro record in binary format.

Here are the entries in the config setting all the above. The are placed in the ``connect-properties`` file Kafka Connect is started with.
Of course if your SchemaRegistry runs on a different machine or you have multiple instances of it you will have to amend the configuration.

.. sourcecode:: bash

    key.converter=io.confluent.connect.avro.AvroConverter
    key.converter.schema.registry.url=http://localhost:8081
    value.converter=io.confluent.connect.avro.AvroConverter
    value.converter.schema.registry.url=http://localhost:8081

Test Database
~~~~~~~~~~~~~

The Sink requires that a database be precreated in MongoDB.

.. sourcecode:: bash

    #from a new terminal
    ➜  cd ~/mongodb/bin

    #start the cli
    ➜  ./mongo

    #list all dbs
    ➜  show dbs

    #create a new database named connect
    ➜  use connect
    #create a dummy collection and insert one document to actually create the database
    ➜  db.dummy.insert({"name":"Kafka Rulz!"})

    #list all dbs
    ➜  show dbs


Starting the Connector
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor and Confluent. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Once the Connect has started we can now use the kafka-connect-tools :ref:`cli <kafka-connect-cli>` to post in our distributed properties file for Kudu.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

     ➜  bin/connect-cli create mongo-sink < conf/source.kcql/mongo-sink.properties

    #Connector `mongo-sink-orders`:
    name=mongo-sink-orders
    connector.class=com.datamountaineer.streamreactor.connect.mongodb.sink.MongoSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.mongo.kcql=INSERT INTO orders SELECT * FROM orders-topic
    connect.mongo.database=connect
    connect.mongo.connection=mongodb://localhost:27017
    connect.mongo.batch.size=10

    #task ids: 0

If you switch back to the terminal you started Kafka Connect in you should see the Mongo Sink being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/connect-cli ps
    mongo-sink


.. sourcecode:: bash

    [2016-11-06 22:25:29,354] INFO MongoConfig values:
        connect.mongo.retry.interval = 60000
        connect.mongo.kcql = INSERT INTO orders SELECT * FROM orders-topic
        connect.mongo.connection = mongodb://localhost:27017
        connect.mongo.error.policy = THROW
        connect.mongo.database = connect
        connect.mongo.sink.batch.size = 10
        connect.mongo.max.retires = 20
     (com.datamountaineer.streamreactor.connect.mongodb.config.MongoConfig:178)
    [2016-11-06 22:25:29,399] INFO
      ____        _        __  __                   _        _
     |  _ \  __ _| |_ __ _|  \/  | ___  _   _ _ __ | |_ __ _(_)_ __   ___  ___ _ __
     | | | |/ _` | __/ _` | |\/| |/ _ \| | | | '_ \| __/ _` | | '_ \ / _ \/ _ \ '__|
     | |_| | (_| | || (_| | |  | | (_) | |_| | | | | || (_| | | | | |  __/  __/ |
     |____/ \__,_|\__\__,_|_|  |_|\___/ \__,_|_| |_|\__\__,_|_|_| |_|\___|\___|_|
      __  __                         ____  _       ____  _       _ by Stefan Bocutiu
     |  \/  | ___  _ __   __ _  ___ |  _ \| |__   / ___|(_)_ __ | | __
     | |\/| |/ _ \| '_ \ / _` |/ _ \| | | | '_ \  \___ \| | '_ \| |/ /
     | |  | | (_) | | | | (_| | (_) | |_| | |_) |  ___) | | | | |   <
     |_|  |_|\___/|_| |_|\__, |\___/|____/|_.__/  |____/|_|_| |_|_|\_\
    . (com.datamountaineer.streamreactor.connect.mongodb.sink.MongoSinkTask:51)
    [2016-11-06 22:25:29,990] INFO Initialising Mongo writer.Connection to mongodb://localhost:27017 (com.datamountaineer.streamreactor.connect.mongodb.sink.MongoWriter$:126)


Test Records
^^^^^^^^^^^^

.. hint::

    If your input topic doesn't match the target use Kafka Streams to transform in realtime the input. Also checkout the
    `Plumber <https://github.com/rollulus/kafka-streams-plumber>`__, which allows you to inject a Lua script into
    `Kafka Streams <http://www.confluent.io/blog/introducing-kafka-streams-stream-processing-made-simple>`__ to do this,
    no Java or Scala required!

Now we need to put some records it to the orders-topic. We can use the ``kafka-avro-console-producer`` to do this.

Start the producer and pass in a schema to register in the Schema Registry. The schema matches the table created earlier.

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

Now if we check the logs of the connector we should see 2 records being inserted to MongoDB:

.. sourcecode:: bash

    [2016-11-06 22:30:30,473] INFO Setting newly assigned partitions [orders-topic-0] for group connect-mongo-sink-orders (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:231)
    [2016-11-06 22:31:29,328] INFO WorkerSinkTask{id=mongo-sink-orders-0} Committing offsets (org.apache.kafka.connect.runtime.WorkerSinkTask:261)

.. sourcecode:: bash

    #Open a new terminal and navigate to the mongodb instalation folder
    ➜ ./bin/mongo
        > show databases
            connect  0.000GB
            local    0.000GB
        > use connect
            switched to db connect
        > show collections
            dummy
            orders
        > db.orders.find()
        { "_id" : ObjectId("581fb21b09690a24b63b35bd"), "id" : 1, "created" : "2016-05-06 13:53:00", "product" : "OP-DAX-P-20150201-95.7", "price" : 94.2 }
        { "_id" : ObjectId("581fb2f809690a24b63b35c2"), "id" : 2, "created" : "2016-05-06 13:54:00", "product" : "OP-DAX-C-20150201-100", "price" : 99.5 }
        { "_id" : ObjectId("581fb2f809690a24b63b35c3"), "id" : 3, "created" : "2016-05-06 13:55:00", "product" : "FU-DATAMOUNTAINEER-20150201-100", "price" : 10000 }
        { "_id" : ObjectId("581fb2f809690a24b63b35c4"), "id" : 4, "created" : "2016-05-06 13:56:00", "product" : "FU-KOSPI-C-20150201-100", "price" : 150 }


Bingo, our 4 rows!


Legacy topics (plain text payload with a json string)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We have found some of the clients have already an infrastructure where they publish pure json on the topic and obviously the jump to follow
the best practice and use schema registry is quite an ask. So we offer support for them as well.

This time we need to start the connect with a different set of settings.

.. sourcecode:: bash

      #create a new configuration for connect
      ➜ cp  etc/schema-registry/connect-avro-distributed.properties etc/schema-registry/connect-avro-distributed-json.properties
      ➜ vi etc/schema-registry/connect-avro-distributed-json.properties

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
      ➜   $CONFLUENT_HOME/bin/confluent stop
      ➜   $CONFLUENT_HOME/bin/confluent start

Use the ``CLI`` to remove the old MongoDB Sink:

.. sourcecode:: bash

    ➜ bin/connect-cli rm  mongo-sink

and start the new Sink with the json properties files to read from the a different topic with json as the payload.

.. sourcecode:: bash

     #start the connector for mongo
    ➜   bin/connect-cli create mongo-sink-orders-json < mongo-sink-orders-json.properties

Use the Confluent CLI to view Connects logs.

.. sourcecode:: bash

    # Get the logs from Connect
    confluent log connect

    # Follow logs from Connect
    confluent log connect -f

.. sourcecode:: bash

        [2016-11-06 23:53:09,881] INFO MongoConfig values:
            connect.mongo.retry.interval = 60000
            connect.mongo.kcql = UPSERT INTO orders_json SELECT id, product as product_name, price as value FROM orders-topic-json PK id
            connect.mongo.connection = mongodb://localhost:27017
            connect.mongo.error.policy = THROW
            connect.mongo.db = connect
            connect.mongo.batch.size = 10
            connect.mongo.max.retires = 20
         (com.datamountaineer.streamreactor.connect.mongodb.config.MongoConfig:178)
        [2016-11-06 23:53:09,927] INFO
          ____        _        __  __                   _        _
         |  _ \  __ _| |_ __ _|  \/  | ___  _   _ _ __ | |_ __ _(_)_ __   ___  ___ _ __
         | | | |/ _` | __/ _` | |\/| |/ _ \| | | | '_ \| __/ _` | | '_ \ / _ \/ _ \ '__|
         | |_| | (_| | || (_| | |  | | (_) | |_| | | | | || (_| | | | | |  __/  __/ |
         |____/ \__,_|\__\__,_|_|  |_|\___/ \__,_|_| |_|\__\__,_|_|_| |_|\___|\___|_|
          __  __                         ____  _       ____  _       _ by Stefan Bocutiu
         |  \/  | ___  _ __   __ _  ___ |  _ \| |__   / ___|(_)_ __ | | __
         | |\/| |/ _ \| '_ \ / _` |/ _ \| | | | '_ \  \___ \| | '_ \| |/ /
         | |  | | (_) | | | | (_| | (_) | |_| | |_) |  ___) | | | | |   <
         |_|  |_|\___/|_| |_|\__, |\___/|____/|_.__/  |____/|_|_| |_|_|\_\
        . (com.datamountaineer.streamreactor.connect.mongodb.sink.MongoSinkTask:51)
        [2016-11-06 23:53:10,270] INFO Initialising Mongo writer.Connection to mongodb://localhost:27017 (com.datamountaineer.streamreactor.connect.mongodb.sink.MongoWriter$:126)


Now it's time to produce some records. This time we will use the simple kafka-consoler-consumer to put simple json on the topic:

.. sourcecode:: bash

    ➜ ${CONFLUENT_HOME}/bin/kafka-console-producer --broker-list localhost:9092 --topic orders-topic-json

    {"id": 1, "created": "2016-05-06 13:53:00", "product": "OP-DAX-P-20150201-95.7", "price": 94.2}
    {"id": 2, "created": "2016-05-06 13:54:00", "product": "OP-DAX-C-20150201-100", "price": 99.5}
    {"id": 3, "created": "2016-05-06 13:55:00", "product": "FU-DATAMOUNTAINEER-20150201-100", "price":10000}

Following the command you should have something similar to this in the logs for your connect:

.. sourcecode:: bash

    [2016-11-07 00:08:30,200] INFO Setting newly assigned partitions [orders-topic-json-0] for group connect-mongo-sink-orders-json (org.apache.kafka.clients.consumer.internals.ConsumerCoordinator:231)
    [2016-11-07 00:08:30,324] INFO Opened connection [connectionId{localValue:3, serverValue:9}] to localhost:27017 (org.mongodb.driver.connection:71)


Let's check the mongo db database for the new records:

.. sourcecode:: bash

    #Open a new terminal and navigate to the mongodb installation folder
    ➜ ./bin/mongo
        > show databases
            connect  0.000GB
            local    0.000GB
        > use connect
            switched to db connect
        > show collections
            dummy
            orders
            orders_json
        > db.orders_json.find()
        { "_id" : ObjectId("581fc5fe53b2c9318a3c1004"), "created" : "2016-05-06 13:53:00", "id" : NumberLong(1), "product_name" : "OP-DAX-P-20150201-95.7", "value" : 94.2 }
        { "_id" : ObjectId("581fc5fe53b2c9318a3c1005"), "created" : "2016-05-06 13:54:00", "id" : NumberLong(2), "product_name" : "OP-DAX-C-20150201-100", "value" : 99.5 }
        { "_id" : ObjectId("581fc5fe53b2c9318a3c1006"), "created" : "2016-05-06 13:55:00", "id" : NumberLong(3), "product_name" : "FU-DATAMOUNTAINEER-20150201-100", "value" : NumberLong(10000) }


Bingo, our 3 rows!

Features
--------

The sink connector will translate the SinkRecords to json and will insert each one in the database. We support to insert modes:
INSERT and UPSERT. All of this can be expressed via KCQL (our own SQL like syntax for configuration. Please see below the section
for Kafka Connect Query Language)

The sink supports:

1. Field selection - Kafka topic payload field selection is supported, allowing you to have choose selection of fields
   or all fields written to MongoDb.
2. Topic to table routing. Your sink instance can be configured to handle multiple topics and collections. All you have to do is to set
   your configuration appropriately. Below you will find an example

.. sourcecode:: bash

    connect.mongo.kcql = INSERT INTO orders SELECT * FROM orders-topic; UPSERT INTO customers SELECT * FROM customer-topic PK customer_id; UPSERT INTO invoiceid as invoice_id, customerid as customer_id, value a SELECT invoice_id, FROM invoice-topic

3. Error policies for handling failures.

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**K** afka **C** onnect **Q** uery **L**, :ref:`KCQL <kcql>` allows for routing and mapping using a SQL like syntax,
consolidating typically features in to one configuration option.

MongoDb sink supports the following:

.. sourcecode:: bash

    INSERT INTO <database>.<target collection> SELECT <fields> FROM <source topic> <PK field name>

Example:

.. sourcecode:: sql

    #Insert mode, select all fields from topicA and write to tableA
    INSERT INTO collectionA SELECT * FROM topicA

    #Insert mode, select 3 fields and rename from topicB and write to tableB with primary key as the field id from the topic
    INSERT INTO tableB SELECT x AS a, y AS b and z AS c FROM topicB PK id


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
    violations and or other exceptions thrown by drivers..

**Retry**

Any error on write to the target database causes the RetryIterable exception to be thrown. This causes the
Kafka connect framework to pause and replay the message. Offsets are not committed. For example, if the database is offline
it will cause a write failure, the message can be replayed. With the Retry policy the issue can be fixed without stopping
the sink.

The length of time the sink will retry can be controlled by using the ``connect.mongo.max.retires`` and the
``connect.mongo.retry.interval``.

Topic Routing
^^^^^^^^^^^^^

The sink supports topic routing that maps the messages from topics to a specific collection. For example map
a topic called "bloomberg_prices" to a collection called "prices". This mapping is set in the ``connect.mongo.kcql`` option.
You don't need to set up multiple sinks for each topic or collection. The same sink instance can be configured to handle multiple collections.
For example your configuration in this case:


.. sourcecode:: bash

    connect.mongo.kcql = INSERT INTO orders SELECT * FROM orders-topic; UPSERT INTO customers SELECT * FROM customer-topic PK customer_id; UPSERT INTO invoiceid as invoice_id, customerid as customer_id, value a SELECT invoice_id, FROM invoice-topic

Field Selection
^^^^^^^^^^^^^^^

The sink supports selecting fields from the source topic or selecting all. There is an option to rename a field as well.
All of this can be easily expressed with KCQL:

 -  Select all fields from topic fx_prices and insert into the fx collection: ``INSERT INTO fx SELECT * FROM fx_prices``.

 -  Select all fields from topic fx_prices and upsert into the fx collection, The assumption is there will be a ticker field in the incoming json:
    ``UPSERT INTO fx SELECT * FROM fx_prices PK ticker``.


 -  Select specific fields from the topic sample_topic and insert into the sample collection:
    ``INSERT INTO sample SELECT field1,field2,field3 FROM sample_topic``.

 -  Select specific fields from the topic sample_topic and upsert into the sample collection:
    ``UPSERT INTO sample SELECT field1,field2,field3 FROM sample_fopic PK field1``.

 -  Rename some fields while selecting all from the topic sample_topic and insert into the sample collection:
    ``INSERT INTO sample SELECT *, field1 as new_name1,field2 as new_name2 FROM sample_topic``.

 -  Rename some fields while selecting all from the topic sample_topic and upsert into the sample collection:
    ``UPSERT INTO sample SELECT *, field1 as new_name1,field2 as new_name2 FROM sample_topic PK new_name1``.

 -  Select specific fields and rename some of them from the topic sample_topic and insert into the sample collection:
    ``INSERT INTO sample SELECT field1 as new_name1,field2, field3 as new_name3 FROM sample_topic``.

 -  Select specific fields and rename some of them from the topic sample_topic and upsert into the sample collection:
    ``INSERT INTO sample SELECT field1 as new_name1,field2, field3 as new_name3 FROM sample_fopic PK new_name3``.

TLS/SSL
-------

TLS/SSL is support by setting ``?ssl=true`` in the ``connect.mongo.connection`` option. The MongoDB driver will then
load attempt to load the truststore and keystore using the JVM system properties.

You will need to set several JVM system properties to ensure that the client is able to validate the SSL certificate
presented by the server:

.. sourcecode:: bash

   javax.net.ssl.trustStore: the path to a trust store containing the certificate of the signing authority
   javax.net.ssl.trustStorePassword: the password to access this trust store

The trust store is typically created with the keytool command line program provided as part of the JDK. For example:

.. sourcecode:: bash

    keytool -importcert -trustcacerts -file <path to certificate authority file> -keystore <path to trust store> -storepass <password>

You will also need to set several JVM system properties to ensure that the client presents an SSL certificate to the MongoDB server:

.. sourcecode:: bash

    javax.net.ssl.keyStore: the path to a key store containing the client’s SSL certificates
    javax.net.ssl.keyStorePassword: the password to access this key store

The key store is typically created with the keytool or the openssl command line program.

Authentication Mechanism
~~~~~~~~~~~~~~~~~~~~~~~~

All authentication methods are supported, X.509, LDAP Plain, Kerberos (GSSAPI), Mongodb-CR and SCRAM-SHA-1. The default as of
MongoDB version 3.0 SCRAM-SHA-1. To set the authentication mechanism set the ``authMechanism`` in the ``connect.mongo.connection`` option.


.. note::

    The mechanism can either be set in the connection string but this requires the password to be in plain text in the connection string
    or via the ``connect.mongo.auth.mechanism`` option.

    If the username is set it overrides the username/password set in the connection string and the ``connect.mongo.auth.mechanism`` has precedence.

e.g.

.. sourcecode:: bash

    # default of scram
    mongodb://host1/?authSource=db1
    # scram explict
    mongodb://host1/?authSource=db1&authMechanism=SCRAM-SHA-1
    # mongo-cr
    mongodb://host1/?authSource=db1&authMechanism=MONGODB-CR
    # x.509
    mongodb://host1/?authSource=db1&authMechanism=MONGODB-X509
    # kerberos
    mongodb://host1/?authSource=db1&authMechanism=GSSAPI
    # ldap
    mongodb://host1/?authSource=db1&authMechanism=PLAIN

Configurations
--------------

Configurations parameters:

``connect.mongo.db``

The target MongoDb database name.

* Data type: string
* Optional : no

``connect.mongo.connection``

The mongodb endpoints connections in the format mongodb://host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]]

* Data type: string
* Optional : no

.. note::

    Setting username and password in the endpoints is not secure, they will be pass to Connect as plain text before being
    given to the driver. Use the ``connect.mongo.username`` and ``connect.mongo.password`` options.

``connect.mongo.batch.size``

The number of records the sink would push to mongo at once (improved performance)

* Data type: int
* Optional : yes
* Default: 100

``connect.mongo.kcql``

Kafka connect query language expression. Allows for expressive topic to collectionrouting, field selection and renaming.

Examples:

.. sourcecode:: sql

    INSERT INTO TABLE1 SELECT * FROM TOPIC1;INSERT INTO TABLE2 SELECT field1, field2, field3 as renamedField FROM TOPIC2


* Data Type: string
* Optional : no

``connect.mongo.username``

The username to use for authentication. If the username is set it overrides the username/password set in the connection
string and the ``connect.mongo.auth.mechanism`` has precedence.

* Data Type: string
* Option: yes
* Default:

``connect.mongo.password``

The password to use for authentication.

* Data Type: string
* Optional: yes
* Default:

``connect.mongo.auth.mechanism``

The mechanism to use for authentication. GSSAPI (Kerberos), PLAIN (LDAP), X.509 or SCRAM-SHA-1.

*   Data Type: string
*   Optional: yes
*   Default: SCRAM-SHA-1

``connect.mongo.error.policy``

Specifies the action to be taken if an error occurs while inserting the data.

There are three available options, **NOOP**, the error is swallowed, **THROW**, the error is allowed to propagate and retry.
For **RETRY** the Kafka message is redelivered up to a maximum number of times specified by the ``connect.mongo.max.retires``
option. The ``connect.mongo.retry.interval`` option specifies the interval between retries.

The errors will be logged automatically.

* Type: string
* Importance: high
* Default: ``throw``

``connect.mongo.max.retires``

The maximum number of times a message is retried. Only valid when the ``connect.mongo.error.policy`` is set to ``TRHOW``.

* Type: string
* Importance: high
* Default: 10

``connect.mongo.retry.interval``

The interval, in milliseconds between retries if the sink is using ``connect.mongo.error.policy`` set to **RETRY**.

* Type: int
* Importance: medium
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

    name=mongo-sink-orders
    connector.class=com.datamountaineer.streamreactor.connect.mongodb.sink.MongoSinkConnector
    tasks.max=1
    topics=orders-topic
    connect.mongo.kcql=INSERT INTO orders SELECT * FROM orders-topic
    connect.mongo.db=connect
    connect.mongo.connection=mongodb://localhost:27017
    connect.mongo.batch.size=10

Schema Evolution
----------------

Upstream changes to schemas are handled by Schema registry which will validate the addition and removal or fields,
data type changes and if defaults are set. The Schema Registry enforces Avro schema evolution rules. More information
can be found `here <http://docs.confluent.io/3.0.1/schema-registry/docs/api.html#compatibility>`_.


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
