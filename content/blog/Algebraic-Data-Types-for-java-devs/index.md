---
title: "Algebraic Data Types - Intro for java devs"
date: "2020-04-14T12:00:00.000Z"
author: "edreyer"
tags:
- blog
- java
- ADT
---

## What’s the problem?

If you’ve ever worked on a java project that involved a database, odds are you used an ORM like Hibernate or JPA. Despite the promise that these systems solve the impedance mismatch problem between the two-dimensional structure of database tables to the richer object graphs of a domain model, it's likely your mapped domain objects pretty closely mirror the structure of the database.

A hidden consequence typical of designs like this is that the structure of the database leaks back into the structure of the domain, influencing the design of your entire application. The entities probably map to a table on a field by field basis. We usually end up with one or more java object graphs that are nothing more than fancy structs for holding data in what has come to be known as an [anemic domain](https://martinfowler.com/bliki/AnemicDomainModel.html). As I will outline here, this is a primary source of bugs in our applications.

Entities often have various states specific to your business problem that are modeled using enums or boolean flags. Here is where our first issue arises. It is common that an entity in a particular state is often accompanied by various other fields of data specific to that state. An entity may go through a series of states as it moves through a workflow, with each state contributing its own particular set of data. Although a class may eventually have all its fields populated, many class fields may be null until various class states come into play. Other fields may never be populated if a class goes through a workflow that involves branching logic. This aspect of how we model our domains is one of the single largest sources of bugs in our applications. How?

Our applications tend to bundle business logic into stateless service classes. This service layer is where the business logic of our application resides. Entities passed as arguments to service methods must be checked to ensure they are in a state applicable to the method. That is, our service methods tend to have various checks to enforce the invariants of our domain.  Is an entity even in a valid state for a given method? It is incumbent on the developer to check. What do we do with fields if they are null? As java engineers, dealing with nulls is always part of the mental process when writing code.  And even so, don’t we still too frequently fall prey to the NullPointerException?

It is incredibly easy to forget to check business invariants. In complex domains, the engineer may not even be aware of invariants that must be enforced. Requirements change over time. As new states and fields are added to an entity, the methods that act on that entity may require updating. The invariants have changed. Again, it is incumbent on the engineer to find and address these issues using the search tools provided by their IDE and reading the code at each point to see if new business rules apply. If they are lucky, there may be a unit test which surfaces an issue arising from the above. In my experience nearly all bugs in a java application are a direct result of these issues. They are usually discovered at runtime, hopefully before they roll to production. 

Algebraic Data Types are an effective solution to these issues.

## Algebraic what?
Algebraic Data Types (or ADTs) are an alternative to traditional OOP methods for defining and encoding a domain.  In contrast to OOP, which traditionally values inheritance and encapsulation as core tenets, ADTs explicitly expose the structure of the domain objects.

*Sidebar: In practice, we don’t really practice encapsulation with our anemic domain entities anyway, do we?  It’s standard procedure to litter our service methods with constructs such as `entity.getFoo().getBar().getBaz()`. There’s no business logic to encapsulate, and we readily expose the inner structure.*

As stated, when it comes to designing your domain model, we tend to follow an OOP approach, usually influenced by the structure of our database.  ML based languages like OCaml and Haskell use a different approach.  The default mechanism for those languages is to describe a domain by composing complex structures from simpler ones.

This is done via the use of ADTs.  ADTs come in two flavors, **Product** and **Sum** types.  The term “algebraic” comes from the fact that types are created by taking the “product” or “sum” of other existing types.  This being a practical guide, we will steer clear of the theory and focus on the practical benefit.  ADTs are typically, and should be, immutable.  They contain methods that operate on their internal data.  Ideally, you’ll create ADT factory methods that prevent instantiation unless the invariants of that ADT are satisfied.

### Product Types
You are already familiar with Product types. They are just immutable POJOS. You can think of a product type as an AND type. A `Person` class has a `firstName` AND a `lastName`. Product types are composed of basic types (`String`, `Integer`, `List`) and other Product and Sum types.  

In Java 14, the product type has been incorporated into the language as a new “kind” of class called a `record`. Oracle has a nice intro to records on [their blog](https://blogs.oracle.com/javamagazine/records-come-to-java).  If you’re stuck on Java 11, or even Java 8, don’t worry we’ve got you covered.  More on this below.

For the purposes of this intro, there isn’t much more to say about Product types.  Sum types, however, are far more interesting.

### Sum Types
To a certain extent you are already familiar with Sum types as well.  Sum types can be thought of as OR types.  Java enums are a great example.  They represent the simplest of sum types.  Consider a `DayOfWeek` enum.  When you have an instance of that, you have either a `Monday` OR `Tuesday`, but not both.  Enums can carry data and methods.  

One potentially new way to think about enums, is that they are a mechanism that allows you to create a new type, and enforce that only a fixed set of instances of that type can be created.  You cannot programmatically add new instances to an enum.  For example, there’s no way to programmatically add a `FoosDay` to our `DayOfWeek` enum.

Sum types can also be stateful.  While it never makes sense to have more than one instance of `DayOfWeek.Monday` (a singleton), sum types can be created with state.  Obviously, you can't use `enum` to model stateful Sum types.

An example will do well to illustrate this.  For these examples we're going to use Scala, which has first class support for ADTs.  In fact, nearly all modern languages have some level of language level support.  Even java is going to get sum types, currently targeted for [Java 15](https://openjdk.java.net/jeps/360).

```scala
sealed trait Listing

object Listing {
  final case class DealerListing private (dealer: Dealer, data: YearMakeModel) extends Listing
  final case class PersonListing private (seller: Person, data: YearMakeModel) extends Listing  
  
  object DealerListing {
    // factory with validation
    def create(dealer: Dealer, data: YearMakeModel): Either<ValidationError, DealerListing> =
      if (!valid(dealer, data)) Either.left(ValidationError("Cannot create DealerListing due to ...")) 
      else Either.right(DealerListing(dealer, data))
    
    def valid(...): Boolean = { ... }
  }
  
  object PersonListing {
    // factory with validation
    def create(person: Person, data: YearMakeModel): Either<ValidationError, PersonListing> =
      if (!valid(person, data)) Either.left(ValidationError("Cannot create PersonListing due to ...")) 
      else Either.right(PersonListing(person, data))
    
    def valid(...): Boolean = { ... }
  }
  
}

```

No need to understand Scala.  I'll highlight the important bits here:

 - We've created the notion of a `Listing`.
 - Unlike in Java, where any public interface can be implemented by any class anywhere, we've restricted subtypes of `Listing` to `DealerListing` and `PersonListing`.
 - We've made the class constructors private, in favor of "static" `create()` factory methods.
 - The factory methods enforce that legal values are supplied. 
 - The return type of our create methods are a disjoint union type (`Either<E, V>`) containing either the desired and fully constructed listing type or a validation error.  (This point alone is worthy of its own blog and may follow this one).
 
 These are the facts of the code.  What are the benefits? For starters, because we enforce the business invariants at time of construction, any business logic can safely assume it's operating on valid data.  But why restrict the subtypes?  This is where Sum types really come into power.  We have to introduce two new concepts, **exhaustive pattern matching** and **destructuring**.

### Exhaustive Pattern Matching
I think it's safe to assume we would all prefer the compiler notify us about problems with our code than to find out about issues at runtime.  Consider the following, a style that is typical in java applications.  

```java
class Listing {
    private YearMakeModel data;
    // boilerplate omitted
}

class DealerListing extends Listing {
    private Dealer dealer;
}

class PersonListing extends Listing {
    private Person person;
}

class ListingService {
    public String listingSellerName(listing Listing) {
	    String result = "";
		if (listing instanceof DealerListing) {
			result = ((DealerListing)listing).getDealer().getDealershipName()
		} else if (listing instanceof PersonListing) {
			result = ((PersonListing)listing).getPerson().getFullName()
		}
		return result;
    }
}
```
What happens if we were to add a new type of Listing?  Let's say `EnterpriseListing`.  We can add the new `EnterpriseListing` class and successfully compile our project.  Our next task, if we're diligent, would be to search for all usages of `Listing` to see if business logic needs to be updated to accommodate our new `EnterpriseListing`.   We'd have to carefully read the code at each search result to see if the logic requires modification.

Wouldn't it be better if, just by adding the new type, all the touch points no longer compiled?  No more searching and reading code.  You'd have instant feedback on what code required updating.  

Scala has a feature called pattern matching.  Pattern matching is exhaustive.  That is, if the developer doesn't provide cases for all possible match combinations, the code won't compile.  The following code shows an example of this.  Here we rewrite the `ListingService.listingSellerName()` method to use our ADTs.

```scala
def listingSellerName(listing: Listing): String = listing match {
    case DealerListing(dealer, _) => dealer.nameOfDealership()
    case PersonListing(person, _) => person.fullName()
  }
```

If you compare this to our java version above, I think you'll find it easier to read.  There's less cognitive load to understand this.  No `if-else` chain.  No `instanceof` checks.  It's devoid of any clutter.

And most importantly, as soon as we add a new `EnterpriseListing` type, this method will fail to compile, thanks to **exhaustive pattern matching**.  In my opinion, this is the single largest benefit of ADTs. We move potential runtime problems to compile time, giving us instant feedback on where our code needs to be updated to accommodate the new type.

You might notice something else about this code.  Look at each `case` statement.  The pattern matcher will match a particular case if the `listing` being matched matches the type.  But what about what looks like an argument in the parenthesis after the class name?  This brings us to...

### Destructuring / Decomposition


Let's look at a single line

```scala
case DealerListing(dealer, _) => dealer.nameOfDealership()
```

The `dealer` variable is actually being defined here, and is available for use after the  arrow (`=>`).  What's happening here is that a `DealerListing` are being decomposed back into the individual arguments used to construct the object in the first place.  In the case, we used an underscore (`_`) for the second argument, representing the `YearMakeModel` slot, because we don't need it in our function.

Remember that one aspect of ADTs is that the structure is made explicit.  Here is how that shows up.  While doing exhaustive pattern matching, we destructure our types into their component parts so that we can act on those parts.  This combination turns out to be a very powerful concept.  It leads to compile time safety.  It leads to concise, efficient, and readable code.  It leads to fewer bugs in production.  It leads to fewer interrupted nights of sleep.

### Awesome!  But how do we do this in Java?
One way is to wait for the next LTS release of Java. That will be Java 17. By then, the java `record` and sealed classes should be out of preview.  You will have to wait for over a year and a half for that.

Or, if you can't use scala (or Haskell, Kotlin, F#, Swift, etc) and you want to use ADTs right now, you can use a nice java library called [Derive4J](https://github.com/derive4j/derive4j).

*sidebar: Why is Oracle adding these features to Java anyway? Nearly all modern languages have some level of first class support for ADTs.  They have their root in functional programming, and again, nearly all modern languages are adding some level of first class support for functional programming.  Over time, the concepts have proven their value and the Java architects agree that their application tends towards more correct applications*

Derive4j uses an annotation processor on specially marked classes and interfaces to generate classes that conform to all the concepts described in this article.

The first step, is to create an annotation interface, that's configured with some reasonable defaults for your project.  Perhaps something like this.

```java
@Data(
    arguments = ArgOption.checkedNotNull,
    value = @Derive(
        inClass = "{ClassName}ADT"
    )
)
public @interface ADT { }
```
Our `ADT` interface says that our generated class names will end in "ADT" and we will enforce that any arguments used to construct our ADTs will not be null.

Using this, we can define our `Listing` ADTs

```java
@ADT
public interface Listing {
	interface Cases<R> {
		R dealerListing(DealerListing dealerListing);
		R personListing(PersonListing personListing);
	}
	
	<R> R match(Cases<R> cases);
}

@ADT
public interface DealerListing {
	interface Cases<R> {
		R create(
			Dealer dealer,
			YearMakeModel data
		);
	}
	<R> R match(Cases<R> cases);
}

@ADT
public interface PersonListing {
	interface Cases<R> {
		R create(
			Person person,
			YearMakeModel data
		);
	}
	<R> R match(Cases<R> cases);
}
```

 - `Listing` is a Sum type.  The annotation processor will generate a `ListingADT` class.
 - `DealerListing` is Product type.  We still use Derive4J to generate this class to ensure all constructor arguments are null checked and that the class is immutable. Same for `PersonListing`

Ok, but how do you use this?

#### Creating instances

```java
public Listing createDealerListing(Dealer dealer, data YearMakeModel) {
	DealerListing dealerListing = DealerListingADT.create(dealer, data);
	return ListingADT.dealerListing(dealerListing);
}
```

We are guaranteed  that our Listing is fully populated with no null data.  

What if we had a business rule that said only create listings for active dealers? We could modify our `DealerListing` type to add a static factory method and use that to create our instance

```java
@ADT
public abstract class DealerListing {
	DealerListing() {} // protected
	
	interface Cases<R> {
		R create(
			Dealer dealer,
			YearMakeModel data 
		);
	}
	<R> R match(Cases<R> cases);

	@ExportAsPublic
	static Either<ValidationError, DealerListing> validateAndCreate(
		Dealer dealer,
		YearMakeModel data) {
		if (!dealer.isActive() {
			return Either.left(new ValidationError("Dealer is not active");
		}
		return Either.right(DealerListingADT.create(dealer, data));
	}
}
```

Here we are enforcing business invariants in our type system.  We can't even construct a `DealerListing` unless all our business rules are satisfied.  Thus, anytime anywhere we are dealing with a `DealerListing` we know it's internally consistent and safe to use.  Neither do we ever have to worry about `null`.

#### Using Instances

Let's rewrite our `ListingService.listingSellerName` method using our java version of our `Listing` ADTs created using Derive4J

```java
public String listingSellerName(Listing listing) {
	return ListingADT.caseOf(listing)
		.dealerListing((dealer, data) -> dealer.nameOfDealership())
		.personListing((person, data) -> person.fullName());
}
```

This is nearly as clean and concise as the scala version!  And just like with scala, if we add a new `EnterpriseListing`, this will no longer compile thanks to exhaustive pattern matching.  Finally, notice we have class destructuring in each lambda, corresponding to each "case".  

Just for sake of comparison, here's our original java method, using a more traditional approach.

```java
    public String listingSellerName(listing Listing) {
	    String result = "";
		if (listing instanceof DealerListing) {
			result = ((DealerListing)listing).getDealer().getDealershipName()
		} else if (listing instanceof PersonListing) {
			result = ((PersonListing)listing).getPerson().getFullName()
		}
		return result;
    }
```

In addition to compile time safety, doesn't the ADT version require less cognitive load to read and understand?  Yet another benefit.

## Wrapping up

In this introductory article, we've introduced the notion of Algebraic Data Types for java developers.  We assumed no prior knowledge on the topic.  We attempted to steer clear of the theory behind these concepts, opting instead to focus on the practical benefits.

We illustrated how ADTs make a shift to using the type system itself to enforce business invariants (as opposed to fields of enums and boolean flags) and how this makes code more robust.

We attempted to illustrate the concepts of exhaustive pattern matching in combination with class destructuring and how they can be used to move errors that occur traditionally at runtime to compile time checks.

Finally, because we're just introducing the topic here, we chose to highlight only a couple benefits to using ADTs.  We hope you find this useful and would love your feedback.

#### references

 1. [https://jnkr.tech/blog/introduction-to-algebraic-data-types](https://jnkr.tech/blog/introduction-to-algebraic-data-types)
 2. [https://pragprog.com/book/swdddf/domain-modeling-made-functional](https://pragprog.com/book/swdddf/domain-modeling-made-functional)
 3. [https://github.com/derive4j/derive4j](https://github.com/derive4j/derive4j)
 4. [https://gitter.im/derive4j/derive4j](https://gitter.im/derive4j/derive4j)
