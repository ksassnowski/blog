---
layout: post
title: "Refactoring Legacy Code - Nullable Constructor Parameters"
date:   2017-11-11 08:00:00 +01:00
categories: php
tags: php testing refactoring
---
One of the first things you usually want to do when working with legacy code is to get them into a test harness. That way you can later make changes to the codebase and verify that you didn’t change or break the existing behaviour. 

The issue usually is that legacy code is not very conducive to testing. This might be for a variety of reasons. But the issue that I have seen the most in the wild is that everything is just so tightly coupled that you really can’t just test _this one thing_. Because that thing requires this thing which requires this other thing and before you know it you require every single class in the application just to test this small part of functionality. 

> You wanted a banana but what you got was a gorilla holding the banana and the entire jungle.
<small>Joe Armstrong - Creator of Erlang</small>

Take this simplified piece of code for example:

```php
class LegacyClass
{
    public function legacyMethod()
    {
        $dependency = new MyDependency();
        // do something...
    }
}
```

Now, let me preface this by saying that I usually am not a big fan of mock-heavy tests. But in the case of a legacy application I really don’t want to pull in a dependency that I can’t control. In some cases it might even make testing close to impossible because of how tightly coupled everything is. What is `MyDependency` doing? It might extend 15 other classes. It might require a database connection. It might send out an email using hardcoded production credentials! Not something I want to happen every time I run a test.

With that in mind, how would you go about testing this method? There is no good way to swap out that dependency in a test. 

## Create an optional constructor parameter
One thing you can do in this case is to declare the dependency as an **optional constructor dependency**. Then in your method call you only instantiate the dependency if it hasn’t been passed in through the constructor.

{% highlight php %}
class LegacyClass
{
    /** @var null|MyDependency */
    private $dep;

    public function __construct(?MyDependency $dep = null)
    {
        $this->dep = $dep;   
    }

    public function legacyMethod()
    {
        $dependency = $this->dep ?: new MyDependency();
        // do something...
    }
}
{% endhighlight &}

This way you can now pass in a stub/mock/fake/whatever in your test but at the same time don’t have to touch any existing code. If we don’t pass in a dependency through the constructor the method simply works the way it did before.

{% highlight php %}
/** @test */
public function it_does_something()
{
    $dependencyMock = $this->createMock(MyDependency::class);
    // Set up your mock here...

    $classUnderTest = new LegacyClass($dependencyMock));
    $result = $classUnderTest->legacyMethod();

    // Assert against the result...
}
{% endhighlight %}

### Injecting dependencies on a per-method basis
If, for some reason, passing in dependencies through the constructor is impractical e.g. `MyDependency` requires parameters that are not available at the time `LegacyClass` gets instantiated, you can follow the same pattern to inject the dependency into the method call instead.

{% highlight php %}
class LegacyClass
{
    public function legacyMethod(?MyDependency $dep = null)
    {
        $dep = $dep ?: new MyDependency();
        // do something...
    }
}
{% endhighlight %}

## Oh god this is even worse!
If you look at the supposedly *better* code above and recoil in horror let me quote one of my favourite passages from the book _Working Effectively with Legacy Code_ by Micheal C. Feathers (emphasis mine).

> When you break dependencies in legacy code, **you often have to suspend your sense of aesthetics a bit**. Some dependencies break cleanly; others end up looking less than ideal from a design point of view. They are like the incision points in surgery: There might be a scar left in your code after your work, but everything beneath it can get better. If later you can cover code around the point where you broke the dependencies, you can heal that scar too.

Remember, this is not the end of the refactoring process but only the very first step. In order to do any kind of non-trivial refactoring, you absolutely need tests in place to assure yourself that your refactoring did not break anything. Then when you later want to do proper dependency injection this class is already prepared for it.

Happy refactoring!