.. _install:

.. toctree::
    :maxdepth: 2

Install
=======

The Stream Reactor components are built around The Confluent Platform. They rely on the Kafka Brokers, Zookeepers and
optionally the Schema Registry provided by this distribution.

The following releases are available:

-  `0.2.2 <https://github.com/datamountaineer/stream-reactor/releases/tag/v0.2.2>`__

+------------------------+------------------------+
| Connector              | Versions               |
+========================+========================+
| BlockChain             | Not applicable         |
+------------------------+------------------------+
| Bloomberg              | blpapi-3.8.8-2         |
+------------------------+------------------------+
| Cassandra              | Driver 3.0.0,          |
|                        | Server 2.6.6           |
+------------------------+------------------------+
| Druid                  | Tranquility 0.7.4      |
+------------------------+------------------------+
| Elastic                | Elastic 2.2.0,         |
|                        | Elastic4s 2.3.0        |
+------------------------+------------------------+
| HazelCast              | HazelCast 3.6.0        |
+------------------------+------------------------+
| HBase                  | HBase Server 1.2.0,    |
|                        | HBase Client 1.2.0     |
+------------------------+------------------------+
| InfluxDB               | InfluxDB 2.3           |
+------------------------+------------------------+
| JMS                    | javax.jms 1.1-rev-1,   |
|                        | active-mq-code 1.26.0  |
+------------------------+------------------------+
| Kudu                   | Kudu Client 0.9.0      |
+------------------------+------------------------+
| MongoDB                | MongoDB 3.3.0          |
+------------------------+------------------------+
| Redis                  | Redis 2.8.1            |
+------------------------+------------------------+
| ReThinkDB              | ReThinkDB 2.3.3        |
+------------------------+------------------------+
| VoltDB                 | VoltDB 6.4             |
+------------------------+------------------------+
| Yahoo                  | yahoofinance-api 1.3.0 |
+------------------------+------------------------+

Install Confluent
~~~~~~~~~~~~~~~~~

Confluent can be downloaded for `here <http://www.confluent.io/download/>`__

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
    bin/zookeeper-server-start etc/kafka/zookeeper.properties &
    bin/kafka-server-start etc/kafka/server.properties &
    bin/schema-registry-start etc/schema-registry/schema-registry.properties &

Stream Reactor Install
~~~~~~~~~~~~~~~~~~~~~~

Download the latest release from `here <https://github.com/datamountaineer/stream-reactor/releases>`__.

Unpack the archive:

.. sourcecode:: bash

    #Stream reactor release 0.2.2
    tar xvf -C stream-reactor-0.2.2-cp-3.0.1.jar stream-reactor

Within the unpacked directory you will find the following structure:

.. sourcecode:: bash

    `-- stream-reactor-0.2.2-3.0.1
        |-- LICENSE
        |-- README.md
        |-- bin
        |   |-- cli.sh
        |   |-- install-ui.sh
        |   |-- sr-cli-linux
        |   |-- sr-cli-osx
        |   |-- start-connect.sh
        |   `-- start-ui.sh
        |-- conf
        |   |-- blockchain-source.properties
        |   |-- bloomberg-source.properties
        |   |-- cassandra-sink.properties
        |   |-- cassandra-source.properties
        |   |-- druid-sink.properties
        |   |-- hazelcast-sink.properties
        |   |-- hbase-sink.properties
        |   |-- influxdb-sink.properties
        |   |-- jms-sink.properties
        |   |-- kudu-sink.properties
        |   |-- mongodb-sink.properties
        |   |-- redis-sink.properties
        |   |-- rethink-sink.properties
        |   |-- rethink-source.properties
        |   |-- voltdb-sink.properties
        |   `-- yahoo-source.properties
        `-- libs
            |-- kafka-connect-blockchain-0.2.2-3.0.1-all.jar
            |-- kafka-connect-bloomberg-0.2.2-3.0.1-all.jar
            |-- kafka-connect-cassandra-0.2.2-3.0.1-all.jar
            |-- kafka-connect-cli-0.5-all.jar
            |-- kafka-connect-druid-0.2.2-3.0.1-all.jar
            |-- kafka-connect-elastic-0.2.2-3.0.1-all.jar
            |-- kafka-connect-hazelcast-0.2.2-3.0.1-all.jar
            |-- kafka-connect-hbase-0.2.2-3.0.1-all.jar
            |-- kafka-connect-influxdb-0.2.2-3.0.1-all.jar
            |-- kafka-connect-jms-0.2.2-3.0.1-all.jar
            |-- kafka-connect-kudu-0.2.2-3.0.1-all.jar
            |-- kafka-connect-mongodb-0.2.2-3.0.1-all.jar
            |-- kafka-connect-redis-0.2.2-3.0.1-all.jar
            |-- kafka-connect-rethink-0.2.2-3.0.1-all.jar
            |-- kafka-connect-voltdb-0.2.2-3.0.1-all.jar
            |-- kafka-connect-yahoo-0.2.2-3.0.1-all.jar
            `-- kafka-socket-streamer-0.2.2-3.0.1-all.jar

The ``libs`` folder contains all the Stream Reactor Connector jars.

The ``bin`` folder contains:

*   ``start-connect.sh`` script loads all the Stream Reactors jars onto the CLASSPATH and starts
    Kafka Connect in distributed mode. The Confluent Platform, Zookeeper, Kafka and the Schema Registry must be started first.
*   :ref:`sr-cli <schema-registry-cli>` GO scripts for interacting with the Schema Registry. Two versions are package,
    for Linux and OSX.
*   :ref:`cli <cli>` script for interacting with Kafka Connects REST API's.
*   ``install-ui.sh`` script to download and install the Schema Registry and Topic Browser UIs from `Landoop <https://www.landoop.com/>`__.
*   ``start-ui.sh`` script to start the Schema Registry and Topic Browser UIs from `Landoop <https://www.landoop.com/>`__.

The ``conf`` folder contains quickstart connector properties files.

.. _dockers:

Docker Install
~~~~~~~~~~~~~~

All the Stream Reactor Connectors, Confluent and UI's for Connect, Schema Registry and topic browsing are available in Dockers.
The Docker images are available in `DockerHub <https://hub.docker.com/>`__ and maintained by our partner `Landoop <https://www.landoop.com/>`__

Pull the latest images:

.. sourcecode:: bash

    docker pull landoop/fast-data-dev
    docker pull landoop/fast-data-dev-connect-cluster

    #UI's
    docker pull landoop/kafka-topics-ui
    docker pull landoop/schema-registry-ui

Fast Data Dev
-------------

This is Docker image for development.

If you need

1.  Kafka Broker
2.  ZooKeeper
3.  Schema Registry
4.  Kafka REST Proxy
5.  Kafka Connect Distributed
6.  Certified DataMountaineer Connectors (ElasticSearch, Cassandra, Redis ..)
7.  Landoop's Fast Data Web UIs : schema-registry , kafka-topics , kafka-connect and
8.  Embedded integration tests with examples

Run with:

.. sourcecode:: bash

    docker run --rm -it --net=host landoop/fast-data-dev

On Mac OSX run:

.. sourcecode:: bash

    docker run --rm -it \
           -p 2181:2181 -p 3030:3030 -p 8081:8081 \
           -p 8082:8082 -p 8083:8083 -p 9092:9092 \
           -e ADV_HOST=127.0.0.1 \
           landoop/fast-data-dev

That's it. Your Broker is at localhost:9092, your Kafka REST Proxy at localhost:8082, your Schema Registry at
localhost:8081, your Connect Distributed at localhost:8083, your ZooKeeper at localhost:2181 and at
`<http://localhost:3030>`__ you will find Landoop's Web UIs for Kafka Topics and Schema Registry, as well as a Coyote test report.

.. figure:: ../images/landoop-docker.png
    :alt:

Fast Data Dev Connect
---------------------

This docker is targeted to more advanced users and is a special case since it doesn't set-up a Kafka cluster,
instead it expects to find a Kafka Cluster with Schema Registry up and running.

The developer can then use this docker image to setup a connect-distributed cluster by just spawning a couple containers.

.. sourcecode:: bash

    docker run -d --net=host \
           -e ID=01 \
           -e BS=broker1:9092,broker2:9092 \
           -e ZK=zk1:2181,zk2:2181 \
           -e SC=http://schema-registry:8081 \
           -e HOST=<IP OR FQDN>
           landoop/fast-data-dev-connect-cluster


Things to look out for in configuration options:

1. It is important to give a full URL (including schema —http://) for schema registry.

2. ID should be unique to the Connect cluster you setup, for current and old instances. This is because Connect stores
data in Brokers and Schema Registry. Thus even if you destroyed a Connect cluster, its data remain in your Kafka setup.

3.  HOST should be set to an IP address or domain name that other connect instances and clients can use to reach the
current instance. We chose not to try to autodetect this IP because such a feat would fail more often than not.
Good choices are your local network ip (e.g 10.240.0.2) if you work inside a local network, your public ip (if you have
one and want to use it) or a domain name that is resolvable by all the hosts you will use to talk to Connect.

If you don't want to run with --net=host you have to expose Connect's port which at default settings is 8083.
There a PORT option, that allows you to set Connect's port explicitly if you can't use the default 8083. Please remember
that it is important to expose Connect's port on the same port at the host. This is a choice we had to make for simplicity's sake.


.. sourcecode:: bash

    docker run -d \
           -e ID=01 \
           -e BS=broker1:9092,broker2:9092 \
           -e ZK=zk1:2181,zk2:2181 \
           -e SC=http://schema-registry:8081 \
           -e HOST=<IP OR FQDN>
           -e PORT=8085
           -p 8085:8085
           landoop/fast-data-dev-connect-cluster

Web Only Mode
-------------

This is a special mode only for Linux hosts, where only Landoop's Web UIs are started and kafka services are expected to be running on the local machine. It must be run with --net=host flag, thus the Linux only requisite:

.. sourcecode:: bash

    docker run --rm -it --net=host \
               -e WEB_ONLY=true \
               landoop/fast-data-dev

This is useful if you already have a cluster with Confluent's distribution install and want a fancy UI.

HBase Connector
~~~~~~~~~~~~~~~

Due to some issues with dependencies, the ElasticSearch connector and the HBase connector cannot coexist. Whilst both are available,
HBase won't work. We do provide the PREFER_HBASE environment variable which will remove ElasticSearch (and the Twitter connector)
to let HBase work:

.. sourcecode:: bash

    docker run --rm -it --net=host \
               -e PREFER_HBASE=true \
               landoop/fast-data-dev

Advanced
--------

The container does not exit with CTRL+C. This is because we chose to pass control directly to Connect, so you check your logs via docker logs.
You can stop it or kill it from another terminal.

Whilst the PORT variable sets the rest.port, the HOST variable sets the advertised host. This is the hostname that
Connect will send to other Connect instances. By default Connect listens to all interfaces, so you don't have to worry
as long as other instances can reach each instance via the advertised host.

Latest Test Results
-------------------

To see the latest tests for the Connectors, in a docker, please vist Landoop's test github `here <https://github.com/Landoop/kafka-connectors-tests>`__
Test results can be found `here <https://coyote.landoop.com/connect/>`__.

An example for BlockChain is:

.. figure:: ../images/blockchain-coyote-top.png
    :alt:

.. figure:: ../images/blockchain-coyote-bottom.png
    :alt:

