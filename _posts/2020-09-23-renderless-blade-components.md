---
layout: post
title: "Renderless Blade Components"
date: 2020-09-23 07:00:00 +01:00
categories: php laravel
tags: php laravel
---

_If you’ve already read Adam Wathan’s excellent blog post about [renderless components in Vue.js](https://adamwathan.me/renderless-components-in-vuejs/), this post will not contain any new concepts. But you might have simply never thought about applying the same idea to blade components._

This post was inspired by a recent [pull request on the Laravel repository.](https://github.com/laravel/framework/pull/34339). The idea was to add a middleware stack to the view layer, similar to how routes go through a list of HTTP middleware. You would then be able to manipulate the HTML of the view before it gets rendered. The example use-case given in the PR was to handle shortcodes, i.e. replacing `[user id="1" field="first_name"]` with the first name of the User with id 1.

While working on something unrelated, I found myself reading through Laravel’s documentation on [Blade components](https://laravel.com/docs/8.x/blade#components). There two ways to define a Blade component: class-based components and anonymous components. For this post, we’re only interested in class-based components.

## Returning views from the render method

When we use a custom Blade component in our templates, the template engine will first attempt to resolve the tag to a registered class-based component. If found, it will then call the `render` method on the class to get the string representation of our component.

In most cases, your render method will look something like this

```php
public function render()
{
    return view('components.my-custom-component');
}
```

Very similar to how a controller might work, we return the view that corresponds to our component.

However, the documentation mentions another thing you can inside the `render` method.

## Returning a Closure

If we want to access the component’s name, attributes or slot, we can return a Closure from the render function instead. The closure will receive a `$data` array as its only argument and needs to return the string representation of our component.

```php
public function render()
{
	  return function (array $data) {
		  // $data['attributes']
        // $data['componentName']
        // $data['slot']

		  return '<p>Component goes brrrrr</p>';
    };
}
```

We’re interested in `$data['slot']` in particular.

### What’s inside `$data['slot']` exactly?

`$data['slot']` contains everything that goes between the opening and closing tag of our component.

```html
<x-my-component>
  <p>Hello world!</p>
</x-my-component>
```

So the slot—or rather, the _contents_ of the slot—for our `<x-my-component>` component would be the string `<p>Hello world!</p>`. We don’t really get back a string, though, but an instance of `Illuminate\Support\HtmlString`.

The `$data['slot']` property allows us to access the component’s content. We can now apply whatever transformation we want to it.

## Example – Censoring swear words

Let’s look at a complete example, shall we?

We want to implement a way to stop our users from using nasty words on our site (there’s no way this could be abused, right?). With everything we’ve learned so far, we can create a Blade component like this:

```php
<?php declare(use_strict=1);

use Illuminate\View\Component;

class CensorSwearWords extends Component
{
	private array $badWords = [
		'poopoo',
		// and other, similarly vile words
	];

	public function render()
	{
		return function (array $data) {
			return str_replace($this->badWords, '*****', $data['slot']);
		};
	}
}
```

Note that this component does not return a view from its `render` method, but instead returns a _render function_. Whatever this function returns is what will be rendered when we use `<x-censor-swear-words>` in our templates.

Inside this render function we implement the actual logic of this component, stripping out bad words (yeah, I’m pretty sure this can’t be abused).

> There is, however, nothing stopping you from returning a view from your render function as well.

This is a fairly naive implementation with a hardcoded list of nasty words right inside the component itself. There’s nothing stopping us from injecting any other dependencies into this class to perform more complicated transformations, however.

You would use this component the same way you use any other blade component.

```html
<x-censor-swear-words>
  <p>I like the <strong>poopoo</strong>.</p>
</x-censor-swear-words>
```

This would output the following HTML:

```html
<p>I like the <strong>*****</strong>.</p>
```

Notice how there is no wrapping `<div>` around the output. That’s because our component is _renderless_. It doesn’t actually have its own template, it only transforms the contents inside of its opening and closing tag.

## Other ideas

Other ideas you could implementing using this pattern (some more useful than others):

- Replacing `@mentions` with a link to the mentioned user’s profile
- Implementing shortcodes (the example given in the PR I mentioned at the beginning of this article).
- Rendering a preview or an excerpt of a comment (so you don’t have to store it as a separate column on the model)

Neat.
