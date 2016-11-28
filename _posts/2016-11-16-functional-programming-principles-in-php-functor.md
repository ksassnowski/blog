---
layout: post
title: "Functional Programming Principles in PHP – Functors"
date:   2016-11-24 08:00:00 +01:00
categories: php
tags: php functional-programming
---
_Disclaimer: I do not recommend for you to write production code like this. This is a thought experiment and should be treated as such. I attempt to apply certain concepts that I learned in the context of Haskell to PHP. The implementation can become fairly obscure and complex. They only serve educational purposes._

Recently, I have been learning Haskell using [this book](http://haskellbook.com/). Especially the later parts of the book pique my interest. In there, the authors talk about the more intimidating topics such as Monads, Monoids and Semigroups.  As I learn about the theory, I am always curious how I can apply this knowledge to other languages. Since in my day job I work in PHP, this is the language I am most interested in. Figuring out if and how I can apply a topic to a completely different programming paradigm helps me deepen my understanding. One of these topics is the `Functor` typeclass. What follows are my notes on trying to apply what I learned to PHP.

## Things we already now
Every PHP developer is familiar with arrays. In PHP, they are the One Datatype To Rule Them All (tm). As such, we should all be familiar with the notion of _mapping_  a function over an array like so:

```php
$arr = [1, 2, 3, 4];

$mapped = array_map(function ($x) {
    return $x + 1;
}, $arr);

// $mapped is now [2, 3, 4, 5]
```

What `array_map` is doing, is to first apply the callback to every element in the array. It then returns a new array containing the result of function applications. In the above example, we just added `1` to every element in the list. Easy stuff.

## Things we might not know
What if I told you that _mapping_ is not specific to arrays at all. In fact, it has nothing to do with arrays per se. That is just an implementation of a much more general pattern.

If you have mainly worked in PHP, this may come as a surprise to you. In the context of PHP, we only really come into contact with mapping when we deal with arrays. Assuming that it is an operation inherit to lists might not seem too far fetched.

Let’s now think about what `array_map` is doing on a more general level. Instead of saying that we iterate over the list, applying the callback to each element, let’s try and find a more general description. Another way to describe `array_map`, is that it takes a function that operates on a single value and makes it work on lists. 

Here is yet another way to look at it. What we are really interested in, is the values _inside_ of our list. We want to apply some operation (or transformation) to these elements. The issue is that our elements are wrapped in this container-like structure, the array. Since our function only operates on single values, we need a way to _reach inside_ this structure and pull out the individual values. That allows us to then apply our function to a single value. After that we put the result of this computation into another list which gets returned at  the very end. That means, that `array_map` is **structure-preserving**. We started with an array and we end up with (1) another array (2) _of the same length_. We changed the values *inside* the structure, but not the structure itself. 

If this all sounds really abstract and confusing, don’t worry. It will hopefully all make sense in a bit.

## Some notation
Let me introduce some notation. I will try to keep this to a minimum. However knowing how to read this notation really helps for the coming parts. 

Suppose we have something like this:

```haskell
add1 :: Int -> Int
```

The way you read this expression is 

> `add1` is a function, that takes an `Int` and returns an `Int`

Here’s another one:

```haskell
add :: Int -> Int -> Int
```

This reads

> `add` is a function, that takes an `Int` and an `Int` and returns an `Int`

Treat the last thing as the return type and everything else as arguments (even though that is not [strictly true](https://wiki.haskell.org/Currying) , it’s good enough for our purposes). One last example to demonstrate what a function looks like that takes another function as an argument (a so-called [higher-order function](https://en.wikipedia.org/wiki/Higher-order_function) ).

```haskell
filter :: (a -> Bool) -> [a] -> [a]
```

> `filter` is a function, that takes another function, that takes an `a` and returns a `Bool`, and a list of `a` and returns a list of `a`.

In this example `a` can be any type. We’re not restricted to, say, `Int` for example.

Ok, I hope that wasn’t too bad. Let’s move on.

## Make it functor…y
I mentioned above, that mapping is not specific to lists. If that’s the case, we should be able to _generalise_ the concept of mapping over something. Turns out we can! This is called a `Functor`. A functor is something that is _mappable_. 

This is an example of a [type class](https://www.haskell.org/tutorial/classes.html) . Think of a type class as something very close to an interface. It defines a list of functions that a type has to implement if it wants to act as an instance of that type class. Here’s the definition of `Functor` in Haskell.

```haskell
class Functor f where
    fmap :: (a -> b) -> f a -> f b
```

Let’s go through this slowly.

First of all, do not get confused by the word `class`. It has nothing to do with classes in OOP. It means class, as in type class. So we’re defining a type class called `Functor`. This type class takes a single type parameter `f`. What does that mean? If you’ve ever worked with Java, you know that you can never have just a `List`. It’s always a list _of something_. `List<String>`, `List<Int>`, `List<List<String>>` and so on. So `List` takes a type parameter. What that means for us, is that in order for something to be a functor, it needs to take exactly one type parameter. In other words, it needs to be a _type constructor_.

The method that we need to implement for something to be a functor is called `fmap`. Let’s see what the function signature tells us.

> `fmap` is a function, that takes a function from `a` to `b`, a functor of `a` and returns a functor of `b`

If you understood that, then pat yourself on the back. If not, read on.

### Deciphering fmap
Remember,  I said that functor means _mappable_. We already know, that we can map over lists. So from that we can deduce that lists are functors.

Let’s get rid of all this abstract nonsense and look at the type signature of `fmap` for lists. That means just replace every occurrence of `f` with `[]`, which is the list constructor.

```haskell
fmap :: (a -> b) -> [a] -> [b]
``` 

Aha! That makes a lot more sense. We take a function from `a` to `b` and a list of `a` and return a list of `b`. That looks like our good friend `array_map`.

```php
array array_map(callable $callback , array $array)
```

The `callable` is our function `(a -> b)`. The `$array` we take as an argument is our `[a]` (or `f a` in the original definition). And we return another array, which corresponds to `[b]` (or `f b`). Of course, since PHP is dynamically typed we lose some information about the types. The PHP equivalent of `fmap` for arrays would look something like this.

```
fmap :: callable -> array -> array
```

But let’s stick with the original definition and just pretend that PHP was statically typed. Trust me, it makes all this much easier to understand.

### Defining an interface
Let’s take the above definition of `Functor` and translate it to a PHP interface. It would look something like this.

```php
interface Functor
{
    public function fmap(callable $fn): Functor
}
```

_Note: I’m using PHP 7 type hinting here to stick more closely to the original definition. It is not strictly needed for any of this to work._

We’re doing this in an object-oriented way. That means our version of `fmap` takes only a single parameter, the callback. The functor instance itself would be the seconds argument. So instead of 

```php
fmap($myFunction, $functor);
``` 

we get 

```php
$functor->fmap($myFunction);
```

Again, due to the dynamic nature of PHP we lose some information about the types. There is no way for us to parameterise our `Functor` interface over a type. But this is as close as we can get.

Now that we have an actual PHP interface, let’s try and write a few classes that implement this interface.

## Creating our own array
The easiest way to get our feet wet with this functor thing is by just reimplementing our trusty array using the functor interface. So let’s try that.

```php
class Arr implements Functor
{
    protected $items;

    public function __construct(array $items)
    {
        $this->items = $items;
    }

    public function fmap(callable $fn): Functor
    {
        return new static(array_map($fn, $this->items));
    }
}
```

Essentially, what we have here is a very simple wrapper around an array. For brevity’s sake, I did not implement any of the array interfaces (like `ArrayAccess`). We don’t need them for this.

As you can see, `fmap` for our little array wrapper is simply `array_map`! The only extra thing we need to do, is wrap the result into another instance of our new class. We have to do that because our functor interface dictates that `fmap` needs to return another `Functor`. Let’s try the example from above with our shiny new class.

```php
$add1 = function ($x) { return $x + 1; };

$arr = new Arr([1, 2, 3, 4]);

$mappedArr = $arr->fmap($add1);
// Arr([2, 3, 4, 5])
```

Works as expected, neat.

## That’s enough (for now)
If you made it all the way to here, congratulate yourself.

Looking back at it, this really didn’t teach us all that much. We basically ended up in the same place as before, but now with a crappy home-brewed implementation of an array. 

The purpose of part 1 was mostly to introduce the idea that *mapping over something* can be generalised. I also tried to gently introduce some notation that will help us understand part 2.

So if this felt a bit underwhelming, do not fret! In the next part we’re going to apply our newfound knowledge about functors to things that aren’t arrays! If that does not excite you, you have no soul (or something like that). 

So see you soon for part 2!
