# Kafka Backup

`kafka-backup` is a tool to backup and restore Kafka Topics **and**
Conumer Offsets. It is architected as two connectors for Kafka
Connect: A sink connector (backing data up) and a source connector
(restoring data).

Currently `kafka-backup` supports backup and restore to/from the file
system.

## Maturity of the project

* Very early stage
* There are some sucessful tests
* Seems to work overall
* There will be definitely breaking changes in the near future
* [ ] Documentation is coming!
* [ ] More tests are coming!


## More Documentation

* [Comparing Kafka Backup Solutions](./docs/Comparing Kafka Backup Solutions.md)

## The need for `kafka-backup`

Backing up Kafka is not trivial. Apart from `kafka-backup` there is
(to the best of our knowledge) no reliable and data-saving way to
backup data from Kafka that also ensures the backup and restore of
consumer offsets.

## Usage

### Set up Kafka Connect to use `kafka-backup`

Kafka Connect is shipped only with a small number of connectors. All
other connectors are added by putting the connector `jar` file to the
`CLASSPATH` where the JVM can find the classes.

#### Non-Containerized environment

See [Build and Run](#build-and-run).

#### Docker

Get the `kafka-backup` jar (See [Build and Run](#build-and-run)) and
add it to your Kafka Connect Dockerfile. For example if you use the
confluent Docker images:

```dockerfile
FROM confluentinc/cp-kafka-connect
COPY ./build/libs/kafka-backup.jar /etc/kafka-connect/jars
```

Run the Connect image as documented (e.g. [here](https://docs.confluent.io/5.0.0/installation/docker/docs/installation/connect-avro-jdbc.html))

### Backup

Configure a Backup Sink Connector
(e.g. create a file `connect-backup-sink.properties`):

```
name=backup-sink
connector.class=de.azapps.kafkabackup.sink.BackupSinkConnector
tasks.max=1
topics.regex=*
key.converter=de.azapps.kafkabackup.common.AlreadyBytesConverter
value.converter=de.azapps.kafkabackup.common.AlreadyBytesConverter
target.dir=/my/backup/dir
# 1GiB
max.segment.size.bytes=1073741824
cluster.bootstrap.servers=localhost:9092
```

#### Configuration options

| Name                        | Required? | Recommended Value                                    | Comment                                                                                                |
|-----------------------------|-----------|------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| `name`                      | ✓         | `backup-sink`                                        | A unique name identifying this connector jobs                                                          |
| `connector.class`           | ✓         | `de.azapps.kafkabackup.sink.BackupSinkConnector`     | Must be this class to use `kafka-backup`                                                               |
| `tasks.max`                 | ✓         | 1                                                    | Must be `1`. Currently no support for multi-task backups                                               |
| `topics`                    | -         |                                                      | Explicit, comma-separated list of topics to back up                                                    |
| `topics.regex`              | -         | `*`                                                  | Topic regex to back up                                                                                 |
| `key.converter`             | ✓         | `de.azapps.kafkabackup.common.AlreadyBytesConverter` | Must be this class to interpret the data as bytes                                                      |
| `value.converter`           | ✓         | `de.azapps.kafkabackup.common.AlreadyBytesConverter` | Must be this class to interpret the data as bytes                                                      |
| `target.dir`                | ✓         | `/my/backup/dir`                                     | Where to store the backup                                                                              |
| `max.segment.size`          | ✓         | `1073741824` (`1 GiB`)                               | Max size of the backup files. When the size is reached, a new file is created. No data is overwritten. |
| `cluster.bootstrap.servers` | ✓         | `my.kafka.cluster:9092`                              | `bootstrap.servers` property to connect to the cluster to back up.                                     |
| `cluster.*`                 | -         | none                                                 | Other consumer configuration options required to connect to the cluster (e.g. SSL settings)            |

#### Enable the Backup Sink

Configure the Sink Connector.

**Using curl:**

```sh
curl -X POST -H "Content-Type: application/json" \
  --data "@path/to/connect-backup-sink.properties"
  http://my.connect.server:8083/connectors
```

**Using [Confluent CLI](https://docs.confluent.io/current/cli/index.html):**

```sh
confluent load backup-source -d path/to/connect-backup-sink.properties
```

#### Monitor the progress

* Watch Kafka Connect logs (e.g. `confluent log connect`)
* Watch the consumer lag for the sink connector. The consumer group is
  probably named `connect-backup-sink`. Use for example
  `kafka-consumer-groups --bootstrap-server localhost:9092 --describe
  --group connect-backup-sink` to monitor it.

### Restore

Configure a Backup Source Connector
(e.g. create a file `connect-backup-source.properties`):

```
name=backup-source
connector.class=de.azapps.kafkabackup.source.BackupSourceConnector
tasks.max=1
topics=topic1,topic2,topic3
key.converter=de.azapps.kafkabackup.common.AlreadyBytesConverter
value.converter=de.azapps.kafkabackup.common.AlreadyBytesConverter
source.dir=/my/backup/dir
batch.size=500
```

#### Configuration Options

| Name              | Required? | Recommended Value                                    | Comment                                                                          |
|-------------------|-----------|------------------------------------------------------|----------------------------------------------------------------------------------|
| `name`            | ✓         | `backup-source`                                      | A unique name identifying this connector jobs                                    |
| `connector.class` | ✓         | `de.azapps.kafkabackup.source.BackupSourceConnector` | Must be this class to use `kafka-backup`                                         |
| `tasks.max`       | ✓         | 1                                                    | Must be `1`. Currently no support for multi-task backups                         |
| `topics`          | ✓         | `topic1,topic2,topic3`                               | A list of topics to restore. Only explicit list of topics is currently supported |
| `key.converter`   | ✓         | `de.azapps.kafkabackup.common.AlreadyBytesConverter` | Must be this class to interpret the data as bytes                                |
| `value.converter` | ✓         | `de.azapps.kafkabackup.common.AlreadyBytesConverter` | Must be this class to interpret the data as bytes                                |
| `source.dir`      | ✓         | `/my/backup/dir`                                     | Location of the backup files.                                                    |
| `batch.size`      | -         | `500`                                                | How many messages should be processed in one batch?                                                                                 |

#### Monitor the restore progress

* Watch the Kafka Connect log for the message `All records
  read. Restore was successful`
* Currently there is no other direct way to detect when the restore finished.

## Build and Run

You should be able to build this with `./gradlew shadowJar`. Once the jar is generated in `build/libs`, include it in `CLASSPATH` (e.g., `export CLASSPATH=.:$CLASSPATH:/fullpath/to/kafka-backup-jar` )

Run: `connect-standalone example-connect-worker.properties
example-kafka-backup-sink.properties`