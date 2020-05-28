---
title: Building a Pure Functional Random Number Generator Part 2
date: "2020-05-28T00:03:19.248Z"
author: "edreyer"
tags:
- java
- functional programming

## Part 2

This topic is broken out into a three part series, each building on the other.  This is Part 2.

 - Part 1 - [A  simple pure functional random number generator](/Building-a-Pure-Functional-Random-Number-Generator/)
 - Part 2 - Finding the abstractions
 - Part 3 - Discovering the State monad (TBD)

### Finding the Abstractions
In [Part 1](/Building-a-Pure-Functional-Random-Number-Generator/), we created a simple pure functional random number generator. Let's build on this by finding some hidden abstractions in that implementation.  We will find and extract those to recreate our functional RNG using a more generalized approach.

When we introduced our `RandomResult` class, we briefly mentioned that it was a tuple of a desired result and a state object. Let's create an abstraction for that tuple and redefine our RNG function to use that abstraction.  We're renaming `RandomResult` to `StateResult` and parameterizing the types of the `state` and `value` fields.

```java
class Step2 {
	static class StateResult<STATE, VALUE> {					1
		STATE state;		
		VALUE value;		
	}
	interface StateFn<STATE, VALUE> 
		extends Function<STATE, StateResult<STATE, VALUE>> {}  	2
		
    static class Seed {
        Integer seed;
    }

	static StateFn<Seed, Integer> randIntFn = seed -> {			3
        Random rand = new Random(seed.seed);
        int x = rand.nextInt();
        return new StateResult(new Seed(x), x);
    };
}
```

1. `StateResult` is essentially the same class as our `RandomResult` from `Step`, but we've parameterized the types. `VALUE` for the output, and `STATE` for some object of state. We can now create instances of the output for any type.

1. We've done the same here for the signature of our RNG function. `StateFn` is the same signature as `Step1.RandomIntFn` but with parameterized types. 

1. Going straight to lambdas, we've recreated our RNG function `randIntFn` using our newly minted parameterized types. We're still using a `Seed` class, but now that type is bound to the `STATE` slot of `StateResult`.

Essentially, our random number generating function conforms to the following shape (signature):

`State -> (State', Value)`

Given some **State** object, calculate a **Value**, and return a tuple of both **State prime** and the calculated **Value**. In our use case, the state object, is essentially an integer, but it could be more complex.  We created a new type called `StateFn` that formalizes this shape into a named function type.

But what does this abstraction buy us? Let's say that in addition to random integers, we also want random doubles, or booleans, or even something a bit more complex, like random points in a 3-dimensional space? How would you go about that? Using OOP, you might override the method. You might copy and paste the original method that produces integers, change the signature to match your desired type, then hack the code until it suits your needs.

We have a more powerful way to accomplish this than copy-paste-hack. 

#### Mapping Over Data
In FP, many constructs allow for mapping over data. That is, they provide you a `map()` method that takes as input a mapping function.  This mapping function accepts an argument of one type and transforms it into another. In FP parlance, anything that provides this capability is a `Functor`. The signatures of these methods are always the same. For example, here is Java's `Optional.map()`

```java
<U> Optional<U> map(Function<? super T, ? extends U> mapper)
```

Here is `Stream`'s
```java
<U> Stream<U> map(Function<? super T, ? extends U> mapper)
```

Change `Optional`'s return type from `Optional<U>` to `Stream<U>` and you have the signature for `Stream.map()`.  The signature remains the same for every type that supports `map()`.  This is true for any `Functor`.

The less than obvious benefit of this is that anytime you see a `map()` method, you can instantly know what it does, regardless of what type it's attached to.

We can leverage `map()` to turn a random integer generator into something that generates random other things.  How?  Let's take another look at our `StateFn` type.

```java
interface StateFn<STATE, VALUE> 
	extends Function<STATE, StateResult<STATE, VALUE>> {}
```

When we `apply()` a `StateFn`, we get back an instance of `StateResult<STATE, VALUE>` where `VALUE` for us has been bound to `Integer`. If we want to generate random booleans, we can start with the Integer version of our random number generator and use `map()` to transform the integer into a boolean.  It would look like this.

```java
StateFn<Seed, Boolean> randBooleanFn = randIntFn
	.map(i -> i % 2 == 0);
```

Isn't this much cleaner, and much more concise than any OOP approach we might have used?  Also, note that nothing actually happens here.  This is just saying, "take my function that produces some type A, and change it into something that produces some type B".  Nothing actually happens until we call `apply()` on our function.

Because `randIntFn` is an instance of `StateFn`, we have to add our `map()` method to that interface.  

We're not going to spend time explaining the `map()` implementation because as we'll see in Part 3, we won't need to write our own version. We include it here so you can see the steps in our how we get from Part 1 through Part 3 of this series.

```java
interface StateFn<STATE, VALUE> 
	extends Function<STATE, StateResult<STATE, VALUE>> {
	
    default <V2> StateFn<STATE, V2> map(
	    Function<? super VALUE, ? extends V2> f
	) {
        return state -> {
            StateResult<STATE, VALUE> result = apply(state); 1
            return new StateResult<>(
	            result.state, 
	            f.apply(result.value)						 2
	        );
        };
    }
}
```
1. We invoke `apply()` our function to get our original result.

1. We build a new `StateResult` by taking the original `result` and applying our mapping function `f()` to transform our `VALUE` into a `V2`.

A couple additional points of interest about this:

- We are adding a function to a `Function`.  That might seem weird, until you realize that `Function` is just like any other interface, and in fact already comes with a few methods of its own.  More precisely, we're adding a new method `map()` to our interface `StateFn`.
- Because our `StateFn` is itself a function type, `map()` must also return a function.

### Function Composition
It's pretty trivial to `map()` our original random integer generator to a boolean generator, as we've shown above.  But what if we want something slightly more complex, like a 3D point in space?  Let's define a `Point` class.

```java
// boilerplate omitted
class Point {
   private int x, y, z;
}
```

We need to generate three random integers to create a `Point`.  We can use function composition to transform our integer generator into a point generator.  To do so, we're going to need to add one more method to our `StateFn` type.  We already added `map()`, but now we're going to need `flatMap()` as well.

`flatMap()` is just the combination of two operations, `flatten()` and `map()`. What `flatten()` does is unwrap, by one level, a doubly nested structure. For example, if you had an instance of an object of type `Optional<Optional<Integer>>`, after `flatten()` you'd just have `Optional<Integer>>`, which is what you really want. If ever you use a `map()` operation, and you end up with a doubly nested type, you probably want `flatMap()` instead.

Once we have this method, this is how we use it.

```java
StateFn<Seed, Point> randPoint =
    randIntFn.flatMap(x ->		1
    randIntFn.flatMap(y ->
    randIntFn.map(z ->			2
        new Point(x, y, z)		3
    )));

Point point = randPoint.apply(new Seed(0.0)).value;	4
```
1. Each `flatMap()` and `map()` invocation is nested inside other lambdas functions.  We aligned them vertically to make it easier to read.
1. The innermost lambda must use `map()` instead of `flatMap()` to get the correct type
1. Inside these nested lambdas, we can [close over](https://riptutorial.com/java/example/14441/java-closures-with-lambda-expressions-) the values `x`, `y`, and `z` to create a new `Point`. 
1. Creating a new point is as simple as invoking `randPoint.apply()` with a seed.

There you have it.  We used function composition to transform a simple function (several times) into a more complex one.

The `flatMap()` method, for our purposes looks similar to our `map()` method, but is just a bit more complex.  Again, we won't explain the `flatMap()` implementation because in Step 3 we'll show how you don't need to write your own.  It's useful to see the all the steps, so here it is.

```java
interface StateFn<STATE, VALUE> 
	extends Function<STATE, StateResult<STATE, VALUE>> {
	//...
	default <V2> StateFn<STATE, V2> flatMap(
		Function<? super VALUE, ? extends StateFn<STATE, V2>> f
	) {
	    return state -> {
	        StateResult<STATE, VALUE> result = apply(state);
	        return f.apply(result.value).apply(result.state);
	    };
	}
}
```

For completeness, here's the entire `Step2` class.  
```java
static class Step2 {
    static class Seed {
        Double seed;
    }

    static class StateResult<STATE, VALUE> {
        STATE state;
        VALUE value;
    }

    static class Point {
        private int x, y, z;
    }

    // STATE -> (VALUE, STATE)
    interface StateFn<STATE, VALUE> 
	    extends Function<STATE, StateResult<STATE, VALUE>> {

        default <V2> StateFn<STATE, V2> map(
	        Function<? super VALUE, ? extends V2> f
	    ) {
            return state -> {
                StateResult<STATE, VALUE> result = apply(state);
                return new StateResult<>(
	                result.state, 
	                f.apply(result.value)
	            );
            };
        }

        default <V2> StateFn<STATE, V2> flatMap(
	        Function<? super VALUE, ? extends StateFn<STATE, V2>> f
	    ) {
            return state -> {
                StateResult<STATE, VALUE> result = apply(state);
                return f.apply(result.value).apply(result.state);
            };
        }

    }

    static StateFn<Seed, Double> randDoubleFn = seed -> {
        Random rand = new Random(seed.seed.intValue());
        double x = Integer.MAX_VALUE * rand.nextDouble();
        return new StateResult<>(new Seed(x), x);
    };

    public static void main(String[] args) {
        StateFn<Seed, Integer> randIntFn   = 
	        randDoubleFn.map(Double::intValue);
	        
        StateFn<Seed, Boolean> randBoolean = 
	        randIntFn.map(i -> i % 2 == 0);

        StateResult<Seed, Integer> r1 = randIntFn.apply(new Seed(0.0));
        StateResult<Seed, Integer> r2 = randIntFn.apply(r1.state);
        StateResult<Seed, Integer> r3 = randIntFn.apply(r2.state);

        asList(r1, r2, r3).forEach(r -> System.out.println(r.value));

        StateFn<Seed, Point> randPoint =
            randIntFn.flatMap(x ->
            randIntFn.flatMap(y ->
            randIntFn.map(z ->
                new Point(x, y, z)
            )));

        Point point = randPoint.apply(new Seed(0.0)).value;
        System.out.println(point);
    }
}
```

### Summing up Part 2

In Part 2,  we discovered an abstraction for a state function with the shape `State -> (State', Value)` and formalized that as a type `StateFn`.

We learned how to create  instances of `StateFn` to perform a specific operation, like generating random integers.

We learned how to leverage `map()` and `flatMap()` methods to transform our state function to produce a value of a different type using a functional approach.  Traditional OOP methods would use method overloading and/or overriding, with a lot of copy-paste.

Finally, consider this an intermediate step that bridges a gap between the concrete function we developed in Part 1, and the generalized utility we discover in Part 3.

In Part 3, we will continue building our example to its conclusion, a powerful utility called the State monad.  Our goal is to leave you with a clear understanding of what this is and how you might use it in your own applications.

