Kafka Connect Blockchain
========================

A Connector to hook into the live streaming providing a real time feed for new bitcoin blocks and transactions provided by
`www.blochain.info <http://www.blockchain.info/>`__ The connector subscribe to notification on blocks, transactions or an address
and receive JSON objects describing a transaction or block when an event occurs. This json is then pushed via kafka connect
to a kafka topic and therefore can be consumed either by a sink or have a live stream processing using for example kafka streaming.

Since is a direct websocket connection the source will only ever use one connector task at any point. there is no point spawning more
and then have duplicate data.

One thing to remember is the subscription API from blockchain doesn't offer an option to start from a given timestam. This means
if the connect worker is down then you will miss some data.

The sink connects to unconfirmed transaction!! Read more about the live data `here <https://blockchain.info/api/>`__

Prerequisites
-------------

- Confluent 3.0.1
- Java 1.8
- Scala 2.11

Setup
-----

Blockchain Setup
~~~~~~~~~~~~~~~~
All you need is having the confluent platform installed. Here is how you would install it

.. sourcecode:: bash

    #make confluent home folder
    ➜  mkdir confluent

    #download confluent
    ➜  wget http://packages.confluent.io/archive/3.0/confluent-3.0.1-2.11.tar.gz

    #extract archive to confluent folder
    ➜  tar -xvf confluent-3.0.1-2.11.tar.gz -C confluent

    #setup variables
    ➜  export CONFLUENT_HOME=~/confluent/confluent-3.0.1

Start the Confluent platform.

.. sourcecode:: bash

    #Start the confluent platform, we need kafka, zookeeper and the schema registry
    ➜  bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    ➜  bin/kafka-server-start etc/kafka/server.properties &
    ➜  bin/schema-registry-start etc/schema-registry/schema-registry.properties &

Build the Connector and CLI
~~~~~~~~~~~~~~~~~~~~~~~~~~~

The prebuilt jars can be taken from `here <https://github.com/datamountaineer/stream-reactor/releases>`__ and
`here <https://github.com/datamountaineer/kafka-connect-tools/releases>`__
or from `Maven <http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22kafka-connect-cli%22>`__

If you want to build the connector, clone the repo and build the jar.

.. sourcecode:: bash

    ##Build the connectors
    ➜  git clone https://github.com/datamountaineer/stream-reactor
    ➜  cd stream-reactor
    ➜  gradle fatJar

    ##Build the CLI for interacting with Kafka connectors
    ➜  git clone https://github.com/datamountaineer/kafka-connect-tools
    ➜  cd kafka-connect-tools
    ➜  gradle fatJar

Source Connector QuickStart
---------------------------

Next we will start the connector in distributed mode. Connect has two modes, standalone where the tasks run on only one host
and distributed mode. Usually you'd run in distributed mode to get fault tolerance and better performance. In distributed mode
you start Connect on multiple hosts and they join together to form a cluster. Connectors which are then submitted are
distributed across the cluster.

Before we can start the connector we need to setup it's configuration. In standalone mode this is done by creating a
properties file and passing this to the connector at startup. In distributed mode you can post in the configuration as
json to the Connectors HTTP endpoint. Each connector exposes a rest endpoint for stopping, starting and updating the
configuration.

Source Connector Configuration
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Create a file called ``blockchain-source.properties`` with the contents below:

.. sourcecode:: bash

    name=blockchain-source
    connector.class=com.datamountaineeer.streamreactor.connect.blockchain.source.BlockchainSourceConnector
    max.tasks=1
    connect.blockchain.source.kafka.topic = blockchain-test

This configuration defines:

1.  The name of the source.
2.  The source class.
3.  The max number of tasks the connector is allowed to created (1 task only).
4.  The topics to write to


Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Connectors can be deployed distributed mode. In this mode one or many connectors are started on the same or different
hosts with the same cluster id. The cluster id can be found in ``etc/schema-registry/connect-avro-distributed.properties.``

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

Now start the connector in distributed mode. We only give it one properties file for the kafka, zookeeper and
schema registry configurations.

First add the connector jar to the CLASSPATH and then start Connect.

.. sourcecode:: bash

    #Add the Connector to the class path
    ➜  export CLASSPATH=kafka-connect-blockchain-0.2-cp-3.0.1.all.jar

.. sourcecode:: bash

    ➜  confluent-3.0.1/bin/connect-distributed confluent-3.0.1/etc/schema-registry/connect-avro-distributed.properties

Once the connector has started lets use the kafka-connect-tools cli to post in our distributed properties file.

.. sourcecode:: bash

    ➜  java -jar build/libs/kafka-connect-cli-0.5-all.jar create blockchain-source < blockchain-source.properties

    #Connector `blockchain-source`:
    name=blockchain-source
    connector.class=com.datamountaineeer.streamreactor.connect.blockchain.source.BlockchainSourceConnector
    max.tasks=1
    connect.blockchain.source.kafka.topic = blockchain-test
    max.tasks=1
    #task ids:

If you switch back to the terminal you started the Connector in you should see the Blockchain source being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ java -jar build/libs/kafka-connect-cli-0.5-all.jar ps
    blockchain-source

.. sourcecode:: bash

    [2016-08-21 20:31:36,398] INFO Finished starting connectors and tasks (org.apache.kafka.connect.runtime.distributed.DistributedHerder:769)
    [2016-08-21 20:31:36,406] INFO

      ____        _        __  __                   _        _
     |  _ \  __ _| |_ __ _|  \/  | ___  _   _ _ __ | |_ __ _(_)_ __   ___  ___ _ __
     | | | |/ _` | __/ _` | |\/| |/ _ \| | | | '_ \| __/ _` | | '_ \ / _ \/ _ \ '__|
     | |_| | (_| | || (_| | |  | | (_) | |_| | | | | || (_| | | | | |  __/  __/ |
     |____/ \__,_|\__\__,_|_|  |_|\___/ \__,_|_| |_|\__\__,_|_|_| |_|\___|\___|_|
      ____  _            _     ____ _           _         ____ by Stefan Bocutiu
     | __ )| | ___   ___| | __/ ___| |__   __ _(_)_ __   / ___|  ___  _   _ _ __ ___ ___
     |  _ \| |/ _ \ / __| |/ / |   | '_ \ / _` | | '_ \  \___ \ / _ \| | | | '__/ __/ _ \
     | |_) | | (_) | (__|   <| |___| | | | (_| | | | | |  ___) | (_) | |_| | | | (_|  __/
     |____/|_|\___/ \___|_|\_\\____|_| |_|\__,_|_|_| |_| |____/ \___/ \__,_|_|  \___\___|



Test Records
^^^^^^^^^^^^

Now we need to see records pushed on the topic. We can use the ``kafka-avro-console-producer`` to do this.


.. sourcecode:: bash

    $ ./bin/kafka-avro-console-consumer --topic blockchain-test \
         --zookeeper localhost:2181 \
         --from-beginning

Now the console is reading blockchain transaction data which would print on the terminal.


Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

TODO