---
layout: post
title:  "Leverage Lambdas for Cleaner Code"
date:   2023-02-22 20:052:00 +0200
---
## Once upon a time in a production code
When I was working on code persisting some domain data, I ended up with the following:
{% highlight java %}
public void processMessage(InsuranceProduct product) throws Exception {
    for (int retry = 0; retry <= MAX_RETRIES; retry++) {
        try {
            upsert(product);
            return;
        } catch (SQLException ex) {
            if (retry >= MAX_RETRIES) {
                throw ex;
            }
            LOG.warn("Fail to execute database update. Retrying...", ex);
            reestablishConnection();
        }
    }
}

private void upsert(InsuranceProduct product) throws SQLException {
    //content not relevant
}

{% endhighlight %}
The `processMessage` is a part of a framework contract and is called to persist every processed message.
The code performs an idempotent database upsert and handles retry logic in case of errors. The main error I was concerned about was a timed-out JDBC connection that needs to be re-established.

I wasn't satisfied with the initial version of `processMessage` from a clean code perspective. I expected something that would instantly reveal its intent without the need to dive into the code. The method is full of low-level details that need to be understood to know what it does. Additionally, I wanted to separate the retry logic from the database operation being retried to make it easy to reuse.

Decided to rewrite it to address the mentioned issues.
## Less procedural, more declarative
The first step is to move the `updateDatabase()` call to a lambda-powered variable.
Let the IDE help with that by using _Introduce Functional Variable_ refactoring. Unfortunately, we get an error message:
> No applicable functional interfaces found

The reason for this is the absence of a functional interface that provides a SAM interface compatible with the upsert method. To address this issue, we need to define a custom functional interface that declares a single abstract method accepting no parameters, returning nothing, and throwing `SQLException`. Here's the interface we need to provide:
{% highlight java %}
@FunctionalInterface
interface SqlRunnable {
    void run() throws SQLException;
}
{% endhighlight %}
With the custom functional interface in place, let's repeat the refactoring. This time, it succeeds. Also, let's move the variable assignment before the _for_ loop:
{% highlight java %}
public void processMessage(InsuranceProduct product) throws Exception {
    final SqlRunnable handle = () -> upsert(product);
    for (int retry = 0; retry <= MAX_RETRIES; retry++) {
        try {
            handle.run();
            return;
        } catch (SQLException ex) {
            if (retry >= MAX_RETRIES) {
                throw ex;
            }
            LOG.warn("Fail to execute database update. Retrying...", ex);
            reestablishConnection();
        }
    }
}
{% endhighlight %}
Use the _Extract Method_ refactoring to move the _for_ loop and its content to a new method named `retryOnSqlException`:
{% highlight java %}
public void processMessage(InsuranceProduct product) throws Exception {
    final SqlRunnable handle = () -> upsert(product);
    retryOnSqlException(handle);
}

private void retryOnSqlException(SqlRunnable handle) throws SQLException {
    //skipped for clarity
}
{% endhighlight %}
The last step is to use the _Inline Variable_ refactoring to inline the `handle` variable.

The final result is below.
{% highlight java %}
public void processMessage(InsuranceProduct product) throws Exception {
    retryOnSqlException(() -> upsert(product));
}
{% endhighlight %}
The framework entry method now clearly states what it is doing. It is only one line long, so there is no cognitive load.

The supporting code contains details on how it fulfills its duty and enables reusability:
{% highlight java %}
private void retryOnSqlException(SqlRunnable handle) throws SQLException {
    for (int retry = 0; retry <= MAX_RETRIES; retry++) {
        try {
            handle.run();
            return;
        } catch (SQLException ex) {
            if (retry >= MAX_RETRIES) {
                throw ex;
            }
            LOG.warn("Fail to execute database update. Retrying...", ex);
            reestablishConnection();
        }
    }
}

@FunctionalInterface
interface SqlRunnable {
    void run() throws SQLException;
}
{% endhighlight %}
## Conclusion
Was it worth the effort? Absolutely. Let's summarize the benefits. 

The `processMessage` method now clearly expresses its intent by utilizing a declarative approach with high-level code. The retry logic is separated from the database operation and placed in its own method, which, thanks to good naming, precisely reveals its intent. Furthermore, the lambda syntax allows for easy reuse of the retry feature with other database operations.