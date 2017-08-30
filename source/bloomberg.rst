Kafka Connect Bloomberg
=======================

Kafka Connect Bloomberg is a Source connector to subscribe to Bloomberg feeds via the Bloomberg labs open API and write to Kafka.

Prerequisites
-------------

-  Bloomberg subscription
- Confluent 3.2
-  Java 1.8
-  Scala 2.11

Setup
-----

Confluent Setup
~~~~~~~~~~~~~~~

Follow the instructions :ref:`here <install>`.

Source Connector QuickStart
~~~~~~~~~~~~~~~~~~~~~~~~~~~

We you start the Confluent Platform, Kafka Connect is started in distributed mode (``confluent start``). 
In this mode a Rest Endpoint on port ``8083`` is exposed to accept connector configurations. 
We developed Command Line Interface to make interacting with the Connect Rest API easier. The CLI can be found in the Stream Reactor download under
the ``bin`` folder. Alternatively the Jar can be pulled from our GitHub
`releases <https://github.com/datamountaineer/kafka-connect-tools/releases>`__ page.

Starting the Connector (Distributed)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Download, unpack and install the Stream Reactor and Confluent. Follow the instructions :ref:`here <install>` if you haven't already done so.
All paths in the quickstart are based in the location you installed the Stream Reactor.

Once the Connect has started we can now use the kafka-connect-tools :ref:`cli <kafka-connect-cli>` to post in our distributed properties file for Redis.
If you are using the :ref:`dockers <dockers>` you will have to set the following environment variable to for the CLI to
connect to the Rest API of Kafka Connect of your container.

.. sourcecode:: bash

   export KAFKA_CONNECT_REST="http://myserver:myport"

.. sourcecode:: bash

    ➜  bin/connect-cli create bloomberg-source < conf/bloomberg-source.properties
    #Connector name=name=`bloomberg-source`
    connector.class=com.datamountaineer.streamreactor.connect.bloomberg.BloombergSourceConnector
    tasks.max=1
    connect.bloomberg.server.host=localhost
    connect.bloomberg.server.port=8194
    connect.bloomberg.service.uri=//blp/mkdata
    connect.bloomberg.subscriptions=AAPL US Equity:LAST_PRICE,BID,ASK;IBM US Equity:BID,ASK,HIGH,LOW,OPEN
    kafka.topic=bloomberg
    connect.bloomberg.buffer.size=4096
    connect.bloomberg.authentication.mode=USER_AND_APPLICATION
    #task ids: 0

The ``bloomberg-source.properties`` file defines:

1.  The connector name.
2.  The class containing the connector.
3.  The number of tasks the connector is allowed to start.
4.  The Bloomberg server host.
5.  The Bloomberg server port.
6.  The Bloomberg service uri.
7.  The subscription keys to subscribe to.
8.  The topic to write to.
9.  The buffer size for the Bloomberg API to buffer events in.
10. The authentication mode.

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
    bloomberg-source

Test Records
^^^^^^^^^^^^

Now we need to see records pushed on the topic. We can use the ``kafka-avro-console-producer`` to do this.

.. sourcecode:: bash

    $ ./bin/kafka-avro-console-consumer --topic bloomberg \
         --zookeeper localhost:2181 \
         --from-beginning

Now the console is reading bloomberg transaction data which would print on the terminal.

Features
--------

The Source Connector allows subscriptions to BPipe mkdata and refdata endpoints to feed data into Kafka.

Configurations
--------------

``connect.bloomberg.server.host``

The bloomberg endpoint to connect to.

* Data type : string
* Optional  : no

``connect.bloomberg.server.port``

The Bloomberg endpoint to connect to.

* Data type : string
* Optional  : no

``connect.bloomberg.service.uri``

Which Bloomberg service to connect to. Can be //blp/mkdata or //blp/refdata.

* Data type : string
* Optional  : no

``connect.bloomberg.authentication.mode``

The mode to authentication against the Bloomberg server. Either APPLICATION_ONLY or USER_AND_APPLICATION.

* Data type : string
* Optional  : no


``connect.bloomberg.subscriptions``

* Data type : string
* Optional  : no

Specifies which ticker subscription to make. The format is TICKER:FIELD,FIELD,..;
e.g.AAPL US Equity:LAST_PRICE;IBM US Equity:BID

``connect.bloomberg.buffer.size``

* Data type : int
* Optional  : yes
* Default   : 2048

The buffer accumulating the data updates received from Bloomberg. If not provided it will default to 2048. If the
buffer is full and a new update will be received it won't be added to the buffer until it is first drained.

``connect.bloomberg.kafka.topic``

The topic to write to.

* Data type : string
* Optional  : no

``connect.progress.enabled``

Enables the output for how many records have been processed.

* Type: boolean
* Importance: medium
* Optional: yes
* Default : false

Example
~~~~~~~

.. sourcecode:: bash

    name=bloomberg-source
    connector.class=com.datamountaineer.streamreactor.connect.bloomberg.BloombergSourceConnector
    tasks.max=1
    connect.bloomberg.server.host=localhost
    connect.bloomberg.server.port=8194
    connect.bloomberg.service.uri=//blp/mkdata
    connect.bloomberg.subscriptions=AAPL US Equity:LAST_PRICE,BID,ASK;IBM US Equity:BID,ASK,HIGH,LOW,OPEN
    kafka.topic=bloomberg
    connect.bloomberg.buffer.size=4096

Schema Evolution
----------------

TODO

Deployment Guidelines
---------------------

TODO

TroubleShooting
---------------

Please review the :ref:`FAQs <faq>` and join our `slack channel <https://slackpass.io/datamountaineers>`_.

