# Kite End-to-End Demo

This module provides an example of logging application events from a webapp to Hadoop
via Flume (using log4j as the logging API), extracting session data from the events using
Crunch, and analyzing the session data with SQL using Impala or Hive.

If you run into trouble, check out the [Troubleshooting section](../README.md#troubleshooting).

## Getting started

1. This example assumes that you have VirtualBox or VMWare installed and have a
   running [Cloudera QuickStart VM][getvm] version 5.1 or later. See the
   [Getting Started](../README.md#getting-started) and
   [Troubleshooting](../README.md#troubleshooting) sections for help.
2. In that VM, check out a copy of this demo so you can build the code and
   follow along:
  * Open "Applications" > "System Tools" > "Terminal"
  * Then run:

```bash
git clone https://github.com/kite-sdk/kite-examples.git
cd kite-examples
cd demo
```

[getvm]: http://www.cloudera.com/content/support/en/downloads/quickstart_vms.html

## Configuring the VM

*   __Enable Flume user impersonation__ Flume needs to be able to impersonate the owner
 of the dataset it is writing to. (This is like Unix `sudo`, see
[Configuring Flume's Security Properties](http://www.cloudera.com/content/cloudera-content/cloudera-docs/CDH5/latest/CDH5-Security-Guide/cdh5sg_flume_security_props.html#topic_4_2_1_unique_1)
for further information.)
    * If you're using Cloudera Manager (the QuickStart VM ships with Cloudera Manager,
      but by default it is not enabled) then this is already configured for you.
    * If you're not using Cloudera Manager, just add the following XML snippet to your
      `/etc/hadoop/conf/core-site.xml` file and then restart the NameNode with
      `sudo service hadoop-hdfs-namenode restart`.

```
<property>
  <name>hadoop.proxyuser.flume.groups</name>
  <value>*</value>
</property>
<property>
  <name>hadoop.proxyuser.flume.hosts</name>
  <value>*</value>
</property>
```

### __Configure the flume agent__

* First, check the value of the `tier1.sinks.sink-1.hdfs.proxyUser` in the `flume.properties`
  file to ensure it matches your login username. The default value is `cloudera`, which is correct
  for the QuickStart VM, but you'll likely need to change this when running the example from another system.
* If you're using Cloudera Manager, configure the Flume agent by following these steps:
    * Select "View and Edit" under the Flume service Configuration tab
    * Click on the "Agent (Default)" category
    * Paste the contents of the `flume.properties` file into the text area for the "Configuration File" property.
    * Save your change
* If you're not using Cloudera Manager, configure the Flume agent by following these steps:
    * Edit the `/etc/default/flume-ng-agent` file and add a line containing `FLUME_AGENT_NAME=tier1`
      (this sets the default Flume agent name to match the one defined in the `flume.properties` file).
    * Run `sudo cp flume.properties /etc/flume-ng/conf/flume.conf` so the Flume agent uses our configuration file.

__NOTE:__ Don't start Flume immediately after updating the configuration. Flume requires that the
dataset alerady be created before it will start correctly.

## Building

To build the project, type

```bash
mvn install
```

This creates the following artifacts:

* a JAR file containing the compiled Avro specific schemas `standard_event.avsc` and
`session.avsc` (in `demo-core`)
* a WAR file for the webapp that logs application events (in `demo-logging-webapp`)
* a JAR file for running the Crunch job to transform events into sessions (in
`demo-crunch`)
* a WAR file for the webapp that displays reports generated by Impala (in
`demo-reports-webapp`)

## Running

### Create the datasets

First we need to create the datasets: one called `events` for the raw events,
and `sessions` for the derived sessions.

We store the raw events metadata in HDFS so Flume can find the schema (it would be nice
if we could store it using HCatalog, so we may lift this restriction in the future).
The sessions dataset metadata is stored using HCatalog, which will allow us to query it
via Hive.

```bash
mvn kite:create-dataset \
  -Dkite.rootDirectory=/tmp/data \
  -Dkite.datasetName=events \
  -Dkite.avroSchemaFile=demo-core/src/main/avro/standard_event.avsc \
  -Dkite.hcatalog=false \
  -Dkite.partitionExpression='[year("timestamp", "year"), month("timestamp", "month"), day("timestamp", "day"), hour("timestamp", "hour"), minute("timestamp", "minute")]'

mvn kite:create-dataset \
  -Dkite.rootDirectory=/tmp/data \
  -Dkite.datasetName=sessions \
  -Dkite.avroSchemaFile=demo-core/src/main/avro/session.avsc
```

A few comments about these commands. The schemas for the `events` and `sessions`
datasets are loaded from local files.

The `-Dkite.partitionExpression` argument is used to specify how the data is partitioned.
Here we partition by time fields, using JEXL to specify the field partitioners.

Note that you can delete the datasets if you created them on a previous attempt with:

```bash
mvn kite:delete-dataset -Dkite.rootDirectory=/tmp/data -Dkite.datasetName=events -Dkite.hcatalog=false
mvn kite:delete-dataset -Dkite.rootDirectory=/tmp/data -Dkite.datasetName=sessions
```

You can check that the data directories were created, using Hue (login as `cloudera` if
 you are logged in to the VM, or as your host login if you are running from your
 machine): [`/tmp/data/default/events`](http://quickstart.cloudera:8888/filebrowser/#/tmp/data/default/events),
 [`/tmp/data/default/sessions`](http://quickstart.cloudera:8888/filebrowser/#/tmp/data/default/sessions).

### Start Flume

* If using Cloudera Manager:
    * Start (or restart) the Flume agent
* If not using Cloudera Manager:
    * Run `sudo /etc/init.d/flume-ng-agent restart` to restart the Flume agent with this new configuration

### Create events

Next we can run the webapps. They can be used in a Java EE 6 servlet
container; for this example we'll start an embedded Tomcat instance using Maven:

```bash
mvn tomcat7:run
```

Navigate to [http://quickstart.cloudera:8034/demo-logging-webapp/](http://quickstart.cloudera:8034/demo-logging-webapp/),
which presents you with a very simple web page for sending messages.

The message events are sent to the Flume agent
over IPC, and the agent writes the events to the HDFS file sink.

Rather than creating lots of events manually, it's easier to simulate two users with
a script as follows:

```bash
./bin/simulate-activity.sh 1 10 > /dev/null &
./bin/simulate-activity.sh 2 10 > /dev/null &
```

### Generate the derived sessions

Wait about 30 seconds for Flume to flush the events to the
[filesystem](http://quickstart.cloudera:8888/filebrowser/#/tmp/data/default/events),
then run the Crunch job to generate derived session data from the events:

```bash
cd demo-crunch
mvn kite:run-tool
```

The `kite:run-tool` Maven goal executes the `run` method of the `Tool`,
in this case `CreateSessions`, which launches a Crunch job on the cluster.

The `Tool` class to run, as well as the cluster settings, are found from the configuration
of the `kite-maven-plugin`.

When it's complete you should see a file in [`/tmp/data/default/sessions`]
(http://quickstart.cloudera:8888/filebrowser/#/tmp/data/default/sessions).

You can also supply a view URI to process the events for a particular minute bucket:

```bash
mvn kite:run-tool -Dkite.args='view:hdfs:/tmp/data/default/events?year=2014&month=8&date=5&hour=17&minute=10'
```

### Run session analysis

The `sessions` dataset is now populated with data, but we need to tell Impala to refresh its metastore so the new `sessions` table will be visible:

```bash
impala-shell -q 'invalidate metadata'
```

One way to explore the results is by using the `demo-reports-webapp` running at
[http://quickstart.cloudera:8034/demo-reports-webapp/](http://quickstart.cloudera:8034/demo-reports-webapp/),
which uses JDBC to run Impala queries for a few pre-defined reports. (Note this only
work with Impala 1.1 or later, see instructions above.)

Another way is to run ad hoc SQL queries using the Hue interfaces to
[Impala](http://quickstart.cloudera:8888/impala/) or [Hive](http://quickstart.cloudera:8888/beeswax/).
Here are some queries to try out:

```
DESCRIBE sessions
```

```
SELECT * FROM sessions
```

```
SELECT AVG(duration) FROM sessions
```
