---
layout: post
title: "Mocking static methods and constructors with Mockito"
---

# Introduction

From time to time, we find ourselves needing a way to mock out static methods or constructors when writing tests. This is usually accomplished by introducing a layer of indirection like strategy or factory pattern. Even if it may not be the right way for the purists, mockito's static/constructor mocking feature is a compelling alternative and a handy tool to keep in your toolbox.

In this post, we will show you how to leverage the feature.

Note that a similar feature has long been present in [PowerMock](https://github.com/powermock/powermock/wiki/Mockito#mocking-static-method), but mockito's design is arguably more convenient and easier-to-use.

# Setup

Add [mockito-inline](https://search.maven.org/search?q=g:org.mockito%20AND%20a:mockito-inline) dependency to your maven pom.xml (or the equivalent in your build system):

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-inline</artifactId>
    <version>3.8.0</version>
    <scope>test</scope>
</dependency>
```

We will also add assertj and junit dependencies. See [pom.xml](https://github.com/maxxedev/mockito-example/blob/main/pom.xml) for complete example.

# Mocking Static Methods
To mock static methods on Foo class, simply call mockito's `mockStatic()` method:
```java
assertEquals("originalRealValue", Foo.staticMethod());
try (MockedStatic mocked = mockStatic(Foo.class)) {
    mocked.when(Foo::staticMethod).thenReturn("fakeMockValue");

    // Foo.staticMethod() behavior is changed only within this try scope
    assertEquals("fakeMockValue", Foo.staticMethod());

    // (optional) verify that Foo.staticMethod() was called
    mocked.verify(Foo::staticMethod);
}

// staticMethod() behavior reverts to normal after exiting try scope
assertEquals("originalRealValue", Foo.staticMethod());
```

Let's look at a couple of examples for clarity.


### Example Problem: java.nio.Path

Suppose you have code with hardwired file paths, perhaps to /etc/hosts or /proc/self/xyz:

```java
public class EtcHostsUtils {

    /**
     * returns /etc/hosts mappings for the given ipAddress
     */
    public static List<String> findEtcHostsMappings(String ipAddress) throws IOException {
        Path path = Paths.get("/etc/hosts");
        ...
    }
}
```

How would you test that with a temporary /etc/hosts file? 

#### Solutions? 
Perhaps you could introduce an indirection with `System.getProperty()`:
```java
Path path = Paths.get(System.getProperty("hosts.file", "/etc/hosts"));
```
<br>

Or convert the static method to an instance method where /etc/hosts would be an instance variable:
```java
public class EtcHostsLoader {
    private Path etcHosts;

    public EtcHostsLoader() { 
        this(Paths.get("/etc/hosts"));
    }

    EtcHostsLoader(Path etcHosts) {
        this.etcHosts = etcHosts;
    }

    public List<String> findEtcHostsMappings(String ipAddress) throws IOException {
        ...
    }
}
```
<br>

Or perhaps inject a `FileSystem` from which a hardwired path could be obtained:
```
public class EtcHostsLoader {}
    private FileSystem fileSystem;

    public EtcHostsLoader(FileSystem fileSystem) {
        this.fileSystem = fileSystem;
    }

    public List<String> findEtcHostsMappings(String ipAddress) throws IOException {
        Path path = fileSystem.get("/etc/hosts");
        ...
    }
}
```
In your tests, you could then inject a fake or [in-memory FileSystem](https://github.com/google/jimfs).

#### Solution
More easily though, we could mock `Paths.get()` method:
```java
public class EtcHostsUtilsTest {

    @TempDir
    Path tempDir;

    @Test
    public void testEtcHostsUtils() throws IOException {
        Path tempFile = tempDir.resolve("test-etc-hosts.txt");
        Files.write(tempFile, asList("192.168.10.10        database-server"));

        // mock (all) static methods on Paths.class
        try (MockedStatic<Paths> mocked = mockStatic(Paths.class)) {

            // when Paths.get() is called with hardwired /etc/hosts argument,
            // instead return a tempFile we created above.
            mocked.when(() -> Paths.get("/etc/hosts"))
                    .thenReturn(tempFile);

            assertThat(EtcHostsUtils.findEtcHostsMappings("192.168.10.10"))
                    .containsExactly("database-server");
        }
    }
}
```

### Example Problem: java.time

As another example, suppose we need to perform some calculations based on some java.time values like `Instant.now()` or `ZoneDateTime.now()`:
```java
public class DateTimeUtils {

    public static long millisTillTomorrow() {
        Instant now = Instant.now();
        Instant tomorrow = now.plus(1, DAYS).truncatedTo(DAYS);
        return Duration.between(now, tomorrow).toMillis();
    }
}
```

Tests would be much easier to write if `Instant.now()` could be a fixed constant value. We could employ the usual tricks like passing in a `Clock`, storing it as a static variable with package-private visibility, or converting to an instance method, and so on.

With static mocking, it's more straightforward:
```java
public class DateTimeUtilsTest {

    @Test
    public void testMillisTillTomorrow() {
        // shortly before Dec 25, 2021
        Instant now = LocalDateTime.of(2021, 12, 24, 23, 29, 40).toInstant(UTC);

        try (MockedStatic<Instant> mocked = mockStatic(Instant.class, CALLS_REAL_METHODS)) {
            mocked.when(() -> Instant.now())
                    .thenReturn(now);

            long expectedMillis = SECONDS.toMillis(20) + MINUTES.toMillis(30);
            assertThat(DateTimeUtils.millisTillTomorrow()).isEqualTo(expectedMillis);
        }
    }
    
}
```

Note that we passed in `CALLS_REAL_METHODS` to `mockStatic()` so that all Instant static methods are mocked to call the real methods. That way, aside from the `now()` method we specifically mocked differently, all other Instant static methods are mocked so that they return real values instead of nulls. This is needed because, as it turns out, Instant _instance_ methods like `plus()` or `truncatedTo()` call Instant _static_ methods.

Or, put simply: call `mockStatic(Foo.class)`. If you see strange exceptions, try `mockStatic(Foo.class, Answers.CALLS_REAL_METHODS)`

### Example Problem: java ManagementFactory

As one last example on mocking static methods, suppose you want to calculate the idle cpu count, perhaps for sizing a ThreadPoolExecutor:
```java
public class CpuUtils {

    public static int getIdleCpuCount() {
        OperatingSystemMXBean operatingSystemMXBean = ManagementFactory.getOperatingSystemMXBean();
        double availableProcessors = operatingSystemMXBean.getAvailableProcessors();
        double averageLoad = operatingSystemMXBean.getSystemLoadAverage();
        int idleProcessors = (int) (availableProcessors - averageLoad);
        if (idleProcessors <= 0) {
            return 1;
        }
        return idleProcessors <= 0 ? 1 : idleProcessors;
    }
}
```

Without a whole lot of indirection, this would be difficult to test. Static mocking to the rescue:
```java
public class CpuUtilsTest {

    @Test
    public void testGetIdleCpuCount() {
        OperatingSystemMXBean mockOsMxBean = mock(OperatingSystemMXBean.class);
        doReturn(10.0).when(mockOsMxBean).getSystemLoadAverage();
        doReturn(16).when(mockOsMxBean).getAvailableProcessors();

        try (MockedStatic<ManagementFactory> mocked = mockStatic(ManagementFactory.class)) {
            mocked.when(() -> ManagementFactory.getOperatingSystemMXBean())
                    .thenReturn(mockOsMxBean);

            assertThat(CpuUtils.getIdleCpuCount()).isEqualTo(6);
        }
    }

}
```


# Mocking Constructors
Mocking constructors is slightly trickier but still follows the same pattern as mocking static methods:
```java
assertEquals("originalRealValue", new Foo().method());
try (MockedConstruction mocked = mockConstruction(Foo.class, this::stubFooWithBar)) {
    String result = mySystemUnderTest.callNewFooMethod();
    assertEquals(result, "bar");

    // (optional) verify mock Foo was called by MySystemUnderTest
    Foo mockFoo = mocked.constructed().get(0);
    verify(mockFoo).method();
}
assertEquals("originalRealValue", new Foo().method());
```

### Example Problem: java.util.Random

Suppose we have a `DiceRoller` class and we want to mock out `new Random().nextInt()`:

```java
public class DiceRoller {

    private final Random random = new Random();
    private int numSides;

    public DiceRoller() {
        this(6);
    }

    public DiceRoller(int numSides) {
        this.numSides = numSides;
    }

    public int roll() {
        return random.nextInt(numSides) + 1;
    }
}
```

Here is how we can mock the constructor:
```java
public class DiceRollerTest {

    @Test
    public void testDiceRoll() {
        try (MockedConstruction<Random> mocked = mockConstruction(Random.class, this::stubRandom)) {
            DiceRoller dicerRoller = new DiceRoller();
            assertThat(dicerRoller.roll()).isEqualTo(6);

            Random mockRandom = mocked.constructed().get(0);
            verify(mockRandom).nextInt(6);
        }
    }

    private void stubRandom(Random mockRandom, Context context) {
        doReturn(5).when(mockRandom).nextInt(6);
    }
}
```

Immediately after `DiceRoller` calls `new Random()`, mockito calls our `stubRandom()` method, giving us a chance to apply custom behavior on the mocked Random instance just created. 


# Summary

You should now be able to see that how powerful static and constructor mocking can be. With great power, of course, comes great responsibility. Use it judiciously!

The examples in this post are available on GitHub:

- [https://github.com/maxxedev/mockito-example](https://github.com/maxxedev/mockito-example)
