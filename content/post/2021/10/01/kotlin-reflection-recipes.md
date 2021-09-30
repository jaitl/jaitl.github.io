---
title: "Kotlin Reflection Recipes"
author: ""
type: ""
date: 2021-10-01T12:00:00+03:00
subtitle: ""
image: ""
tags: []
private: false
---
Collections of recipes for Kotlin Reflection.
<!--more-->

## Original reflection types
KType is
KClass<> is

## Recipe 1: How to get KClass<>
From a type:
```kotlin
val kClass: KClass<String> = String::class
```
From an instance:
```kotlin
val str = "my test"
val kClass: KClass<String> = str::class
```

## Recipe 2: How to get KType
### Method 1: From KClass<>
Perfectly works with simple types.
```kotlin
val kClass: KClass<String> = String::class
val kType: KType = kClass.createType()
```

This method badly works with Generics. You will loose type parameters information
```kotlin
val kClass: KClass<String> = List::class
val kType: KType = kClass.createType()
```

### Method 2: typeOf<> method
Method [typeOf<>](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/type-of.html) is experemental now. Probably it will be stable in Kotlin 1.6.
```kotlin
val kType: KType = typeOf<String>
```
This method supports generics types and saves type parameters information.
```kotlin
val kType: KType = typeOf<List<String>>
```

### Method 3: KParameter

### Method 4: KProperty

## Recipe 3: Class traversal

## Recipe 4: Refined paremeter
Save metadata....

## Recipe 5: Create class by constructor

## Recipe 6: Create class by strings' name

