> This is a translation of [Functors, Applicatives, And Monads In Pictures](http://adit.io/posts/2013-04-17-functors,_applicatives,_and_monads_in_pictures.html) from Haskell into Java. Hopefully this should make the article much easier to understand for people who don't know Haskell.
I will be using java's jshell to demonstrate code snippets:
```shell
$ jshell
|  Welcome to JShell -- Version 17.0.10
|  For an introduction type: /help intro
```

Here’s a simple value:

![](http://adit.io/imgs/functors/value.png)

And we know how to apply a function to this value: 

![](http://adit.io/imgs/functors/value_apply.png)

In Java shell it will look like this:
```shell
jshell> Function<Integer, Integer> add3 = x -> x + 3;
add3 ==> $Lambda$20/0x0000023581009a00@28c97a5

jshell> var result = add3.apply(2)
result ==> 5
```

Simple enough. Lets extend this by saying that any value can be in a context. For now you can think of a context as a box that you can put a value in:


![](http://adit.io/imgs/functors/value_and_context.png)

Now when you apply a function to this value, you’ll get different results **depending on the context**. This is the idea that Functors, Applicatives, Monads, Arrows etc are all based on. The Maybe data type defines two related contexts:

![](http://adit.io/imgs/functors/context.png)

In Java, instead of Maybe, we have the `Optional<T>` class which
models the absence of value `T`. The following table maps Haskell definitions to Java classes

| Haskell    | Java              |
| ---------- | ----------------  |
| `Maybe`    | `Optional<T>`     | 
| `Just 2`   | `Optional.of(2)`  |
| `Nothing`  | `Optional.empty()`|

```shell
jshell> var just2 = Optional.of(2)
just2 ==> Optional[2]

jshell> var nothing = Optional.<Integer>empty();
nothing ==> Optional.empty
```

<details>

  <summary>We can implement Maybe in Java by own</summary>

```java
sealed interface Maybe<T> {

  default <R> Maybe<R> map(Function<T, R> mapper) {
    if (this instanceof Just<T> just) {
      return Maybe.just(mapper.apply(just.t));
    } else {
      return Maybe.nothing();
    }
  }

  default <R> Maybe<R> flatMap(Function<T, Maybe<R>> mapper) {
    if (this instanceof Just<T> just) {
      return mapper.apply(just.t);
    } else {
      return Maybe.nothing();
    }
  }
  
  static <T> Maybe<T> just(T t) {
    return new Just<>(t);
  }

  @SuppressWarnings("unchecked")
  static <T> Maybe<T> nothing() {
    return (Maybe<T>) Nothing.INSTANCE;
  }

  record Just<T>(T t) implements Maybe<T> {
  }

  final class Nothing<T> implements Maybe<T> {
    static final Nothing<?> INSTANCE = new Nothing<>();
  }
}
```

</details>

<br>

In a second we’ll see how function application is different when something is a Just a versus a Nothing. First let’s talk about Functors!

# Functors

When a value is wrapped in a context, you can’t apply a normal function to it:

![](http://adit.io/imgs/functors/no_fmap_ouch.png)

This is where `map` comes in (`fmap` in Haskell). `map` is from the street, `map` is hip to contexts. `map` knows how to apply functions to values that are wrapped in a context. For example, suppose you want to apply `add3` to `just2`. Use `map`:

```shell
jshell> var just5 = just2.map(add3);
just5 ==> Optional[5]
```

![](http://adit.io/imgs/functors/fmap_apply.png)

**Bam!** `map` shows us how it’s done! But how does `map` know how to apply the function?

# Just what is a Functor, really?

Functor is an Abstract Base Class (typeclass in Haskell). Here’s the definition:

![](http://adit.io/imgs/functors/functor_def.png)

In Java it could be a functional interface:

```java
@FunctionalInterface
public interface Functor<T> {
  <R> Functor<R> map(Function<T, R> mapper);
}
```

A Functor is any data type that defines how `map` applies to it. Here’s how `map` works:

![](http://adit.io/imgs/functors/fmap_def.png)

So we can do this in Java:

```shell
jshell> just2.map(x -> x + 3);
$10 ==> Optional[5]
```

And `map` magically applies this function, because `Optional` is a `Functor`. It specifies how `map` applies to the empty and non-empty Optional's:

Here’s what is happening behind the scenes when we write `just2.map(x -> x + 3)`:

![](http://adit.io/imgs/functors/fmap_just.png)

So then you’re like, alright `map`, please apply `x -> x + 3` to a Nothing?

![](http://adit.io/imgs/functors/fmap_nothing.png)

```shell
jshell> nothing.map(x -> x + 3);
$14 ==> Optional.empty
```

![](http://adit.io/imgs/functors/bill.png)

_Bill O’Reilly being totally ignorant about the Maybe functor_

Like Morpheus in the Matrix, `map` knows just what to do; you start with `Nothing`, and you end up with `Nothing`! `map` is zen. Now it makes sense why the `Optional` data type exists. For example, here’s how you work with a database record in a language without `Optional`:

```java
var post = Post.find_by_id(1);
if (post != null) {
  return post.getTitle();
} else {
  return "N/A";
}
```

But in Java with Optional:

```java
return Optional.ofNullable(Post.find_by_id(1))
.map(Post::getTitle)
.orElse("N/A")
```

If `find_by_id()` returns a post, we will get the title with `getTitle()`. If it returns `Nothing`, we will return `Nothing`! Pretty neat, huh? 

Here’s another example: what happens when you apply a function to a list?

![](http://adit.io/imgs/functors/fmap_list.png)

In Java with Stream API:

```java
jshell> Stream.of(2,4,6).map(add3).toList();
$15 ==> [5, 7, 9]
```

Okay, okay, one last example: what happens when you apply a function to another function?

![](http://adit.io/imgs/functors/function_with_value.png)

Here’s a function applied to another function:

![](http://adit.io/imgs/functors/fmap_function.png)

The result is just another function!

```java
jshell> var add5 = add3.andThen(x -> x + 2);
add5 ==> java.util.function.Function$$Lambda$25/0x000002358105d928@621be5d1

jshell> add5.apply(4)
$18 ==> 9
```

So functions are Functors too!

# Applicatives

Applicatives take it to the next level. With an applicative, our values are wrapped in a context, just like Functors:

![](http://adit.io/imgs/functors/value_and_context.png)

But our functions are wrapped in a context too!

![](http://adit.io/imgs/functors/function_and_context.png)

Yeah. Let that sink in. Applicatives don’t kid around. Applicative knows how to apply a function wrapped in a context to a value wrapped in a context:

![](http://adit.io/imgs/functors/applicative_just.png)

Java's Optional is not applicative but we can write aplicative function for it:

```java
  <T, R> Optional<R> applicative(Optional<Function<T, R>> func, Optional<T> value) {
    return func.flatMap(f -> value.flatMap(v -> Optional.ofNullable(f.apply(v))));
  }
```
Let's test it

```java
jshell> applicative(Optional.of(add3), just2);
$20 ==> Optional[5]

jshell> applicative(Optional.empty(), just2);
$21 ==> Optional.empty

jshell> applicative(Optional.of(add3), nothing);
$22 ==> Optional.empty
```

Using `<*>` in Haskell can lead to some interesting situations.

![](http://adit.io/imgs/functors/applicative_list.png)

In Java we do not have such operator `<*>`, but we can achive the same result in the following way:

```java
jshell> Stream.of(mult2, add3).flatMap(f -> Stream.of(1,2,3).map(f)).toList();
$29 ==> [2, 4, 6, 4, 5, 6]
```
Here’s something you can do with Applicatives that you can’t do with Functors. How do you apply a function that takes two arguments to two wrapped values?


`Applicative` pushes `Functor` aside. “Big boys can use functions with any number of arguments,” it says. “Armed with <$> and <*> in Haskell , I can take any function that expects any number of unwrapped values. Then I pass it all wrapped values, and I get a wrapped value out! AHAHAHAHAH!”

And hey! There’s a method called `lift_a2` that does the same thing:

Java can't handle any number of parameters in aplicative calls, but anyway we can create
`lift_a2` method:

```java
  <A, B, C> Optional<C> lift_a2(Optional<A> optA, Optional<B> optB, BiFunction<A, B, C> bifunc) {
    return optA.flatMap(a -> optB.flatMap(b -> Optional.of(bifunc.apply(a, b))));
  }
```
Let's test it

```
jshell> lift_a2(just2, Optional.of(3), (x, y) -> x * y);
$31 ==> Optional[6]

jshell> lift_a2(just2, nothing, (x, y) -> x * y);
$32 ==> Optional.empty
```

# Monads

How to learn about Monads:

1. Get a PhD in computer science.
2. Throw it away because you don’t need it for this section!

Monads add a new twist.

Functors apply a function to a wrapped value:

![](http://adit.io/imgs/functors/fmap.png)

Applicatives apply a wrapped function to a wrapped value:

![](http://adit.io/imgs/functors/applicative.png)

Monads apply a function that returns a wrapped value to a wrapped value. Monads have a function `flatMap` (>>= in Haskell) (pronounced “bind”) to do this.

Let’s see an example. Good ol’ Maybe is a monad:

![](http://adit.io/imgs/functors/context.png)

Just a monad hanging out

Suppose `half` is a function that only works on even numbers:

```java
Function<Integer, Optional<Integer>> half = i -> i%2 == 0 ? Optional.of(i/2) : Optional.empty();

```

![](http://adit.io/imgs/functors/half.png)

What if we feed it a wrapped value?

![](http://adit.io/imgs/functors/half_ouch.png)

We need to use `flatMap` (`>>=` in Haskell) to shove our wrapped value into the function. Here’s a photo of `flatMap`:

![](http://adit.io/imgs/functors/plunger.jpg)

Here’s how it works:

```java
jshell> Optional.of(4).flatMap(half);
$47 ==> Optional[2]

jshell> Optional.of(2).flatMap(half)
$48 ==> Optional[1]

jshell> Optional.of(1).flatMap(half)
$49 ==> Optional.empty
```

What’s happening inside? Monad is another Abstract Base Class (typeclass in Haskell). Here’s a partial definition for Java:

```java
public interface Monad<T> {
	<R> Monad<R> flatMap(Function<? super T, ? extends Monad<R>> mapper);
}
```

Where bind (`flatMap` in Java, `>>=` in Haskell) is:

![](http://adit.io/imgs/functors/bind_def.png)

So Java's `Optional<T>` actually is a Monad:

Here it is in action with a Just 3!

![](http://adit.io/imgs/functors/monad_just.png)

```
jshell> Optional.of(3).flatMap(half)
$51 ==> Optional.empty
```

And if you pass in a `Nothing` it’s even simpler:

![](http://adit.io/imgs/functors/monad_nothing.png)

```
jshell> nothing.flatMap(half);
$53 ==> Optional.empty
```

You can also chain these calls:


![](http://adit.io/imgs/functors/monad_chain.png)

```java
jshell> Optional.of(20).flatMap(half).flatMap(half).flatMap(half);
$50 ==> Optional.empty
```
Cool stuff! So now we know that Optional is a Functor and a Monad. Optional is not Applicative itself, but we can provide applicative functions if needed.

## IO Monad

Now let’s mosey on over to another example: the `IO` monad:

![](http://adit.io/imgs/functors/io.png)

Java does not have IO monad in it's standard library. So for the sake of this translation I will use my own minimal IO monad implementation.

```java
public interface IO<T> {

  T run();
  
  default <R> IO<R> flatMap(Function<T, IO<R>> mapper) {
    return () -> mapper.apply(run()).run();
  }
}
```

Simply speaking it is just Java's `Supplier<T>` interface with defined `flatMap` method.

Specifically three functions. `getLine` takes no arguments and gets user input:

![](http://adit.io/imgs/functors/getLine.png)

```java
  IO<String> getLine() {
    return () -> "read line";  //skipping real reading from console
  }
```

`readFile` (`readFile` in Haskell) takes a string (a filename) and returns that file’s contents:

![](http://adit.io/imgs/functors/readFile.png)

```java
  IO<String> readFile(String fileName) {
    return () -> "file content"; //skipping real file reading
  }
```

`printLine` (`putStrLn` in Haskell) takes a string and prints it:

![](http://adit.io/imgs/functors/putStrLn.png)

```java 
  IO<Void> printLine(String text) {
    return () -> {
      System.out.println(text);
      return null;
    };
  }
```

All three functions take a regular value (or no value) and return a wrapped value. We can chain all of these using `flatMap`!

![](http://adit.io/imgs/functors/monad_io.png)

```
   var io = getLine()
    .flatMap(s -> readFile(s))
    .flatMap(s -> printLine(s));
    
    io.run();
```

Aw yeah! Front row seats to the monad show!

Haskell also provides us with some syntactical sugar for monads, called do notation:

```haskell
foo = do
    filename <- getLine
    contents <- readFile filename
    putStrLn contents
```

Java does not have a `do` notation, alas.

# Conclusion

1. A functor is a data type that implements the `Functor` abstract base class.
2. An applicative is a data type that implements the `Applicative` abstract base class.
3. A monad is a data type that implements the `Monad` abstract base class.
4. A `Maybe` implements all three, so it is a functor, an applicative, and a monad.
5. Java's `Optional<T>` implements two of them, `Functor` and `Monad`.

What is the difference between the three?

![](http://adit.io/imgs/functors/recap.png)

* **functors**: you apply a function to a wrapped value using `map` or `%`
* **applicatives**: you apply a wrapped function to a wrapped value using `*` or `lift`
* **monads**: you apply a function that returns a wrapped value, to a wrapped value using ´|´ or `bind`

So, dear friend (I think we are friends by this point), I think we both agree that monads are easy and a SMART IDEA(tm). Now that you’ve wet your whistle on this guide, why not pull a Mel Gibson and grab the whole bottle. Check out LYAH’s [section on Monads](http://learnyouahaskell.com/a-fistful-of-monads). There’s a lot of things I’ve glossed over because Miran does a great job going in-depth with this stuff.