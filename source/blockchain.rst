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

- Confluent 3.3
- Java 1.8
- Scala 2.11

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Source Connector QuickStart
---------------------------

We you start the Confluent Platform, Kafka Connect is started in distributed mode (``confluent start``). 
In this mode a Rest Endpoint on port ``8083`` is exposed to accept connector configurations. 
We developed Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

Starting the Connector (Distributed)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Download, unpack and install the Stream Reactor and Confluent. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Once the Connect has started we can now use the kafka-connect-tools :ref:`cli <kafka-connect-cli>` to post in our distributed properties file for BlockChain.

.. sourcecode:: bash

    ➜  bin/connect-cli create blockchain-source < conf/blockchain-source.properties

    #Connector `blockchain-source`:
    name=blockchain-source
    connector.class=com.datamountaineeer.streamreactor.connect.blockchain.source.BlockchainSourceConnector
    max.tasks=1
    connect.blockchain.kafka.topic = blockchain-test
    max.tasks=1
    #task ids:

The ``blockchain-source.properties`` file defines:

1.  The name of the source.
2.  The Source class.
3.  The max number of tasks the connector is allowed to created (1 task only).
4.  The topics to write to.

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
    blockchain-source

.. sourcecode:: bash

    # Get connects logs
    connect log connect

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

Configurations
--------------

``connect.progress.enabled``

Enables the output for how many records have been processed.

* Type: boolean
* Importance: medium
* Optional: yes
* Default : false

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


