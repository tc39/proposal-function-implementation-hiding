# `Function.prototype.toString()` censorship proposal

## The problem

JavaScript's `Function.prototype.toString()` method reveals the source text originally used to create the function. This causes two main issues:

### Encapsulation leakage

Revealing the source text of a function gives callers unnecessary insight into its implementation details. They can introspect on a function's implementation and react to it, causing what would otherwise likely be non-breaking changes to become breaking ones.

A historical example of this in action is AngularJS's reliance on `f.toString()` to figure out the parameter names of a function. It would then use these as part of its dependency injection framework. This made what was previously a non-breaking change, modifying the name of a parameter, into a breaking one. In particular, it made it impossible to use minification tools in combination with this mode of AngularJS.

Another unnecessary insight gained by `f.toString()` is how a function was created. Most dramatically, it can be used to detect whether an implementation is "native" or not, by looking for the pattern of `[native code]`. This makes high-fidelity polyfilling difficult; indeed, some zealous polyfill libraries have gone as far as to replace `Function.prototype.toString` to prevent detection. (TODO: is this true? Jordan said it is but I can't find the code.) It can also be used to detect whether the implementation is done via a `class`, a `function`, an arrow function, a method, etc. It would be preferable if these were mostly-undetectable implementation details, allowing more confidence in refactoring between them.

### Memory usage

Having to store the source text of a function for later lookup causes unnecessary memory usage. Especially on memory-constrained platforms, such as embedded devices or [Android Go phones](https://docs.google.com/presentation/d/1snSOAvzlbHsBaH1nelwf2JQq4KYOd6eWWIIO3fOlajs/edit#slide=id.g26319d7823_6_247), the extra space required for storing source text is quite significant.

TODO: get some more data on how bad this is.

## Toward a solution

We'd like to provide a way to "censor" the output of `Function.prototype.toString()`, allowing both of these issues to be addressed.

There are a few major questions that come up in designing such a solution:

* What is the scope of the censorship? Some candidates include realm, source file, and individual function.
* What is the mechanism for censoring? Some candidates include an external-to-JavaScript flag, a source-text-level parsing-time annotation, or a dynamic switch on the function object itself.
* What is the censored output? Probably this should be something simple like `function $functionName() { [native code] }`; this allows cloaked polyfills. We could also contemplate, however, using some new sigil like `[censored code]`.

In practice, we expect people mostly want the scope of the censorship to be per source file, or at least per large block of code. That is, we expect entire libraries to be written with censored `f.toString()`. However, given the still-common practices of bundling and concatenation, we may need something more granular so that people can mix together uncensored and censored code in a single source file, for cases where programs depend on at least some functions having uncensored `f.toString()` output.

Another important constraint is that you must not be able to "uncensor" a function after censoring it. Doing so would require implementations to hold the source text around anyway, negating the memory usage benefits of censorship.

Below are a few proposals.

## Proposals

### A new pragma

A new pragma would be introduced, perhaps `"use no Function.prototype.toString"`. Like `"use strict"`, it could be placed at either the source file level or the per-function level.

Similar to the strict pragma, this new pragma would apply "inclusively downward", so that the everything within the scope, plus the function itself when in function scope, gets censored. For example:

```js
function foo() {
    const x = () => {};

    const y = () => {
        "use no Function.prototype.toString";
        class Z {
            m() {}
        }
    };
}
```

In this example, `foo` and `x` remain uncensored, while `y`, `Z`, and `Z.prototype.m` are censored.

This proposal draws heavily on the strengths of the existing strict mode pragma:

* It allows easy censorship of an entire source file.
* At the same time, it allows easy bundling together of uncensored and censored code, by using anonymous-function-wrapper blocks.

Some potential complications:

* Like `"use strict"`, this cannot be opted out of from within a censored scope. That seems fine though.
* It might be slightly confusing that in the above example `y` is censored; one might think "the censorship starts inside `y`". But, `"use strict"` has an analogous problem ("the strictness starts inside `y`"), and mostly people seem fine.
* We would need to introduce pragmas into class bodies, which is unprecedented since currently they are always strict.
* For memory-conscious developers using uncensored third-party libraries, they would need to do some source file preprocessing to censor them.

### An external-to-JavaScript switch

In this proposal, the JavaScript specification would mostly not be modified. It would only add some clause to the definition of `Function.prototype.toString()` which allows implementations to return an appropriate, well-defined censored string under host-defined conditions.

Then, individual hosts would expose (or not) the ability to censor functions, source files, realms, or programs. For example, you could imagine running

```bash
$ node --no-fn-tostring myapp.js
```

or in the browser, using a whole-page pragma such as

```html
<meta http-equiv="script-tostring" content="off">
```

or using a per-resource header such as

```
Script-ToString: off
```

on individual JavaScript files.

The main advantage of this method over the pragma is that it allows relatively easy imposition of broad censorship across a whole program or source file, without source text manipulation. But there is of course a tradeoff: when external metadata controls program behavior, code becomes less portable. In other words, with this solution, it's not obvious whether it's safe to censor a source file or not; you basically have to try it and see if anything breaks.

### A one-time censorship method

In this proposal, functions get a new method, `f.censor()`, which permanently censors their stringified value. You'd use it like so:

```js
function foo() {
    // ...
}

console.assert(foo.toString().includes("..."));

foo.censor();

console.assert(foo.toString() === "function foo() { [ native code ] }");
```

Note that we do not propose a method that returns a new, censored version because of all the difficulties involved in "cloning" functions: e.g., how would reinstall a method with the correct [[HomeObject]] after creating a new censored version of it?

This proposal seems less good than the others:

* It is difficult to en-masse censor many functions. In particular, it is impossible to censor functions which are kept alive by a third-party library in one of its closures, without nontrivial source text modification.
* Censorship is done at runtime, not at parse time, so the implementation needs to store the source text for some interval.

Let's not do this.

### `delete Function.prototype.toString`

In the past, people have asked if perhaps engines could just detect the pattern of

```js
delete Function.prototype.toString;
```

 early in a source file, and perform appropriate optimizations.

 Unfortunately this does not work in any multi-realm environment:

```js
delete Function.prototype.toString;

function foo() {
    // ...
}

const otherGlobal = frames[0];
console.assert(otherGlobal.Function.prototype.toString.call(foo).includes("..."));
```

It's also a very blunt tool, usable only on the realm level, and thus probably only by application developers. We'd like whatever we come up with to work for library developers as well.
