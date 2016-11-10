---
layout: post
title:  "Understanding filterM"
date:   2016-11-06 21:19:00 +0100
categories: haskell
---

At the recent Haskell Meetup I went to, someone posed this interesting challenge:

> Given a list of integers, produce a list of all possible sums that can be created using the integers from the list.

Turns out, in Haskell this is a one-liner:

```haskell
map sum $ filterM (const [True, False]) [1..5]
```

Which returns

```
[15,10,11,6,12,7,8,3,13,8,9,4,10,5,6,1,14,9,10,5,11,6,7,2,12,7,8,3,9,4,5,0]
```

To me as a Haskell beginner, this is straight up witchcraft. How can this line possibly produce something this complex. So a couple of us decided to dive into the implementation of `filterM` to see how it all works.  What follows are our findings (a bit cleaned up).

## First impressions
The first thing throwing me for a loop, is that a function called `filterM` is returning *more* elements than in the initial list. That’s not what _filtering_ means in my mind. I should always end up with less, or at least with the same amount of elements than before.

Secondly, I assume that the `M` in `filterM` stands for Monad. So that instantly makes it a bit more scary to me. While I (sort of) understand what Monads are – at least from a theoretical point of view – I don’t yet have intuition for them. It’s not immediately obvious to me, why Monads are needed in this particular case.

## Understanding the type signature
Looking at the type signature for `filterM`  gives us this:

```haskell
filterM :: Monad m => (a -> m Bool) -> [a] -> m [a]
```

_Note that the type signature `ghci` gives you actually differs from the one on Hoogle. I have chosen to use the type signature that `ghci` provides me._

As mentioned before, monads are still a bit scary to me. What I always end up doing, is pluck in the actual Monad I’m dealing with, and see what the type signature looks like. So let’s do that.

We’re passing in `const [True, False]` into `filterM`. So our monad in this case would be the list monad `[]`. Let’s see what that gives us.

```
filterM :: (a -> [Bool]) -> [a] -> [[a]]
```

Ok that makes a bit more sense. One thing that strikes me as interesting, is that this is returning a nested list.

Let’s continue.

## Playing human interpreter
Since I still don’t have a clue what this function is supposed to do, I decided to just evaluate the function myself by plucking in actual values. Let’s take a look at our `filterM` call again.

_I’m leaving off the map call and only focusing on the filterM expression here._

```haskell
filterM (const [True, False]) [1..5]
```

`filterM` can be implemented something like this (credits to [this guy](https://byorgey.wordpress.com/2007/06/26/deducing-code-from-types-filterm/) ):

```haskell
filterM :: Monad m => (a -> m Bool) -> [a] -> m [a]
filterM p []     = return []
filterM p (x:xs) =
    let rest = filterM p xs in
        do b <- p x
           if p then liftM (x:) rest
           else                 rest
```

_This is not actually how it is [implemented](http://hackage.haskell.org/package/base-4.9.0.0/docs/src/Control.Monad.html#filterM), but this implementation makes it much easier to understand._

Since we’re not passing in an empty list, let’s just skip that part for now and instead focus on the second part of the pattern match.

Immediately we have a recursive call on the tail of the list. Let’s ignore that for now and move on.

```haskell
do b <- p x
   if b then liftM (x:) rest
   else                 rest
```

Ok, so what’s going on here. Looks like we’re applying `p` to `x`, which is the head of our input list `[1..5]`.  Substituting the variables for their actual values gives us

```haskell
do b <- const [True, False] 1
-- ...
```

`const` is a function of type `a -> b -> a`. So in other words, it takes two parameters and simply returns the first one.  The first argument in this case is `[True, False]`. So `const [True, False] 1` returns `[True, False]`. In fact, `const [True, False] anything` will return `[True, False]`. This expression will _always_ evaluate to `[True, False]`.

Moving on, that leaves us with

```haskell
do b <- [True, False]
    if b then liftM (1:) rest
    else                 rest
```

What does `b <- [True, False]` do? We’re in a do-expression, so this is just syntactic sugar around bind (`>>=`) calls. Let’s desugar and see what we end up with.

```haskell
[True, False] >>= \x -> if b then liftM (1:) rest else rest
```

Ok, so depending on what `b` is, we either execute `liftM (x:) rest` or simply `rest`. `rest` is the result of our initial recursive call `filterM p xs` which has type `m [a]` since that is the return type of `filterM`.

Right so what does `liftM (x:) rest` do? `liftM` has this type signature

```haskell
liftM :: Monad m => (a -> b) -> m a -> m b
```

That looks familiar… What’s the type signature of `fmap` again?

```haskell
fmap :: Functor f => (a -> b) -> f a -> f b
```

Swap out Monad with Functor and `m` with `f` and you have the same function. So `liftM` is basically `fmap`, but for monads. In other words, it takes a function that works on “unwrapped” values, and makes it work on values that are in a monad.

So back to our example. We had this expression

```haskell
if b then liftM (1:) rest else rest
```

If b is `True` we want to cons `1` onto `rest`. Since `rest` is of type `[[Int]]` and `x` is of type `Int` we need to lift the cons function `(:)` into the monad before we can apply it. Makes sense.

Depending on the value of `b` we either return a list with `1` added to the front, or one without the `1`. There’s our filter!

That still leaves the question what `b` actually _is_. We had the following piece of code

```haskell
[True, False] >>= \x -> if x then liftM (1:) rest else rest
```

It seems like we need to take a look at how `(>>=)` is implemented for the list monad in order for this snippet to make sense. Here it is.

```haskell
(>>=) :: Monad m => m a -> (a -> m b) ->  m b
xs >>= f = concat $ fmap f xs
```

Aha! Bind for lists is implemented in terms of `fmap`. So what’s really going on is that we’re feeding both `True` and `False` into our little if-expression one after the other.  But what does it mean? Let’s see what the expression above evaluates to:

```haskell
[True, False] >>= \x -> if x then liftM (1:) rest else rest

[[1],[]]
```

What if we were to simulate the recursive call, and pass in `2` as our value. In this case `rest` would be equal to `[[1], []]`.

```haskell
[True, False] >>= \x -> if x then liftM (2:) [[1],[]] else [[1],[]]

[[2,1],[2],[1],[]]
```

I think I’m starting to see it. One more.

```haskell
[True, False] >>= \x -> if x then liftM (3:) [[2,1],[2],[1],[]] else [[2,1],[2],[1],[]]

[[3,2,1],[3,2],[3,1],[3],[2,1],[2],[1],[]]
```

_Note, that this not actually how the evaluation goes. We’re evaluating from left to right for simplicities sake. However, `filterM` is actually implemented in terms of `foldr`.  So evaluate it from right to left, starting with the `5`._

For every element in our input list, we return a list _with_ that element and one _without_ the element. Due to the recursive nature of this algorithm this gives us back all possible subsets of our initial list.

After some research, I found out, that this is called the [power set](https://en.wikipedia.org/wiki/Power_set) of a list (or set, in set theory).

## Wrapping it up
Going back to the original problem we can now see, that 

```haskell
filterM (const [True, False]) [1..5]
```

is returning all possible subsets that can be formed using the numbers 1 to 5 (including the empty set).

```
[[1,2,3,4,5],[1,2,3,4],[1,2,3,5],[1,2,3],[1,2,4,5],[1,2,4],[1,2,5],[1,2],[1,3,4,5],[1,3,4],[1,3,5],[1,3],[1,4,5],[1,4],[1,5],[1],[2,3,4,5],[2,3,4],[2,3,5],[2,3],[2,4,5],[2,4],[2,5],[2],[3,4,5],[3,4],[3,5],[3],[4,5],[4],[5],[]]
```

All that’s now left to do in order to answer the original problem is calculating the sum of each subset. Which leaves us with the initial solution.

```haskell
map sum $ filterM (const [True, False]) [1..5]
```

```
[15,10,11,6,12,7,8,3,13,8,9,4,10,5,6,1,14,9,10,5,11,6,7,2,12,7,8,3,9,4,5,0]
```

Neat.
