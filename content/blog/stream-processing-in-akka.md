+++
date = "2017-08-29T21:37:16-05:00"
title = "Creating a Stock Market Trading Bot Using Akka Streams"
description = "Example usage of Akka Streams to solve complex asynchronous problems."
tags = [ "akka", "streams", "Scala" ]
categories = [ "Development", "Scala", "How To" ]
+++

To provide an interesting example of utilizing a subset of Akka's streaming capabilities I am going to show how to make
a simple fictitious stock market trading bot controlled by some form of an API endpoint.

This trade bot is not an attempt at becoming Warren Buffet. It is just a fun example to work through some Akka stream concepts.

Here is how the trade bot will work / the functionality we will implement,

User makes a request to initiate the trade bot

**User's request includes the following,**

* The ticker symbol of the stock to track
* The limit of trades per day
* The buy price
* The sell price
* An email address to send notifications to

**The bot will stream the quotes for the requested ticker symbol**

**If a quote price drops below the buy price a buy trade will occur**

**If a quote price increases above the sell price a sale trade will occur**

**If a trade occurs it will send an email notification**

**A user can at anytime update the trading bot with different parameters**

**A user can stop the trade bot at anytime**

First let's create the data case class's we will be working with as well as a kill switch that will be explained later.

```scala
final case class StockQuote(symbol: String, price: Int)
sealed trait Trade {
  val stockQuote: StockQuote
  val shares: Int
}
final case class BuyTrade(stockQuote: StockQuote, shares: Int) extends Trade
final case class SellTrade(stockQuote: StockQuote, shares: Int) extends Trade

var tradeBotKillSwitch: Option[KillSwitch] = None
```

Now we will define our fictitious Email API Interface.

```scala
def sendEmail(emailAddress: String, emailMessage: String): Future[Boolean] = {
  println(emailMessage)
  Future.successful(true)
}

def sendTradeEmail(emailAddress: String)(tradeMade: Trade) = {
  sendEmail(emailAddress, s"Trade: $tradeMade")
}
```

Next we will create a helper function that will generate Quote objects for us.

```scala
def getNextStockQuote(tickerSymbol: String, priceChange: Int) = {
  if (priceChange % 2 == 0 && currentStockPrice - priceChange > 0) {
    currentStockPrice = currentStockPrice - priceChange
  } else {
    currentStockPrice = currentStockPrice + priceChange
  }
  val nextStockQuote = StockQuote(tickerSymbol, currentStockPrice)
  println(s"Stock Quote:  $nextStockQuote")
  nextStockQuote
}
```

Here is the fake Stock streaming API interface we will use.

```scala
def getQuoteStreamForStock(buyPrice: Int, sellPrice: Int)(tickerSymbol: String) = {
  currentStockPrice = (buyPrice + sellPrice) / 2
  Source.fromIterator(() => Iterator.continually(getNextStockQuote(tickerSymbol, Random.nextInt(10))))
    .throttle(1, 1.second, 1, ThrottleMode.shaping)
    .take(100)
}

def makeTrade(tradeToMake: Trade): Future[Boolean] = {
  Future.successful(true)
}
```

The first thing it does is set an initial stock price (since we won't have one to work from), using the initial buy and
sell prices supplied by the user. It then creates a stream source from a continuous iterator that calls our `getNextStockQuote`
function with a random price change between 0 and 10. This will simulate our fluctuating stock prices. We attach a
throttle to the stream source to simulate a slower stream that may be coming from an API. Lastly we add a `take(100)`
to set an arbitrary bounds on the stream. Both the throttle and the take can be experimented with to see their effects.

Then we will write a utility function to create a trade for us based on a buy price, a sell price, and a quote.

```scala
def createTrade(buyPrice: Int, sellPrice: Int)(stockQuote: StockQuote): Option[Trade] = {
  if (stockQuote.price < buyPrice) {
    Some(BuyTrade(stockQuote, 1))
  } else if (stockQuote.price > sellPrice) {
    Some(SellTrade(stockQuote, 1))
  } else {
    None
  }
}
```

If the stock price is below our buy price (buy low), we create a `BuyTrade`. If the stock price is above our sell price
(sell high), we create a `SellTrade`. Otherwise we won't make a trade on the quote.

Our next function will do most of our interesting business logic. This is our fake API endpoint for starting the
trade bot and manipulating the stock stream.

```scala
def startTradeBot(tickerSymbol: String, tradesPerDayLimit: Int, buyPrice: Int, sellPrice: Int, notificationEmailAddress: String): Unit = {
  println(s"Starting new trade bot for $tickerSymbol at $tradesPerDayLimit per day, buy at $buyPrice sell at $sellPrice")
  tradeBotKillSwitch.foreach(_.shutdown())

  val tradeBotTradeCreator = createTrade(buyPrice, sellPrice)(_)
  val tradeBotEmailCreator = sendTradeEmail(notificationEmailAddress)(_)

  val newTradeBotKillSwitch = getQuoteStreamForStock(buyPrice, sellPrice)(tickerSymbol)
    .viaMat(KillSwitches.single)(Keep.right)
    .map(tradeBotTradeCreator)
    .mapConcat(_.toList)
    .throttle(tradesPerDayLimit, 1.day, tradesPerDayLimit, ThrottleMode.shaping)
    .mapAsync(tradesPerDayLimit)( trade => makeTrade(trade).map(TradeResult(_, trade)) )
    .filter(_.success)
    .mapAsync(tradesPerDayLimit) ( tradeResult => tradeBotEmailCreator(tradeResult.trade) )
    .toMat(Sink.ignore)(Keep.left)
    .run()

  tradeBotKillSwitch = Some(newTradeBotKillSwitch)
}
```

There is A LOT going on in this function, lets break it down.

```scala
tradeBotKillSwitch.foreach(_.shutdown())
```

If there is an existing trade bot running we will stop it by using the stream's kill switch.

http://doc.akka.io/docs/akka/2.5.3/scala/stream/stream-dynamic.html

```scala
val tradeBotTradeCreator = createTrade(buyPrice, sellPrice)(_)
val tradeBotEmailCreator = sendTradeEmail(notificationEmailAddress)(_)
```

In these two lines we partially apply the `createTrade` and the `sendTradeEmail` methods with the knowledge we already
know for this trade bot, the buy price, sell price, and the notification email address.

```scala
val newTradeBotKillSwitch = getQuoteStreamForStock(buyPrice, sellPrice)(tickerSymbol)
```

Normally we would not have to pass buyPrice and sellPrice into this function but we need a basis price to start with
which is why I treated them as partial application. If we were using an actual streaming API we would only need the
ticker symbol as well as some other possible configurations.

The kill switch will be returned from actually calling `run()` to materialize our stream.

```scala
.viaMat(KillSwitches.single)(Keep.right)
```

This line of code sets up our kill switch into our materializer. If the kill switch is called any elements in the stream
below this line of code will finish through our flow but no new elements will enter.

```scala
.map(tradeBotTradeCreator)
```

The first action we want to take on our stream source is to Map the quote into a trade based off of the configured
trade bot's buy and sell prices.

```scala
.mapConcat(_.toList)
```

The stream has now been converted into a stream of `Option[Trade]`'s instead of a stream of `Quote`'s. What we really want
is a stream of `Trade`'s, we want to filter out the `None`'s from our stream. The way to achieve this is by using
`mapConcat` which is the equivalent of using `flatMap` since `Option[T]` can be treated like a `Seq`.

http://doc.akka.io/docs/akka/2.5.3/java/stream/stream-quickstart.html#flattening-sequences-in-streams

```scala
.throttle(tradesPerDayLimit, 1.day, tradesPerDayLimit, ThrottleMode.shaping)
```

This line of code is a bit easier to understand. We want to limit our stream of Trade's to only happen by our
`tradesPerDayLimit` and we will allow bursts of `tradesPerDayLimit` because we don't care how fast they happen, we only
care about how many are executed in a given day.

http://doc.akka.io/docs/akka/2.5.3/java/stream/stream-quickstart.html#time-based-processing

Now we know we have the trade's we want to make (the buy and sell's that are valid based on the configured trade bot),
and we also know that we will never execute more than the specified number per day because of the throttle.

It is time to make our trade!

```scala
.mapAsync(tradesPerDayLimit)( trade => makeTrade(trade).map(TradeResult(_, trade)) )
```

The trade is asynchronous which is why we call `.mapAsync`. We specify `tradesPerDayLimit` because that is the greatest
parallelism we are able to achieve. We map the result of the Trade into a TradeResult so that we keep the data about
the trade itself flowing through our stream.

```scala
.filter(_.success)
```

We only want to continue through our stream if the trade was successful.

```scala
.mapAsync(tradesPerDayLimit) ( tradeResult => tradeBotEmailCreator(tradeResult.trade) )
```

After any successful trades we will email their results, which is also an asynchronous call.

```scala
.toMat(Sink.ignore)(Keep.left)
.run()
```

These final 2 lines materialize our stream using a sink where we ignore the streaming elements, we want to keep the left
result of materializing our stream which is our kill switch. And then we actually run our materialized stream.

```scala
tradeBotKillSwitch = Some(newTradeBotKillSwitch)
```

The final bit of code saves off the kill switch for the newly executing trade bot.

Because of the way we setup our kill switch we can guarantee there will only ever be one trade bot executing at a time
and we can shutdown any trade bot and start running a different one.

The last function will be a simple fake API endpoint to stop any currently running trade bot.

```scala
def stopTradeBot(): Unit = {
  println("Stopping trade bot")
  tradeBotKillSwitch.foreach(_.shutdown())
}
```

To test using our new trade bot we can execute the following code and observe the trade bot in action.

```scala
startTradeBot("GOOGL", 10, 900, 950, "foobar@gmail.com")
Thread.sleep(5000)
startTradeBot("TSLA", 10, 330, 340, "foobar@gmail.com")
Thread.sleep(5000)
stopTradeBot()
startTradeBot("AAPL", 1, 163, 165, "foobar@gmail.com")
```

You can see the code in it's entirety at the bottom of this post.

I hope this toy example illuminates some of the power of Akka streams and the cool things you can do with them in a small
amount of code, (this example was under 100 lines without whitespace).

Cheers!

Aaron

<script src="https://gist.github.com/amast09/032949a2b8f106f7101b52784a1d59ba.js"></script>