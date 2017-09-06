Kafka Connect FTP
-----------------

.. note:: All credits for this connector go to Eneco's team, where this connector was forked from!

Monitors files on an FTP server and feeds changes into Kafka.

Provide the remote directories and on specified intervals, the list of files in the directories is refreshed.
Files are downloaded when they were not known before, or when their timestamp or size are changed.
Only files with a timestamp younger than the specified maximum age are considered.
Hashes of the files are maintained and used to check for content changes.
Changed files are then fed into Kafka, either as a whole (update) or only the appended part (tail), depending on the configuration.
Optionally, file bodies can be transformed through a pluggable system prior to putting it into Kafka.

Data Types
~~~~~~~~~~

Each Kafka record represents a file, and has the following types.

*   The format of the keys is configurable through `ftp.keystyle=string|struct`.
    It can be a `string` with the file name, or a `FileInfo` structure with `name: string` and `offset: long`.
    The offset is always `0` for files that are updated as a whole, and hence only relevant for tailed files.
*   The values of the records contain the body of the file as `bytes`.

Configuration
~~~~~~~~~~~~~

In addition to the general configuration for Kafka connectors (e.g. name, connector.class, etc.) the following options are available.


``ftp.address``

host\[:port\] of the ftp server.

* Type: string
* Importance: high
* Optional: no

``ftp.user``

Username to connect with.

* Type: string
* Importance: high
* Optional: no

``ftp.password``

Password to connect with.

* Type: string
* Importance: high
* Optional: no           

``ftp.refresh``

iso8601 duration that the server is polled.

* Type: string
* Importance: high
* Optional: no   

``ftp.file.maxage``       

iso8601 duration for how old files can be.

* Type: string
* Importance: high
* Optional: no   

``ftp.keystyle``         

SourceRecord keystyle, `string` or `struct`, see above.

* Type: string
* Importance: high
* Optional: no  

``ftp.monitor.tail``          

Comma separated list of path:destinationtopic to tail.

* Type: string
* Importance: high
* Optional: yes  

``ftp.monitor.update``      

Comma separated list of path:destinationtopic to monitor for updates.

* Type: string
* Importance: high
* Optional: yes  

``ftp.sourcerecordconverter``

Source Record converter class name.

* Type: string
* Importance: high
* Optional: yes  
An example file:

.. sourcecode:: bash

    name=ftp-source
    connector.class=com.datamountaineer.streamreactor.connect.ftp.source.FtpSourceConnector
    tasks.max=1

    #server settings
    ftp.address=localhost:21
    ftp.user=ftp
    ftp.password=ftp

    #refresh rate, every minute
    ftp.refresh=PT1M

    #ignore files older than 14 days.
    ftp.file.maxage=P14D

    #monitor /forecasts/weather/ and /logs/ for appends to files.
    #any updates go to the topics `weather` and `error-logs` respectively.
    ftp.monitor.tail=/forecasts/weather/:weather,/logs/:error-logs

    #keep an eye on /statuses/, files are retrieved as a whole and sent to topic `status`
    ftp.monitor.update=/statuses/:status

    #keystyle controls the format of the key and can be string or struct.
    #string only provides the file name
    #struct provides a structure with the filename and offset
    ftp.keystyle=struct

Tailing Versus Update as a Whole
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

The following rules are used.

-   *Tailed* files are *only* allowed to grow. Bytes that have been appended to it since a last inspection are yielded. Preceding bytes are not allowed to change;
-   *Updated* files can grow, shrink and change anywhere. The entire contents are yielded.


Data Converters
~~~~~~~~~~~~~~~

Instead of dumping whole file bodies (and the danger of exceeding Kafka's `message.max.bytes`), one might
want to give an interpretation to the data contained in the files before putting it into Kafka.
For example, if the files that are fetched from the FTP are comma-separated values (CSVs), one
might prefer to have a stream of CSV records instead.
To allow to do so, the connector provides a pluggable conversion of `SourceRecords`.
Right before sending a `SourceRecord` to the Connect framework, it is run through an object that implements:

.. sourcecode:: scala

    package com.datamountaineer.streamreactor.connect.ftp

    trait SourceRecordConverter extends Configurable {
        def convert(in:SourceRecord) : java.util.List[SourceRecord]
    }


(for the Java people, read: `interface` instead of `trait`).

The default object that is used is a pass-through converter, an instance of:

.. sourcecode:: scala

    class NopSourceRecordConverter extends SourceRecordConverter{
        override def configure(props: util.Map[String, _]): Unit = {}
        override def convert(in: SourceRecord): util.List[SourceRecord] = Seq(in).asJava
    }


To override it, create your own implementation of `SourceRecordConverter`, put the jar into your `$CLASSPATH` and instruct the connector to use it via the .properties:

.. sourcecode:: bash
    
    connect.ftp.sourcerecordconverter=your.name.space.YourConverter
