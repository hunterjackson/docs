.. _faq:

FAQS
====

.. toctree::
   :maxdepth: 1

Classpath Issues
----------------

The Connectors throw dependency issues such as Guava class not found. Update the ``plugins.path`` option in the
``connect-avro-distributed.properties`` to point to the directory containing the Connector jars files. Kafka Connect 0.11
has introduced isolated classpath loaders.

Connection Refused when posting Configs
---------------------------------------

If you see this

.. sourcecode:: bash

    confluent-3.3.0|⇒ bin/connect-cli ps
    java.net.ConnectException: Connection refused (Connection refused)

Kafka Connect is not up. Connect can be slow to start especially is loading a lot of plugins like the Stream Reactor. Confluents CLI will
report Connect is up when running `confluent start` but Connect's api is not ready to recieve connectors yet. Wait until the Connect logs 
had reported that all plugins have loaded an try again.

Also the `connect-cli` assumes you are on the same host a Connect cluster worker that you are targeting, if not set the following environment
variable:

.. sourcecode:: bash

    export KAFKA_CONNECT_REST="http://myserver:myport"

Kudu - No records in Impala table
---------------------------------

If the KCQL statement is set to autocreate, tables that are created are not visible in Impala. You need to map them first.  Please refer to the 
`Kudu Documentation <https://kudu.apache.org/docs/kudu_impala_integration.html#_impala_databases_and_kudu>`__. 

If you have created your table in Impala as a managed table you need to fully qualify the table name in the KCQL statement with the
impala namespace and database .i.e.

.. sourcecode:: sql

    INSERT INTO impala::default.my_table SELECT * FROM my_topic    

I have JSON. Can I still use DataMountaineer Sink Connectors?
-------------------------------------------------------------

Kafka Connect has two converters for both the key and payload from Kafka. These are Json and Avro, the Json converter is
part of the Kafka distribution and the Avro converter from Confluents schema registry. These converters convert the records
in Kafka to SinkRecords, most of our Sinks rely of the data in Kafka being Avro and written with the Confluent Avro Serializers.
This is `best practice`. This allows the Connectors to receive SinkRecords with schemas for the payloads to mapping, filtering
can take place based on the :ref:`The KCQL routing querying <kcql>` provided.

However, it is possible sink Json messages from Kafka with some of our Sinks by using the JsonConverter from Kafka. If your Json messages
have a `schema` field the converter will deliver the records to the Sink with a schema. If no `schema` tag is present the
records will be delivered with a schema of type ``SCHEMA.String``.

.. note::

    You must be using at least Cassandra 3.0.9 to have JSON support!


Can I run on multiple nodes?
----------------------------

Yes, Kafka Connect has two modes, standalone and distributed. Both allow for scaling by setting the `max.tasks` property.

In distributed mode each work joins a Connect cluster defined in the `etc/schema-registry/connect-avro-distributed.properties`
file that is part of the Confluent distribution. Within this file a property called ``group.id`` controls this.

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

Redis authentication
--------------------

If your redis server is requiring the connection to be authenticated you will need to provide an extra setting:

    .. sourcecode:: bash

        connect.redis.connection.password=$REDIS_PASSWORD

Don't set the value to empty if no password is required.

InfluxDb Port already in use
----------------------------

InfluxDB starts an Admin web server listening on port 8083 by default. For this quickstart this will collide with Kafka
Connects default port of 8083. Since we are running on a single node we will need to  edit the InfluxDB config.

.. sourcecode:: bash

    #create config dir
    sudo mkdir /etc/influxdb
    #dump the config
    influxd config > /etc/influxdb/influxdb.generated.conf

Now change the following section to a port 8087 or any other free port.

.. sourcecode:: bash

    [admin]
    enabled = true
    bind-address = ":8087"
    https-enabled = false
    https-certificate = "/etc/ssl/influxdb.pem"

How get multiple worker on different hosts to for a Connect Cluster
-------------------------------------------------------------------

For workers to join a Connect cluster, set the `group.id` in the `$CONFLUENT_HOME/etc/schema-registry/connect-avro-distributed.properties`
file.

.. sourcecode:: bash

    # The group ID is a unique identifier for the set of workers that form a single Kafka Connect
    # cluster
    group.id=connect-cluster

HBase Sink isn't connecting to Zookeeper Quroum
-----------------------------------------------

Ensure you have your HBase clusters ``hbase-site.xml`` in your classpath.

.. sourcecode:: bash

    export CLASSPATH=hbase-site.xml