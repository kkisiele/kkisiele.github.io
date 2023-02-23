---
layout: post
title:  "Leverage Lambda Syntax for Cleaner Code"
date:   2023-02-22 20:052:00 +0200
---

## Once upon a time in a production code
When I was working on code persisting some domain data, I ended up with the following:
{% highlight java %}
for (int retry = 0; retry <= MAX_RETRIES; retry++) {
    try {
        updateDatabase();
        return;
    } catch (SQLException ex) {
        if (retry >= MAX_RETRIES) {
            throw ex;
        }
        LOG.warn("Fail to execute database update. Retrying...", ex);
        reestablishConnection();
    }
}
{% endhighlight %}
The code basically performs a database update and handles retry logic in case of errors. The main error I worry about is a timed-out JDBC connection, which needs to be reestablished.

Somehow, I didn't like the initial version from a clean code perspective. I expected something that would give instant clarity about what it's doing without the need to dive into the details.

## Less procedural, more declarative
The first step is to move updateDatabase() call to lambda powered variable. The naive approach might look like:
{% highlight java %}
Runnable handle = () -> updateDatabase();
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
{% endhighlight %}
Unfortunately, the above does not work. The Runnable.run() method doesn't allow any checked exception.
We must provide custom functional interface which deals with SQLException:
{% highlight java %}
@FunctionalInterface
interface SqlRunnable {
    void run() throws SQLException;
}
{% endhighlight %}

Now, with Extract Method refactoring we move the for loop and its content to a new method named retryOnSqlException:
{% highlight java %}
SqlRunnable handle = () -> updateDatabase();
retryOnSqlException(handle);
{% endhighlight %}
Let's use Inline Variable to inline handle variable.

We end-up with the following method content:
{% highlight java %}
retryOnSqlException(() -> updateDatabase());
{% endhighlight %}
And some supporting code:
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
