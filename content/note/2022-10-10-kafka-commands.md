# Kafka

# Утилиты для кафки
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


## Управление топиками:
```bash
./bin/kafka-topics.sh --bootstrap-server 10.10.100.1:6667 --create --topic test.test --partitions 2 --replication-factor 1

./bin/kafka-topics.sh --bootstrap-server 10.10.100.1:6667 --delete --topic test.test
```

## Список топиков:
```bash
./bin/kafka-topics.sh --bootstrap-server=10.10.100.1:6667 --list
kcat -L -b 10.10.100.1:6667
```

## Описание топика:
```bash
./bin/kafka-topics.sh --bootstrap-server=10.10.100.1:6667 --describe --topic test.test
```

## Запись в топик
```bash
kcat -P -b localhost -t test.test
./bin/kafka-console-producer.sh --bootstrap-server 10.10.100.1:6667 --topic test.test
```

## Чтение из топика
```bash
kcat -C -b localhost -t test.test
./bin/kafka-console-consumer.sh --bootstrap-server 10.10.100.1:6667 --topic test.test
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
