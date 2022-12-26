---
title: "Батчевое сохранение в Spring Data JPA"
author: ""
type: ""
date: 2022-12-26T12:00:00+03:00
subtitle: ""
image: ""
tags: []
private: false
---
В этой статье я исследую стратегии генерации идентификаторов для записей на предмет их совместимости с батчевой вставкой в БД. В качестве ORM используется Spring Data JPA, а в качестве БД - PostgreSQL.
<!--more-->

У меня появилась задача выполнять сохранение данных в батчевом режиеме. В качестве базы данных в проекте у меня используется PostgreSQL.

Я провел теоретическое исследование, в результате которого выяснилось, что для батчевой вставки (batch insert) данных идентификаторы должны генерироваться с использованием стратегии `GenerationType.SEQUENCE`, которая в свою очередь использует отдельную сущность в БД под названием `sequence` для батчевой генерации идентификаторов (primary key).

Теоретическое исследование статей в интернете показывает, что стратегия `GenerationType.IDENTITY` позволяющая использовать стандартный тип `serial` для авто-генерации идентификаторов не работает с батчевой вставкой. Судя по документации в случае `GenerationType.IDENTITY` спринг игнорирует батчевую вставку даже если она включена в настройках.

> When we want to use batching for inserts, we should be aware of the primary key generation strategy. If our entities use the GenerationType.IDENTITY identifier generator, Hibernate will silently disable batch inserts.

[baeldung.com](https://www.baeldung.com/jpa-hibernate-batch-insert-update#id-generation-strategy)

Я не люблю плодить дополнительные сущности, я бы хотел и дальше использовать тип `serial` для идентификаторов не создавая дополнительно сущность `sequence`, поэтому в этой статье я хочу провести практическое исследование и выяснить возможно ли использовать стратегию `GenerationType.IDENTITY` для генерирования идентификаторов при батчевой вставке.

## Включение батчевой вставки и логирования запросов

```yaml
spring:
  jpa:  
    show-sql: true  
      properties:  
        hibernate:  
          generate_statistics: true
          order_inserts: true
          jdbc.batch_size: 100
```

Флаги:
* `show-sql` - включает логирование запросов
* `generate_statistics` - включает генерирование статистики о запросах
* `order_inserts` - включает сортировку запросов по имени таблицы. В случае если инсерты не отсортированы по имени таблицы, они не могут быть объеденины в один батч и будут разделены на несколько батчей
* `batch_size` - максимальный размер батча

## GenerationType.IDENTITY

Начнем эксперимент со стратегии `GenerationType.IDENTITY`. Ниже приветен код sql создания таблицы для сущности, код сущности, репозитория и теста, полный код можно найти на [Github](https://github.com/jaitl/spring-jpa-batch-insert).

### SQL код таблицы
```sql
create table identity_table  
(  
    id    serial primary key,  
    name  varchar(100)  NOT NULL,  
    ts    timestamp    default now()  
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
public interface IdentityRepository extends JpaRepository<IdentityEntity, Integer> {  
}
```

### Тест
Размер батча установлен в 100, количество записей которые сохраняются в бд установлена в 1000.

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

Результат:
```
114701 nanoseconds spent preparing 1 JDBC statements;
1995614 nanoseconds spent executing 1 JDBC statements;
0 nanoseconds spent executing 0 JDBC batches;
```

Как можно видеть из лога, приложением было выполнено 1000 запросов в БД, а количество батчей - 0. Из чего можно сделать вывод, что батчевые вставки не выполняются в случае стратегии `GenerationType.IDENTITY` и документация не врет.

## GenerationType.SEQUENCE
Теперь перейдем к стратегии `GenerationType.SEQUENCE`. Ниже приветен код sql создания таблицы для сущности, код сущности, репозитория и теста, полный код можно найти на [Github](https://github.com/jaitl/spring-jpa-batch-insert).

### SQL код таблицы
Как можно заметить тип поля идентификатора - `integer`, потому что для инкрементальной генерации идентификаторов hibernate будет использовать последовательность `sequence_id_auto_gen`.

```sql
create table sequence_table  
(  
    id    integer primary key,  
    name  varchar(100)  NOT NULL,  
    ts    timestamp    default now()  
);  
  
create sequence sequence_id_auto_gen increment 100;
```

### SequenceEntity

Параметр `allocationSize` указывает количество айдишников, которые будут сгенерированы за один запрос к последовательности `sequence_id_auto_gen` в БД. Размер это параметра прямо влияет на производительность батчевой вставки, подробности в разделе с результатами.

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
public interface SequenceRepository extends JpaRepository<SequenceEntity, Integer> {  
}
```

### Тесты
Первый тест, размер батча установлен в 100, количество записей которые сохраняются в бд установлено в 1000.

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
```
4588674 nanoseconds spent preparing 101 JDBC statements;
215632040 nanoseconds spent executing 100 JDBC statements;
41919958 nanoseconds spent executing 1 JDBC batches;
194059063 nanoseconds spent executing 1 flushes (flushing a total of 1000 entities and 0 collections);

```
Результаты при `allocationSize = 50`
```
930578 nanoseconds spent preparing 21 JDBC statements;
33927601 nanoseconds spent executing 20 JDBC statements;
45626454 nanoseconds spent executing 1 JDBC batches;
176865388 nanoseconds spent executing 1 flushes (flushing a total of 1000 entities and 0 collections);
```

Результаты при `allocationSize = 100`
```
766926 nanoseconds spent preparing 11 JDBC statements;
40908813 nanoseconds spent executing 10 JDBC statements;
53419380 nanoseconds spent executing 1 JDBC batches;
214674739 nanoseconds spent executing 1 flushes (flushing a total of 1000 entities and 0 collections);
```

Из лога можно сделать два вывода:
1. Батчевая вставка действительно работает.
2. Количество запросов к БД зависит от трех параметров:
	1. Количества сущностей на вставку
	2. Размера батча
	3. Параметра `allocationSize`, т.е. количества идентификаторов, которые генерируются за один запрос к `sequence` в БД

Теперь рассмотрим откуда берется такое количество запросов к БД на примере первого лога.
Параметр `allocationSize` установлен в значение 10, следовательно нужно сделать 1000/10 = 100 запросов к БД что бы сгенерировать 1000 идентификаторов. А затем еще один запрос к БД что бы сохранить батч из 1000 сущностей. В итоге мы получаем 100 + 1 запросов к БД. Но почему батч всего один и на 1000 сущностей? При размере батча в 100 из 1000 сущностей мы должны были получить 10 батчей, следовательно 10 батчевых запросов на вставку.

Как можно видеть в коде теста, я сначала генерирую 1000 сущностей, затем сохраняю их разом в БД. Судя по всему hibernate не делит их на отдельные батчи по 100 сущностей, а сразу сохраняет одним батчем в 1000 сущностей. Что бы проверить эту теорию я написал еще один тест:

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

В результате запуска теста при `allocationSize = 10` мы видим 10 подобных записей в логе:
```
638321 nanoseconds spent preparing 11 JDBC statements;
26781131 nanoseconds spent executing 10 JDBC statements;
7646157 nanoseconds spent executing 1 JDBC batches;
45586938 nanoseconds spent executing 1 flushes (flushing a total of 100 entities and 0 collections);
```

Следовательно, предыдущая теория подтверждена, при передаче N записей в метод `saveAll` hibernate не делит их на отдельные саб-батчи, а сохраняет все разом. Тогда возникает следующий вопрос, что будет если передать меньше сущностей чем установленно в настройке `batch_size`? Я написал третий тест что бы это выяснить:

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

В результате запуска теста при `allocationSize = 10` мы видим 20 подобных записей в логе:
```
260032 nanoseconds spent preparing 6 JDBC statements;
8648810 nanoseconds spent executing 5 JDBC statements;
3471284 nanoseconds spent executing 1 JDBC batches;
15285352 nanoseconds spent executing 1 flushes (flushing a total of 50 entities and 0 collections);
```

Как видно из логов теперь размер батча равен 50, ровно столько я и передаю в метод `saveAll` в третьем тесте.
Из этого можно сделать вывод, что в случае использования метода `saveAll`, параметр `batch_size` игнорируется и размер батча равен количеству сущностей, которое передается в метод `saveAll`.

Осталось проверить какой будет размер батча, если сохранять сущности через метод `save` по одной. Для этого я написал четвертый тест:

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

В результате запуска теста при `allocationSize = 10` мы получаем 1000 подобных записей в логе:
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

Из анализа этого лога видно следующее:
1. Размер батча 1 сущность, нет агрегации запросов в батчи по 100 штук, как указано в конфиге
2. hibernate сначала делает запрос на получение 10 идентификаторов, затем последовательно использует их для вставки записей, пока не вставит 10 записей, затем он запрашивает еще 10 идентификаторов и т.д.


## Вывод
Думаю не стоит лишний раз писать, что батчевая вставка работает быстрее одиночной вставки, этот вывод мы опустим.

Целью статьи было исследовать действительно ли при стратегии генерации идентификаторов `GenerationType.IDENTITY` не работает батчевая вставка. Как можно видеть из результатов - это чистая правда, батчевая вставка не работает в данном случае, даже если она явно включена в настройках. Spring игнорирует эту опцию и не пишет в логи сообщения об этом.

При использовании стратегии `GenerationType.SEQUENCE` нужно обращать внимание не только на количество сущностей и размер батча, но и на параметр `allocationSize`, он так же оказывает влияние на количество запросов к БД. В идеале значение этого параметра должно равняться размеру батча, что бы на весь батч был один запрос в БД на генерацию идентификаторов.

В случае сохранения коллекции записей через метод `saveAll`, параметр `batch_size` игнорируется, а размер батча равен количеству сущностей в передаваемой коллекции. В случае сохранения сущностей по одной через метод `save`, размер батча равен 1.