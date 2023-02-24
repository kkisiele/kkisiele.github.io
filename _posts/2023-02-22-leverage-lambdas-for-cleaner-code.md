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
The `processMessage` is part of a framework contract and is called for every processed message to be persisted.
The code performs an idempotent database upsert and handles retry logic in case of errors. The main error I worried about was a timed-out JDBC connection, which needs to be re-established.

Somehow, I didn't like the initial version from a clean code perspective. I expected something that would give **instant clarity** of its intent without the need to dive into the code itself. The `processMessage` is full of low level details and one needs to understand them to know what it's about. Also, wanted to separate the retry logic from the retried database operation to make the retry logic easy to reuse.

Decided to rewrite it to address the mentioned issues.
## Less procedural, more declarative
The first step is to move `updateDatabase()` call to lambda powered variable. 
Let the IDE help with that by using _Introduce Functional Variable_. Unfortunatelly, we get
> No applicable functional interfaces found

The reason for that is lack of functional interface which provides SAM interface compatible with upsert method. It should declare single abstract method, which accepts no parameters, returns nothing and throws `SQLException`.

We need to provide it by ourselves:
{% highlight java %}
@FunctionalInterface
interface SqlRunnable {
    void run() throws SQLException;
}
{% endhighlight %}
The custom functional interface is in place, so let's repeat the refactoring. Now succeeds. Also, move the variable assignment before the _for_ loop.
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
Use _Extract Method_ refactoring to move the _for_ loop and its content to a new method named `retryOnSqlException`:
{% highlight java %}
public void processMessage(InsuranceProduct product) throws Exception {
    final SqlRunnable handle = () -> upsert(product);
    retryOnSqlException(handle);
}
private void retryOnSqlException(SqlRunnable handle) throws SQLException {
    //skipped for clarity
}
{% endhighlight %}
The last step is to use _Inline Variable_ to inline the `handle` variable.

The final result looks as follow:
{% highlight java %}
public void processMessage(InsuranceProduct product) throws Exception {
    retryOnSqlException(() -> upsert(product));
}

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
Was it worth the effort? Definitely. Let's recap the benefits. 

The `processMessage` method now expresses the intent much clearly by utilizing a declarative approach with a high-level code. Details regarding the retry logic are placed in its own method, which, thanks to good naming, precisely describes the method's purpose.
The lambda syntax separates the retry logic and database operation and enables the reuse of the retry feature.