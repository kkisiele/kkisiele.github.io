---
title: Use declarative approach for better code
layout: post
---
From time to time when browse the codebase or review the pull requests I encounter the code
structure similar to the following:
```java
    public Credentials getCredentials() {
        Credentials credentials = env.getCredentials();
        if (credentials != null) {
            return credentials;
        }
        credentials = userProperties.getCredentials();
        if (credentials != null) {
            return credentials;
        }
        return defaults.getCredentials();
    }
```
It basically evaluates few data providers and returns the first value satisfying the given condition
(not null) till the last return statement where some default is returned (or exception is thrown).

I bet it's an implementation that most programmers would commit and push out. It represents
an imperative paradigm mixing _what_ to achieve (return credentials) and _how_ to do that (control flow).
And it's the easiest one to come with.

I see two issues with this code.
It lacks **instant clarity** what it's doing. One must spend a few dozen seconds to grasp what it's about.
Not a big deal if it's the only code you inspect, but more often it's just a part of bigger investigation.
The other thing I notice, it's way **too long** that it could be. I prefer the very compact methods and in Kotlin the
single-expression methods are my favourite.

On the other end is a declarative paradigm expressing what to achieve without specifying how.
The declarative approach is a way to follow in order to simplify the code down to fewer LOC and
better code clarity.

```java
    public Credentials getCredentials() {
        return firstNonNull(
                env::getCredentials,
                userProperties::getCredentials,
                defaults::getCredentials
        );
    }
```
Unfortunately, Java doesn't provide such method in the JDK. You either have to write one by yourself or use a third-party library.
The popular [Apache Commons Lang](https://commons.apache.org/proper/commons-lang/) has
[ObjectUtils.getFirstNonNull](https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/ObjectUtils.html#getFirstNonNull-java.util.function.Supplier...-)
for that and your own generic implementation can look similar to the following:
```java
    public static <T> T firstNonNull(Supplier<T>... suppliers) {
        for (var supplier : suppliers) {
            T result = supplier.get();
            if (result != null) {
                return result;
            }
        }
        return null;
    }
```
At the end, let me advertise the Kotlin in which thanks to the Elvis operator (?:) the code can
be compacted to one line-of-code:
```kotlin
    fun getCredentials(): Credentials? = env.credentials ?: userProperties.credentials ?: defaults.credentials
```