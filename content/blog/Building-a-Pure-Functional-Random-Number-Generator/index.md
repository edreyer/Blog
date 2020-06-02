---
title: Building a Pure Functional Random Number Generator
date: "2020-05-26T00:36:51.617Z"
author: "edreyer"
tags:
- java
- lambda
- functional programming
---

This topic is broken out into a three part series, each building on the other.  This is Part 1.

 - Part 1 - A  simple pure functional random number generator
 - Part 2 - [Finding the abstractions](/Building-a-Pure-Functional-Random-Number-Generator-Part-2/)
 - Part 3 - [Discovering the State monad](/Building-a-Pure-Functional-Random-Number-Generator-Part-3/)

## Part 1

### Motivation
Functional Programming is a paradigm that's been around for decades, but in recent years has quickly gained popularity. The reasons for this can be attributed to the need for engineers to create correct concurrent and parallel applications. 

For years, we got "free" performance gains just by waiting a year and running our software on new and much faster hardware. In recent years, the speed gains have become more incremental. Instead, we're given more CPU cores to work with. To take advantage of this, we must write concurrent and parallel applications.

Creating correct and bug-free multi-threaded applications is exceedingly difficult using traditional Java primitives. Books have been written detailing the myriad ways in which our valiant attempts at creating multi-threaded applications can go bad.

In [Java Concurrency In Practice](https://jcip.net/), one statement is oft repeated. To paraphrase, "*problem X goes away when using immutable data*". It turns out the use of shared mutable state is one primary driver for what makes writing correct multi-threaded applications so hard. When all state data is immutable, entire classes of problems just go away.

The use of immutable data is one of the core tenets of Functional Programming (FP). Functional programs are easier to test and to prove correct. It turns out functional programs can easily be parallelized, largely due to this aspect. 

Having practiced OOP for many years, my experience in trying to grok FP to be a long and sometimes frustrating experience. I've found online resources to be either trivial (e.g. *"A pure function is..."*) or opaque (e.g. *"Tagless Final"?*). Essentially, it felt like being given a whole new toolbox, with a whole new set of tools, some of which I might understand to a degree, but none of which I know how or where to use.

The goal here is to start simply enough with a question. How can I create pure functional random number generator (RNG)? From this starting point, we can methodically create an abstraction of that concept and in the process discover something called the State Monad. More importantly, by the end of this process we should have a decent understanding of what that is, and how to use it. 

We're going to take a step-by-step approach.  We will move pretty quickly past the trivial into slightly more advanced topics.  The goal is not only introduce new concepts but provide enough material so that you, dear reader, have some understanding of how and when you might use some of these concepts. Let's get started. 

### A Basic Functional Random Number Generator 
Our RNG will initially be implemented as a simple method on a class. We're not going to actually create a random number generator from scratch, but we'll make use of java's `java.util.Random`.  This will just be an implementation detail of our method.

Starting from first principals, in FP parlance, a **pure function** is simply a function that given some input **X**, will **always** return the same output **Y**. No exceptions. Literally, no `Exception`s. Here is an example of a very basic pure function in java.

```java
int addOne(int x) {
    return x + 1;
}
```

Regardless of what value for `x` you pass, you will always get back an incremented value (apart from `x == Integer.MAX_VALUE`, but even with that you always will get the same result).

Consider java's `java.util.Random`.  It doesn't need an input to produce a value. Not only that, it mutates its internal state each time a random value is produced. :

```java
Random rand = new Random();    1
int randomInt = rand.nextInt() 2                  
```
1 -- No data required to construct an instance

2 -- No input required to generate the next random integer.

Yet, it's still possible to use `Random` to produce a pure functional random number generating function. One that, given an input, will produce the same output.  

Here's how. Consider the following.

```java
// all boilerplate omitted
class Step1 {
    static class Seed {
        Integer seed;
    }
	static int randomInteger(Seed seed) {
        Random rand = new Random(seed.seed);  1
        return rand.nextInt();				  2
    }
}
```
1 -- We've create a `Seed` class that just wraps an Integer value. We use this value to seed our `Random` instance. The ability to seed our random is the key to this whole thing. It allows us to get the same output for a given input. This works because `Random` is really a [pseudo-random generator](https://docs.oracle.com/javase/8/docs/api/java/util/Random.html).  

2 -- Using our seeded `Random` generate a "predictable" random number. It's predictable in that for a given seed value, we will always get the same result.

Here's how we can use our new function to generate a series of random numbers:

```java
class Step1 {
//...
    public static void main(String ...args) {
        int r1 = randomInteger(new Seed(0));
        int r2 = randomInteger(new Seed(r1);
        int r3 = randomInteger(new Seed(r2);
        Arrays.asList(r1, r2, r3).forEach(System.out::println);
    }
}
```

This application will always generate the same three numbers every time it's run because we always use the same starting seed.  The `randomInteger` method is a pure, in accordance with the rules of FP.  But it's still a method on a class.  In FP, functions are values.  How can we convert the class method into a function value? Let's do that now.

### Sprinkling in some FP
We're going to make a few surgical changes that might not make sense right away, but will set us up for what's to come.

First, I'm going to change the return type of our `randomInteger()` method from `int` to the following `RandomResult` class:
```java
class Step1 {
//...
	static class RandomResult {
	    Seed seed;				1
	    Integer rand;			2
	}
}
```
This type is a tuple of two values:
1 -- `seed` is a new seed value, to be used as input when we want to acquire a new random integer.
2 -- `rand` is our generated random integer

Here is our method updated with this new type:
```java
static RandomResult randomInteger(Seed seed) {
    Random rand = new Random(seed.seed);
    int x = rand.nextInt();
    return new RandomResult(x, new Seed(x));   1
}
```
1 -- We use the generated value `x` as both the random result, and as the basis for our new `Seed` instance.

If you're scratching your head on why we would do this, we'll expand on this in Part 2, but for now think of `rand` as the desired value, and `seed` as an updated container for application state. In this rather simple use case, they can just be the same value. 

### Method to Function
Above, we essentially have an OOP approach to creating a stateless RNG. It's a stateless method.  It's stateless because the state is passed in on each invocation, and an updated state is returned in the result. Even though we use a stateful object internally, it's discarded on each invocation.

Let's convert this class method to a function value. We can do that with the following modifications to our `Step1` class:

```java
class Step1 {
//...
	interface RandomIntFn 
		extends Function<Seed, RandomResult> {} 1

    static final RandomIntFn randomIntFn = 
	    Step1::randomInteger;  					2
}
```
1 -- Yes, you can extend `Function`. By doing that here, we're creating a shorter name. It's like an alias to the longer  `Function<Seed, RandomResult>`.

2 -- `randomIntFn` is a variable, whose type is a function. We can assign the class method `Step1.randomInteger` to our variable `randomIntFn` because its signature matches that of our `RandomIntFn` type. 

What we've done is to convert a class method to a value. This value happens to be a function type, but can be treated like any other value, e.g. `String`, or `Integer`. It can be passed around like a value. It can be provided as input to or returned from other methods/functions. This forms a basis for function composition.

Here is the entire `Step1` class, with an updated `main()` method, using our new function value instead of the class method.

```java
// All boilerplate omitted
class Step1 {
    static class Seed {
        Integer seed;
    }
    static class RandomResult {
	    Seed seed;				
	    Integer rand;			
	}
	static RandomResult randomInteger(Seed seed) {
	    Random rand = new Random(seed.seed);
	    int x = rand.nextInt();
	    return new RandomResult(new Seed(x), x);   
	}
	
	interface RandomIntFn extends Function<Seed, RandomResult> {} 
    static final RandomIntFn randomIntFn = Step1::randomInteger;  
    
    public static void main(String ...args) {
        RandomResult r1 = randomIntFn.apply(new Seed(0));
        RandomResult r2 = randomIntFn.apply(r1.seed);
        RandomResult r3 = randomIntFn.apply(r2.seed);
        Arrays.asList(r1.rand, r2.rand,r3.rand)
	        .forEach(System.out::println);
    }        	
}
```

In Part 2, we will build on this by finding the abstractions in this simple, but concrete solution.
