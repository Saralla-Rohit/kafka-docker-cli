# Apache Kafka Learning Notes

**Date:** 22 October 2026  
**OS:** Fedora Linux  
**Kafka version:** 4.3.1  
**Deployment:** Docker Compose  
**Mode:** KRaft (no ZooKeeper)

## 1. Project Overview

This project is about learning Apache Kafka by running it in Docker using the official `apache/kafka:4.3.1` image and a Docker Compose configuration.

So far, we have started Kafka, created a topic, and tested communication between a producer and a consumer.

## 2. Kafka Concepts

- **Kafka:** A distributed event-streaming platform.
- **Broker:** The Kafka server that stores and serves messages.
- **Topic:** A named stream of messages, such as `eating`.
- **Partition:** A topic is divided into partitions for scalability and parallel processing.
- **Producer:** An application that publishes messages to a topic.
- **Consumer:** An application that reads messages from a topic.
- **Offset:** A message's position within a partition.
- **Consumer group:** A group of consumers that coordinate to read topic partitions.
- **KRaft:** Kafka's built-in metadata management mechanism, which removes the need for ZooKeeper.

## 3. Docker Compose Setup

The Kafka container is named `kafka` and uses this image:

```text
apache/kafka:4.3.1
```

Kafka is configured to run in KRaft mode, with a single node acting as both broker and controller.

The relevant configuration includes:

```properties
listeners=PLAINTEXT://:9092,CONTROLLER://:9093
controller.quorum.voters=1@localhost:9093
controller.listener.names=CONTROLLER
```

The advertised client listener is configured as:

```text
PLAINTEXT://localhost:9092
```

- Port `9092` is used for client connections.
- Port `9093` is used for controller communication.
- Kafka's data directory was already formatted when we inspected the container logs.

### Useful Docker Compose commands

Run these commands from the directory containing your `compose.yaml` or `docker-compose.yml` file.

Start Kafka:

```bash
docker compose up -d
```

Check running containers:

```bash
docker ps
```

View Kafka logs:

```bash
docker logs kafka
```

Follow logs continuously:

```bash
docker logs -f kafka
```

Stop the Compose services:

```bash
docker compose down
```

**Note:** Do not use `docker compose down -v` casually, because removing volumes can delete persisted Kafka data.

## 4. Enter the Kafka Container

From the Fedora terminal:

```bash
docker exec -it kafka /bin/sh
```

Navigate to Kafka's installation directory:

```sh
cd /opt/kafka
```

List the contents:

```sh
ls
```

The directory contains:

```text
LICENSE
NOTICE
bin
config
libs
logs
```

Kafka's command-line tools are located in the `bin` directory.

Exit the container shell:

```sh
exit
```

**Important:** The commands in the following sections are run inside the container from `/opt/kafka`, unless otherwise specified.

## 5. Create and Manage Topics

### List existing topics

```sh
bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

### Create the `eating` topic

We successfully created the `eating` topic with three partitions and a replication factor of one.

```sh
bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create \
  --topic eating \
  --partitions 3 \
  --replication-factor 1
```

Successful output:

```text
Created topic eating.
```

### Describe the topic

```sh
bin/kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --describe \
  --topic eating
```

This command displays topic details, including:

- Partition count
- Partition leaders
- Replicas
- In-sync replicas

### Why replication factor is one

Our current setup uses a single Kafka broker. Therefore, the replication factor is set to `1`.

A replication factor of `3` generally requires at least three suitable brokers.

## 6. Start a Kafka Producer

A console producer allows us to type messages and publish them to a topic.

From `/opt/kafka`, execute:

```sh
bin/kafka-console-producer.sh \
  --bootstrap-server localhost:9092 \
  --topic eating
```

The producer displays a prompt:

```text
>
```

Type messages and press Enter:

```text
hello
hii
```

Each line is sent as a separate message.

Keep the producer running while testing the consumer. Press `Ctrl+C` to exit.

### Correct command syntax

The following is correct:

```sh
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic eating
```

Common mistakes we encountered:

**Incorrect script name:**

```sh
bin/kafka-producer-start.sh
```

This script was not found. Use `kafka-console-producer.sh` for the console producer.

**Missing option prefix:**

```sh
bin/kafka-console-producer.sh bootstrap-server localhost:9092 --topic eating
```

The correct option is `--bootstrap-server`, with two leading hyphens.

**Missing topic:**

```sh
bin/kafka-console-producer.sh --bootstrap-server localhost:9092
```

This is incomplete because the producer requires a topic. Add `--topic eating`.

## 7. Start a Kafka Consumer

Open a second Fedora terminal.

Enter the container:

```bash
docker exec -it kafka /bin/sh
```

Navigate to Kafka:

```sh
cd /opt/kafka
```

Start the consumer:

```sh
bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic eating
```

Keep the consumer running.

Now type a message into the producer terminal:

```text
hello
```

The consumer should display:

```text
hello
```

During our practice, the consumer displayed messages including:

```text
hello
hii
```

This demonstrated producer-to-consumer message delivery.

### Read messages from the beginning

To request messages from the beginning of the topic, stop the consumer and run:

```sh
bin/kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic eating \
  --from-beginning
```

Consumer offset and group settings can affect which messages are displayed. For a repeatable demonstration, use a new consumer group or configure the offset-reset behavior explicitly.

## 8. Run Kafka Commands Directly from Fedora

You do not have to enter an interactive shell every time.

### List topics

```bash
docker exec kafka sh -lc \
  'cd /opt/kafka && bin/kafka-topics.sh --bootstrap-server localhost:9092 --list'
```

### Describe the `eating` topic

```bash
docker exec kafka sh -lc \
  'cd /opt/kafka && bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic eating'
```

### Start the producer

```bash
docker exec -it kafka sh -lc \
  'cd /opt/kafka && bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic eating'
```

### Start the consumer

```bash
docker exec -it kafka sh -lc \
  'cd /opt/kafka && bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic eating'
```

These commands assume the container is running and Kafka is accepting connections.

## 9. Understanding Kafka Messages and Offsets

Kafka stores messages in topic partitions.

For example, when we publish:

```text
hello
hii
```

The message values are the actual text `hello` and `hii`.

An offset represents the position of a record within its partition. It is not the message value.

Because our topic has three partitions, each partition maintains its own sequence of offsets. Offsets are not one global sequence shared by the entire topic.

Use the console consumer to read message values and the topic-description command to inspect partition metadata.

## 10. Troubleshooting Notes

### `la: not found`

The container's shell does not necessarily define the `la` alias.

Use:

```sh
ls
```

### `kafka-producer-start.sh: not found`

Use the correct console producer:

```sh
bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic eating
```

### `Missing required argument "[topic]"`

Add the required topic option:

```sh
--topic eating
```

### `Error while validating the bootstrap address`

Check that the address option includes both hyphens:

```sh
--bootstrap-server localhost:9092
```

### Connection warnings after restarting Kafka

We observed temporary connection warnings during a Kafka restart. The topic creation command subsequently succeeded.

We also observed recurring KRaft broker-heartbeat timeouts in the logs. If these persist, inspect the current logs and Kafka health rather than repeatedly restarting the container or deleting data.

Useful Fedora commands:

```bash
docker ps
docker logs --since 2m kafka
docker stats kafka --no-stream
```

Do not reformat Kafka storage or delete volumes as a first troubleshooting step.

## 11. Useful Commands Summary

| Task | Command |
|---|---|
| Start Compose services | `docker compose up -d` |
| List containers | `docker ps` |
| View Kafka logs | `docker logs kafka` |
| Follow logs | `docker logs -f kafka` |
| Enter the container | `docker exec -it kafka /bin/sh` |
| Navigate to Kafka | `cd /opt/kafka` |
| List topics | `bin/kafka-topics.sh --bootstrap-server localhost:9092 --list` |
| Describe topic | `bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic eating` |
| Start producer | `bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic eating` |
| Start consumer | `bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic eating` |
| Read from beginning | Add `--from-beginning` to the consumer command |

## 12. Current Progress

- [x] Started Kafka 4.3.1 using Docker Compose.
- [x] Used the Dockerized Kafka container in KRaft mode.
- [x] Entered the container and navigated to `/opt/kafka`.
- [x] Created the `eating` topic with three partitions and replication factor one.
- [x] Listed and explored Kafka command-line tools.
- [x] Started
