---
layout: post
title: "Hacking the Method Name"
---

# Problem

Inspired by a [recent post](https://committing-crimes.com/articles/2026-08-04-java-nameof/) on easily obtaining method parameter name by identifier, here is a technique to obtain a method's name by method reference.


Imagine you have a class with getters that follow typical Java Bean naming convention or Java Record naming convention:

```java
class Book {
   ...
   public String getAuthor() { ... }
   public String title() { ... }
}
```

How can we easily obtain `"title"` or `"author"` strings? Perhaps you want to directly parse CSV content, database results, or call a REST API:

```java
String title = csvContents.get(nameOf(Book::title));
String author = dbQueryResults.get(beanNameOf(Book::getAuthor));
List<Book> books = bookClient.findByAttr(singletonMap(nameOf(Book::getAuthor), "Shakespeare"));
```


# Solution

We can implement such a `nameOf(...)` functionality in Java 8+ by serializing a method reference to [SerializedLambda](https://docs.oracle.com/javase/8/docs/api/java/lang/invoke/SerializedLambda.html) which lets us easily read the method's name:

   * Declare a serializable getter functional interface: `interface Getter<T, R> extends Function<T, R>, Serializable {}`
   * When serializing it, Java will create a SerializedLambda as a replacement object.
   * Intercept that SerializedLambda object and read the method's name

Here is the code:

```java
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.io.OutputStream;
import java.io.Serializable;
import java.lang.invoke.SerializedLambda;
import java.util.function.Function;

public class GetterTool {

    @FunctionalInterface
    public interface Getter<T, R> extends Function<T, R>, Serializable {}

    public static <T, R> String nameOf(final Getter<T, R> methodRef) {
        return toSerializedLambda(methodRef).getImplMethodName();
    }

    private static <T, R> SerializedLambda toSerializedLambda(final Object methodRef) {
        try {
            try (final CustomObjectOutputStream stream = new CustomObjectOutputStream()) {
                stream.writeObject(methodRef);
                return (SerializedLambda) stream.interceptedObject;
            }
        } catch (final IOException e) {
            throw new RuntimeException(e);
        }
    }

    private static class CustomObjectOutputStream extends ObjectOutputStream {
        private Object interceptedObject = null;

        public CustomObjectOutputStream() throws IOException {
            super(nullOutputStream());
            enableReplaceObject(true);
        }

        @Override
        protected Object replaceObject(final Object obj) {
            if (this.interceptedObject == null) {
                this.interceptedObject = obj;
            }
            return obj;
        }

        private static OutputStream nullOutputStream() {
            return new OutputStream() {
                @Override public void write(final int b) { }
                @Override public void write(final byte[] b, final int off, final int len) { }
                @Override public void close() { }
            };
        }
    }
}
```

And we can use it like this:

```java
@Test
void test() {
    assertEquals("title", GetterTool.nameOf(Book::title));
}
```

After implementing `nameOf()`, it's straightforward to implement further transformations like `beanNameOf()` to remove the `get` prefix.


# Other methods


So this technique works for zero-argument getter methods but what if we want to obtain the name of other methods like:

```
List<Book> findByAttr(Map<String, Object> attr);
Book save(String title, String author);
```

Method references need to target a functional interface, so we would need to define corresponding serializable functional interfaces. Only then overloaded `nameOf()` methods can be defined and called:

```java
// for methods like findByAttr(...)
public interface Function2<T1, T2, R> extends Serializable {
    R apply(T1 t1, T2 t2);
}

// for methods like save(...)
public interface Function3<T1, T2, T3, R> extends Serializable {
    R apply(T1 t1, T2 t2, T3 t3);
}

<T1, T2, R> String nameOf(final Function2<T1, T2, R> methodRef);
<T1, T2, T3, R> String nameOf(final Function3<T1, T2, T3, R> methodRef);
```

Very tedious! Fortunately, someone has done the tedious work and in fact made it better. We can easily obtain, not only the method name, but also the actual `java.lang.reflect.Method`:

```java
Method m1 = toMethod(BookClient::save);
Method m2 = toMethod(BookClient::findByAttr);
```

Take a look at this library: [https://github.com/Hervian/safety-mirror/tree/master#type-safe-method-creation](https://github.com/Hervian/safety-mirror/tree/master#type-safe-method-creation)
