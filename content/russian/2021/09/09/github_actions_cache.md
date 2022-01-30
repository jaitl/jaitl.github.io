---
title: "Github Actions кеширование зависимостей"
author: ""
type: ""
date: 2021-09-09T19:43:15+03:00
subtitle: ""
image: ""
tags: []
private: false
---
В этой заметке я подготовил пару сниппетов кода для быстрого включения кеширования в Github Actions.
Кеширвоание зависимостей позволяет существенно повысить скорость сборки проекта.

<!--more-->
Статья из моего telegram канала: [Senior's Blog](https://t.me/seniorsITBlog). Подписывайтесь на канал ;-)

Github Actions умеет кешировать файлы. 
Например, вот так можно кешировать `Gradle Wrapper` и зависимости приложения, в результате они не будут скачиваться при каждой сборке:
```
# Cache Gradle dependencies
- name: Setup Gradle Dependencies Cache
  uses: actions/cache@v2
  with:
    path: ~/.gradle/caches
    key: ${{ runner.os }}-gradle-caches-${{ hashFiles('**/*.gradle', '**/*.gradle.kts') }}

# Cache Gradle Wrapper
- name: Setup Gradle Wrapper Cache
  uses: actions/cache@v2
  with:
    path: ~/.gradle/wrapper
    key: ${{ runner.os }}-gradle-wrapper-${{ hashFiles('**/gradle/wrapper/gradle-wrapper.properties') }}
```

## JVM
В случае если вы используете [`actions/setup-java`](https://github.com/actions/setup-java),
то его вторая версия уже поддерживает кеширование зависимостей и враперов, достаточно ее включить:
```
  - name: Set up JDK
    uses: actions/setup-java@v2
    with:
      distribution: zulu
      java-version: 11
      cache: gradle
```
Поддерживаются gradle и maven.

[Подробнее в офф документации](https://docs.github.com/en/actions/guides/caching-dependencies-to-speed-up-workflows)
