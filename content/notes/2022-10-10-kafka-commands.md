---
title: "Kafka: команды и настройки"
slug: "kafka-commands"
author: ""
type: ""
date: 2022-10-10T12:00:00+03:00
subtitle: ""
image: ""
tags: []
private: false
---
В заметке собраны команды и настройки для Kafka, которые я часто использую.

<!--more-->
# Утилиты
## kcat
Если вы часто работаете с Kafka, то вы скорее всего знакомы с официальным cli клиентом:
```bash
./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test.test --from-beginning
./bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

Существует более лаконичный cli клиент [kcat](https://github.com/edenhill/kcat):
```bash
kcat -C -b localhost -t test.test
kcat -L -b localhost
```

kcat работает в разы быстрее официального клиента

# Основные команды
## Управление топиками:
```bash
./bin/kafka-topics.sh --bootstrap-server 10.10.100.1:6667 --create --topic test.test --partitions 2 --replication-factor 1

./bin/kafka-topics.sh --bootstrap-server 10.10.100.1:6667 --delete --topic test.test
```

## Список топиков:
```bash
./bin/kafka-topics.sh --bootstrap-server=10.10.100.1:6667 --list
```
```bash
kcat -L -b 10.10.100.1:6667
```

## Описание топика:
```bash
./bin/kafka-topics.sh --bootstrap-server=10.10.100.1:6667 --describe --topic test.test
```

## Запись в топик
```bash
./bin/kafka-console-producer.sh --bootstrap-server 10.10.100.1:6667 --topic test.test
```
```bash
kcat -P -b localhost -t test.test
```

## Чтение из топика
```bash
./bin/kafka-console-consumer.sh --bootstrap-server 10.10.100.1:6667 --topic test.test
```
```bash
kcat -C -b localhost -t test.test
```

## Изменение конфигов топика
```bash
./bin/kafka-configs.sh --bootstrap-server 10.10.100.1:6667 --alter --entity-type topics --entity-name test.test --add-config retention.ms=-1
```

```bash
./bin/kafka-configs.sh --bootstrap-server 10.10.100.1:6667 --alter --entity-type topics --entity-name test.test --add-config retention.bytes=-1
```

```bash
./bin/kafka-configs.sh --bootstrap-server 10.10.100.1:6667 --alter --entity-type topics --entity-name test.test --add-config retention.ms=-1
```

```bash
./bin/kafka-configs.sh --bootstrap-server 10.10.100.1:6667 --alter --entity-type topics --entity-name test.test --add-config cleanup.policy=compact
```

```bash
./bin/kafka-configs.sh --bootstrap-server 10.10.100.1:6667 --alter --entity-type topics --entity-name test.test --delete-config retention.bytes --delete-config retention.ms
```

# Аутентификация
## Конфиг для java
```
kafka.organization.consumer.security.protocol=SASL_PLAINTEXT
kafka.organization.consumer.sasl.mechanism=SCRAM-SHA-256
kafka.organization.consumer.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="user" password="password";
```

## Пример запроса с аутентификацией
### Получение сообщений (consumer)
```bash
kcat -C -b 10.10.100.1:6667 -X sasl.username=user -X sasl.password=password -t test.test-X security.protocol=sasl_plaintext -X sasl.mechanism=SCRAM-SHA-256
```

### Отправка сообщений (producer)
```bash
kcat -P -b 10.10.100.1:6667 -X sasl.username=user -X sasl.password=password -t test.test -X security.protocol=sasl_plaintext -X sasl.mechanism=SCRAM-SHA-256
```
