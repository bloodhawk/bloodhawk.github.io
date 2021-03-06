A functor is an object that implements a `map` method. A monad is a special type of functor that implements a `flatMap` method.

At first glance, this doesn't seem very interesting. However, let's go over some examples of monads in Scala that can completely change how you think about programming. Introducing the `Try` and `Option` monads.

From the documentation

> The Try type represents a computation that may either result in an exception or return a successfully computed value.
> Option represents optional values. Instances of Option are either an instance of Some or the object None. The most idiomatic way to use an Option instance is to treat it as a collection or monad and use map,flatMap, filter, or foreach

Let's look at the code below to discuss these monads

```scala
class AnalyticsServlet extends ScalatraServlet with DBSessionSupport {
  val MILLI_IN_HALF_HOUR = 1800000L
  val logger: Logger = LoggerFactory.getLogger(getClass)

  private def parseLong(s: String): Option[Long] = {
    val parsedValue = Try { Some(s.toLong) }
    parsedValue.getOrElse(None)
  }

  private def getMilli(t: Option[String]): Option[Long] = {
    t.flatMap(parseLong)
      .filter(x => x > MILLI_IN_HALF_HOUR && x < System.currentTimeMillis())
  }

  get("/analytics") {

    val milli = getMilli(params.get("timestamp"))
    milli.getOrElse(halt(400, "You must provide a valid timestamp"))

    val analyticAggResult = milli
      .map(x => (new Timestamp(x - MILLI_IN_HALF_HOUR), new Timestamp(x + MILLI_IN_HALF_HOUR)))
      .map(x => AnalyticDAO.getBetween(x._1, x._2))
      .get

    analyticAggResult.getOrElse({
      logger.error(analyticAggResult.toString)
      InternalServerError()
    })
  }
  ...
```

Here are some simple methods implementing a pretty standard REST endpoint to get at some analytics for the hour.
You could call this doing: `GET /analytics?timestamp=1556386553`

The first thing to understand is that `params.get` method returns an `Option` type which can either be of `Some(value)` or `None`. `Some` and `None` are also monads as they implement `.map` and `.flatMap`. They are in essence child monads of the parent `Option` monad. This hints at the power of how polymorphism is not lost when switching over to the usage of monads.

What is interesting about `Some` and `None` and any monad for that matter is that we can chain operations and essentially abort halfway through the chain without problems. For example, lets say I pass in a timestamp that is not an integer. What do you think happens? Give it some thought before moving on.

Well, our `getMilli` method controls this. `getMilli` first checks that a `timestamp` is a proper `long` by making use of the `parseLong` and `filter` methods on our `Option` monad. If the `filter` method predicate does not pass (returns false), it will return a `None` rather than a `Some(value)`. What is interesting about this is that the `None` implements `.map`, `.flatMap` etc (because it is a monad). For None because there is no wrapped value they serve more as identity functions than anything else. If `getMilli` returns a `None` then the `map` calls will do nothing as they have no value to unpack and operate on. If it was a `Some(value)` then `map` would need to unpack the value, run the function over it, and then repack the value to be returned. Another special function is `getOrElse`. `getOrElse` operates on the context of the functor. Is the functor `None` or `Some`? If `None` we need to return the expression passed into the method, if it is `Some`, we should return the value wrapped inside the `Some`.

The same concepts above are used with `Try`. A `Try` can be a `Success` or a `Failure`. `Success` is analogous to `Some`, and `Failure` is analogous to `None`. This allows you to think about success and failures as simply data rather than typical exceptions. They also operate in the same way as `Option` does. In fact, you can convert a `Try` to an `Option` with `toOption` method if you want to treat it as such.

You may be asking yourself now. What is the purpose of `flatMap` then? Well, what happens if my `map` returns a `Some` or a `None`? I don't want a container inside a container. That is the job of `flatMap` -- to flatten the container down from something like `Some(Some(value))` to `Some(value)`. Or, a more horrible situation a `Some(None)` which seems like a bad situation waiting to happen. `flatMap` would take `Some(None)` down to a `None`.

With these two simple monads, we have enabled the creation of powerful control flows and got rid of the exception concept. Namely, we can now easily prevent the infamous `NullPointerException`. Play with these concepts. They are truly powerful and will change how you think about programming.
