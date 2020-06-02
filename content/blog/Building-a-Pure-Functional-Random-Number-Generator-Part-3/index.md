---
title: Building a Pure Functional Random Number Generator Part 3
date: "2020-06-02T02:03:15.418Z"
author: "edreyer"
tags:
- java
- functional prograrmming
---

This topic is broken out into a three part series, each building on the other.  This is Part 3.

 - Part 1 - [A  simple pure functional random number generator](/Building-a-Pure-Functional-Random-Number-Generator/)
 - Part 2 - [Finding the abstractions](/Building-a-Pure-Functional-Random-Number-Generator-Part-2/)
 - Part 3 - Discovering the State monad

### Monads are...
...a somewhat difficult concept to grasp at first. There are plenty of online resources devoted to explaining what this thing is.  Some do a better job than others. This being an article on practical application, we're going to steer away from the theory behind the monad.  

For our purpose, a monad is class that wraps a value.  The wrapped class also provides a context, via methods like `map()` that operate on the wrapped value from within the wrapping class.  These wrapping classes typically parameterize the type they wrap using generics. E.g.: `Optional<T>`, `Stream<T>`, `CompletableFutuer<T>`. These examples all monads provided by the JDK.

Each monad provides what is termed an "effect".  What is an "effect"?  That's probably explained best by a few examples.  The effect `Optional<T>` provides is "the effect of existence".  That is, it allows you to safely operate on a value regardless of whether that value exists or not.  The effect of `CompletableFuture<T>` is asynchronous computation.  At some point in the future it will provide you the value of the specified type.

There are many monads that exist in various FP libraries beyond those that the JDK provides.  Each provides a different effect.  It is more appropriate to say that monads are discovered rather than invented.  In fact, we've been building towards one during the course of this series.  

### The State Monad
The monad we are about to discover is called the State monad.  The State monad, like others, is parameterized to a type.  It's pretty easy to intuit the usefulness of a monad like `Optional`, but the `State` monad is somewhat harder to grasp at first.

Like other monads, the State monad wraps a value.  In this case, it's a bit more specialized.  The value it wraps is a function.  The shape of that function is:

`State -> (State', Value)`

Look familiar?  If not, please go back and re-read Part 2.  

In Part 2, we created our own specialization of `Function`.  We created type, called `StateFn`, whose signature matches this shape.  To make this useful, we also had to develop and write our own `map()` and `flatMap()` methods.  The State monad provides that for us.  And the good news is that we don't have to write our own State monad.  There are libraries that provides this for us.

A particularly robust functional library for Java is called [Lambda](https://palatable.github.io/lambda/). Among many other things, it provides a `State` monad.  

But what does the `State` monad actually do?  What is its effect?  In functional programming, functions are stateless.  But many operations required to model your business domain are inherently stateful.  The effect of the state monad is to provide a context for operating on stateful operations while remaining pure.  It's also really easy to use in that it frees you from having to worry about that State.

Starting once again with a simple pure function that generates random integers, how can we use the State monad to generate a series of random `Point`s?

*Lambda provides a drop-in replacement for java's `Function` that supports arities from 0 to 8 (`Fn0`, `Fn1`, ... `Fn8`).  These functions have a much richer API, compared to `Function`.  We will make use of those in our solution.*

Let's throw in a new wrinkle.  Let's modify our `State` object with a new field.  Not only will it carry the random `seed` but we'll add a field 

```java
// Boilerplate omitted
class Step3 {
    static class RndState {
        int seed;
        int rndCount;
    }
    static Fn1<RndState, Tuple2<Integer, RndState>> rndIntF = 
	    state -> {												1
	        Random r = new Random(state.getSeed());
	        int i = r.nextInt();
	        return tuple(
		        i, 
		        new RndState(i, state.rndCount + 1)				2
		    );			  
	    };
}
```

1. No need to create a special `StateFn` now.  Here we are using the primitives provided by the Lambda library to create our function state function.  Notice the signature conforms to: `State -> (Value, State)`
1. New twist, not only do we create a new State with a new seed, but we also increment the number of random integers we've created.

In order to setup our State monad, we just private a reference to this function to State's factory method

```java
import com.jnape.palatable.lambda.functor.builtin.State;
...
// S -> (V, S')
State<RndState, Integer> rndIntS = State.state(rndIntF);
```

The left side of the equals looks very similar to our state function (`StateFn<Seed, Integer>`).  This was intentional.  It shows how close we were to discovering the State monad.  The `State` monad wraps the state function.  Our solution, by comparison, used inheritance by extending `Function`.  We had to write our own supporting methods `map()` and `flatMap()`.  The `State` monad provides these for us.

Using our `rndIntS` state monad reference, let's transform that into one that produces `Point`s.

```java
State<RndState, Point> rndPointS =
	rndIntS.flatMap(x ->
	rndIntS.flatMap(y ->
	rndIntS.fmap(z -> new Point(x, y, z))));	1		
```
1. In Lambda, the `fmap()` function is the same as `map()` in Java's monadic types.  This was taken from Haskell.

Again, this should look familiar to what we did in Part 2.

### Point Generators

Let's see how to use this.  We're going to use our new State monad instance `rndPointS` plus a couple other utilities provided by Lambda to create two different `Point` generators.

```java
Iterable<Point> unlimitedPointGenerator = unfoldr(	1
    state -> just(rndPointS.run(state)),
    new RndState(0, 0)
);

Iterable<Point> limitedPointGenerator = unfoldr(
    state -> state.rndCount < 30 
	    ? just(rndPointS.run(state)) 
	    : nothing(),  								2
    new RndState(0, 0)								3
);
```

1. `unfoldr` -- or "unfold right" is a generator.  Given an initial value, and a way to create more values, from that starting point, it can be used to create any number of values.  In this case, we created an infinite generator of `Points`, expressed as an `Iterator`.
2. Unlike the infinite generator, this one is limited to 10 points.  We're using the State object's `rndCount` field to limit the number of points created.  Three integers per `Point` accounts for the limit of 30.
3. This is the initial state.  The `State` monad handles all the state propagation for you.

Another way to limit to 10 points is to use the `unlimitedPointGenerator` and just take the first 10.
```java
Iterable<Point> tenPoints = take(10, unlimitedPointGenerator);
```

Using each of these to generate points, we can print them out
```java
tenPoints.forEach(System.out::println);					
System.out.println("---------------");
limitedPointGenerator.forEach(System.out::println);		
```

Here's the full class for `Step3`, including imports
```java
import java.util.Random;

import com.jnape.palatable.lambda.adt.hlist.Tuple2;
import com.jnape.palatable.lambda.functions.Fn1;
import com.jnape.palatable.lambda.functor.builtin.State;

import static com.jnape.palatable.lambda.adt.Maybe.just;
import static com.jnape.palatable.lambda.adt.Maybe.nothing;
import static com.jnape.palatable.lambda.adt.hlist.HList.tuple;
import static com.jnape.palatable.lambda.functions.builtin.fn2.Take.take;
import static com.jnape.palatable.lambda.functions.builtin.fn2.Unfoldr.unfoldr;
import static com.jnape.palatable.lambda.functor.builtin.State.state;

static class Step3 {
    static class RndState {
        int seed;
        int rndCount;
    }

    static Fn1<RndState, Tuple2<Integer, RndState>> rndIntF = state -> {
        Random r = new Random(state.getSeed());
        int i = r.nextInt();
        return tuple(i, new RndState(i, state.rndCount + 1));
    };

    // S -> (V, S')
    static State<RndState, Integer> rndIntS = state(rndIntF);

    static class Point {
        private int x, y, z;
    }

    static State<RndState, Point> rndPointS =
        rndIntS.flatMap(x ->
        rndIntS.flatMap(y ->
        rndIntS.fmap(z -> new Point(x, y, z))));

    public static void main(String[] args) {
        Iterable<Point> unlimitedPointGenerator = unfoldr(
            state -> just(rndPointS.run(state)),
            new RndState(0, 0)
        );
        Iterable<Point> limitedPointGenerator = unfoldr(
            state -> state.rndCount < 30 
	            ? just(rndPointS.run(state)) 
	            : nothing(),
            new RndState(0, 0));

        Iterable<Point> tenPoints = take(10, unlimitedPointGenerator);

        tenPoints.forEach(System.out::println);
        System.out.println("---------------");
        limitedPointGenerator.forEach(System.out::println);
    }
}
```

### Wrapping up
This concludes our three part series. We started with a question, "How can we take an inherently stateful thing, like a random number generator, and create a pure functional version?"  From there, we generalized the solution, ultimately discovering the State monad.  A powerful tool in the FP toolbox.  

Our goal was to create an article on functional programming that moved past the trivial into something closer to real world use.  We steered clear of FP theory, and focused on practical and pragmatic examples.  The theory is important to understand, to a degree, but for those of us who learn by example, we hope this was worth your time.

Even though we took a step-by-step approach to arrive at our final point, don't worry if you're still scratching your head a bit.  If you come from an OOP background, these topics feel very foreign at first.  You might find that it will sink in on your second, or even third time through this series.

If you're particularly interested in grokking this material, we encourage you to crack open your IDE and attempt to get our examples working.  You could even add to them and create your own combinations.  We hope that you can find something here that you can use in your own work.

And thank you for reading.  We appreciate your time and interest.
