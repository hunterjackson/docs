.. _kcql:

Kafka Connect Query Language
============================

The Kafka Connect Query Language is implemented in antlr4 grammar files.

Why ?
-----

While working on our sink/sources we ended up producing quite complex configuration in order to support the functionality
required. Imagine a Sink where you Source from different topics and from each topic you want to cherry pick the payload
fields or even rename them. Furthermore you might want the storage structure to be automatically created and/or even
evolve or you might add new support for the likes of bucketing (Riak TS has one such scenario). Imagine the JDBC sink
with a table which needs to be linked to two different topics and the fields in there need to be aligned with the table
column names and the complex configuration involved ...or you can just write this

.. sourcecode:: sql

    routes.query = "INSERT INTO transactions SELECT field1 as column1, field2 as column2, field3 FROM topic_A;
                INSERT INTO transactions SELECT fieldA1 as column1, fieldA2 as column2, fieldC FROM topic_B;"

Kafka Connect Query Language
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

There are two paths supported by this DSL. One is the INSERT and take the following form:

.. code-block:: bash

    INSERT INTO $TARGET
    SELECT *|columns
    FROM   $TOPIC_NAME
           [IGNORE columns]
           [AUTOCREATE]
           [PK columns]
           [AUTOEVOLVE]
           [BATCH = N]
           [CAPITALIZE]
           [PARTITIONBY cola[,colb]]
           [DISTRIBUTEBY cola[,colb]]
           [CLUSTERBY cola[,colb]]
           [TIMESTAMP cola|sys_current]
           [WITHFORMAT TEXT|JSON|AVRO|BINARY]
           [STOREAS $YOUR_TYPE([key=value, .....])]

If you follow our connectors @Datamountaineer you will find depending on the Connect Sink only some of the the options
are used. You will find all our documentation here

The second path is SELECT only. We have the Socket Streamer</> which allows you to peek into KAFKA via websocket and
receive the payloads in real time!

.. sourcecode:: bash

    SELECT *|columns
    FROM   $TOPIC_NAME
           [IGNORE columns]
           [WITHFORMAT TEXT|JSON|AVRO|BINARY]
           [WITHGROUP $YOUR_CONSUMER_GROUP]
           [WITHPARTITION (partition),[(partition, offset)]
           [SAMPLE $RECORDS_NUMBER EVERY $SLIDE_WINDOW

Examples
^^^^^^^^

.. sourcecode:: bash

    SELECT field1 FROM mytopic                    // Project one avro field named field1
    SELECT field1 AS newName                      // Project and renames a field
    SELECT *  FROM mytopic                        // Select everything - perfect for avro evolution
    SELECT *, field1 AS newName FROM mytopic      // Select all & rename a field - excellent for avro evolution
    SELECT * FROM mytopic IGNORE badField         // Select all & ignore a field - excellent for avro evolution
    SELECT * FROM mytopic PK field1,field2        //Select all & with primary keys (for the sources where primary keys are required)
    SELECT * FROM mytopic AUTOCREATE              //Select all and create the target Source (table for databases)
    SELECT * FROM mytopic AUTOEVOLVE              //Select all & reflect the new fields added to the avro payload into the target