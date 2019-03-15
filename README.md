# Function implementation hiding proposal

A proposal for a new directive, tentatively `"hide implementation"`, which provides a way for developers to indicate that implementation details should not be exposed. This has benefits for authors of library code who would like to refactor without fear of breaking consumers relying on their implementation details and authors of encapsulated code who would like to provide certain encapsulation guarantees.

In practice, this directive currently hides the source text revealed by `Function.prototype.toString` and position information and calling behavior revealed by `Error.prototype.stack`. It may be expanded in the future as new implementation leakages are discovered or added to the language.

This proposal is at stage 2 in the [TC39 process](https://tc39.github.io/process-document/).

## The problem

### `Function.prototype.toString`

JavaScript's `Function.prototype.toString` method reveals the source text originally used to create the function. Revealing the source text of a function gives callers unnecessary insight into its implementation details. They can introspect on a function's implementation and react to it, causing what would otherwise likely be non-breaking changes to become breaking ones. They can extract secret values from the source text to compromise an application's attempts at encapsulation.

A historical example of this in action is AngularJS's reliance on `f.toString()` to inspect the parameter names of a function. It would then use these as part of its dependency injection framework. This made what was previously a non-breaking change—modifying the name of a parameter—into a breaking one. In particular, it made it impossible to use minification tools in combination with this mode of AngularJS.

Another unnecessary insight gained by `f.toString()` is how a function was created. Most dramatically, it can be used to detect whether an implementation is "native" or not, by looking for the pattern of `[native code]`. This makes high-fidelity polyfilling difficult; indeed, some zealous polyfill libraries have gone as far as to replace `Function.prototype.toString` or introduce an own-property `polyfillFn.toString()` to prevent detection ([1](https://github.com/zloirock/core-js/blob/9f051803760c02b306aae2595621bb7ef698fc29/modules/_redefine.js#L28), [2](https://github.com/paulmillr/es6-shim/blob/8d7aec1403751686dbbd3c4fa13a7bb584a75bf3/es6-shim.js#L139)). This is only possible because engines have the magic implementation-hiding ability which is not available to developers. Finally, `f.toString()` can also be used to detect whether the implementation is done via a `class`, a `function`, an arrow function, a method, etc. It would be preferable if these were mostly-undetectable implementation details, allowing more confidence in refactoring between them.

### `Error.prototype.stack`

JavaScript's (non-standard though de facto) `Error.prototype.stack` getter reveals calling behavior in the presence/absence of a stack frame or, in the case of recusive functions, the frame count for a particular function. If this calling behaviour is dependent upon secret values, whether present in the function source text or just lexically available to the function, it can result in these secrets being partially or fully exposed. Additionally, `Error.prototype.stack` reveals position (line/column number) information that restricts refactoring in a similar way to exposing the source text itself.

## The solution

The solution is to provide a way to modify the output of the above functions, to prevent them from exposing implementation details. This is done via a new directive, tentatively `"hide implementation"`. Like `"use strict"`, it could be placed at either the source file level or the per-function level.

Similar to the `"use strict"` directive, this new directive would apply "inclusively downward", so that everything within the scope, plus the function itself when in function scope, gets hidden. For example:

```js
function foo() {
  const x = () => {};

  const y = () => {
    "hide implementation";
    class Z {
      m() {}
    }
  };
}
```

In this example, `foo` and `x` remain unhidden, while `y`, `Z`, and `Z.prototype.m` are considered hidden.

_Note: historically, folks have also explored out-of-band, application-level switches for hiding implementation details, often motivated by memory use. See [the appendix](#appendix-out-of-band-memory-saving-switches) for more discussion on that._

### Why a directive prologue?

This proposal draws heavily on the strengths of JavaScript's [existing directive prologue support](https://tc39.github.io/ecma262/#directive-prologue):

* It allows easy hiding of the implementation details of all functions in an entire source file.
* At the same time, it allows easy bundling together of hidden and non-hidden code, by using anonymous-function-wrapper blocks.
* It is backward-compatible, allowing easy deployment of code that makes a best-effort to hide its implementation. `"hide implementation"` will be a no-op in engines that do not implement this proposal.
* It is lexical, thus allowing tooling to easily and statically determine whether a function's implementation is hidden. For example, an inliner would know it should not inline an implementation-hidden function into an implementation-exposed function.

Notably, this proposal also introduces a directive prologue for class bodies, where they did not previously exist (presumably since they are always strict and `"use strict"` was the only meaningful built-in directive).

## Rejected alternatives

### A one-time hiding method

In this alternative, functions get a new method, `f.hide()`, which permanently hides the function's implementation details from `Function.prototype.toString` and `Error.prototype.stack`. You'd use it like so:

```js
function foo() {
  // ...
}

console.assert(foo.toString().includes("..."));

foo.hide();

console.assert(foo.toString() === "function foo() { [ native code ] }");
```

This alternative seems less good than the directive:

* It is difficult to en-masse hide many functions. The directive allows hiding an entire source file at once.
* You can now hide anyone's functions, not just ones that you created and control.
* It is non-lexical, thus requiring tools that operate on source code to rely on heuristics to tell if a function is implementation-hidden or not.
* In general, it makes this a property of the function, and not of the source text, which seems like the wrong level of abstraction, and harder to reason about.

### A clone-creating hiding method

The idea here is similar to the previous one, except that `f.hide()` returns a new, hidden function instead of modifying the function it is called on. The caller than needs to only hand out references to the clone, and not the original.

This suffers from three of the same drawbacks:

* It is difficult to en-masse hide many functions.
* It is non-lexical.
* It makes this a property of the function, and not of the source text.

and introduces a couple of new ones:

* Any "cloning" of functions is difficult to spec and reason about. See [this discussion of `toMethod()`](https://github.com/allenwb/ESideas/blob/master/dcltomethod.md#the-tomethod-solution), a proposal from the run up to ES2015. It similarly made function-clones-with-one-tweak, and was rejected by TC39 for its complexity.
* Some functions are not easy to replace with their clones. For example, methods need to be installed with their correct [[HomeObject]], which is not currently possible.

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

It's also a very blunt tool, usable only on the realm level, and thus probably only by application developers. The directive is targeted at library developers; application developers are better served by the out-of-band solution.

This proposal has since been expanded to include hiding of more than just `Function.prototype.toString` results, which makes solutions like this even less feasible.

## FAQs

### Should this hide the function name and length as well?

Some of the same arguments for hiding source text and stack trace information also apply to a function's `name` and `length` properties. The language already has a mechanism for hiding these via `Object.defineProperty`:

```js
function foo(a, b, c) { }
console.assert(foo.name === "foo");
console.assert(foo.length === 3);

Object.defineProperty(foo, "name", { value: "" });
Object.defineProperty(foo, "length", { value: 0 });
console.assert(foo.name === "");
console.assert(foo.length === 0);
```

See discussion on this topic in [#2](https://github.com/domenic/proposal-function-implementation-hiding/issues/2).

### Shouldn't this kind of meta-API involve a well-known symbol?

It's tempting, given APIs like `Symbol.isConcatSpreadable` or `Symbol.iterator`, to think that changing the behavior of an object with regard to some language feature should always be done by installing a well-known symbol onto the object.

This does not work very well for our use case. In particular, any kind of symbol flag would be reversible: e.g.

```js
function foo() { /* ... */ }
console.assert(foo.toString.includes("..."));

foo[Symbol.hideImplementation] = true;
console.assert(!foo.toString.includes("..."));

foo[Symbol.hideImplementation] = false;
console.assert(foo.toString.includes("...")); // oops
```

This basically makes the hiding toothless, so that you do not gain the encapsulation or memory usage benefits.

You could try to patch around it by saying that only setting it to `true` works, and setting it to `false` or deleting the property does nothing. But that's just strange. We already have a mechanism for changing the state of an object in a non-reversible way: call a method on it. Thus, the `foo.hide()` proposal above (which has its own problems).

### How does this affect devtools or other methods of inspecting a function's implementation details?

This proposal, like all JavaScript language proposals, only impacts the behavior of JavaScript code. Developer tools are outside the scope, and this proposal does not intend to change how they behave in any way. So, functions with their implementation hidden via the proposal's directive can still be inspected by devtools, if the devtools choose to expose this information.

Concretely, this proposal only impacts two things:

* The JavaScript string values produced by `Function.prototype.toString()`.
* The JavaScript values produced by not-yet-standardized properties like `Error.prototype.stack` or other parts of the [error stacks proposal](https://github.com/tc39/proposal-error-stacks).

The proposed specification text does _not_ impact any of the following:

* What the developer sees on their screen when they do `console.log(someFunction)` (which doesn't have to be related to JavaScript-exposed `.toString()` return values).
* What error stack traces the developer eventually sees on their screen when an unhandled exception or rejection occurs (which don't have to be related to JavaScript-exposed `.stack` strings).
* What the developer sees on their screen when they look at the source code or network-response panels of devtools.
* Whether the developer can set breakpoints or pause on uncaught exceptions inside functions.
* ... etc.

That said, developer tools teams are not governed by any language specification; they made independent decisions on their user experience. They could at any point decide that code from `example.com` gets hidden from devtools, or code that is minified gets hidden from devtools. Or, indeed, they may decide that code whose implementation is hidden from JavaScript via this proposal gets hidden from devtools. That is their decision, and no language specification can impact their UX. All we can say is that the proposal champions hope that devtools continue to make code maximally introspectable.

## Appendix: out-of-band memory-saving switches

Historically, there have been a number of ideas, motivations, and proposals in this space. In the January 2018 TC39 meeting, we realized that there were two related proposals:

* (This proposal) An encapsulation mechanism, applied in-band with the source text, especially suited for libraries.
* A memory-saving mechanism, applied out-of-band with the source text, especially suited for applications.

The idea behind the latter is that in order to allow `Function.prototype.toString` to give back the expected results, engines need to hold large amounts of data in memory: namely, the original source text of the application, and all its dependencies. The hope was that by allowing application authors to globally turn off source text storage, the applications would be able to consume less memory.

Since applications are boostrapped by a the host, this would most likely be done with host-specific mechanisms, e.g. a HTTP header, or a `<meta>` tag, or a Node.js command line flag. The host would then tie into the JavaScript language specification via [HostHasSourceTextAvailable](https://tc39.github.io/ecma262/#sec-hosthassourcetextavailable). See [previous revisions of this document](https://github.com/domenic/proposal-function-implementation-hiding/blob/134802869ce99933973e9b8c19d7fd99a92a352f/README.md#an-external-to-javascript-switch) for exploration of this space.

The conclusion of the January 2018 TC39 meeting was to split these efforts, focusing this proposal on the in-band mechanism, and separately pursuing out-of-band mechanisms with individual host environments. However, shortly afterward discussions with JavaScript engine implementers revealed the basic premise behind the memory-saving mechanism was flawed. It wouldn't, in fact, save memory! At least in several modern JavaScript engines, the source text needs to be retained anyway for lazy compilation. So, making it unavailable to `Function.prototype.toString` would not allow the engine to reclaim the memory after all.

As such, work on the out-of-band memory-saving switch for `Function.prototype.toString` has not progressed. It may pick up again, perhaps if engines change their lazy compilation techniques. And in that case, this appendix will be updated to direct you toward those efforts.
