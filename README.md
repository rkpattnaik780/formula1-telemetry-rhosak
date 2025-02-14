# Formula 1 telemetry on Red Hat OpenShift Streams for Apache Kafka

This repository provides a guide to deploy the [Formula 1 - Telemetry with Apache Kafka](https://github.com/ppatierno/formula1-telemetry-kafka) project on a fully managed Kafka instance from the [Red Hat OpenShift Streams for Apache Kafka](https://www.redhat.com/en/technologies/cloud-computing/openshift/openshift-streams-for-apache-kafka) service.

![Overview](./images/f1-telemetry-overview.png)

 ## Prerequisites

 The main pre-requisites are:

 * Having an account on [Red Hat Hybrid Cloud](https://cloud.redhat.com/).
 * Having the `rhoas` CLI tool installed by following instructions [here](https://github.com/redhat-developer/app-services-guides/tree/main/rhoas-cli#installing-the-rhoas-cli).
 * Logging into your own Red Hat Hybrid Cloud account via `rhoas login` command by following instructions [here](https://github.com/redhat-developer/app-services-guides/tree/main/rhoas-cli#logging-in-to-rhoas).
 * [jq](https://github.com/stedolan/jq) tool.

## Create Apache Kafka instance

Create the Apache Kafka instance by running the following command.

```shell
rhoas kafka create --name formula1-kafka --wait
```

It specifies the `--name` option with the name of the instance and instructs the command to run synchronously by waiting for the instance to be ready using the `--wait` option.
The command will exit when the instance is ready providing some related information in JSON format like following.

```shell
✔️  Kafka instance "formula1-kafka" has been created:
{
  "bootstrap_server_host": "formula-k-c-rountr-f----l-fp-g.bf2.kafka.rhcloud.com:443",
  "cloud_provider": "aws",
  "created_at": "2022-05-08T09:37:36.095647Z",
  "href": "/api/kafkas_mgmt/v1/kafkas/c9rountr8f0102l1fp6g",
  "id": "c9rountr8f0102l1fp6g",
  "instance_type": "developer",
  "kafka_storage_size": "10Gi",
  "kind": "Kafka",
  "multi_az": false,
  "name": "formula1-kafka",
  "owner": "ppatiern",
  "reauthentication_enabled": true,
  "region": "us-east-1",
  "status": "ready",
  "updated_at": "2022-05-08T09:42:17.916266Z",
  "version": "3.0.1"
}
```

To confirm that everything is fine, verify the status of the Kafka instance by running the following command.

```shell
rhoas status kafka
```

The output will show status and bootstrap URL of the Kafka instance.

```shell
Kafka
--------------------------------------------------------------------------------
ID:                     c9rountr8f0102l1fp6g
Name:                   formula1-kafka
Status:                 ready
Bootstrap URL:          formula-k-c-rountr-f----l-fp-g.bf2.kafka.rhcloud.com:443
```

## Create topics

The Formula 1 project relies on some topics for storing and analyzing the telemetry data.
They can be customized via environment variables across the different applications but by running the following script you can create them with the default names.

```shell
./create-topics.sh
```

Each command will print the topic configuration.
Check that all topics are created by running the following command.

```shell
rhoas kafka topic list
```

The output will show the list of the topics on the Kafka instance.

```shell
NAME (5)                         PARTITIONS   RETENTION TIME (MS)   RETENTION SIZE (BYTES)  
-------------------------------- ------------ --------------------- ------------------------ 
f1-telemetry-drivers                      1   604800000             -1 (Unlimited)          
f1-telemetry-drivers-avg-speed            1   604800000             -1 (Unlimited)          
f1-telemetry-drivers-laps                 1   604800000             -1 (Unlimited)          
f1-telemetry-events                       1   604800000             -1 (Unlimited)          
f1-telemetry-packets                      1   604800000             -1 (Unlimited)
```

## Create service accounts and ACLs

### UDP to Apache Kafka

Create a service account for the UDP to Apache Kafka application by running the following command.

```shell
rhoas service-account create --short-description formaula1-udp-kafka --file-format env --output-file ./formula1-udp-kafka.env
```

This will generate a file containing the credentials for accessing the Kafka instance as environment variables.

```shell
## Generated by rhoas cli
RHOAS_SERVICE_ACCOUNT_CLIENT_ID=srvc-acct-adg23480-dsdf-244a-gt65-d4vd65784dsf
RHOAS_SERVICE_ACCOUNT_CLIENT_SECRET=f8g93220-9619-55ed-c23d-a2356c1fds9c
RHOAS_SERVICE_ACCOUNT_OAUTH_TOKEN_URL=https://identity.api.openshift.com/auth/realms/rhoas/protocol/openid-connect/token
```

The UDP to Apache Kafka application needs the rights to write on the `f1-telemetry-drivers`, `f1-telemetry-events` and `f1-telemetry-packets` topics.
To simplify let's grent access as a producer on topics starting with `f1-` prefix.

```shell
source ./formula1-udp-kafka.env
rhoas kafka acl grant-access -y --producer --service-account $RHOAS_SERVICE_ACCOUNT_CLIENT_ID --topic-prefix f1-
```

In this way the following ACLs entries will be created for the corresponding service account.

```shell
PRINCIPAL                                        PERMISSION   OPERATION   DESCRIPTION              
------------------------------------------------ ------------ ----------- ------------------------- 
srvc-acct-adg23480-dsdf-244a-gt65-d4vd65784dsf   allow        describe    topic starts with "f1-"  
srvc-acct-adg23480-dsdf-244a-gt65-d4vd65784dsf   allow        write       topic starts with "f1-"  
srvc-acct-adg23480-dsdf-244a-gt65-d4vd65784dsf   allow        create      topic starts with "f1-"  
srvc-acct-adg23480-dsdf-244a-gt65-d4vd65784dsf   allow        write       transactional-id is "*"  
srvc-acct-adg23480-dsdf-244a-gt65-d4vd65784dsf   allow        describe    transactional-id is "*" 
```

### Apache Kafka to InfluxDB

Create a service account for the Apache Kafka to InfluxDB application by running the following command.

```shell
rhoas service-account create --short-description formula1-kafka-influxdb --file-format env --output-file ./formula1-kafka-influxdb.env
```

This will generate a file containing the credentials for accessing the Kafka instance as environment variables.

```shell
## Generated by rhoas cli
RHOAS_SERVICE_ACCOUNT_CLIENT_ID=srvc-acct-abc1234-dsdf-244a-gt65-d4vd65784dsf
RHOAS_SERVICE_ACCOUNT_CLIENT_SECRET=g6d12345-9619-55ed-c23d-a2356c1fds9c
RHOAS_SERVICE_ACCOUNT_OAUTH_TOKEN_URL=https://identity.api.openshift.com/auth/realms/rhoas/protocol/openid-connect/token
```

The Apache Kafka to InfluxDB application needs the rights to read from the `f1-telemetry-drivers`, `f1-telemetry-events` and `f1-telemetry-drivers-avg-speed` topics.
To simplify let's grent access as a consumer on topics starting with `f1-` prefix.

```shell
source ./formula1-kafka-influxdb.env
rhoas kafka acl grant-access -y --consumer --service-account $RHOAS_SERVICE_ACCOUNT_CLIENT_ID --topic-prefix f1- --group all
```

In this way the following ACLs entries will be created for the corresponding service account.

```shell
PRINCIPAL                                        PERMISSION   OPERATION   DESCRIPTION              
------------------------------------------------ ------------ ----------- ------------------------- 
srvc-acct-abc1234-dsdf-244a-gt65-d4vd65784dsfa   allow        describe    topic starts with "f1-"  
srvc-acct-abc1234-dsdf-244a-gt65-d4vd65784dsfa   allow        read        topic starts with "f1-"  
srvc-acct-abc1234-dsdf-244a-gt65-d4vd65784dsfa   allow        read        group is "*"
```

### Apache Kafka Streams

Create a service account for the Apache Kafka Straems applications by running the following command.

```shell
rhoas service-account create --short-description formula1-kafka-streams --file-format env --output-file ./formula1-kafka-streams.env
```

This will generate a file containing the credentials for accessing the Kafka instance as environment variables.

```shell
## Generated by rhoas cli
RHOAS_SERVICE_ACCOUNT_CLIENT_ID=srvc-acct-93a071fc-7709-43ad-b0bd-431ce4f640ae
RHOAS_SERVICE_ACCOUNT_CLIENT_SECRET=5d614086-5562-4f19-9f83-e5e616e6e045
RHOAS_SERVICE_ACCOUNT_OAUTH_TOKEN_URL=https://identity.api.openshift.com/auth/realms/rhoas/protocol/openid-connect/token
```

The Apache Kafka Streams applications need the rights to read from and write to multiple topics as `f1-telemetry-drivers`, `f1-telemetry-drivers-avg-speed` and `f1-telemetry-drivers-laps`.

To simplify let's grent access as a producer and consumer on topics starting with `f1-` prefix.

```shell
source ./formula1-kafka-streams.env
rhoas kafka acl grant-access -y --producer --consumer --service-account $RHOAS_SERVICE_ACCOUNT_CLIENT_ID --topic-prefix f1- --group all
```

In this way the following ACLs entries will be created for the corresponding service account.

```shell
PRINCIPAL (7)                                    PERMISSION   OPERATION   DESCRIPTION              
------------------------------------------------ ------------ ----------- ------------------------- 
srvc-acct-93a071fc-7709-43ad-b0bd-431ce4f640ae   allow        describe    topic starts with "f1-"  
srvc-acct-93a071fc-7709-43ad-b0bd-431ce4f640ae   allow        read        topic starts with "f1-"  
srvc-acct-93a071fc-7709-43ad-b0bd-431ce4f640ae   allow        read        group is "*"             
srvc-acct-93a071fc-7709-43ad-b0bd-431ce4f640ae   allow        write       topic starts with "f1-"  
srvc-acct-93a071fc-7709-43ad-b0bd-431ce4f640ae   allow        create      topic starts with "f1-"  
srvc-acct-93a071fc-7709-43ad-b0bd-431ce4f640ae   allow        write       transactional-id is "*"  
srvc-acct-93a071fc-7709-43ad-b0bd-431ce4f640ae   allow        describe    transactional-id is "*"  
```

## Running Formula1 applications

### UDP to Apache Kafka

Start the UDP to Apache Kafka application by running the following command.

```shell
./formula1-run.sh \
./formula1-udp-kafka.env \
$(rhoas kafka describe --name formula1-kafka | jq -r .bootstrap_server_host) \
<PATH_TO_JAR>
```

The `<PATH_TO_JAR>` is the path to the application JAR (i.e. `/home/ppatiern/github/formula1-telemetry-kafka/udp-kafka/target/f1-telemetry-udp-kafka-1.0-SNAPSHOT-jar-with-dependencies.jar`)

### Apache Kafka to InfluxDB

Start the Apache Kafka to InfluxDB application by running the following command.

```shell
./formula1-run.sh \
./formula1-kafka-influxdb.env \
$(rhoas kafka describe --name formula1-kafka | jq -r .bootstrap_server_host) \
<PATH_TO_JAR>
```

The `<PATH_TO_JAR>` is the path to the application JAR (i.e. `/home/ppatiern/github/formula1-telemetry-kafka/kafka-influxdb/target/f1-telemetry-kafka-influxdb-1.0-SNAPSHOT-jar-with-dependencies.jar`)

### Apache Kafka Streams

Start one of the Apache Kafka Streams based applications by running the following command.

```shell
export F1_STREAMS_INTERNAL_REPLICATION_FACTOR=-1
./formula1-run.sh \
./formula1-kafka-streams.env \
$(rhoas kafka describe --name formula1-kafka | jq -r .bootstrap_server_host) \
<PATH_TO_JAR>
```

The `<PATH_TO_JAR>` is the path to the application JAR (i.e. `/home/ppatiern/github/formula1-telemetry-kafka/streams-avg-speed/target/f1-telemetry-streams-avg-speed-1.0-SNAPSHOT-jar-with-dependencies.jar`)

#### Quarkus

The Quarkus based Apache Kafka Streams application uses a different set of configuration parameters and can be started by running the following command.

```shell
source ./formula1-kafka-streams.env
export KAFKA_BOOTSTRAP_SERVERS=$(rhoas kafka describe --name formula1-kafka | jq -r .bootstrap_server_host)
export QUARKUS_KAFKA_STREAMS_SASL_MECHANISM=PLAIN
export QUARKUS_KAFKA_STREAMS_SECURITY_PROTOCOL=SASL_SSL
export QUARKUS_KAFKA_STREAMS_SASL_JAAS_CONFIG="org.apache.kafka.common.security.plain.PlainLoginModule required username=\"${RHOAS_SERVICE_ACCOUNT_CLIENT_ID}\" password=\"${RHOAS_SERVICE_ACCOUNT_CLIENT_SECRET}\";"
java -jar <PATH_TO_JAR>
```

The `<PATH_TO_JAR>` is the path to the `quarkus-run.jar` (i.e. `/home/ppatiern/github/formula1-telemetry-kafka/quarkus-streams-avg-speed/target/quarkus-app/quarkus-run.jar`)

## Cleaning

In order to clean the deployment, you can run the following script to delete the service accounts and the local env files.

```shell
./delete-service-accounts.sh
```

Next step deleting the topics.

```shell
./delete-topics.sh
```

Then finally delete the Apache Kafka instance.

```shell
rhoas kafka delete -y --name formula1-kafka
```