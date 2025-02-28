---
layout: post
title: "Loading Spring properties into Immutables and corresponding problems"
date: 2025-02-27 10:00:00 +0100
categories: jekyll update
---

# Introduction
In Spring Boot, handling configuration properties is something that you do all the time.
While Spring makes it easy to bind properties to Java classes, getting it to work with
Immutables ([immutables.github.io](https://immutables.github.io/)) isn’t straightforward and well-documented.
There’s not so much information out there, and some of what you can find is outdated.

In this article I would like to give you some simple examples of loading Spring
properties into Immutables-based classes and show what problems you can encounter.

I would like to jump ahead a bit and say that I wouldn't recommend using Immutables for loading and storing
Spring properties, **especially if your goal is to have an immutable configuration class**.
Java Records or [Lombok](https://projectlombok.org/) are much better suited for achieving this
(if you are targeting to reduce boilerplate code).

> **_NOTE:_**  In the examples, Spring version is `3.4.3`, Immutables version is `2.10.1` and
> Google Guava version is `33.4.0-jre` (I will highlight cases, where Guava is present in the classpath)

# Main issues and blockers
We will use this yaml properties file for all the examples in the section:
```yaml
list-property:
  - item1
  - item2
  - item3
```

#### Problem 1

By default, generated private constructors for `@Value.Immutable` don't have any nullability checks or immutability guarantees.
Therefore, for the next class:

```java
@Value.Immutable
@ConfigurationProperties
public interface ConfigPropertiesExample1 {
  List<String> listProperty();
}
```

The generated constructor will be:

```java
private ImmutableConfigPropertiesExample1(List<String> listProperty) {
  this.listProperty = listProperty;
}
```

And this constructor is used by Spring to inject properties. Spring injects `ArrayList` which is
mutable, so you can change the content after you obtain the objects using the corresponding getter method.

#### Problem 2

If we enforce Immutables to create a constructor with nullability checks and immutability guarantees, then `List` or `Set` 
fields are backed by `Iterable`, so for the next class:

```java
@Value.Style(of = "new", allParameters = true)
@Value.Immutable
@ConfigurationProperties
public interface ConfigPropertiesExample2 {
  List<String> listProperty();
}

```

The generated constructor will be:

```java
public ImmutableConfigPropertiesExample2(Iterable<String> listProperty) {
  this.listProperty = createUnmodifiableList(false, createSafeList(listProperty, true, false));
}
```

But Spring can't inject properties into `Iterable` parameters, so the application startup fails with:
```java
Failed to bind properties under '' to denshchikov.dmitry.app.ImmutableConfigPropertiesExample2:

    Reason: java.lang.NullPointerException: Cannot invoke "java.lang.Iterable.iterator()" because "iterable" is null
```

#### Problem 3
If we enforce Immutables to use `List` instead of `Iterable` by utilizing `@Value.Default` and 
`strictBuilder = true`, then the generated constructor again doesn't have immutability guarantees.

For the next class:
```java
@Value.Immutable
@Value.Style(of = "new", allParameters = true, strictBuilder = true)
@ConfigurationProperties
public interface ConfigPropertiesExample3 {
  @Value.Default
  @SuppressWarnings("immutables:untype")
  default List<String> listProperty() {
    return List.of();
  }
}
```

The generated constructor is:
```java
public ImmutableConfigPropertiesExample3(List<String> listProperty) {
  this.listProperty = Objects.requireNonNull(listProperty, "listProperty");
}
```
And we again can change the content of the list that will be injected by Spring.


#### Problem 4
If we enforce Immutables to use `List` instead of `Iterable` by utilizing 
`builtinContainerAttributes = false`, then the generated constructor again doesn't have immutability guarantees.

Original class:
```java
@Value.Immutable
@Value.Style(of = "new", allParameters = true, builtinContainerAttributes = false)
@ConfigurationProperties
public interface ConfigPropertiesExample4 {
  List<String> listProperty();
}
```

Generated constructor:
```java
public ImmutableConfigPropertiesExample4(List<String> listProperty) {
  this.listProperty = Objects.requireNonNull(listProperty, "listProperty");
}
```
And we again can change the content of the list that will be injected by Spring.

#### Problem 5
If we use Google Guava and thereby enforce Immutables to use `ImmutableList`, then the generated constructor 
guarantees immutability.

So, for the next class:
```java
@Value.Immutable
@ConfigurationProperties
public interface ConfigPropertiesExample5 {
  List<String> listProperty();
}
```

The generated constructor is:
```java
private ImmutableConfigPropertiesExample5(ImmutableList<String> listProperty) {
  this.listProperty = listProperty;
}
```

But Spring can't inject properties into Guava's immutable collections, so the application startup fails with:
```java
Failed to bind properties under 'list-property' to com.google.common.collect.ImmutableList<java.lang.String>:

Property: list-property[2]
Value: "item3"
Origin: class path resource [application.yaml] - 7:5
Reason: failed to convert java.util.ArrayList<?> to com.google.common.collect.ImmutableList<java.lang.String> (caused by java.lang.InstantiationException)
```

# Key takeaways so far
With Immutables it's impossible to get an **immutable** configuration class without some workarounds if you at some point want to have 
`List` or `Set` fields. 

You can perform a small workaround and use array as a type for injected field and then utilize
`@Value.Derived` to eagerly compute a field with a desired type during construction:

```java
@Value.Immutable
@ConfigurationProperties
public interface ConfigPropertiesExample6 {
  String[] listPropertyAsArray();

  @Value.Derived
  default List<String> listProperty() {
    return List.of(listPropertyAsArray());
  }
}
```

It's a bit awkward, but it works. The properties file in this case will be:
```yaml
list-property-as-array:
  - item1
  - item2
  - item3
```

You may also want to utilize `@Value.Redacted` and `@Value.Auxiliary` to hide the unnecessary attribute.


# Examples of successful configuration classes and corresponding properties files

Below I will provide two examples of configuration classes that contain various different fields along with their corresponding
yaml files. The first one will be a fully mutable configuration class. And the second one is immutable 
(but of course, with some workarounds that we discussed).

#### Mutable configuration class
Below we utilize `@Value.Modifiable`, so the generated class will have setters that will be used by Spring for 
properties binding. In comparison with approach from [Problem 1](#problem-1) where we used `@Value.Immutable`, here
you can change all the fields since you have all the setters.
```java
@Value.Modifiable
@Value.Style(strictBuilder = true)
@ConfigurationProperties
public interface ConfigPropertiesExample7 {
  Integer integerProperty();

  String stringProperty();

  @Value.Default
  default List<String> listProperty() {
    return new ArrayList<>();
  }

  @Value.Default
  default Set<String> setProperty() {
    return new HashSet<>();
  }

  Map<String, String> mapProperty();

  Map<String, List<String>> multiMapProperty();
}
```

```yaml
integer-property: 192

string-property: "abc"

list-property:
  - item1
  - item2
  - item3

set-property:
  - item1
  - item2
  - item3

map-property:
  key1: value1
  key2: value2

multimap-property:
  key1:
    - value1
    - value2
  key2:
    - value3
    - value4
```

#### Immutable configuration class
Here I would like to highlight that you don't need a special workaround to parse `Map<String, List<String>>` because the
generated constructor doesn't utilize `Iterable` in this case and Spring can successfully inject properties.
```java
@Value.Immutable
@ConfigurationProperties
public interface ConfigPropertiesExample8 {
  Integer integerProperty();

  String stringProperty();

  String[] listPropertyAsArray();

  String[] setPropertyAsArray();

  @Value.Derived
  default List<String> listProperty() {
    return List.of(listPropertyAsArray());
  }

  @Value.Derived
  default Set<String> setProperty() {
    return Set.of(setPropertyAsArray());
  }

  Map<String, String> mapProperty();

  Map<String, List<String>> multiMapProperty();
}
```

```yaml
integer-property: 192

string-property: "abc"

list-property-as-array:
  - item1
  - item2
  - item3

set-property-as-array:
  - item1
  - item2
  - item3

map-property:
  key1: value1
  key2: value2

multimap-property:
  key1:
    - value1
    - value2
  key2:
    - value3
    - value4
```

# Conclusion
Using Immutables-based classes for binding Spring properties is probably not the best choice due to the limitations 
that we discussed in [Main issues and blockers](#main-issues-and-blockers). But it's still possible in case if you want 
to follow a project style. Especially if you can avoid using `List` or `Set` fields. Or if it's fine for you to use 
the array type as a mediator.

There are plenty of ways to arrange Immutables' annotation attributes, and you can play a bit more to find the one
that suits you best. But the main limitations are still there, and you unfortunately can't avoid them by just using 
the annotation attributes.

You can find the full project in my repository: [https://github.com/DmitryDenshchikov/spring-properties-to-immutables-demo](https://github.com/DmitryDenshchikov/spring-properties-to-immutables-demo)

It contains all the configuration classes and the properties file. You can enable them by uncommenting necessary ones 
in the `@EnableConfigurationProperties` class and uncommenting them in the main method (for being able to see the 
loaded content).
