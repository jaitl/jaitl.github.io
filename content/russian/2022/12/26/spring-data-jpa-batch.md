---
title: "Батчевое сохранение данных в Spring Data JPA"
author: ""
type: ""
date: 2022-12-26T12:00:00+03:00
subtitle: ""
image: ""
tags: []
private: false
---
В этой статье я исследую стратегии генерации идентификаторов для сущностей на предмет их совместимости с батчевой вставкой в БД. В качестве ORM используется Spring Data JPA, а в качестве БД - PostgreSQL.
<!--more-->

Я провел теоретическое исследование в результате которого выяснилось, что для батчевой вставки (batch insert) данных идентификаторы должны генерироваться с использованием стратегии `GenerationType.SEQUENCE`, которая в свою очередь использует отдельную сущность в БД под названием [sequence](https://www.postgresql.org/docs/10/sql-createsequence.html) для батчевой генерации идентификаторов (`primary key`).

Теоретическое исследование статей в интернете показывает, что стратегия `GenerationType.IDENTITY` позволяющая использовать стандартный тип `serial` для авто-генерации идентификаторов не работает с батчевой вставкой. Судя по документации в случае `GenerationType.IDENTITY` Spring игнорирует батчевую вставку даже если она включена в настройках.

> When we want to use batching for inserts, we should be aware of the primary key generation strategy. If our entities use the GenerationType.IDENTITY identifier generator, Hibernate will silently disable batch inserts.

[baeldung.com](https://www.baeldung.com/jpa-hibernate-batch-insert-update#id-generation-strategy)

Я не люблю плодить дополнительные сущности и хотел бы дальше использовать тип `serial` для идентификаторов не создавая дополнительно сущность `sequence`, поэтому в этой статье я хочу провести практическое исследование и выяснить возможно ли использовать стратегию `GenerationType.IDENTITY` для генерирования идентификаторов при батчевой вставке.

## Включение батчевой вставки и логирования запросов
[application.yml](https://github.com/jaitl/spring-jpa-batch-insert/blob/main/src/main/resources/application.yml):
```yaml
spring:
  jpa:  
    properties:  
      hibernate:  
        generate_statistics: true
        order_inserts: true
        jdbc.batch_size: 100
```

Флаги:
* `generate_statistics` - включает генерирование статистики о запросах на уровне `Hibernate`
* `order_inserts` - включает сортировку запросов по имени таблицы. В случае если инсерты не отсортированы по имени таблицы, они не могут быть объеденины в один батч и будут разделены на несколько батчей
* `batch_size` - размер батча

Для более точного логирования я использую библиотеку [datasource-proxy](https://github.com/jdbc-observations/datasource-proxy), которая оборачивает спринговый `datasource` в `datasource-proxy` и логирует запросы на уровне драйвера. Настраивается эта библиотека с помощью [DatasourceProxyBeanPostProcessor](https://github.com/jaitl/spring-jpa-batch-insert/blob/main/src/main/java/pro/jaitl/spring/jpa/batch/log/DatasourceProxyBeanPostProcessor.java).

## GenerationType.IDENTITY
Начнем эксперимент со стратегии `GenerationType.IDENTITY`. Ниже приведен sql код создания таблицы для сущности, код сущности, репозитория и теста, полный код можно найти на [Github](https://github.com/jaitl/spring-jpa-batch-insert).

### SQL код таблицы
```sql
create table identity_table  
(  
    id    serial        primary key,  
    name  varchar(100)  NOT NULL,  
    ts    timestamp     default now()  
);
```

### IdentityEntity
```java
@Entity
@Table(name = "identity_table")  
public class IdentityEntity {  
  @Id  
  @GeneratedValue(strategy = GenerationType.IDENTITY)  
  private Integer id;  
  
  private String name;  
  
  private Instant ts;  
}
```

### IdentityRepository
```java
@Repository  
public interface IdentityRepository extends JpaRepository<IdentityEntity, Integer> {  }
```

### Тест
Размер батча установлен в 100, количество записей которые сохраняются в бд - 1000.

[Код теста](https://github.com/jaitl/spring-jpa-batch-insert/blob/main/src/test/java/pro/jaitl/spring/jpa/batch/IdentityRepositoryBatchTest.java)
```java
@SpringBootTest  
class IdentityRepositoryBatchTest {  
  
  @Autowired  
  private IdentityRepository repository;  
  
  @Test  
  public void test() {  
    long size = 1000;  
    List<IdentityEntity> identityEntities = new ArrayList<>();  
  
    for (int i = 0; i < size; i += 1) {  
      IdentityEntity entity = new IdentityEntity();  
      entity.setName("Test identity number: " + i);  
      entity.setTs(Instant.now());  
      identityEntities.add(entity);  
    }  
  
    repository.saveAll(identityEntities);  
    assertEquals(size, repository.count());  
  }  
}
```

Лог с результатом от `Hibernate`:
```
114701 nanoseconds spent preparing 1 JDBC statements;
1995614 nanoseconds spent executing 1 JDBC statements;
0 nanoseconds spent executing 0 JDBC batches;
```

Лог с результатом от `datasource-proxy`:
```
Type:Prepared, Batch:False, QuerySize:1, BatchSize:0
Query:["insert into identity_table (name, ts) values (?, ?)"]
Params:[(Test identity number: 0,2022-12-27 11:01:20.637046)]
```

Как можно видеть из логов, приложением было выполнено 1000 запросов в БД, а количество батчей - 0. Следовательно, батчевые вставки не выполняются в случае стратегии `GenerationType.IDENTITY` и документация не врет (ну а вдруг бы врала, всякое бывает).

## GenerationType.SEQUENCE
Теперь перейдем к стратегии `GenerationType.SEQUENCE`. Ниже приведен sql код создания таблицы для сущности, код сущности, репозитория и теста, полный код можно найти на [Github](https://github.com/jaitl/spring-jpa-batch-insert).

### SQL код таблицы
Как можно заметить тип поля идентификатора - `integer`, потому что для инкрементальной генерации идентификаторов `Hibernate` будет использовать последовательность `sequence_id_auto_gen`, следовательно тип `serial` для идентификатора ненужен.

```sql
create table sequence_table  
(  
    id    integer       primary key,  
    name  varchar(100)  NOT NULL,  
    ts    timestamp     default now()  
);  
  
create sequence sequence_id_auto_gen increment 100;
```

Последовательность `sequence_id_auto_gen` имеет параметр `increment`, который должен быть согласован с параметром `allocationSize` из аннотации `@SequenceGenerator`.

### SequenceEntity

Параметр `allocationSize` указывает количество идентификаторов, которые будут сгенерированы за один запрос в БД к последовательности `sequence_id_auto_gen`. Размер это параметра прямо влияет на производительность батчевой вставки, исследование ниже.

```java
@Entity  
@Table(name = "sequence_table")  
public class SequenceEntity {  
  @Id  
  @SequenceGenerator(name = "sequence_id_auto_gen", allocationSize = 100)  
  @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "sequence_id_auto_gen")  
  private Integer id;  
  
  private String name;  
  
  private Instant ts;  
}
```

### SequenceRepository
```java
@Repository  
public interface SequenceRepository extends JpaRepository<SequenceEntity, Integer> {  }
```

### Тесты
#### Первый тест: размер батча 100, количество записей 1000, вставляется сразу 1000 сущностей

[Код тестов](https://github.com/jaitl/spring-jpa-batch-insert/blob/main/src/test/java/pro/jaitl/spring/jpa/batch/SequenceRepositoryBatchTest.java)
```java
@SpringBootTest  
class SequenceRepositoryBatchTest {  
  
  @Autowired  
  private SequenceRepository repository;  
  
  @Test  
  public void test1() {  
    long size = 1000;  
    List<SequenceEntity> sequenceEntities = new ArrayList<>();  
  
    for (int i = 0; i < size; i += 1) {  
      SequenceEntity entity = new SequenceEntity();  
      entity.setName("Test identity number: " + i);  
      entity.setTs(Instant.now());  
      sequenceEntities.add(entity);  
    }  
  
    repository.saveAll(sequenceEntities);  
    assertEquals(size, repository.count());  
  }  
}
```

Результаты при `allocationSize = 10`

Логи от `Hibernate`:
```
9654469 nanoseconds spent preparing 102 JDBC statements;
243283459 nanoseconds spent executing 101 JDBC statements;
99123235 nanoseconds spent executing 10 JDBC batches;
311794956 nanoseconds spent executing 1 flushes (flushing a total of 1000 entities and 0 collections);
```
Логи от `datasource-proxy`:
```
Name:MyDS, Connection:7, Time:12, Success:True
Type:Prepared, Batch:True, QuerySize:1, BatchSize:100
Query:["insert into sequence_table (name, ts, id) values (?, ?, ?)"]
```


Результаты при `allocationSize = 50`

Логи от `Hibernate`:
```
2575582 nanoseconds spent preparing 22 JDBC statements;
64088523 nanoseconds spent executing 21 JDBC statements;
166666099 nanoseconds spent executing 10 JDBC batches;
479391860 nanoseconds spent executing 1 flushes (flushing a total of 1000 entities and 0 collections);
```
Логи от `datasource-proxy`:
```
Name:MyDS, Connection:7, Time:13, Success:True
Type:Prepared, Batch:True, QuerySize:1, BatchSize:100
Query:["insert into sequence_table (name, ts, id) values (?, ?, ?)"]
```

Результаты при `allocationSize = 100`

Логи от `Hibernate`:
```
1226502 nanoseconds spent preparing 12 JDBC statements;
45371272 nanoseconds spent executing 11 JDBC statements;
81173417 nanoseconds spent executing 10 JDBC batches;
280084078 nanoseconds spent executing 1 flushes (flushing a total of 1000 entities and 0 collections);
```
Логи от `datasource-proxy`:
```
Name:MyDS, Connection:7, Time:6, Success:True
Type:Prepared, Batch:True, QuerySize:1, BatchSize:100
Query:["insert into sequence_table (name, ts, id) values (?, ?, ?)"]
```

Из лога можно сделать два вывода:
1. Батчевая вставка действительно работает.
2. Количество запросов к БД зависит от трех параметров:
	1. Количества сущностей на вставку.
	2. Размера батча.
	3. Параметра `allocationSize`, т.е. количества идентификаторов, которые генерируются за один запрос в БД к `sequence_id_auto_gen`.

Теперь рассмотрим откуда берется 101 + 10 запросов к БД на примере лога первого результата:
1. `Hibernate` делает 1 запрос к `sequence_id_auto_gen` что бы получить текущее значение последовательности.
2. Параметр `allocationSize` установлен в значение 10, следовательно выполняется 1000/10 = 100 запросов к БД что бы сгенерировать 1000 идентификаторов по 10 идентификаторов за запрос. 
3. Параметр `batch_size` установлен в 100, следовательно выполняется 10 батчевых запросов по 100 сущностей в каждом, что бы сохранить батч из 1000 сущностей.

В итоге мы получаем 1 + 100 + 10 = 111 запросов к БД.

#### Второй тест: размер батча 100, количество записей 1000, вставляются по 100 сущностей за раз
Как можно видеть в коде первого теста, я сначала генерирую 1000 сущностей, затем сохраняю их разом в БД, а `Hibernate` сам делит их на 10 батчей по 100 сущностей. Во втором тесте я генерирую и сохраняю по 100 сущностей за раз.

```java
@Test
public void test2() throws Exception {
    for (int k = 0; k < 10; k += 1) {
        List<SequenceEntity> sequenceEntities = new ArrayList<>();
        for (int i = 0; i < 100; i += 1) {
            SequenceEntity entity = new SequenceEntity();
            entity.setName("Test identity number: " + (k + i));
            entity.setTs(Instant.now());
            sequenceEntities.add(entity);
        }
        repository.saveAll(sequenceEntities);
    }
    assertEquals(1000, repository.count());
}
```

В результате запуска теста при `allocationSize = 100` мы видим 10 подобных записей в логе от `Hibernate`:
```
564714 nanoseconds spent preparing 3 JDBC statements;
5325416 nanoseconds spent executing 2 JDBC statements;
12250282 nanoseconds spent executing 1 JDBC batches;
75982849 nanoseconds spent executing 1 flushes (flushing a total of 100 entities and 0 collections);
```
А так же 10 подобных записей в логе от `datasource-proxy`:
```
Name:MyDS, Connection:7, Time:9, Success:True
Type:Prepared, Batch:True, QuerySize:1, BatchSize:100
Query:["insert into sequence_table (name, ts, id) values (?, ?, ?)"]
```

Из анализа логов следует что мы так же получаем 10 батчей по 100 сущностей в каждом.

#### Третий тест: размер батча 100, количество записей 1000, вставляются по 50 сущностей за раз
В третьем тесте я решил проверить, что будет если передать меньше сущностей чем установленно в настройке `batch_size`.

```java
@Test
public void test3() {
    for (int k = 0; k < 20; k += 1) {
        List<SequenceEntity> sequenceEntities = new ArrayList<>();
        for (int i = 0; i < 50; i += 1) {
            SequenceEntity entity = new SequenceEntity();
            entity.setName("Test identity number: " + (k + i));
            entity.setTs(Instant.now());
            sequenceEntities.add(entity);
        }
        repository.saveAll(sequenceEntities);
    }
    assertEquals(1000, repository.count());
}
```

В результате запуска теста при `allocationSize = 100` мы видим 20 подобных записей в логе от `Hibernate`:
```
309752 nanoseconds spent preparing 3 JDBC statements;
3794706 nanoseconds spent executing 2 JDBC statements;
7992646 nanoseconds spent executing 1 JDBC batches;
15285352 nanoseconds spent executing 1 flushes (flushing a total of 50 entities and 0 collections);
...
67387 nanoseconds spent preparing 1 JDBC statements;
0 nanoseconds spent executing 0 JDBC statements;
4282457 nanoseconds spent executing 1 JDBC batches;
15285352 nanoseconds spent executing 1 flushes (flushing a total of 50 entities and 0 collections);
```
А так же 20 подобных записей в логе от `datasource-proxy`:
```
Name:MyDS, Connection:7, Time:6, Success:True
Type:Prepared, Batch:True, QuerySize:1, BatchSize:50
Query:["insert into sequence_table (name, ts, id) values (?, ?, ?)"]
```

Как видно из логов, теперь размер батча равен 50, ровно столько я и передаю в метод `saveAll` в третьем тесте. Следовательно, `Hibernate` не объединяет мелкие батчи в большие для достижения размера батча из параметра `batch_size`.

Так же из интересного можно проанализировать запросы к `sequence_id_auto_gen` на получения идентификаторов:
1. В первом логе мы видим два запроса к `sequence_id_auto_gen`: один на получение начального значения и один на получение 100 идентификаторов.
2. Во втором логе мы видим 0 запросов к `sequence_id_auto_gen`, потому что в первом запросе мы получили 100 идентификаторов, а использовали 50, так как количество сохраняемых сущностей равняется 50, следовательно осталось еще 50 идентификаторов для второго батча и еще один запрос к `sequence_id_auto_gen` делать ненужно.

#### Четвертый тест: сохранение по 1 записи
Осталось проверить какой будет размер батча, если сохранять сущности через метод `save` поштучно. Для этого я написал четвертый тест:

```java
@Test
public void test4() {
    long size = 1000;
    for (int i = 0; i < size; i += 1) {
        SequenceEntity entity = new SequenceEntity();
        entity.setName("Test identity number: " + i);
        entity.setTs(Instant.now());
        repository.save(entity);
    }
    assertEquals(size, repository.count());
}
```

В результате запуска теста при `allocationSize = 100` мы получаем 1000 подобных записей в логе от `Hibernate`:
```
12684 nanoseconds spent preparing 2 JDBC statements;
1794385 nanoseconds spent executing 1 JDBC statements;
1466591 nanoseconds spent executing 1 JDBC batches;
1788887 nanoseconds spent executing 1 flushes (flushing a total of 1 entities and 0 collections);
...
21034 nanoseconds spent preparing 1 JDBC statements;
0 nanoseconds spent executing 0 JDBC statements;
1222957 nanoseconds spent executing 1 JDBC batches;
1572550 nanoseconds spent executing 1 flushes (flushing a total of 1 entities and 0 collections);
```
А так же 1000 подобных записей в логе от `datasource-proxy`:
```
Name:MyDS, Connection:733, Time:1, Success:True
Type:Prepared, Batch:True, QuerySize:1, BatchSize:1
Query:["insert into sequence_table (name, ts, id) values (?, ?, ?)"]
```

Из анализа этого лога видно, что размер батча - 1, следовательно агрегации в большие батчи нет.

## Вывод
Думаю не стоит лишний раз писать, что батчевая вставка работает быстрее одиночной вставки, этот вывод мы опустим.

Статистика от `Hibernate` соответствует метрикам от `datasource-proxy`, так что для поверхностного анализа необязательно подключать и настраивать библиотеку `datasource-proxy`.

Целью статьи было исследовать действительно ли при стратегии генерации идентификаторов `GenerationType.IDENTITY` не работает батчевая вставка. Как можно видеть из результатов - это чистая правда. Батчевая вставка не работает в данном случае, даже если она явно включена в настройках. Spring игнорирует эту настройку и не пишет в логи предупреждения об этом.

При использовании стратегии `GenerationType.SEQUENCE` следует обращать внимание не только на количество сущностей и размер батча, но и на параметр `allocationSize`, так как он тоже оказывает влияние на количество запросов к БД. В идеале значение этого параметра должно равняться размеру батча, что бы идентификаторы для всего батча сгенерировались за один запрос в БД к `sequence_id_auto_gen`.

При сохранении батча через `saveAll` видим следующее поведение:
1. Если размер батча больше `batch_size`, то он делится на несколько более мелких батчей по `batch_size` сущностей в каждом.
2. Если размер батча меньше `batch_size`, то батч не агрегируется в более большой батч, а отправляется в БД в том размере что есть.

В случае сохранения сущностей по одной через метод `save` размер батча равен 1, т.е. сущности так же не агрегируются в более большой батч.

## Версии
Версии всех зависимостей можно посмотреть в [build.gradle](https://github.com/jaitl/spring-jpa-batch-insert/blob/main/build.gradle)

Основные версии:
* PostgreSQL 10
* Spring Boot 2.7.7

## Полезные ссылки:
* [Исходный код примеров](https://github.com/jaitl/spring-jpa-batch-insert)
* [Batch Insert/Update with Hibernate/JPA](https://www.baeldung.com/jpa-hibernate-batch-insert-update)
* [Batching in Hibernate/JPA](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#batch)
