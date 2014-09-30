Java is one of the most popular programming languages around, but no one seems
to enjoy using it. Well, Java is actually an alright programming language, and
since Java 8 came out recently, I decided to compile a list of libraries, 
practices, and tools to make using Java better. 

This article was originally posted on 
[my blog](http://blog.seancassidy.me/better-java.html).

# Style

Traditionally, Java was programmed in a very verbose enterprise JavaBean style.
The new style is much cleaner, more correct, and easier on the eyes.

## Structs

One of the simplest things we as programmers do is pass around data. The
traditional way to do this is to define a JavaBean:

```java
public class DataHolder {
    private String data;

    public DataHolder() {
    }

    public void setData(String data) {
        this.data = data;
    }

    public String getData() {
        return this.data;
    }
}
```

This is verbose and wasteful. Even if your IDE automatically generated this
code, it's a waste. So, [don't do this][dontbean].

Instead, I prefer the C struct style of writing classes that merely hold data:

```java
public class DataHolder {
    public final String data;

    public DataHolder(String data) {
        this.data = data;
    }
}
```

This is a reduction in number of lines of code by a half. Further, this class
is immutable unless you extend it, so we can reason about it easier as we know
that it can't be changed.

If you're storing objects like Map or List that can be modified easily, you
should instead use ImmutableMap or ImmutableList, which is discussed in the 
section about immutability.

### The Builder Pattern

If you have a rather complicated object that you want to build a struct for,
consider the Builder pattern.

You make a subclass in your object which will construct your object. It uses
mutable state, but as soon as you call build, it will emit an immutable
object.

Imagine we had a more complicated *DataHolder*. The builder for it might look
like:

```java
public class ComplicatedDataHolder {
    public final String data;
    public final int num;
    // lots more fields and a constructor

    public static class Builder {
        private String data;
        private int num;
        
        public Builder data(String data) {
            this.data = data;
            return this;
        }

        public Builder num(int num) {
            this.num = num;
            return this;
        }

        public ComplicatedDataHolder build() {
            return new ComplicatedDataHolder(data, num); // etc
        }  
    }
}
```

Then to use it:

```java
final ComplicatedDataHolder cdh = new ComplicatedDataHolder.Builder()
    .data("set this")
    .num(523)
    .build();
```

There are [better examples of Builders elsewhere][builderex] but this should
give you a taste for what it's like. This ends up with a lot of the boilerplate
we were trying to avoid, but it gets you immutable objects and a very fluent
interface.

## Dependency injection

This is more of a software engineering section than a Java section, but one of
the best ways to write testable software is to use [dependency injection][di]
(DI). Because Java strongly encourages OO design, to make testable software,
you need to use DI.

In Java, this is typically done with the [Spring Framework][spring]. It has a
either code-based wiring or XML configuration-based wiring. If you use the XML 
configuration, it's important that you [don't overuse Spring][springso] because
of its XML-based configuration format.  There should be absolutely no logic or
control structures in XML. It should only inject dependencies.

Good alternatives to using Spring is Google and Square's [Dagger][dagger]
library or Google's [Guice][guice]. They don't use Spring's XML 
configuration file format, and instead they put the injection logic in
annotations and in code.

## Avoid Nulls

Try to avoid using nulls when you can. Do not return null collections when you
should have instead returned an empty collection. If you're going to use null, 
consider the [@Nullable][nullable] annotation. [IntelliJ IDEA][intellij] has 
built-in support for the @Nullable annotation.

If you're using [Java 8][java8], you can use the excellent new 
[Optional][optional] type. If a value may or may not be present, wrap it in
an *Optional* class like this:

```java
public class FooWidget {
    private final String data;
    private final Optional<Bar> bar;

    public FooWidget(String data) {
        this(data, Optional.empty());
    }

    public FooWidget(String data, Optional<Bar> bar) {
        this.data = data;
        this.bar = bar;
    }

    public Optional<Bar> getBar() {
        return bar;
    }
}
```

So now it's clear that *data* will never be null, but *bar* may or may not be
present. *Optional* has methods like *isPresent*, which may make it feel like
not a lot is different from just checking *null*. But it allows you to write
statements like:

```java
final Optional<FooWidget> fooWidget = maybeGetFooWidget();
final Baz baz = fooWidget.flatMap(FooWidget::getBar)
                         .flatMap(BarWidget::getBaz)
                         .orElse(defaultBaz);
```

Which is much better than chained if null checks. The only downside of using
Optional is that the standard library doesn't have good Optional support, so
dealing with nulls is still required there.

## Immutable-by-default

Unless you have a good reason to make them otherwise, variables, classes, and
collections should be immutable.

Variable references can be made immutable with *final*:

```java
final FooWidget fooWidget;
if (condition()) {
    fooWidget = getWidget();
} else {
    try {
        fooWidget = cachedFooWidget.get();
    } catch (CachingException e) {
        log.error("Couldn't get cached value", e);
        throw e;
    }
}
// fooWidget is guaranteed to be set here
```

Now you can be sure that fooWidget won't be accidentally reassigned. The *final*
keyword works with if/else blocks and with try/catch blocks. Of course, if the
*fooWidget* itself isn't immutable you could easily mutate it.

Collections should, whenever possible, use the Guava [ImmutableMap][immutablemap],
[ImmutableList][immutablelist], or [ImmutableSet][immutableset] classes. These
have builders so that you can build them up dynamically and then mark them 
immutable by calling the build method.

Classes should be made immutable by declaring fields immutable (via *final*)
and by using immutable collections. Optionally, you can make the class itself 
*final* so that it can't be extended and made mutable.

## Avoid lots of Util classes

Be careful if you find yourself adding a lot of methods to a Util class.

```java
public class MiscUtil {
    public static String frobnicateString(String base, int times) {
        // ... etc
    }

    public static void throwIfCondition(boolean condition, String msg) {
        // ... etc
    }
}
```

These classes, at first, seem attractive because the methods that go in them
don't really belong in any one place. So you throw them all in here in the
name of code reuse.

The cure is worse than the disease. Put these classes where they belong, or 
if you must have common methods like this, consider [Java 8][java8]'s default
methods on interfaces. Then you could lump common actions into interfaces. 
And, since they're interfaces, you can implement multiple of them.

```java
public interface Thrower {
    default void throwIfCondition(boolean condition, String msg) {
        // ...
    }

    default void throwAorB(Throwable a, Throwable b, boolean throwA) {
        // ...
    }
}
```

Then every class which needs it can simply implement this interface.

## Formatting

Formatting is so much less important than most programmers make it out to be.
Does consistency show that you care about your craft and does it help others
read? Absolutely. But let's not waste a day adding spaces to if blocks so that
it "matches".

If you absolutely need a code formatting guide, I highly recommend
[Google's Java Style][googlestyle] guide. The best part of that guide is the
[Programming Practices][googlepractices] section. Definitely worth a read.

### Javadoc

Documenting your user facing code is important. And this means 
[using examples][javadocex] and using sensible descriptions of variables,
methods, and classes.

The corollary of this is to not document what doesn't need documenting. If you
don't have anything to say about what an argument is, or if it's obvious,
don't document it. Boilerplate documentation is worse than no documentation at
all, as it tricks your users into thinking that there is documentation.

## Streams

[Java 8][java8] has a nice [stream][javastream] and lambda syntax. You could
write code like this:

```java
final List<String> filtered = list.stream()
    .filter(s -> s.startsWith("s"))
    .map(s -> s.toUpperCase())
    .collect(Collectors.toList());
```

Instead of this:

```java
final List<String> filtered = new ArrayList<>();
for (String str : list) {
    if (str.startsWith("s") {
        filtered.add(str.toUpperCase());
    }
}
```

This allows you to write more fluent code, which is more readable.

# Deploying

Deploying Java properly can be a bit tricky. There are two main ways to deploy
Java nowadays: use a framework or use a home grown solution that is more
flexible.

## Frameworks

Because deploying Java isn't easy, frameworks have been made which can help.
Two of the best are [Dropwizard][dropwizard] and [Spring Boot][springboot].
The [Play framework][play] can also be considered one of these deployment 
frameworks as well.

All of them try to lower the barrier to getting your code out the door. 
They're especially helpful if you're new to Java or if you need to get things
done fast. Single JAR deployments are just easier than complicated WAR or EAR
deployments.

However, they can be somewhat inflexible and are rather opinionated, so if
your project doesn't fit with the choices the developers of your framework
made, you'll have to migrate to a more hand-rolled configuration.

## Maven

**Good alternative**: [Gradle][gradle].

Maven is still the standard tool to build, package, and run your tests. There
are alternatives, like Gradle, but they don't have the same adoption that Maven 
has. If you're new to Maven, you should start with
[Maven by Example][mavenexample].

I like to have a root POM with all of the external dependencies you want to
use. It will look something [like this][rootpom]. This root POM has only one
external dependency, but if your product is big enough, you'll have dozens.
Your root POM should be a project on its own: in version control and released
like any other Java project.

If you think that tagging your root POM for every external dependency change
is too much, you haven't wasted a week tracking down cross project dependency
errors.

All of your Maven projects will include your root POM and all of its version
information.  This way, you get your company's selected version of each
external dependency, and all of the correct Maven plugins. If you need to pull
in external dependencies, it works just like this:

```xml
<dependencies>
    <dependency>
        <groupId>org.third.party</groupId>
        <artifactId>some-artifact</artifactId>
    </dependency>
</dependencies>
```

If you want internal dependencies, that should be managed by each individual
project's **<dependencyManagement>** section. Otherwise it would be difficult
to keep the root POM version number sane.

### Dependency Convergence

One of the best parts about Java is the massive amount of third party
libraries which do everything. Essentially every API or toolkit has a Java SDK
and it's easy to pull it in with Maven.

And those Java libraries themselves depend on specific versions of other
libraries. If you pull in enough libraries, you'll get version conflicts, that
is, something like this:

    Foo library depends on Bar library v1.0
    Widget library depends on Bar library v0.9

Which version will get pulled into your project?

With the [Maven dependency convergence plugin][depconverge], the build will 
error if your dependencies don't use the same version. Then, you have two
options for solving the conflict:

1. Explicitly pick a version for Bar in your *dependencyManagement* section
2. Exclude Bar from either Foo or Widget

The choice of which to choose depends on your situation: if you want to track
one project's version, then exclude makes sense. On the other hand, if you
want to be explicit about it, you can pick a version, although you'll need to
update it when you update the other dependencies.

## Continuous Integration

Obviously you need some kind of continuous integration server which is going
to continuously build your SNAPSHOT versions and tag builds based on git tags.

[Jenkins][jenkins] and [Travis-CI][travis] are natural choices.

Code coverage is useful, and [Cobertura][cobertura] has 
[a good Maven plugin][coberturamaven] and CI support. There are other code
coverage tools for Java, but I've used Cobertura.

## Maven repository

You need a place to put your JARs, WARs, and EARs that you make, so you'll
need a repository.

Common choices are [Artifactory][artifactory] and [Nexus][nexus]. Both work,
and have their own [pros and cons][mavenrepo].

You should have your own Artifactory/Nexus installation and 
[mirror your dependencies][artifactorymirror] onto it. This will stop your
build from breaking because some upstream Maven repository went down.

## Configuration management

So now you've got your code compiled, your repository set up, and you need to
get your code out in your development environment and eventually push it to
production. Don't skimp here, because automating this will pay dividends for a
long time.

[Chef][chef], [Puppet][puppet], and [Ansible][ansible] are typical choices.
I've written an alternative called [Squadron][squadron], which I, of course,
think you should check out because it's easier to get right than the
alternatives.

Regardless of what tool you choose, don't forget to automate your deployments.

# Libraries

Probably the best feature about Java is the extensive amount of libraries it 
has. This is a small collection of libraries that are likely to be applicable
to the largest group of people.

## Missing Features

Java's standard library, once an amazing step forward, now looks like it's
missing several key features.

### Apache Commons

[The Apache Commons project][apachecommons] has a bunch of useful libraries.

**Commons Codec** has many useful encoding/decoding methods for Base64 and hex
strings. Don't waste your time rewriting those.

**Commons Lang** is the go-to library for String manipulation and creation, 
    character sets, and a bunch of miscellaneous utility methods.

**Commons IO** has all the File related methods you could ever want. It has 
[FileUtils.copyDirectory][copydir], [FileUtils.writeStringToFile][writestring],
[IOUtils.readLines][readlines] and much more.

### Guava

[Guava][guava] is Google's excellent here's-what-Java-is-missing library. It's
almost hard to distill everything that I like about this library, but I'm
going to try.

**Cache** is a simple way to get an in-memory cache that can be used to cache
network access, disk access, memoize functions, or anything really. Just
implement a [CacheBuilder][cachebuilder] which tells Guava how to build your
cache and you're all set!

**Immutable** collections. There's a bunch of these: [ImmutableMap][immutablemap],
[ImmutableList][immutablelist], or even [ImmutableSortedMultiSet][immutablesorted]
 if that's your style.

I also like writing mutable collections the Guava way:

```java
// Instead of
final Map<String, Widget> map = new HashMap<>();

// You can use
final Map<String, Widget> map = Maps.newHashMap();
```

There are static classes for [Lists][lists], [Maps][maps], [Sets][sets] and
more. They're cleaner and easier to read.

If you're stuck with Java 6 or 7, you can use the [Collections2][collections2]
class, which has methods like filter and transform. They allow you to write
fluent code without [Java 8][java8]'s stream support.


Guava has simple things too, like a **Joiner** that joins strings on 
separators and a [class to handle interrupts][uninterrupt] by ignoring them.

### Gson

Google's [Gson][gson] library is a simple and fast JSON parsing library. It
works like this:

```java
final Gson gson = new Gson();
final String json = gson.toJson(fooWidget);

final FooWidget newFooWidget = gson.fromJson(json, FooWidget.class);
```

It's really easy and a pleasure to work with. The [Gson user guide][gsonguide]
has many more examples.

### Java Tuples

One of my on going annoyances with Java is that it doesn't have tuples built
into the standard library. Luckily, the [Java tuples][javatuples] project fixes
that.

It's simple to use and works great:

```java
Pair<String, Integer> func(String input) {
    // something...
    return Pair.with(stringResult, intResult);
}
```

### Joda-Time

[Joda-Time][joda] is easily the best time library I've ever used. Simple,
straightforward, easy to test. What else can you ask for? 

You only need this if you're not yet on Java 8, as that has its own new 
[date time][java8datetime] library that doesn't suck.

### Lombok

[Lombok][lombok] is an interesting library. Through annotations, it allows you
to reduce the boilerplate that Java suffers from so badly.

Want setters and getters for your class variables? Simple:

```java
public class Foo {
    @Getter @Setter private int var;
}
```

Now you can do this:

```java
final Foo foo = new Foo();
foo.setVar(5);
```

And there's [so much more][lombokguide]. I haven't used Lombok in production
yet, but I can't wait to.

### Play framework

**Good alternatives**: [Jersey][jersey] or [Spark][spark]

There are two main camps for doing RESTful web services in Java: 
[JAX-RS][jaxrs] and everything else.

JAX-RS is the traditional way. You combine annotations with interfaces and
implementations to form the web service using something like [Jersey][jersey].
What's nice about this is you can easily make clients out of just the 
interface class.

The [Play framework][play] is a radically different take on web services on
the JVM: you have a routes file and then you write the classes referenced in
those routes. It's actually an [entire MVC framework][playdoc], but you can
easily use it for just REST web services.

It's available for both Java and Scala. It suffers slightly from being 
Scala-first, but it's still good to use in Java.

If you're used to micro-frameworks like Flask in Python, [Spark][spark] will
be very familiar. It works especially well with Java 8.

### SLF4J

There are a lot of Java logging solutions out there. My favorite is
[SLF4J][slf4j] because it's extremely pluggable and can combine logs from many
different logging frameworks at the same time. Have a weird project that uses
java.util.logging, JCL, and log4j? SLF4J is for you.

The [two-page manual][slf4jmanual] is pretty much all you'll need to get
started.

### jOOQ

I dislike heavy ORM frameworks because I like SQL. So I wrote a lot of
[JDBC templates][jdbc] and it was sort of hard to maintain. [jOOQ][jooq] is a
much better solution.

It lets you write SQL in Java in a type safe way:

```java
// Typesafely execute the SQL statement directly with jOOQ
Result<Record3<String, String, String>> result = 
create.select(BOOK.TITLE, AUTHOR.FIRST_NAME, AUTHOR.LAST_NAME)
    .from(BOOK)
    .join(AUTHOR)
    .on(BOOK.AUTHOR_ID.equal(AUTHOR.ID))
    .where(BOOK.PUBLISHED_IN.equal(1948))
    .fetch();
```

Using this and the [DAO][dao] pattern, you can make database access a breeze.

## Testing

Testing is critical to your software. These packages help make it easier.

### jUnit 4

[jUnit][junit] needs no introduction. It's the standard tool for unit testing
in Java.

But you're probably not using jUnit to its full potential. jUnit supports
[parametrized tests][junitparam], [rules][junitrules] to stop you from writing
so much boilerplate, [theories][junittheories] to randomly test certain code,
and [assumptions][junitassume].

### jMock

If you've done your dependency injection, this is where it pays off: mocking
out code which has side effects (like talking to a REST server) and still
asserting behavior of code that calls it.

[jMock][jmock] is the standard mocking tool for Java. It looks like this:

```java
public class FooWidgetTest {
    private Mockery context = new Mockery();

    @Test
    public void basicTest() {
        final FooWidgetDependency dep = context.mock(FooWidgetDependency.class);
        
        context.checking(new Expectations() {{
            oneOf(dep).call(with(any(String.class)));
            atLeast(0).of(dep).optionalCall();
        }});

        final FooWidget foo = new FooWidget(dep);

        Assert.assertTrue(foo.doThing());
        context.assertIsSatisfied();
    }
}
```

This sets up a *FooWidgetDependency* via jMock and then adds expectations. We
expect that *dep*'s *call* method will be called once with some String and that
*dep*'s *optionalCall* method will be called zero or more times.

If you have to set up the same dependency over and over, you should probably
put that in a [test fixture][junitfixture] and put *assertIsSatisfied* in an
*@After* fixture.

### AssertJ

Do you ever do this with jUnit?

```java
final List<String> result = some.testMethod();
assertEquals(4, result.size());
assertTrue(result.contains("some result"));
assertTrue(result.contains("some other result"));
assertFalse(result.contains("shouldn't be here"));
```

This is just annoying boilerplate. [AssertJ][assertj] solves this. You can
transform the same code into this:

```java
assertThat(some.testMethod()).hasSize(4)
                             .contains("some result", "some other result")
                             .doesNotContain("shouldn't be here");
```

This fluent interface makes your tests more readable. What more could you want?

# Tools

## IntelliJ IDEA

**Good alternatives**: [Eclipse][eclipse] and [Netbeans][netbeans]

The best Java IDE is [IntelliJ IDEA][intellij]. It has a ton of awesome
features, and is really the main thing that makes the verbosity of Java
bareable. Autocomplete is great, 
[the inspections are top notch][intellijexample], and the refactoring
tools are really helpful.

The free community edition is good enough for me, but there are loads of great
features in the Ultimate edition like database tools, Spring Framework support
and Chronon.

### Chronon

One of my favorite features of GDB 7 was the ability to travel back in time
when debugging. This is possible with the [Chronon IntelliJ plugin][chronon]
when you get the Ultimate edition.

You get variable history, step backwards, method history and more. It's a
little strange to use the first time, but it can help debug some really
intricate bugs, Heisenbugs and the like.

## JRebel

Continuous integration is often a goal of software-as-a-service products. What
if you didn't even need to wait for the build to finish to see code changes
live?

That's what [JRebel][jrebel] does. Once you hook up your server to your JRebel
client, you can see changes on your server instantly. It's a huge time savings
when you want to experiment quickly.

## The Checker Framework

Java's type system is pretty weak. It doesn't differentiate between Strings
and Strings that are actually regular expressions, nor does it do any
[taint checking][taint]. However, [the Checker Framework][checker]
does this and more.

It uses annotations like *@Nullable* to check types. You can even define 
[your own annotations][customchecker] to make the static analysis done even
more powerful.

## Eclipse Memory Analyzer

Memory leaks happen, even in Java. Luckily, there are tools for that. The best
tool I've used to fix these is the [Eclipse Memory Analyzer][mat]. It takes a
heap dump and lets you find the problem.

There's a few ways to get a heap dump for a JVM process, but I use
[jmap][jmap]:

```bash
$ jmap -dump:live,format=b,file=heapdump.hprof -F 8152
Attaching to process ID 8152, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 23.25-b01
Dumping heap to heapdump.hprof ...
... snip ...
Heap dump file created
```

Then you can open the *heapdump.hprof* file with the Memory Analyzer and see
what's going on fast.

# Resources

Resources to help you become a Java master.

## Books

* [Effective Java](http://www.amazon.com/Effective-Java-Edition-Joshua-Bloch/dp/0321356683)
* [Java Concurrency in Practice](http://www.amazon.com/Java-Concurrency-Practice-Brian-Goetz/dp/0321349601)

## Podcasts

* [The Java Posse](http://www.javaposse.com/)


[immutablemap]: http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/ImmutableMap.html
[immutablelist]: http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/ImmutableList.html
[immutableset]: http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/collect/ImmutableSet.html
[depconverge]: https://maven.apache.org/enforcer/enforcer-rules/dependencyConvergence.html
[copydir]: http://commons.apache.org/proper/commons-io/javadocs/api-release/org/apache/commons/io/FileUtils.html#copyDirectory(java.io.File,%20java.io.File)
[writestring]: http://commons.apache.org/proper/commons-io/javadocs/api-release/org/apache/commons/io/FileUtils.html#writeStringToFile(java.io.File,%20java.lang.String)
[readlines]: http://commons.apache.org/proper/commons-io/javadocs/api-release/org/apache/commons/io/IOUtils.html#readLines(java.io.InputStream)
[guava]: http://code.google.com/p/guava-libraries/
[cachebuilder]: http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/cache/CacheBuilder.html
[immutablesorted]: http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableSortedMultiset.html
[uninterrupt]: http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/util/concurrent/Uninterruptibles.html
[lists]: http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/Lists.html
[maps]: http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/Maps.html
[sets]: http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/Sets.html
[collections2]: http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/Collections2.html
[rootpom]: https://gist.github.com/cxxr/10787344
[mavenexample]: http://books.sonatype.com/mvnex-book/reference/index.html
[jenkins]: http://jenkins-ci.org/
[travis]: https://travis-ci.org/
[cobertura]: http://cobertura.github.io/cobertura/
[coberturamaven]: http://mojo.codehaus.org/cobertura-maven-plugin/usage.html
[nexus]: http://www.sonatype.com/nexus
[artifactory]: http://www.jfrog.com/
[mavenrepo]: http://stackoverflow.com/questions/364775/should-we-use-nexus-or-artifactory-for-a-maven-repo
[artifactorymirror]: http://www.jfrog.com/confluence/display/RTF/Configuring+Artifacts+Resolution
[gson]: https://code.google.com/p/google-gson/
[gsonguide]: https://sites.google.com/site/gson/gson-user-guide
[joda]: http://www.joda.org/joda-time/
[lombokguide]: http://jnb.ociweb.com/jnb/jnbJan2010.html
[play]: http://www.playframework.com/
[chef]: http://www.getchef.com/chef/
[puppet]: http://puppetlabs.com/
[ansible]: http://www.ansible.com/home
[squadron]: http://www.gosquadron.com
[googlestyle]: http://google-styleguide.googlecode.com/svn/trunk/javaguide.html
[googlepractices]: http://google-styleguide.googlecode.com/svn/trunk/javaguide.html#s6-programming-practices
[di]: http://en.wikipedia.org/wiki/Dependency_injection
[spring]: http://projects.spring.io/spring-framework/
[springso]: http://programmers.stackexchange.com/questions/92393/what-does-the-spring-framework-do-should-i-use-it-why-or-why-not
[java8]: http://www.java8.org/
[javastream]: http://blog.hartveld.com/2013/03/jdk-8-33-stream-api.html
[slf4j]: http://www.slf4j.org/
[slf4jmanual]: http://www.slf4j.org/manual.html
[junit]: http://junit.org/
[junitparam]: https://github.com/junit-team/junit/wiki/Parameterized-tests
[junitrules]: https://github.com/junit-team/junit/wiki/Rules
[junittheories]: https://github.com/junit-team/junit/wiki/Theories
[junitassume]: https://github.com/junit-team/junit/wiki/Assumptions-with-assume
[jmock]: http://jmock.org/
[junitfixture]: https://github.com/junit-team/junit/wiki/Test-fixtures
[initializingbean]: http://docs.spring.io/spring/docs/3.2.6.RELEASE/javadoc-api/org/springframework/beans/factory/InitializingBean.html
[apachecommons]: http://commons.apache.org/
[lombok]: http://projectlombok.org/
[javatuples]: http://www.javatuples.org/
[dontbean]: http://www.javapractices.com/topic/TopicAction.do?Id=84
[nullable]: http://code.google.com/p/google-guice/wiki/UseNullable
[optional]: http://www.oracle.com/technetwork/articles/java/java8-optional-2175753.html
[jdbc]: http://docs.spring.io/spring/docs/4.0.3.RELEASE/javadoc-api/org/springframework/jdbc/core/JdbcTemplate.html
[jooq]: http://www.jooq.org/
[dao]: http://www.javapractices.com/topic/TopicAction.do?Id=66
[gradle]: http://www.gradle.org/
[intellij]: http://www.jetbrains.com/idea/
[intellijexample]: http://i.imgur.com/92ztcCd.png
[chronon]: http://blog.jetbrains.com/idea/2014/03/try-chronon-debugger-with-intellij-idea-13-1-eap/
[eclipse]: https://www.eclipse.org/
[dagger]: http://square.github.io/dagger/
[guice]: https://code.google.com/p/google-guice/
[netbeans]: https://netbeans.org/
[mat]: http://www.eclipse.org/mat/
[jmap]: http://docs.oracle.com/javase/7/docs/technotes/tools/share/jmap.html
[jrebel]: http://zeroturnaround.com/software/jrebel/
[taint]: http://en.wikipedia.org/wiki/Taint_checking
[checker]: http://types.cs.washington.edu/checker-framework/
[customchecker]: http://types.cs.washington.edu/checker-framework/tutorial/webpages/encryption-checker-cmd.html
[builderex]: http://jlordiales.wordpress.com/2012/12/13/the-builder-pattern-in-practice/
[javadocex]: http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/collect/ImmutableMap.Builder.html
[dropwizard]: https://dropwizard.github.io/dropwizard/
[jersey]: https://jersey.java.net/
[springboot]: http://projects.spring.io/spring-boot/
[spark]: http://www.sparkjava.com/
[assertj]: http://joel-costigliola.github.io/assertj/index.html
[jaxrs]: http://en.wikipedia.org/wiki/Java_API_for_RESTful_Web_Services
[playdoc]: http://www.playframework.com/documentation/2.3.x/Anatomy
[java8datetime]: http://www.oracle.com/technetwork/articles/java/jf14-date-time-2125367.html



