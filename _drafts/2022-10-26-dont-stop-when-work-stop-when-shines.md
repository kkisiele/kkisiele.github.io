---
layout: post
title:  "Don't stop when works, stop when shines"
date:   2022-10-26 16:00:00 +0200
---

How often do you finish with working source code without giving it a try to polish? By pushing too early you might miss the opportunity to create an extraordinary piece of code you will be proud of. In the post, I am presenting a hands-on example of refactoring turning ordinary, everyone-could-write code into maintenance-oriented, very high-quality work.

## The Story Begins
John is assigned the task to implement new functionality in the legacy stock trading platform.
The product owner requests the following
> As a user, I can see the average (weighted) price of the stocks from my portfolio, so I know how the buy price relates to the market price.

He is an experienced programmer, so the task was straightforward and took less time than expected.
The main domain piece of the finished code looks as follows:
{% highlight java %}
public Map<StockSymbol, BigDecimal> weightedAverage(List<UserStockTransaction> stocks) {
    Map<StockSymbol, List<UserStockTransaction>> stocksBySymbol = new HashMap<>();
    for (UserStockTransaction stock : stocks) {
        stocksBySymbol.computeIfAbsent(stock.getSymbol(), stockSymbol -> new ArrayList<>())
            .add(stock);
    }
    Map<StockSymbol, BigDecimal> result = new HashMap<>();
    for (Map.Entry<StockSymbol, List<UserStockTransaction>> entry : stocksBySymbol.entrySet()) {
        BigDecimal totalPrice = entry.getValue().stream()
                .map(userStock -> userStock.getPricePerShare().multiply(userStock.getQuantity()))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        BigDecimal totalQuantity = entry.getValue().stream()
                .map(userStock -> userStock.getQuantity())
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        result.put(entry.getKey(), totalPrice.divide(totalQuantity, MathContext.DECIMAL128));
    }
    return result;
}
{% endhighlight %}
John fills in the pull request. He is ready to click the submit button, but some noise outside the window distracts him... it is
just a horde of motorcyclists. While backing to the pull request, he notices the dusty "Clean Code" which serves as a monitor stand.

Suddenly John reminds himself of all Uncle Bob's arguments about software craftsmanship, professionalism and the like nonsense.
He experiences conscience pangs. The reviewers are in a distant time zone and will be at work in a couple of hours. So there is no need to hurry up. He decides to polish it a bit.

## Course of Action
He reviews the code and gives it some thought.
The method name says _what_ it is doing. It returns a weighted average for provided `UserStockTransaction` items.
What he wonders is whether some of his colleagues will grasp his intentions of _how_ the mehod is fulfilling its duties? At least without spending a minute or two.

Realizes the method content is very low-level and full of implementation details. It is not instantly clear _how_ it is working.
John would like something closer to the natural language thus even his always-busy scrum master could understand. The goal (despite Java's clunky syntax) is to have the code as similar to the following as possible:
{% highlight java %}
public Map<StockSymbol, BigDecimal> weightedAverage(List<UserStockTransaction> stocks) {
    // groups UserStock by StockSymbol
    // for each group calculates weighted average
    // returns weighted average per StockSymbol
}
{% endhighlight %}
A messenger notification pops up. It is Kevin, a new programmer he onboarded a couple of days ago. Kevin was assigned first serious task and wants to know how John manage with his? It turns out the fellow's gig is similar in some parts. It needs to perform the same calculation (weighted average) in the bond module. John don't like being interrupted when in the _flow state_, so reply that let him know when be done. In the meantime, he sets the messenger status to "Do Not Disturb".

He thinks for a second... have read dozen times about _Don't Repeat Principle_ and knows that code copy is not an option, at least in this case. He will need to extract the weighted average calculation and make it generic.

## Declarative Approach
You start with something simple and move the inner guts of the `for` loop into dedicated method.
Use the Extract Method for that and name it appropriate. There is already weightedAverage(List) method, so you name it slightly different: `calculateWeightedAverage`.

The result is below:
{% highlight java %}
public Map<StockSymbol, BigDecimal> weightedAverage(List<UserStockTransaction> stocks) {
    Map<StockSymbol, List<UserStockTransaction>> stocksBySymbol = new HashMap<>();
    for (UserStockTransaction stock : stocks) {
        stocksBySymbol.computeIfAbsent(stock.getSymbol(), stockSymbol -> new ArrayList<>()).add(stock);
    }
    Map<StockSymbol, BigDecimal> result = new HashMap<>();
    for (Map.Entry<StockSymbol, List<UserStockTransaction>> entry : stocksBySymbol.entrySet()) {
        result.put(entry.getKey(), calculateWeightedAverage(entry.getValue()));
    }
    return result;
}
{% endhighlight %}
The first `for` loop just groups the provided data by the stock symbol. When you dive into
the `Java Stream API` you will discover its abilities to handle such cases by leveraging the `groupingBy` Collector.

Let's use it to replace the imperative code with more declarative approach:
{% highlight java %}
public Map<StockSymbol, BigDecimal> weightedAverage(List<UserStockTransaction> stocks) {
    Map<StockSymbol, List<UserStockTransaction>> stocksBySymbol = stocks.stream()
            .collect(groupingBy(UserStockTransaction::getSymbol));
    Map<StockSymbol, BigDecimal> result = new HashMap<>();
    for (Map.Entry<StockSymbol, List<UserStockTransaction>> entry : stocksBySymbol.entrySet()) {
        result.put(entry.getKey(), calculateWeightedAverage(entry.getValue()));
    }
    return result;
}
{% endhighlight %}
Now,  I have the feeling that the remaining `for` loops akward here. The method mixes both imperative and declarative code.
I am browsing the some [examples][baeldung] and notice the following:
{% highlight java %}
posts.stream()
    .collect(groupingBy(BlogPost::getType, averagingInt(BlogPost::getLikes)));
{% endhighlight %}
After some trails and errors, it turns out it can be expressed in the following way:
{% highlight java %}
public Map<StockSymbol, BigDecimal> weightedAverage(List<UserStockTransaction> stocks) {
    return stocks.stream().collect(
            groupingBy(UserStockTransaction::getSymbol,
                    collectingAndThen(toList(), this::calculateWeightedAverage))
    );
}
{% endhighlight %}
Definitely, looks much cleaned now. The declarative approach gives programmer the **instant clarity** how the method implements the functionality.
Not to mention the result code is much more concise.

## Reusability
Let's focus on calculateWeightedAverage method presented below.
The goal is a generic weighted average implementation, not tied to any particular business use-case. It will allow to reuse it.
{% highlight java %}
private BigDecimal calculateWeightedAverage(List<UserStockTransaction> stocks) {
    BigDecimal totalPrice = stocks.stream()
            .map(userStock -> userStock.getPricePerShare().multiply(userStock.getQuantity()))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    BigDecimal totalQuantity = stocks.stream()
            .map(userStock -> userStock.getQuantity())
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    return totalPrice.divide(totalQuantity, MathContext.DECIMAL128);
}
{% endhighlight %}
By taking above into account it is obvious we need a dedidcated class for that.
A class has name and the following come to my mind:
* WeightedAverage (suprise!)
* Financial
* Statistics

I like the first and third ones. I pick the last one, because weighted average is part of statistics science.
Although WeightedAverage might be more appropriate from the class name look-up perspective, especially in the spacious codebase.

We move calculateWeightedAverage to the Statistics class and create dedicated value object in place of UserStockTransaction.
Also, change the method name calculateWeightedAverage -> weightedAverage. The prefixes like get, find, fetch, calculate are just noise and doesn't bring any real value.

The generic implementation is as follows:
{% highlight java %}
public class Statistics {
    public static BigDecimal weightedAverage(List<WeightedValue> values) {
        BigDecimal totalValues = values.stream()
                .map(value -> value.getValue().multiply(value.getWeight()))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        BigDecimal totalWeight = values.stream()
                .map(value -> value.getWeight())
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        return totalValues.divide(totalWeight, MathContext.DECIMAL128);
    }
}

public final class WeightedValue {
    private final BigDecimal value;
    private final BigDecimal weight;

    public WeightedValue(BigDecimal value, BigDecimal weight) {
        this.value = value;
        this.weight = weight;
    }

    public BigDecimal getValue() {
        return value;
    }

    public BigDecimal getWeight() {
        return weight;
    }
    
    //skipped equals, hashCode, toString methods
}
{% endhighlight %}
We are left with calculateWeightedAverage method which needs to be rewritten to use the Statistics.weightedAverage:
{% highlight java %}
private BigDecimal calculateWeightedAverage(List<UserStockTransaction> stocks) {
    List<WeightedValue> weightedValues = stocks.stream()
            .map(stock -> new WeightedValue(stock.getPricePerShare(), stock.getQuantity()))
            .collect(toList());
    return weightedAverage(weightedValues);
}
{% endhighlight %}

## Flexibility
One thing I don't like in the new calculateWeightedAverage method is the data conversion step (UserStockTransaction -> WeightedValue).
It is cumbersome from the caller point of view thus making the API inconvienient to use. How to improve that? Lambda to the rescue.

Let's get rid of the conversion step and make the Statistics.calculateWeightedAverage handles that in a generic-way. The weightedAverage will accept
any list of objects and a lambda to convert list's item to WeightedValue.
{% highlight java %}
private BigDecimal calculateWeightedAverage(List<UserStockTransaction> stocks) {
    return weightedAverage(stocks, stock -> new WeightedValue(stock.getPricePerShare(), stock.getQuantity()));
}
{% endhighlight %}
In case of not many extracted values (two in that case) we can go even further and provide lambda for each value w
{% highlight java %}
private BigDecimal calculateWeightedAverage(List<UserStockTransaction> stocks) {
    return weightedAverage(stocks, UserStockTransaction::getPricePerShare, UserStockTransaction::getQuantity);
}
{% endhighlight %}
To handle that in weightedAverage, some language magic is needed:
{% highlight java %}
public class Statistics {
    public static <T> BigDecimal weightedAverage(List<T> values, Function<T, BigDecimal> toValue, Function<T, BigDecimal> toWeight) {
        BigDecimal totalValues = values.stream()
                .map(domain -> toValue.apply(domain).multiply(toWeight.apply(domain)))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        BigDecimal totalWeight = values.stream()
                .map(domain -> toWeight.apply(domain))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        return totalValues.divide(totalWeight, MathContext.DECIMAL128);
    }
}
{% endhighlight %}
## Custom Collector
Let me recap weightedAverage method:
{% highlight java %}
public Map<StockSymbol, BigDecimal> weightedAverage(List<UserStockTransaction> stocks) {
    return stocks.stream().collect(
            groupingBy(UserStockTransaction::getSymbol,
                    collectingAndThen(toList(), this::calculateWeightedAverage))
    );
}
{% endhighlight %}
Can we improve it? I would like to rewrite the collectingAndThen part in a more direct way, similar how the built-in Collectors are used:
{% highlight java %}
players.stream().collect(
    groupingBy(Player::getTeam,
        averagingInt(Player::getAge))
)
{% endhighlight %}
It turns out we need to provide a custom Collector handling weighted average:
{% highlight java %}
public Map<StockSymbol, BigDecimal> weightedAverage(List<UserStockTransaction> stocks) {
    return stocks.stream().collect(
            groupingBy(UserStockTransaction::getSymbol,
                    weightedAverage(UserStockTransaction::getPricePerShare, UserStockTransaction::getQuantity))
    );
}
{% endhighlight %}
The custom Collector is tied to the Statistics class so I place it here as a static nested class:
{% highlight java %}
public class Statistics {
    public static class Collectors {
        public static <T> Collector<T, ?, BigDecimal> weightedAverage(Function<T, BigDecimal> toValue, Function<T, BigDecimal> toWeight) {
            return collectingAndThen(toList(), list -> Statistics.weightedAverage(list, toValue, toWeight));
        }
    }
}
{% endhighlight %}

## Retrospective
{% highlight java %}
// BEFORE
public Map<StockSymbol, BigDecimal> weightedAverage(List<UserStockTransaction> stocks) {
    Map<StockSymbol, List<UserStockTransaction>> stocksBySymbol = new HashMap<>();
    for (UserStockTransaction stock : stocks) {
        stocksBySymbol.computeIfAbsent(stock.getSymbol(), stockSymbol -> new ArrayList<>()).add(stock);
    }
    Map<StockSymbol, BigDecimal> result = new HashMap<>();
    for (Map.Entry<StockSymbol, List<UserStockTransaction>> entry : stocksBySymbol.entrySet()) {
        BigDecimal totalPrice = entry.getValue().stream()
                .map(userStock -> userStock.getPricePerShare().multiply(userStock.getQuantity()))
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        BigDecimal totalQuantity = entry.getValue().stream()
                .map(userStock -> userStock.getQuantity())
                .reduce(BigDecimal.ZERO, BigDecimal::add);
        result.put(entry.getKey(), totalPrice.divide(totalQuantity, MathContext.DECIMAL128));
    }
    return result;
}
// AFTER
public Map<StockSymbol, BigDecimal> weightedAverage(List<UserStockTransaction> stocks) {
    return stocks.stream().collect(
            groupingBy(UserStockTransaction::getSymbol,
                    weightedAverage(UserStockTransaction::getPricePerShare, UserStockTransaction::getQuantity))
    );
}
{% endhighlight %}

[baeldung]: https://www.baeldung.com/java-groupingby-collector
