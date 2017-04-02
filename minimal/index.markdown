---
layout: minimal
title:  "Minimal: A simple Java API that exposes minimalist interfaces for common functions including identity generation, JSON parsing and reflection."
---

# Introduction

Minimal is a simple Java API that exposes minimalist interfaces for common functions including identity generation, JSON parsing and reflection.

To get started, add the following dependency to your depedency list:

```xml
<dependency>
    <groupId>fm.pattern</groupId>
    <artifactId>minimal</artifactId>
    <version>1.0.0</version>
</dependency>
```

# Identity Generation

The id generator uses a cryptographically strong random number generator to produce sequences of alphanumeric strings that can be used as unique identifiers.

By default, the id generator will produce a 25 character length alphanumeric string, with a mix of uppercase and lowercase characters:

```java
IdGenerator.generateId() => "jDWatQsPtRNOMhYAnVWUDF3LE"
```

You can specify the length of the id to generate by providing a *length* argument:

```java
IdGenerator.generateId(5) => "pciIi"
IdGenerator.generateId(50) => "Sf3hcRp8yIveqZbkx3AMTpomZzbDVHfxyOIyGzjZCaPaGdc0NC"
```

You can add a prefix to generated ids. The id generator will add an underscore between the specified prefix and the (default 25 character length) alphanumeric sequence.

```java
IdGenerator.generateId("txn") => "txn_Lh2LRvtK4euznW686ScmNNkw4"
```

You can specify a prefix and the length of the id to generate:

```java
IdGenerator.generateId("txn", 10) => "txn_qGoxTSqc30"
IdGenerator.generateId("txn", 30) => "txn_6CKbkcPQgk8OCONGu82Tv2rv664P2H"
```

# JSON Parsing

The JSON API exposes a JavaScript-inspired interface on top of the excellent [Jackson library](https://github.com/FasterXML/jackson) for JSON parsing.

#### **JSON.stringify() - serialize a Java value into a JSON string.**

```java
Person person = new Person("John", "Smith");
JSON.stringify(person) => {"firstName":"John","lastName":"Smith"}
```

This method will throw a *JsonParsingException* at runtime if the Java value cannot be serialized.

**Tip:** If you override the toString() implementation to generate JSON it becomes simple to produce well formatted JSON logs, and really useful when debugging and troubleshooting.

```java
public class Account {

    @Override
    public String toString() {
        return JSON.stringify(this);
    }

}

Account account = new Account();

// Logs a well formatted JSON string with the account's current state.
logger.info(account);

// Prints a well formatted JSON string with the account's current state instead of it's hash code.
System.out.pritnln(account);
```

#### **JSON.parse() - deserialize a JSON string into a Java value.**

```java
Person person = JSON.parse("{\"firstName\":\"John\",\"lastName\":\"Smith\"}", Person.class);

System.out.pritnln(person.getFirstName()); => "John"
System.out.pritnln(person.getLastName()); => "Smith"
```

This method will throw a *JsonParsingException* at runtime if the JSON string cannot be deserialized.

# Reflection
The Reflection API provides a map-like interface on top of the Java Reflection API for getting and setting property values on Java objects. We'll run through a number of examples  using the following object model:

```java
public class Entity {

    private String id;
    ...

}

public class Address extends Entity {

    private String suburb;
    ...

}

public class Person extends Entity {

    private String name;
    private Address address;
    ...

}


Address address = new Address();
address.setId("1234abcd");
address.setSuburb("Bondi");

Person person = new Person();
person.setId("abcd12345");
person.setName("Sally");
person.setAddress(address);
```

## Getting Values
The get API returns a property value for a given property name. If no such property value exists, a *null* value is returned. Property values returned from the get API will be of type Object by default, but you can specify the return type explicitly to override the default behaviour.

```java
// Retrieves the 'name' property from the Person object.
Reflection.get(person, "name"); => "Sally"

// Retrieves the 'id' property from the Person object superclass.
Reflection.get(person, "id"); => "abcd12345"

// Retrieves the 'suburb' property from the composed Address object.
Reflection.get(person, "address.suburb"); => "Bondi"

// Retrieves the 'id' property from the composed Address object superclass.
Reflection.get(person, "address.id"); => "1234abcd"

// Retrieves the 'address' property from the Person object, returned as an Address through an explicit cast.
Address address = (Address) Reflection.get(person, "address");

// Retrieves the 'address' property from the Person object, returned as an Address by specifying a return type.
Address address = Reflection.get(person, "address", Address.class);
```

## Setting Values
The set API sets a property value for a given property name. An updated object instance is returned when a property value is set successfully, and null otherwise.

```java
// Sets the 'name' property on the Person object.
Reflection.set(person, "name", "Suzan");

// Sets the 'id' property on the Person object superclass.
Reflection.set(person, "id", "11111");

// Sets the 'suburb' property on the composed Address object.
Reflection.set(person, "address.suburb", "St. Kilda");

// Sets the 'id' property on the composed Address object superclass.
Reflection.set(person, "address.id", "22222");

// Setting an invalid property value will return a null instance.
Person updated = Reflection.set(person, "invalid_property", "12345");
System.out.println(updated) => null

// Setting a valid property value will return the updated instance.
Person updated = Reflection.set(person, "name", "Katherine");
System.out.println(updated) => {"id":"11111", "name":"Katherine", ...}
```
