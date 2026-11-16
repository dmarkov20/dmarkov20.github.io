---
title: "JSON Parsing in Java: From Nashorn to JEP 540"
description: "Exploring JSON parsing in Java, from Nashorn to JEP 540."
categories: [Java, JSON]
tags: [json, nashorn, jackson, gson, json-p, jep540]
toc: true
---

*Looking back at how JSON has been handled in Java, comparing a few existing options, and trying the new API proposed for JDK 28.*

## How it started

Recently, I was thinking about JSON parsing in Java and remembered that there was once a way to do it using Nashorn, the JavaScript engine bundled with the JDK.

That got me wondering how this approach compares with the libraries we use today, and whether it would still work with modern JDKs.

The question became more interesting when I came across **JEP 540 — Simple JSON API**, targeted for JDK 28.

After all these years, Java is finally getting a JSON processing API directly in the JDK. Well, an incubating one for now.

I decided to put together a few examples and see how the different approaches behave. Not to find the fastest parser or declare one library better than another, but to look at their APIs and some details that are easy to overlook when working with ordinary JSON documents.

## JSON and the JDK

JSON has been around for a long time and is used practically everywhere: REST APIs, configuration files, messages exchanged between services, and so on.

Java has had built-in support for XML processing for many years. JSON is a different story.

There have been several mature JSON libraries in the Java ecosystem, but Java SE itself has not provided a dedicated general-purpose JSON processing API.

For most developers, this hasn't been a serious problem. Adding Jackson or Gson to a project is straightforward, and both offer far more than basic JSON parsing.

Still, sometimes all you need is to read a small JSON document, extract a couple of values, or produce a simple response. Having to add a dependency for that can feel unnecessary.

Interestingly, this isn't the first attempt to address the issue.

Back in 2014, **JEP 198 — Light-Weight JSON API** proposed adding JSON support to the JDK. That proposal was never delivered and was eventually withdrawn.

Now, more than a decade later, the idea has returned with JEP 540.

But before getting to that, let's look at an older and slightly unusual approach.

## Nashorn: parsing JSON using JavaScript

Nashorn was introduced in JDK 8 as a JavaScript engine, replacing the older Rhino-based implementation.

Since JavaScript already provides `JSON.parse()`, it was possible to use Nashorn for parsing JSON from Java.

For example:

```java
ScriptEngine engine = new ScriptEngineManager()
        .getEngineByName("nashorn");

Bindings bindings = engine.createBindings();
bindings.put("jsonText", json);

Object result = engine.eval(
    "Java.asJSONCompatible(JSON.parse(jsonText))",
    bindings
);
```

The interesting part is `Java.asJSONCompatible()`.

It's a Nashorn-specific extension that exposes JavaScript objects and arrays through Java's familiar `Map` and `List` interfaces.

So you could parse JSON without adding a dedicated JSON library to your project.

Was this a good idea for production code? Probably not if JSON parsing was the only reason for introducing a JavaScript engine. But it worked, and it's an interesting example of functionality that was available indirectly through the JDK.

Nashorn was deprecated in JDK 11 and removed from the JDK in version 15.

However, Nashorn itself hasn't disappeared. It continues as a standalone OpenJDK project and can still be used with modern Java versions.

I included both versions in my examples: the original JDK-bundled Nashorn and the standalone engine.

The APIs are very similar, and so is their JSON parsing behaviour.

There is one important detail, though, which I'll come back to shortly: **JavaScript's numeric representation.**

## What we normally use today

For the comparison, I chose three established alternatives.

### Jackson

Probably the most familiar option for Java backend development.

Jackson supports tree-based parsing through `JsonNode`, mapping JSON to Java objects, and streaming large documents.

In my example, I used the tree model:

```java
JsonNode root = mapper.readTree(json);

String name = root.required("name").textValue();
```

Jackson offers considerable flexibility, particularly around object mapping and parser configuration.

### Gson

Google's Gson provides a relatively simple API for JSON serialization and deserialization.

For tree-based parsing:

```java
JsonElement root = gson.fromJson(
    json, JsonElement.class
);

String name = root.getAsJsonObject()
                  .get("name")
                  .getAsString();
```

For many smaller applications, this is perfectly sufficient.

### Jakarta JSON Processing (JSON-P)

JSON-P is worth including because, unlike Jackson and Gson, it is a standardized API defined by Jakarta EE.

It is not part of Java SE, and a standalone application needs an implementation. I used Eclipse Parsson.

```java
try (JsonReader reader = Json.createReader(
        new StringReader(json))) {

    JsonObject root = reader.readObject();
    String name = root.getString("name");
}
```

JSON-P provides both an object model and a streaming API. Object binding is covered separately by Jakarta JSON-B.

All three approaches are mature and well-established. For basic JSON input, there isn't much to distinguish them apart from API style and configuration.

The differences become more interesting when the input is less straightforward.

## A few things I found while testing

I wanted the examples to do more than parse a simple JSON object.

All six implementations use the same input files and cover the same test cases. Where necessary, I configured the parsers explicitly, for example to reject trailing JSON values.

I also added tests for malformed JSON, duplicate keys, missing properties, explicit nulls, Unicode, and numeric precision.

A few results were worth looking at more closely.

### 1. Duplicate keys

Consider this JSON:

```json
{
  "value": 1,
  "value": 2
}
```

RFC 8259 says that object member names **SHOULD** be unique. It doesn't make uniqueness an absolute grammar requirement, which leaves room for different parser behaviour.

In my tests:

- Both Nashorn implementations kept the last value, `2`.
- Jackson did the same with the configuration used in the example.
- Gson also kept `2`, even in strict parsing mode.
- JSON-P with Parsson 1.1.9 kept `2`.
- JEP 540 rejected the document with `JsonParseException`.

This is a useful distinction.

A parser accepting valid JSON syntax doesn't necessarily mean it handles duplicate member names the way another parser does.

If different components process the same input differently, the result can be more than just an inconvenience.

### 2. What happens after the first JSON value?

Another test used two consecutive JSON objects:

```json
{"value": 1} {"value": 2}
```

This isn't a valid single JSON document.

Both Nashorn implementations rejected it. So did Gson in strict mode, Jackson with `FAIL_ON_TRAILING_TOKENS` enabled, and JEP 540.

But JSON-P, using Parsson 1.1.9 and `JsonReader.readValue()`, returned the first object without reporting the second one.

That was one of the more interesting findings.

It doesn't mean that JSON-P in general accepts invalid documents. The observed behaviour is specific to the implementation and method I tested.

But it illustrates an important point: **successfully reading a JSON value doesn't necessarily mean the entire input has been validated.**

Depending on the API, checking that there is nothing left after the first value may be the caller's responsibility.

### 3. Missing properties versus explicit null

Consider:

```json
{
  "name": null
}
```

There are two different situations here:

- The property `name` exists and its value is JSON `null`.
- The property doesn't exist at all.

These cases aren't equivalent, even though some APIs return Java `null` for both when using a simple lookup.

Nashorn, for example, exposes objects as map-like structures. Calling `get()` returns Java `null` in both cases, but `containsKey()` allows us to distinguish them.

Other APIs expose this distinction differently.

Jackson has null and missing nodes. Gson uses `JsonNull` for an explicit null. JSON-P has `JsonValue.NULL`.

JEP 540 provides `tryGet()`, which returns an empty `Optional` for a missing member and an `Optional` containing `JsonNull` for an explicitly null member.

I like this distinction being explicit in the API.

### 4. Numeric precision

This is probably the most interesting difference between Nashorn and dedicated Java JSON parsers.

Take the integer:

```text
9007199254740993
```

JavaScript numbers use IEEE 754 binary64, which cannot represent every integer beyond 2⁵³ − 1 exactly. In this case, the original value is rounded during parsing.

When parsed through Nashorn's `JSON.parse()`, the result becomes:

```text
9007199254740992
```

The value has already been rounded before Java receives it.

Converting it to `long` afterwards cannot recover the original number.

Jackson, Gson, and JSON-P can preserve the original integer, provided the appropriate numeric representations or accessors are used.

JEP 540 also preserves it:

```java
JsonValue root = Json.parse(json);

long value = root.get("largeInteger").asLong();
```

The conversion returns the exact integer.

However, calling `asDouble()` on the same value produces the rounded binary64 representation.

So there is a difference between losing precision **while parsing** and losing it **while converting to a particular Java type**.

I also tested an integer just outside the range of `long`:

```text
9223372036854775808
```

With the JEP 540 implementation I tested, calling `asLong()` throws `JsonValueException`, as expected.

But the original decimal representation remains available, so it can be converted to `BigInteger` without losing precision.

This behaviour is useful when working with JSON containing large identifiers or high-precision values.

And it's a good reminder that numeric convenience methods deserve some attention. Not every conversion is exact, and not every API reports information loss in the same way.

## JEP 540: finally a JSON API in the JDK

This brings us to JEP 540.

The proposal introduces a small JSON API in the incubating module:

```text
jdk.incubator.json
```

It's targeted for JDK 28, with a focus on parsing, navigating, and generating JSON documents.

The basic usage is straightforward:

```java
import jdk.incubator.json.Json;
import jdk.incubator.json.JsonValue;

JsonValue root = Json.parse(json);

String name = root.get("name").asString();

String language = root.get("metadata")
                      .get("language")
                      .asString();
```

The API uses an immutable JSON value model, with types representing objects, arrays, strings, numbers, booleans, and null.

One thing I appreciate is that navigation doesn't require repeated explicit casts between JSON object and array types.

The API also takes a fairly strict approach to parsing. It rejects duplicate object member names and malformed documents rather than offering a collection of lenient parsing options.

It's important to understand what JEP 540 is **not** trying to do.

It doesn't attempt to replace Jackson or Gson.

There is no automatic mapping between arbitrary Java objects and JSON documents, and the API doesn't provide streaming processing.

It's intended for relatively simple JSON operations, where bringing in a larger library may not be justified.

That seems like a reasonable scope to me.

If I'm writing a small utility, working on JDK tooling, or reading a configuration document, I can see the benefit of having this functionality available without another dependency.

For a backend application already using Jackson for object mapping, I don't see much reason to replace it.

Since the API is still incubating, the `jdk.incubator.json` module must be explicitly enabled using `--add-modules jdk.incubator.json` when compiling and running class-path applications.

The API may still change. My examples were compiled and tested against OpenJDK `28-ea+23-1886`, and the repository records this build explicitly.

## The examples

I put the implementations together in a small GitHub repository:

[engineering-examples — Java JSON comparison](https://github.com/dmarkov20/engineering-examples/tree/main/java/json)

It contains six examples:

1. Nashorn bundled with JDK 8
2. Standalone Nashorn
3. Jackson
4. Gson
5. Jakarta JSON-P with Parsson
6. JEP 540 Simple JSON API

Each example uses the same fixtures, with tests covering the cases discussed above.

The examples are intentionally small and independent. The idea is to make it easy to inspect the code, try a particular API, and reproduce the results.

With the required JDKs and Maven configured, the entire comparison can be run from the repository root:

```bash
java/json/examples.sh run all
java/json/examples.sh test all
```

The README includes the toolchain requirements and instructions for running individual examples.

I haven't tried to benchmark these libraries. That would be a separate exercise, and I don't think performance numbers would add much to this particular comparison without a properly designed benchmark.

## Final thoughts

I started with a simple question about Nashorn and ended up looking at several approaches to JSON parsing in Java.

The comparison was interesting, particularly when it came to numeric precision and how different parsers handle less straightforward input.

As for JEP 540, I don't expect it to replace established libraries such as Jackson. But I can certainly see the benefit of having a lightweight JSON API available directly in the JDK, especially for smaller tools and utilities.

I'm curious to see how the API develops beyond the incubator stage, and whether it will eventually become a standard part of Java SE.

**I'd be interested to hear what others think about JEP 540. Is this something you've been missing in Java, or are existing libraries already enough?**

---

### References

- [JEP 540 — Simple JSON API (Incubator)](https://openjdk.org/jeps/540)
- [JEP 198 — Light-Weight JSON API](https://openjdk.org/jeps/198)
- [JEP 335 — Deprecate the Nashorn JavaScript Engine](https://openjdk.org/jeps/335)
- [JEP 372 — Remove the Nashorn JavaScript Engine](https://openjdk.org/jeps/372)
- [OpenJDK Nashorn project](https://github.com/openjdk/nashorn)
- [RFC 8259 — The JavaScript Object Notation Data Interchange Format](https://www.rfc-editor.org/rfc/rfc8259.html)
- [JEP 540 early-access API documentation](https://download.java.net/java/early_access/jdk28/docs/api/jdk.incubator.json/jdk/incubator/json/package-summary.html)
