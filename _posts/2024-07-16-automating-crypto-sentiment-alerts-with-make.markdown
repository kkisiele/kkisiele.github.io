---
layout: post
title: "Automating Crypto Sentiment Alerts with Make (Integromat)"
feature-image: "/uploads/66941b146355bb0881d3e258_Automating%20Crypto%20Sentiment%20Alerts%20with%20Make%20(Integromat).jpg"
date: 2024-07-16 10:00:00 +0200
category: automation
---
The post describes automating crypto sentiment alerts using Make (Integromat) and the Fear and Greed Index API. It explains how to set up notifications when market sentiment indicates fear which ofter signal a good buying opportunity.

## Disclaimer
The information provided in this post is for informational and educational purposes only and should
not be construed as investment advice, financial advice, trading advice, or any other sort of
advice. The content is based on the author’s personal opinions and experiences and is not intended
to be a recommendation or solicitation to buy, sell, or hold any cryptocurrencies or any other
financial instruments. The cryptocurrency market is highly volatile and can result in significant
financial loss. Always do your own research and consult with a qualified financial advisor before
making any investment decisions.
 
## Introduction
Investing in the cryptocurrency market can be highly profitable but also fraught with risks and
volatility. One of the key indicators that savvy investors look for is market sentiment. When the
sentiment is dominated by fear, it can often signal a good buying opportunity.

In this blog post, we will explore how to use Make (if not have an account please [register](https://www.make.com/en/register?pc=kkisiele) from my ref link) to send
notifications when the crypto market sentiment indicates fear. We will leverage the API from [alternative.me](https://alternative.me/crypto/fear-and-greed-index/) to track sentiment and automatically send notifications when it falls below a set threshold.
## Understanding Market Sentiment
Market sentiment is essentially the overall attitude of investors toward a particular market or
asset. In the crypto market, sentiment can swing wildly between fear and greed. The Fear and Greed
Index quantifies this sentiment on a scale from 0 to 100, where lower values indicate fear and
higher values indicate greed.

Historically, buying during periods of fear has been a profitable strategy, as prices are often lower.
By setting up a system to monitor this index, investors can be alerted to potential buying opportunities when fear dominates the market.

## Using the Fear and Greed Index API
To get started, you'll need to make requests to the Fear and Greed Index. Use a dedicated
JSON-based API to avoid struggling with HTML parsing. A sample response from this API looks as
follows:
{% highlight json %}
// GET https://api.alternative.me/fng/?limit=1
// Response
{
	"name": "Fear and Greed Index",
	"data": [
		{
			"value": "40",
			"value_classification": "Fear",
			"timestamp": "1551157200",
			"time_until_update": "68499"
		}
	],
	"metadata": {
		"error": null
	}
}{% endhighlight %}    
## Solution Outline
In your Make scenario, configure an HTTP GET request to the API endpoint. Parse the JSON response to
extract the sentiment value. Set a condition to check if this value falls below a predetermined
threshold, indicating fear. If the condition is met, trigger a notification to be sent to your
preferred communication channel, such as email, SMS, or a messaging app.
## Iteration 1
The first scenario is presented below. It makes a Greed&Fear API request to get the current
sentiment and sends notifications if the sentiment value is below a given threshold. The detailed
steps are described below:

![interface](/uploads/it1.1.png)

The first step is to declare the Fear&Greed notification threshold. Here, I am setting it to
30, but you can define it based on your observations of historical data:

![interface](/uploads/6693faa4aa83e5de2ed29fdf_it1.2.png)

Next, we make a request to the Fear&Greed Index API to get the current sentiment value, which
is used in the next parts of the scenario. The only parts to set up here are the <em>URL</em> and
<em>Parse response</em> sections. Because I like to be specific, I provide URL parameters, although
they are optional.

![interface](/uploads/6693fc6d40d3d6d8463f9738_it1.3.png)

The next important element is a filter, placed between the Iterator and the Router. The filter
ensures that the router and the connected nodes execute only once the condition is met. In this
case, the current sentiment value (fetched via the API) must be less than or equal to the defined
threshold value.

![interface](/uploads/6693fc7b4a1b7b5226e37fde_it1.4.png)

You might choose the notification methods you prefer. I chose email and Slack notifications, and my
email setup is:

![interface](/uploads/6693fc844734f1b0a2d8efd7_it1.5.png)

The scenario runs once a day, which is sufficient since the Fear&Greed Index is calculated every 24 hours.

![interface](/uploads/6693fc9056691536d5797ea2_it1.6.png)

This initial scenario is quite good but has one drawback. During a long-term fear in the market, you
might receive notifications every day, which can be overwhelming and undesirable. We will address
that in the next part.

## Iteration 2
In this part, we will improve the scenario by storing the time of the last sent notification. This
will ensure we don’t send repeated notifications too often. Once a notification has been sent, the
next one (if it meets our threshold) will not be sent sooner than a week later.

To handle this, we use the built-in Data store. There are two new elements: storing the timestamp of
the last sent notification (module 9) and using it at the beginning of the scenario to ensure it
runs no sooner than a week since the last notification. The overall scenario view is below:

![interface](/uploads/6693fce7dcb28a776ae0e3f3_1.png)

The data store setup is straightforward as it contains only two fields. The last notification date is
represented as a Number because it stores the UNIX timestamp value, which is the number of seconds
since midnight on January 1, 1970, GMT. There is a Make built-in <em>timestamp</em> function to get
it. The second field stores the Fear&Greed Index value.

![interface](/uploads/6693fcf2a6a869ad7e31a3b3_2.png)

I added a new variable called <em>MinimumNotificationDelayInSeconds</em> in module 12. This variable
is designed to control the minimum amount of time, in seconds, that must elapse before a
notification can be sent.

![interface](/uploads/6693fcfe48004a4fe08e7289_3.png)

Now we need to get the last notification date from the data store. I use the &quot;one&quot; key to
always fetch the same record:

![interface](/uploads/693fd099bdd0ca2bf1d5125_4.png)

In the filter, we need to check if the number of seconds between the current timestamp and the last
notification timestamp is greater than the defined <em>MinimumNotificationDelayInSeconds</em>
variable. Only then will the rest of the scenario run.

![interface](/uploads/6693fd13fc2a063fdea56325_5.png)

## Conclusion
By leveraging Make and the Fear and Greed Index API, you can automate the process of monitoring
crypto market sentiment and receive timely notifications when fear dominates the market. This can
help you make informed investment decisions and potentially capitalize on buying opportunities.
While market sentiment is a useful indicator, it should be used in conjunction with other analyses
and strategies to ensure a comprehensive approach to cryptocurrency investing.
