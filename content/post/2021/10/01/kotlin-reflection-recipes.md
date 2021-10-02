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
Read these articles before proceeding:
1. [Official documentation about reflection](https://kotlinlang.org/docs/reflection.html)
2. [Reflection with Kotlin article](https://www.baeldung.com/kotlin/reflection)

Two basic classes in Kotlin reflection is KClass and KType.

[KClass](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-class/) represents a class. It contains information about a class name, constructors, members, and so on.

[KType](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-type/) represents a type. It contains `KClass` and `type arguments` for generics types.


You can find and play all examples from this article in [this repo](https://github.com/jaitl/kotlin-reflection-examples).

## [Recipe 1](https://github.com/jaitl/kotlin-reflection-examples/blob/main/examples/src/test/kotlin/pro/jaitl/kotlin/reflection/KClassTest.kt): How to get KClass
### Method 1: From a type
```kotlin
val kClass: KClass<String> = String::class
```

### Method 2: From an instance
```kotlin
val str = "my test"
val kClass: KClass<String> = str::class
```

### Method 3: From a KType
```kotlin
val kType: KType = typeOf<String>()
val kClass: KClass<String> = kType.classifier as KClass<String>
```

## [Recipe 2](https://github.com/jaitl/kotlin-reflection-examples/blob/main/examples/src/test/kotlin/pro/jaitl/kotlin/reflection/KTypeTest.kt): Where to get KType
### Method 1: From KClass
#### Simple type
```kotlin
val kClass: KClass<String> = String::class
val kType: KType = kClass.createType()
println(kType) // kotlin.String
```

#### Generic type with a type argument
```kotlin
// type for string
val kClassString: KClass<String> = String::class
val kTypeString: KType = kClassString.createType()

// type for list with strings
val kClass: KClass<List<*>> = List::class
val kType: KType = kClass.createType(listOf(KTypeProjection(KVariance.INVARIANT, kTypeString)))
println(kType) // kotlin.collections.List<kotlin.String>
```

#### Generic type with the star type
Unlike previous example, here we loose all information about the `type argument`.
```kotlin
val kClass: KClass<List<*>> = List::class
val kType: KType = kClass.starProjectedType
assertEquals(KTypeProjection(null, null), kType.arguments.first())
println(kType) // kotlin.collections.List<*>
```

### Method 2: From the typeOf<> method
Method [typeOf<>](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/type-of.html) is experemental now. Probably it will be stable in Kotlin 1.6.
#### Simple type
```kotlin
val kType = typeOf<String>()
println(kType) // kotlin.String
```

#### Generic type with a type argument
The `typeOf<>()` method supports generic types and returns type parameters information.
```kotlin
val kType = typeOf<List<String>>()
println(kType) // kotlin.collections.List<kotlin.String>
```

### Method 3: From constructor parametres
```kotlin
data class MyData(val list: List<String>)

val clazz = MyData::class
val constructor = clazz.primaryConstructor!!
val kType: KType = constructor.parameters.first().type
println(kType) // kotlin.collections.List<kotlin.String>
```

### Method 4: From member properties
```kotlin
data class MyData(val list: List<String>)

val clazz = MyData::class
val parameters = clazz.memberProperties
val kType: KType = parameters.first().returnType
println(kType) // kotlin.collections.List<kotlin.String>
```

There are other ways how to get `KType`. I gave you only several examples to show basic principles.

## [Recipe 4](https://github.com/jaitl/kotlin-reflection-examples/blob/main/examples/src/test/kotlin/pro/jaitl/kotlin/reflection/GenericTypeArgumentTest.kt): How to get a type argument from a generic type
`KClass` hasn't information about `type arguments` only `KType` has. So before getting `type arguments` you have to have `KType` for your type.

```kotlin
val kType = typeOf<Map<String, Int>>()
val arguments = kType.arguments
val keyArgument = arguments.first().type
val valueArgument = arguments.last().type
println("key type argument: $keyArgument") // key type argument: kotlin.String
println("value type argument: $valueArgument") // value type argument: kotlin.Int
```

## [Recipe 5](https://github.com/jaitl/kotlin-reflection-examples/blob/main/examples/src/test/kotlin/pro/jaitl/kotlin/reflection/RefinedFunctionTest.kt): Refined type parameters
Read these articles before proceeding:
1. [Official documentation about inline functions and refined type parameters](https://kotlinlang.org/docs/inline-functions.html).
2. [Stackoverflow question about refined type parameters](https://stackoverflow.com/questions/45949584/how-does-the-reified-keyword-in-kotlin-work)

### Pass KClass as an argument
```kotlin
fun printType(clazz: KClass<*>) {
    println(clazz.qualifiedName)
}

printType(String::class) // kotlin.String
```

### Get KClass from a refined parameter
```kotlin
inline fun <reified T : Any>printType() {
    println(T::class.qualifiedName)
}

printType<String>() // kotlin.String
```

## Recipe 6: Class traversal

## Recipe 7: Create class by constructor

## Recipe 8: Create class by name

