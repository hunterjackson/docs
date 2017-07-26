Kafka Connect Blockchain
========================

A Connector to hook into the live streaming providing a real time feed for new bitcoin blocks and transactions provided by
`www.blockhain.info <http://www.blockchain.info/>`__ The connector subscribe to notification on blocks, transactions or an address
and receive JSON objects describing a transaction or block when an event occurs. This json is then pushed via kafka connect
to a kafka topic and therefore can be consumed either by a Sink or have a live stream processing using
for example Kafka Streams.

Since is a direct websocket connection the Source will only ever use one connector task at any point. There is no point spawning more
and then have duplicate data.

One thing to remember is the subscription API from blockchain doesn't offer an option to start from a given timestamp. This means
if the connect worker is down then you will miss some data.

The Sink connects to unconfirmed transaction!! Read more about the live data `here <https://blockchain.info/api/>`__

Prerequisites
-------------

- Confluent 3.2
- Java 1.8
- Scala 2.11

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Source Connector QuickStart
---------------------------

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

Once the connector has started we can now use the kafka-connect-tools cli to post in our distributed properties file for BlockChain.

.. sourcecode:: bash

    ➜  bin/cli.sh create blockchain-source < conf/blockchain-source.properties

    #Connector `blockchain-source`:
    name=blockchain-source
    connector.class=com.datamountaineeer.streamreactor.connect.blockchain.source.BlockchainSourceConnector
    max.tasks=1
    connect.blockchain.source.kafka.topic = blockchain-test
    max.tasks=1
    #task ids:

The ``blockchain-source.properties`` file defines:

1.  The name of the source.
2.  The Source class.
3.  The max number of tasks the connector is allowed to created (1 task only).
4.  The topics to write to.

If you switch back to the terminal you started the Connector in you should see the Blockchain Source being accepted and the
task starting.

We can use the CLI to check if the connector is up but you should be able to see this in logs as-well.

.. sourcecode:: bash

    #check for running connectors with the CLI
    ➜ bin/cli.sh ps
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

