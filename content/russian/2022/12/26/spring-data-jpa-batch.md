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

Я провел теоретическое исследование, в результате которого выяснилось что для батчевой вставки (batch insert) данных идентификаторы должны генерироваться с использованием стратегии `GenerationType.SEQUENCE`, которая в свою очередь использует отдельную сущность в БД под названием `sequence` для батчевой генерации идентификаторов.

Исследование показывает, что стратегия `GenerationType.IDENTITY` позволяющая использовать стандартный тип serial для авто-генерации инеднификаторов не работает с батчевой вствкой. Судя по документации в случае `GenerationType.IDENTITY` спринг автоматически без предупреждения отключает батчевую вставку, если оно включено.

> When we want to use batching for inserts, we should be aware of the primary key generation strategy. If our entities use the GenerationType.IDENTITY identifier generator, Hibernate will silently disable batch inserts.

[baeldung.com](https://www.baeldung.com/jpa-hibernate-batch-insert-update#id-generation-strategy)

Я не люблю плодить дополнительные сущности, я бы хотел и дальше использовать тип `serial` для идентификаторов, поэтому в этой статье я хочу провести практическое исследование и выяснить возможно ли использовать стратегию `GenerationType.IDENTITY` для генерирования идентификаторов при батчевой вставке.

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
* `show-sql` - включает логирование запросов.
* `generate_statistics` - включает генерирование статистики о запросах.
* `order_inserts` - включает сортировку запросов по имени таблицы. В случае если инсерты не отсортированы по имени таблицы, они не могут быть объеденены в один батч и будут разделены на несколько батчей.
* `batch_size` - максимальный размер батча.

## Эксперимент со стратегией GenerationType.IDENTITY

Начнем эксперимент со стратегии `GenerationType.IDENTITY`. Ниже приветен код sql создания таблицы для сущности, код сущности, репозитория и теста, полный код можно найти на Github.

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

### Test
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

### Результаты
Размер батча установлен в 100, количество записей которые сохраняются в бд - 1000.

```
18167 nanoseconds spent acquiring 1 JDBC connections;
0 nanoseconds spent releasing 0 JDBC connections;
10675034 nanoseconds spent preparing 1000 JDBC statements;
974877739 nanoseconds spent executing 1000 JDBC statements;
0 nanoseconds spent executing 0 JDBC batches;
```

Как можно видеть из лога, приложением было выполнено 1000 запросов в БД, а количество батчей - 0. Из чего можно сделать вывод, что батчевые запросы не выполняются даже при включенов в настройках батчевом режиме вставки.

## Эксперимент со стратегией GenerationType.SEQUENCE
Теперь перейдем к стратегии `GenerationType.SEQUENCE`. Ниже приветен код sql создания таблицы для сущности, код сущности, репозитория и теста, полный код можно найти на Github.

### SQL код таблицы
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

Параметр `allocationSize` указывает количество айдишников, которые будут сгенерированны за один запрос к `sequence` в БД. Размер это параметра прямо влияет на производительность батчевой вставке, подробности в разделе с результаатми.

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

### Test
```java
@SpringBootTest  
class SequenceRepositoryBatchTest {  
  
  @Autowired  
  private SequenceRepository repository;  
  
  @Test  
  public void test() {  
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

### Результаты
Размер батча установлен в 100, количество записей которые сохраняются в бд - 1000.

Результаты `allocationSize = 10`
```
94599456 nanoseconds spent executing 101 JDBC statements;
34484916 nanoseconds spent executing 1 JDBC batches;
```
Результаты `allocationSize = 50`
```
36643832 nanoseconds spent executing 21 JDBC statements;
34662042 nanoseconds spent executing 1 JDBC batches;
```

Результаты при `allocationSize = 100`
```
26290291 nanoseconds spent executing 11 JDBC statements;
27156916 nanoseconds spent executing 1 JDBC batches;
```

Из лога можно сделать два вывода:
1. Батчевая вставка действительно работает.
2. Количество запросов к БД зависит от трех параметров:
	1. Количества сущностей на вставку
	2. Размера батча
	3. Параметра `allocationSize`, т.е. количества идентификаторов, которые генерируются за один запрос к `sequence` в БД

 Если установить параметр `allocationSize` в небольшое значение, например в 10, то при размере батча в 100, потребуется 10 вызовов к БД на генерацию идентификаторов для одного батча.

## Вывод
Думаю не стоит лишний раз писать, что батчевая вставка работает быстрее одиночной вставки, этот вывод мы опустим.

Целью статьи было иссделовать действительно ли при стратегии генерации идентификаторов `GenerationType.IDENTITY` не работает батчевая вставка. Как можно видеть из результатов - это чистай правда, батчевая вставка не работает в данном случае, даже если она явно включена в настройках. Спринг игнорирует эту опцию и не пишет в логи сообщения об этом.

При использовании стратегии `GenerationType.SEQUENCE` нужно обращать внимание не только на количество сущностей и размер батча, но и на параметр `allocationSize`, он так же оказывает влияние на количество запросов к БД. В идеале значение этого параметра должно равняться размеру батча, что бы на весь батч был один запрос в БД на генерацию идентификаторов.