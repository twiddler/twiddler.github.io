---
layout: post
title: Arrows, Functions, and My Sanity
categories: [javascript, typescript, conventions]
---

A practical look at how I decide between arrow functions and function declarations. Cleaner code, fewer surprises, and a happier me.

# Immediate returns

Arrow functions are great for returning things:

```js
const labels = options.map((option) => option.label);
```

"Collect the label of every option. Got it."

We can write the same algorithm with a conventional function:

```js
const labels = options.map(function (option) {
  return option.label;
});
```

"Alright, we are mapping a function, so anything can happen here … Ah wait, it immediately returns a property. Just one more look to make sure I got that right … Okay, got it."

I read a lot of code, so these little extra steps add up and drain me quicker than they need to.

**If I return something immediately, I use an arrow function.**

(Albeit at top-level, I prefer `function foo() { return 42 }` over `const foo = () => 42`.)

# Binding `this`

I remember `this` giving me a hard time when I was getting started with JavaScript. Nowadays, I rarely (if ever) use `this`, so pardon this fairly synthetic example that hopefully still gets the point across:

```js
function foo() {
  this.n = 42;

  return {
    double() {
      return this.n * 2;
    },
  };
}
```

What do you expect when calling `foo().double()`?

```console
> foo().double()
NaN
```

Wait, what? Should it not be `84`? Let's try

```js
function foo() {
  this.n = 42;

  return {
    double: () => this.n * 2,
  };
}
```

```console
> foo().double()
84
```

Ok, that works. But why?

> Arrow functions differ in their handling of `this`: they inherit `this` from the parent scope at the time they are defined. This behavior makes arrow functions particularly useful for callbacks and preserving context.

--_[this - JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)_

We can still make this work without arrow functions:

```js
function foo() {
  this.n = 42;

  const this_ = this;

  return {
    double() {
      return this_.n * 2;
    },
  };
}
```

```console
> foo().double()
84
```

Again, I prefer code that is easier for me to grasp. Binding `const this_ = this` is an extra step that I have to resolve whenever I read `this_`. I prefer the arrow function.

**If I need a certain `this`, I use an arrow function.**

# `const` vs `function`

Some time after arrow functions were introduced, people started slapping them everywhere:

```js
const toggle = (b) => !b;

const Button = () => {
  return {
    label: "My button",
    onClick: toggle,
  };
};
```

That just makes my head go "Erm, what?". When I think of something constant, it is things like

```js
const foo = "bar";

const n = 42;

const products = [
  { name: "blueberry ice cream", price: "4.20" },
  { name: "dragonfruit", price: "6.00" },
];
```

I do not think of a function. Yes, a function (or the reference to one) is technically constant, it is just not what _intuitively_ comes to my mind when thinking of a constant.

**If something is a function, it should be denoted with `function`.**

(Note that `function foo(…) {…}` gets hoisted, while `const foo = function(…) {…}` does not. But this is almost always what you want, anyway.)

# Conclusion

Whatever piece of code I write, I will almost certainly read it again sooner or later, and when that time comes, my not-so-constant memory will have been read from and written to multiple times, so _I'd better write this in the most intuitive way I can_ just so I will not have a harder time understanding it than need be.

When it comes to arrow functions and conventional functions, this is what is easiest to me:

1. By default, denote a function with `function`.
2. If the function immediately returns something and is not a top-level function, write it as an arrow function.
3. If the function needs a certain `this`, write it as an arrow function.
