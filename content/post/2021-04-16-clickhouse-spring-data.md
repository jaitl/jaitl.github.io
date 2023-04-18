---
title: "Working With ClickHouse From Spring Data Using MySql Driver"
author: ""
type: ""
date: 2021-04-16T12:12:00+03:00
subtitle: ""
image: ""
tags: []
url: /post/2021/04/16/clickhouse-spring-data
private: true
---
Some time ago I got a task to write a service that inserts data into ClickHouse. My team uses Kotlin and Spring Framework so I decided to try Spring Data JDBC as ORM framework for ClickHouse. After some research, I found that ClickHouse has a [MySql interface](https://clickhouse.tech/docs/en/interfaces/mysql). Thus, probably, Spring Data JDBC is able to communicate with ClickHouse using MySql driver.

<!--more-->

Unfortunately, Spring Data JDBC doesn't work with ClickHouse out of the box thought MySql driver. Several problems have to be solved before this approach will work out.

This article is about overcoming the problems that I encountered while I was implementing the task and solutions for them.

---

***Update***: after this article has been written I found out about [ClickHouse dialect for Spring Data JDBC](https://github.com/pelenthium/clickhouse-dialect-spring-boot-starter). It seems more correct way using ClickHouse from Spring Data. Examine that library before implementing my approach.

---

***Disclaimer***: I write about a rare use case for ClickHouse. ClickHouse is OLAP DB and cannot be used as OLTP DB so unless you are confident this approach is right for you don't repeat it. ClickHouse doesn't support UPDATE/DELETE and many others SQL statements, be careful when you choose ClickHouse for your project.

---

## Demo Project
For this article, I apply Kotlin, Spring Boot, and [the official docker image](https://hub.docker.com/r/yandex/clickhouse-server) of ClickHouse. You can find source code on my GitHub in [this repo](https://github.com/jaitl/clickhouse-spring-data-demo). In the `README.MD` file there is a guide that explains to you how to run and test the demo project.

### API and DB Schema for the Demo Project
I’ve created a simple API and simple DB schema in the demo project for demonstrating basic principles. My API has only two methods: ***create an item*** and ***get all items*** because ClickHouse hasn’t support update and delete operations therefore I can’t implement all CRUD methods.

`DataItem` is my data model for the `items` table. Items from the `items` table will be mapped to `DataItem`.

```kotlin
data class DataItem(
    val itemId: String,
    val timestamp: Timestamp,
    val data: String,
    val list: StringList
)
```

```sql
CREATE TABLE spring_demo.items
(
    item_id String,
    timestamp DateTime64,
    data String,
    list Array(String)
)
ENGINE = MergeTree()
PRIMARY KEY (item_id);
```

Below are endpoints for my API:

```
POST v1/demo/item
GET v1/demo/item
```

`POST` method will create a new item with `DataItem` model and `GET` method will return all items from the `items` table.

---

Let’s take a look at the problems that I met and how to resolve them.

## Problem 1. Transaction
Spring Data perform commit transactions after each query but ClickHouse doesn’t support transactions so the exception will be thrown during execution queries to ClickHouse.

To resolve this problem I disabled committing transactions by overriding the `PlatformTransactionManager` class with empty `commit` and `rollback` methods.

```kotlin
@Configuration
class ApplicationConfiguration {
    @Bean
    fun transactionManager(): PlatformTransactionManager {
        return object : PlatformTransactionManager {
            override fun getTransaction(definition: TransactionDefinition?): TransactionStatus =
                SimpleTransactionStatus()

            override fun commit(status: TransactionStatus) {
            }

            override fun rollback(status: TransactionStatus) {
            }
        }
    }
}
```

## Problem 2. Debug Logging Level
The MySql driver will crash with the exception when DEBUG level is enabled for Spring Data JDBC. It happens because the driver tried to execute the `WARNINGS` command to fetch additional information for debugging from DB but ClickHouse doesn’t support this statement.

To resolve this problem I set the logger level to `INFO` for Spring Data JDBC logger.

```xml
<logger name="org.springframework.jdbc" level="INFO" />
```

## Problem 3. Array Data Type
MySql doesn’t support array data type therefore MySql driver throws an exception when I persist a list or an array to DB.

To resolve this problem I created a custom wrapper around the list data type and custom converters for the custom wrapper.

```kotlin
data class StringList(val list: List<String>)

@ReadingConverter
class StringListReadingConverter : Converter<String, StringList> {
    override fun convert(source: String): StringList {
        val list = source.removePrefix("[").removeSuffix("]").replace("'", "").split(",")
        return StringList(list)
    }
}

@WritingConverter
class StringListWritingConverter : Converter<StringList, String> {
    override fun convert(source: StringList): String {
        return source.list.joinToString(",", "[", "]") { "'$it'" }
    }
}
```

Don’t forget to register the custom conversions.


```kotlin
@Configuration
class DbConfiguration : AbstractJdbcConfiguration() {
    @Bean
    override fun jdbcCustomConversions(): JdbcCustomConversions {
        return JdbcCustomConversions(
            mutableListOf(
                StringListWritingConverter(),
                StringListReadingConverter()
            )
        )
    }
}
```

If you meet the problem with unsupported data types you have to create a custom converter for each one.

## Problem 4. Primary ID Generation
ClickHouse hasn’t auto-increment data type for an `ID`. ClickHouse supposes the `ID` field is assigned by the user. The problem arises when Spring Data tries to update an item instead of creating a new one if you fill out the `ID` field of the item within your code.

To resolve this problem I inherited my model from `Persistable` interface and override the `isNew` method so that it always returns `true`.

```kotlin
data class DataItem(
    val itemId: String,
    val timestamp: Timestamp,
    val data: String,
    val list: StringList
) :
    Persistable<String> {
    override fun getId(): String = itemId
    override fun isNew(): Boolean = true
}
```

---

## Conclusion
As you can see there are some difficulties when you work with ClickHouse from Spring Data JDBC. Fortunately, all the difficulties can be solved and you can enjoy inserting data to ClickHouse with a powerful ORM library without mapping your entities to SQL statements by hand.

Thanks for reading. I hope this was helpful. If you have any questions, feel free to leave a response.

## Recourses
1. [Source Code of the demo project](https://github.com/jaitl/clickhouse-spring-data-demo)
2. [ClickHouse MySql interface](https://clickhouse.tech/docs/en/interfaces/mysql)
3. [ClickHouse dialect for Spring Data JDBC](https://github.com/pelenthium/clickhouse-dialect-spring-boot-starter)
