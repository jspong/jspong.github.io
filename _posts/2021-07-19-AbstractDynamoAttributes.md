---
layout: post
title: "Storing abstract and generic attributes in DynamoDB"
---

# Intro

[Amazon DynamoDB](https://aws.amazon.com/dynamodb) is a great, simple [document-oriented database](https://en.wikipedia.org/wiki/Document-oriented_database).
For those coming from traditional [Relational Databases](https://en.wikipedia.org/wiki/Relational_database), this switch can be jarring. 
Storing [normalized data](https://en.wikipedia.org/wiki/Database_normalization) has become second-nature, and the idea of dumping a whole object into a field can feel very uncomfortable.

Thankfully, this unstructured nature - when used properly - can have its benefits. If you have predictable query patterns, the ability to store whole objects in your database can
simplify development greatly, and the distributed web-scale means your application can easily scale from a dozen users to millions without having to worry about sharding.

Unfortunately, because DynamoDB ultimately has to serialize its data into strings, maps, lists, or numeric primitives, Dynamo can easily be confused by object-oriented programming models. In particular, lists of custom types (particular inheritance hierarchies) introduce a number of roadblocks to DynamoDB's naive JSON-processing which can be easily overcome with the right patterns.  

# A Nonsense Use case: the `Foo` Manager.

To keep things general, coming up with a better alternative to "a Dog is an Aminal, but so is a Cat!" OOP example, I will not be giving a "real-world" use case.
We will deal with [Metasyntactic variables](https://en.wikipedia.org/wiki/Metasyntactic_variable) like `Foo`, `Bar`, and `Qux`. 

## Phase 1: An Object with Primitive Attributes.

It's extremely simple to store an object with primitive attributes in DynamoDB: annotate your class with [`@DynamoDBTable`](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/dynamodbv2/datamodeling/DynamoDBTable.html), your fields with [`@DynamoDBHashKey`](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/dynamodbv2/datamodeling/DynamoDBHashKey.html)/[`@DynamoDBAttribute`](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/dynamodbv2/datamodeling/DynamoDBAttribute.html), and let [`DynamoDBMapper`](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/DynamoDBMapper.html) handle the rest:

```java

@DynamoDBTable(tableName="Foos")
public class Foo {

	@DynamoDBHashKey(attributeName="id")
	public String id;

	@DynamoDBAttribute(attributeName="name")
	public String name;

}

```

## Phase 2: Every Foo Has a Bar

Sometimes it's not that simple and you want to have structured attributes that are modeled with custom object types. DynamoDB can store maps, but we don't want to have to write our own code converting back-and-forth between Json maps and our Java objects. Thankfully, the [Jackson](https://github.com/FasterXML/jackson) library handles serializing and deserializing objects to/from Json strings and Dynamo has the [`@DynamoDBTypeConvertedJson`](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/dynamodbv2/datamodeling/DynamoDBTypeConvertedJson.html) attribute to use Jackson to serialize your objects to JSON and then deserialize them on read:

```java

@DynamoDBTable(tableName="Foos")
public class Foo {

	/* ... other attributes */

	@DynamoDBAttribute(attribute="barFriend")
	@DynamoDBTypeConvertedJson
	public Bar barFriend;
}

```

## Phase 3: What if every Foo has lots of Bars?

```java

@DynamoDBTable(tableName="Foos")
public class Foo {

	/* ... other attributes */

	@DynamoDBAttribute(attribute="barFriends")
	@DynamoDBTypeConvertedJson
	public List<Bar> barFriends;
}

```

This seems to work OK when you write your document, but when you try to read it back, you see the following error: `java.lang.ClassCastException: java.util.LinkedHashMap cannot be cast to Bar`.

So what happened?

### An Overview of Type Erasure

Java Generics were introduced in J2SE 5.0, well after the initial `Object` collections were introduced.
To avoid making breaking changes the powers that be decided that the new generic collections should be compatible with the originals.
To reach that end the compiler "erases" the type so that the JVM sees them as `Object`s until they are cast to a variable.
The benefit is that compile-time enforcement is still possible, but the drawback is that there is no way to use reflection to determine what type of `Object`s should be stored in the collection.
When serializing Jackson has the run-time information of the object, but when deserializing it only knows the structure, and defaults to `Map<String, Object>` unless explicitly told otherwise.

When you inherit a generic class or implement a generic interface, providing a type so that your new class is no longer generic, the JVM *does* need this information so that it can provide type-safe attributes and polymorphic compatibility with the generic:

```java

public class StringGetter implements Provider<String> {

    @Override
    public String get() {
		return "some string";
	}

}
```

`StringChecker` classes need to behave in two conflicting contexts:

1. They need to be able to be used anywhere `Provider<String>` can be used.
2. They need to be able to be called explicitly like `String string = myStringProvider.get()`.

To handle this the JVM stores some metadata that Jackson exploits in its `TypeReference<T>` class. The details of how this class is implemented are interesting but not necessary for this document.

## Phase 3 revisited: how to deserialize a list of custom objects?

The `@DynamoDBTypeConvertedJson` annotation has a `targetType` argument where you can pass a `Class` that should be used instead of the default `Void`. Jackson uses this to know what type to deserialize to.
Unfortunately, because of Type Erasure, `List<Bar>` does not have a class of its own. Instead it's class is `List`. Barring the other documentation Jackson will fill this list with `Map<String, Object>` instances when it parses the string.

Thankfully, DynamoDB also allows you to provide a specific class that implements [`DynamoDBTypeConverter<S, T>`](https://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/dynamodbv2/datamodeling/DynamoDBTypeConverter.html) to the annotation. What we need is a converter that knows how to parse a `String` into a `List<Bar>`.

Also, thankfully, this class is very simple to implement. We need to have a Jackson `ObjectMapper` and pass it `TypeReference<List<Bar>>` when it deserializes the string:

```java

public class BarListConverter implements DynamoDBTypeConverter<String, List<Bar>> {

	private static final ObjectMapper objectMapper = new ObjectMapper();

	@Override
	public String convert(List<Bar> barList) {
		objectMapper.writeValueAsString(barList);
	}

	@Override
	public List<Bar> unconvert(String s) {
		return objectMapper.readValue(s, new TypeReference<List<Bar>>() {});
	}

}
	
```

Note that we instantiate a private `TypeReference<List<Bar>>` class here because the JVM needs the metadata from the *inheritance* not from the generic itself.

Now we can simply annotate `Foo.barList`:

```java

@DynamoDBTable(tableName="Foos")
public class Foo { 

	/* ... other attributes */

	@DynamoDBAttribute(attributeName="barList")
	@DynamoDBTypeConverted(converter=BarListConverter.class)
	public List<Bar> barList;
}

```

# Phase 4: Generalize the conversion process

It stands to reason that if you're doing this for one type then you'll be doing it for many. While this is a small piece of code, it can be very helpful for many reasons to [not repeat yourself](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself). If we look at another implementation, we can see some immediate generalizations:

```java

public class QuxMapConverter implements DynamoDBTypeConverter<String, Map<String, Qux>> {

	private static final ObjectMapper objectMapper = new ObjectMapper();

	@Override
	public String convert(Map<String, Qux> quxMap) {
		objectMapper.writeValueAsString(quxMap);
	}

	@Override
	public Map<String, Qux> unconvert(String s) {
		return objectMapper.readValue(s, new TypeReference<Map<String, Qux>>() {});
	}
}

```

There are three differences in these classes:

1. The name of the class.
2. The second type variable (which is the same as the argument type in `convert` and the return type for `unconvert`).
3. The `TypeReference` object passed to `objectMapper.readValue`.

Number 1 is unavoidable: you gotta name your classes.

Number 2 means we need to keep one of the generic types unfilled. This means we need an abstract class to handle the rest.

Number 3 is unavoidable (well, probably doable with some *really* gross metaprogramming, but don't do that to yourself or your readers). This means we need an abstract method to get the `TypeReference`

```java

public abstract class DynamoDBJacksonTypeConverter<T> implements DynamoDBTypeConverter<String, T> {

	private static final ObjectMapper objectMapper = new ObjectMapper();

	protected abstract TypeReference<T> getTypeReference();

	@Override
	public String convert(T t) {
		objectMapper.writeValueAsString(t);
	}

	@Override
	public T unconvert(String s) {
		return objectMapper.readValue(s, getTypeReference());
	}

}

```

Then your implementations are simple subclasses that just need to provide the `TypeReference`. I like to store them as static child classes of the types themselves and call them `ListConverter`, `MapConverter`, or whatever:

```java

public class Bar {
	/* implementation */

	public static class ListConverter extends DynamoDBJacksonTypeConverter<List<Bar>> {
		@Override
		protected TypeReference<List<Bar>> getTypeReference() {
			return new TypeReference<List<Bar>>() {};
		}
	}
}

```

So now back to `Foo`:

```java

@DynamoDBTable(tableName="Foos")
public class Foo {
	/* ... other attributes */

	@DynamoDBAttribute(attributeName="barList")
	@DynamoDBTypeConverted(converter=Bar.ListConverter.class)
	public List<Bar> barList;
}

```

Now the `DynamoDBMapper` class will know how to properly serialize and deserialize your `barList`!

# Phase 5: Every Foo has a subclass of Qux

Jackson also needs help deserializing class hierarchies because it needs information about which subtype was serialized. To do this, you can tell it to store a `@Class` attribute in the object, and annotate the base class with all the subclass information. This inversion of the hierarchy is annoying, but necessary, and also has the side-effect that you cannot subclass the type outside its defining package.

```java

@JsonIgnoreProperties(ignoreUnknown=true)
@JsonTypeInfo(use=JsonTypeInfo.Id.CLASS)
@JsonSubTypes(
	@JsonSubTypes.Type(value=Quy.class, name="Quy"),
	@JsonSubTypes.Type(value=Quz.class, name="Quz"))
public class Qux {
}
```

Now you can annotate your `Qux` member with `@DynamoDbTypeConvertedJson` and Jackson will be able to store and infer the subtype information.
