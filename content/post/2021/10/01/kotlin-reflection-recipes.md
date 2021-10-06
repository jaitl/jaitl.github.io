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
Kotlin Reflection has poor documentation so I decided to gather together all code samples that I used when I wrote my library.
I hope this article will help you to save time.

This article contains a collection of recipes for Kotlin Reflection.
You can find and play with all samples from the article in my [Github repository](https://github.com/jaitl/kotlin-reflection-examples).
<!--more-->

## Original reflection types
Read these articles before proceeding:
1. [Official documentation about reflection](https://kotlinlang.org/docs/reflection.html)
2. [Reflection with Kotlin article](https://www.baeldung.com/kotlin/reflection)

`KClass` and a `KType` are two originals classes where Kotlin reflection begins.

[KClass](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-class/) represents a class. It contains information about a class name, constructors, members, and so on.

[KType](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/-k-type/) represents a type. It contains a `KClass` and a `type argument` for generics types.

## [Recipe 1](https://github.com/jaitl/kotlin-reflection-examples/blob/main/examples/src/test/kotlin/pro/jaitl/kotlin/reflection/KClassTest.kt): How to get a KClass
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

## [Recipe 2](https://github.com/jaitl/kotlin-reflection-examples/blob/main/examples/src/test/kotlin/pro/jaitl/kotlin/reflection/KTypeTest.kt): How to get a KType
### Method 1: From a KClass
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
Unlike the previous example, here we lose all information about a type argument.
```kotlin
val kClass: KClass<List<*>> = List::class
val kType: KType = kClass.starProjectedType
assertEquals(KTypeProjection(null, null), kType.arguments.first())
println(kType) // kotlin.collections.List<*>
```

### Method 2: From the typeOf<> method
The [typeOf<>()](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/type-of.html) method is experimental now. It will probably be stable in [Kotlin 1.6](https://youtrack.jetbrains.com/issue/KT-45396).
#### Simple type
```kotlin
val kType = typeOf<String>()
println(kType) // kotlin.String
```

#### Generics type with a type argument
The `typeOf<>()` method supports generic types and returns correct type parameters information.
```kotlin
val kType = typeOf<List<String>>()
println(kType) // kotlin.collections.List<kotlin.String>
```

### Method 3: From a constructor parameter
```kotlin
data class MyData(val list: List<String>)

val clazz = MyData::class
val constructor = clazz.primaryConstructor!!
val kType: KType = constructor.parameters.first().type
println(kType) // kotlin.collections.List<kotlin.String>
```

### Method 4: From a member property
```kotlin
data class MyData(val list: List<String>)

val clazz = MyData::class
val parameters = clazz.memberProperties
val kType: KType = parameters.first().returnType
println(kType) // kotlin.collections.List<kotlin.String>
```

There are other ways to get a `KType` I gave you only several examples to show the basic principles.

## [Recipe 4](https://github.com/jaitl/kotlin-reflection-examples/blob/main/examples/src/test/kotlin/pro/jaitl/kotlin/reflection/GenericTypeArgumentTest.kt): How to get a type argument from a generic type
A `KClass` doesn't contain information about a `type argument` only a `KType` contains. So before getting the `type argument` you have to have the `KType` for your type.

```kotlin
val kType = typeOf<Map<String, Int>>()
val arguments = kType.arguments
val keyArgument = arguments.first().type
val valueArgument = arguments.last().type
println("the key type argument: $keyArgument") // the key type argument: kotlin.String
println("the value type argument: $valueArgument") // the value type argument: kotlin.Int
```

## [Recipe 5](https://github.com/jaitl/kotlin-reflection-examples/blob/main/examples/src/test/kotlin/pro/jaitl/kotlin/reflection/RefinedFunctionTest.kt): Refined type parameters
Read these articles before proceeding:
1. [Official documentation about inline functions and refined type parameters](https://kotlinlang.org/docs/inline-functions.html).
2. [StackOverflow question about refined type parameters](https://stackoverflow.com/questions/45949584/how-does-the-reified-keyword-in-kotlin-work)

### Pass a KClass as an argument
```kotlin
fun printType(clazz: KClass<*>) {
    println(clazz.qualifiedName)
}

printType(String::class) // kotlin.String
```

### Get a KClass from a refined parameter
```kotlin
inline fun <reified T>printType() {
    println(T::class.qualifiedName)
}

printType<String>() // kotlin.String
```

### Get a KClass from an argument
```kotlin
inline fun <reified T>printType(obj: T) {
    println(T::class.qualifiedName)
}

printType("test") // kotlin.String
```

## [Recipe 6](https://github.com/jaitl/kotlin-reflection-examples/blob/main/examples/src/test/kotlin/pro/jaitl/kotlin/reflection/ClassTraverseTest.kt): How to traverse a class
A class for examination:
```kotlin
annotation class FirstTestAnnotation
annotation class SecondTestAnnotation

@FirstTestAnnotation
@SecondTestAnnotation
class MyClass(
    @field:FirstTestAnnotation val int: Int,
    @field:SecondTestAnnotation val string: String
) {

    data class InternalClassDouble(val double: Double)
    data class InternalClassString(val string: String)

    fun method1ReturnUnit() {

    }

    fun method2ReturnsString(): String = string
}
```

### Properties traverse
```kotlin
val data = MyClass(123, "test")
val clazz = data::class

val properties = clazz.memberProperties

properties.forEach { println("name: ${it.name}, value: ${it.getter.call(data)}") }
```

The code will print:
```
name: int, value: 123
name: string, value: test
```

### Declared Methods traverse
```kotlin
val data = MyClass(123, "test")
val clazz = data::class

val methods = clazz.declaredMemberFunctions

methods.forEach { println("name: ${it.name}, returns: ${it.call(data)}") }
```

The code will print:
```
name: method1ReturnUnit, returns: kotlin.Unit
name: method2ReturnsString, returns: test
```

### All Methods traverse
```kotlin
val data = MyClass(123, "test")
val clazz = data::class

val methods = clazz.memberFunctions

methods.forEach { println("name: ${it.name}") }
```

The code will print:
```
name: method1ReturnUnit
name: method2ReturnsString
name: equals
name: hashCode
name: toString
```

### A method call
```kotlin
val data1 = MyClass(123, "test")
val data2 = MyClass(321, "tset")

val clazz = MyClass::class

val equalsMethod = clazz.memberFunctions.find { it.name == "equals" }!!

val result = equalsMethod.call(data1, data2)

println("data1 equals data2: $result")
```

The code will print:
```
data1 equals data2: false
```

### Nested classes traverse
```kotlin
val data = MyClass(123, "test")
val clazz = data::class

val nestedClasses = clazz.nestedClasses

nestedClasses.forEach { println("${it.qualifiedName}") }
```

The code will print:
```
MyClass.InternalClassDouble
MyClass.InternalClassString
```

### Class annotations traverse
```kotlin
val clazz = MyClass::class
clazz.annotations.forEach { println("${it.annotationClass}") }
```

The code will print:
```
class FirstTestAnnotation
class SecondTestAnnotation
```

### Field annotations traverse
Read these articles before proceeding:
1. [Annotation use-site targets](https://kotlinlang.org/docs/annotations.html#annotation-use-site-targets)
2. [Stackoverflow question about annotations](https://stackoverflow.com/questions/46512924/kotlin-get-field-annotation-always-empty)

```kotlin
val clazz = MyClass::class
val properties = clazz.memberProperties

properties.forEach { property ->
    println("property name: ${property.name}")
    property.javaField?.annotations?.forEach { println("annotation class: ${it.annotationClass}") }
}
```

The code will print:
```
property name: int
annotation class: class FirstTestAnnotation
property name: string
annotation class: class SecondTestAnnotation
```

## [Recipe 7](https://github.com/jaitl/kotlin-reflection-examples/blob/main/examples/src/test/kotlin/pro/jaitl/kotlin/reflection/CreateClassTest.kt): How to create a class
### Using a class constructor
#### Array with parameters
```kotlin
val paramsData = arrayOf(1234, "test", Instant.now())

val clazz = SimpleClass::class
val constructor = clazz.primaryConstructor!!

val dataClass: SimpleClass = constructor.call(*paramsData)
println(dataClass) // SimpleClass(int=1234, string=test, instant=2021-10-01T15:00:00Z)
```
#### Named parameters
```kotlin
val paramsData = mapOf("int" to 1234, "string" to "test", "instant" to Instant.now())

val clazz = SimpleClass::class
val constructor = clazz.primaryConstructor!!
val args = constructor.parameters.associateWith { paramsData[it.name] }

val dataClass: SimpleClass = constructor.callBy(args)
println(dataClass) // SimpleClass(int=1234, string=test, instant=2021-10-01T15:00:00Z)
```

### Using a class name
Kotlin has no method to create a `KClass` by name, so you have to use the `Class.forName` method from Java then convert obtained `Class` to the `KClass`. 
To be consistent, you have to use the `class.java.name` method from Java to get the class name instead of the `class.qualifiedName` method from Kotlin, because they return different names.

```kotlin
val expectedClass = Adt.AdtOne(1.11, "test")
val javaName = expectedClass::class.java.name
println("kotlin name: ${expectedClass::class.qualifiedName}")
println("java name: $javaName")

val clazz = Class.forName(javaName).kotlin
val constructor = clazz.primaryConstructor!!

val actualClass = constructor.call(expectedClass.double, expectedClass.string)
println(actualClass)

assertEquals(Adt.AdtOne::class, actualClass::class)
assertEquals(expectedClass, actualClass)
```

The code will print:
```
kotlin name: Adt.AdtOne
java name: Adt$AdtOne
AdtOne(double=1.11, string=test)
```

As you can see from the output Kotlin's `class.qualifiedName` returns a different name than Java's `class.java.name`.
