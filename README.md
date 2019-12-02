# Function implementation hiding proposal

A proposal for a pair of new directives, tentatively `"sensitive"` and `"hide source"`, which provide a way for developers to indicate that certain implementation details should not be exposed to other user code. This has benefits for authors of library code who would like to refactor without fear of breaking consumers relying on their implementation details, authors of security-sensitive code, and authors of polyfills, among others.

In practice, the `"hide source"` directive hides the source text revealed by `Function.prototype.toString` and the file attribution and position information revealed by `Error.prototype.stack`. The `"sensitive"` directive hides the source text revealed by `Function.prototype.toString` and omits the function entirely from `Error.prototype.stack`. The `"sensitive"` directive is intended to be expanded in the future as new security-impacting information leakages are discovered or added to the language.

This proposal is at stage 2 in the [TC39 process](https://tc39.github.io/process-document/), and was last presented to the committee in [July, 2019](https://docs.google.com/presentation/d/1lWH97DxTLU3_1EJA-F19uIzagZQx7PZmys7WyNXw3cY/edit#slide=id.p).


## The problem

### `Function.prototype.toString`

JavaScript's `Function.prototype.toString` method reveals the source text originally used to create the function. Revealing the source text of a function gives callers unnecessary insight into its implementation details. They can introspect on a function's implementation and react to it, causing what would otherwise be harmless refactorings to become breaking changes. They can also extract secret values from the source text to compromise an application's attempts at encapsulation.

A historical example of this in action is AngularJS's reliance on `f.toString()` to inspect the parameter names of a function. It would then use these as part of its dependency injection framework. This made what was previously a non-breaking change &mdash; modifying the name of a parameter &mdash; into a breaking one. In particular, this kind of usage of `Function.prototype.toString` makes it impossible to use otherwise-semantics-preserving minification tools in combination with this mode of AngularJS.

Another unnecessary insight gained by `f.toString()` is how a function was created. Most dramatically, it can be used to detect whether an implementation is "native" or not, by looking for the pattern of `[native code]`. This makes high-fidelity polyfilling difficult; indeed, some zealous polyfill libraries have gone as far as to replace `Function.prototype.toString` or introduce an own-property `polyfillFn.toString()` to prevent detection ([1](https://github.com/zloirock/core-js/blob/9f051803760c02b306aae2595621bb7ef698fc29/modules/_redefine.js#L28), [2](https://github.com/paulmillr/es6-shim/blob/8d7aec1403751686dbbd3c4fa13a7bb584a75bf3/es6-shim.js#L139)). This is only possible because engines have an implementation-hiding capability which is not available to developers. Finally, `f.toString()` can also be used to detect whether the implementation is done via a `class`, a `function`, an arrow function, a method, etc. It would be preferable if these were mostly-undetectable implementation details, allowing more confidence in refactoring between them.

### `Error.prototype.stack`

JavaScript's (non-standard though de facto) `Error.prototype.stack` getter reveals calling behavior in the presence/absence of a stack frame or, in the case of recursive functions, the number of times a particular function appears in a stack trace. If this calling behaviour is dependent upon secret values, whether present in the function source text or just lexically available to the function, it can result in these secrets being partially or fully exposed. Additionally, `Error.prototype.stack` reveals position (line/column number) information and file attribution information that restricts refactoring in a similar way to `Function.prototype.toString`'s exposure of the function source text.


## The solution

The solution is to provide a way to modify the output of the above functions, to prevent them from exposing implementation details. This is done via the above-described new directives, tentatively `"hide source"` and `"sensitive"`. Like `"use strict"`, they can be applied to either an entire source file or per-function.

Similar to the `"use strict"` directive, these new directives apply "inclusively downward", so that everything within the scope, plus the function itself when in function scope, gets hidden. For example:

```js
function foo() {
  const x = () => {};

  const y = () => {
    "hide source";
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
* It is backward-compatible, allowing easy deployment of code that makes a best-effort to hide its implementation. `"hide source"` will be a no-op in engines that do not implement this proposal.
* It is lexical, thus allowing tooling to easily and statically determine whether a function's implementation is hidden. For example, an inliner would know it should not inline an implementation-hidden function into an implementation-exposed function.


## Examples

A stack trace:

```
$ node
Welcome to Node.js v12.13.0.
Type ".help" for more information.
> console.log((new Error).stack)
Error
    at repl:1:14
    at Script.runInThisContext (vm.js:116:20)
    at REPLServer.defaultEval (repl.js:404:29)
    at bound (domain.js:420:14)
    at REPLServer.runBound [as eval] (domain.js:433:12)
    at REPLServer.onLine (repl.js:715:10)
    at REPLServer.emit (events.js:215:7)
    at REPLServer.EventEmitter.emit (domain.js:476:20)
    at REPLServer.Interface._onLine (readline.js:316:10)
    at REPLServer.Interface._line (readline.js:693:8)
undefined
> 
```

The same stack trace if `bound` was marked as `"sensitive"`:

```
$ node
Welcome to Node.js v12.13.0.
Type ".help" for more information.
> console.log((new Error).stack)
Error
    at repl:1:14
    at Script.runInThisContext (vm.js:116:20)
    at REPLServer.defaultEval (repl.js:404:29)
    at REPLServer.runBound [as eval] (domain.js:433:12)
    at REPLServer.onLine (repl.js:715:10)
    at REPLServer.emit (events.js:215:7)
    at REPLServer.EventEmitter.emit (domain.js:476:20)
    at REPLServer.Interface._onLine (readline.js:316:10)
    at REPLServer.Interface._line (readline.js:693:8)
undefined
> 
```

The same stack trace if `bound` was marked as `"hide source"`:

```
$ node
Welcome to Node.js v12.13.0.
Type ".help" for more information.
> console.log((new Error).stack)
Error
    at repl:1:14
    at Script.runInThisContext (vm.js:116:20)
    at REPLServer.defaultEval (repl.js:404:29)
    at bound (<anonymous>)
    at REPLServer.runBound [as eval] (domain.js:433:12)
    at REPLServer.onLine (repl.js:715:10)
    at REPLServer.emit (events.js:215:7)
    at REPLServer.EventEmitter.emit (domain.js:476:20)
    at REPLServer.Interface._onLine (readline.js:316:10)
    at REPLServer.Interface._line (readline.js:693:8)
undefined
> 
```


## Rejected alternatives

### A one-time hiding function

In this alternative, we introduce new functions such as `Error.hideFromStackTraces` or `Function.prototype.hideSource`, which permanently opt the functions into one of the new hidden behaviours. You'd use it like so:

```js
function foo() {
  // ...
}

console.assert(foo.toString().includes("..."));

foo.hideSource();

console.assert(foo.toString() === "function foo() { [ native code ] }");
```

This alternative seems less good than the directive:

* It is difficult to en-masse hide many functions. The directive allows hiding an entire source file at once.
* You can now hide anyone's functions, not just ones that you created and control.
* It is non-lexical, thus requiring tools that operate on source code to rely on heuristics to tell if a function is implementation-hidden or not.
* In general, it makes this a property of the function, and not of the source text, which seems like the wrong level of abstraction, and harder to reason about.

### A clone-creating hiding function

The idea here is similar to the previous one, except that `f.hideSource()` returns a *new* hidden function instead of modifying the function it is called on. The caller then needs to only hand out references to the clone, and not the original.

This suffers from many of the same drawbacks:

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

### Using a well-known symbol

It's tempting, given APIs like `Symbol.isConcatSpreadable` or `Symbol.iterator`, to think that changing the behavior of an object with regard to some language feature should always be done by installing a well-known symbol onto the object.

This does not work very well for our use case. This approach suffers from all of the same issues as the "one-time hiding function" approach. Additionally, any kind of symbol marking would be reversible: e.g.

```js
function foo() { /* ... */ }
console.assert(foo.toString.includes("..."));

foo[Symbol.hideSource] = true;
console.assert(!foo.toString.includes("..."));

foo[Symbol.hideSource] = false;
console.assert(foo.toString.includes("...")); // oops
```

This basically makes the hiding toothless, so that you do not gain the desired encapsulation benefits. You could try to patch around it by saying that only setting it to `true` works, and setting it to `false` or deleting the property does nothing.


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

Because it is already possible to do and it would not be universally desirable for the target use cases of the `"hide source"` directive, this behaviour is not included with the directive(s). See discussion on this topic in [#2](https://github.com/domenic/proposal-function-implementation-hiding/issues/2).

### How does this affect devtools or other methods of inspecting a function's implementation details?

This proposal, like all JavaScript language proposals, only impacts the behaviour of JavaScript code. Developer tools are outside the scope, and this proposal does not intend to change how they behave in any way. So, functions with their implementation hidden via the proposal's directives can still be inspected by any privileged API, such as those used by devtools, if the API chooses to expose this information.

Concretely, this proposal only impacts two things:

* The JavaScript string values produced by `Function.prototype.toString()`.
* The JavaScript values produced by not-yet-standardized properties like `Error.prototype.stack` or other parts of the [error stacks proposal](https://github.com/tc39/proposal-error-stacks).

The proposed specification text does _not_ necessarily impact any of the following:

* What the developer sees on their screen when they do `console.log(function(){})` (which doesn't have to be related to the JavaScript-exposed `.toString()` result).
* What the developer sees on their screen when they do `console.log(new Error)` (which doesn't have to be related to the JavaScript-exposed `.stack` result).
* What error stack traces the developer eventually sees on their screen when an unhandled exception or rejection occurs (which don't have to be related to the JavaScript-exposed `.stack` result).
* What a devtools user sees on their screen when they look at the source code or network-response panels of devtools.
* Whether a devtools user can set breakpoints or pause on uncaught exceptions inside functions.
* ... etc.

That said, developer tools teams are not governed by any language specification; they made independent decisions on their user experience. They could at any point decide that code from `example.com` gets hidden from devtools, or code that is minified gets hidden from devtools. Or, indeed, they may decide that code whose implementation is hidden from JavaScript via this proposal gets hidden from devtools. That is their decision, and no language specification can impact their UX. All we can say is that the proposal champions hope that devtools continue to make code maximally introspectable.

### Will everyone just end up using this everywhere?

We don't think so. Unlike `"use strict"`, which generally makes your code better, `"hide source"` is a specialized mechanism for function authors which need very high levels of encapsulation and freedom to refactor.

Encapsulation is generally a sliding scale. Some library authors are content with underscore-prefixed properties, declaring that if a consumer depends on such properties, they might get broken. Some authors go further and use symbol-keyed properties to create a speed bump. Whereas others use `WeakMap`s or private fields, in order to ensure that consumers _cannot_ take a dependency on such implementation details, or probe into secret values stored within a class.

This proposal is in the vein of the latter scenario. It ensures consumers cannot use `Function.prototype.toString` or `Error.prototype.stack` to create refactoring-hostile dependencies, or to expose secret values. We believe these cases are important enough to deserve a proposal, but not ubiquitous enough to fear a sprinkling of directive prologues throughout every JavaScript file.

Finally, despite historical indications that this directive may provide memory savings (and thus get adoption from everyone who cares about memory, i.e., everyone), those indications have proven false. See [the appendix](#appendix-out-of-band-memory-saving-switches) for more on that. Even if the situation changes, we think the out-of-band mechanisms explored in the appendix will be a more attractive mechanism for realizing those memory savings, leaving this proposal focused on the more specialized encapsulation case.

### In what ways might the "sensitive" directive expand in the future?

The `"sensitive"` directive provides a declarative, textually-bounded opt-in with the end goal of making it more likely for JavaScript programmers of all skill levels to write security-sensitive code without introducing unwanted confidentiality violations. To understand the bounds, we must define which aspects of the program are given a confidentiality guarantee, and which parts of the program are considered "within" the confidentiality boundary. Confidentiality is provided for

1. the source text of the confidential region
1. the local bindings of the confidential region
	1. if, for example, the [Function.prototype.environment proposal](https://es.discourse.group/t/function-prototype-environment/108) made its way into the language, it would not be available for functions marked as `"sensitive"`
1. if the confidential region is a function, the calling behaviour of that function
	1. in practice, this means that the function does not appear in stack reflection mechanisms

Program code is considered within the confidentiality boundary if and only if it is textually within the confidential region. Code that is evaluated by a direct call to `eval` within the confidentiality boundary is also considered to be within the confidentiality boundary.

Note that limitations imposed on code outside of the confidentiality boundary of a confidential region may also apply to code within its confidentiality boundary due to language design/usability constraints. For example, a function marked as `"sensitive"` will not appear in stack frames, regardless of whether the code doing the reflection exists within the appropriate confidentiality boundary.

### Why is there no "preserve source" directive?

There may be a legitimate need for a function to be declared within the lexical closure of another function that includes a "hide source" directive, but to not inherit the implementation hiding behaviour of its ancestor. Most code should not require this, as the function declaration could be extracted to a function and the bindings it cares about passed in. But for cases where a direct call to `eval` is used alongside the kinds of reflection that are disabled by implementation hiding directives, and no assumptions can be made about the contents of the string being evaluated, it is not possible to textually extract the function declaration.

This is a niche use case, and may be evaluated for inclusion in the language in a later proposal.

## Appendix: out-of-band memory-saving switches

Historically, there have been a number of ideas, motivations, and proposals in this space. In the January 2018 TC39 meeting, we realized that there were two related proposals:

* (This proposal) An encapsulation mechanism, applied in-band with the source text, especially suited for libraries.
* A memory-saving mechanism, applied out-of-band with the source text, especially suited for applications.

The idea behind the latter is that in order to allow `Function.prototype.toString` to give back the expected results, engines need to hold large amounts of data in memory: namely, the original source text of the application, and all its dependencies. The hope was that by allowing application authors to globally turn off source text storage, the applications would be able to consume less memory.

Since applications are boostrapped by a the host, this would most likely be done with host-specific mechanisms, e.g. a HTTP header, or a `<meta>` tag, or a Node.js command line flag. The host would then tie into the JavaScript language specification via [HostHasSourceTextAvailable](https://tc39.github.io/ecma262/#sec-hosthassourcetextavailable). See [previous revisions of this document](https://github.com/domenic/proposal-function-implementation-hiding/blob/134802869ce99933973e9b8c19d7fd99a92a352f/README.md#an-external-to-javascript-switch) for exploration of this space.

The conclusion of the January 2018 TC39 meeting was to split these efforts, focusing this proposal on the in-band mechanism, and separately pursuing out-of-band mechanisms with individual host environments. However, shortly afterward discussions with JavaScript engine implementers revealed the basic premise behind the memory-saving mechanism was flawed. In engines that preserve the source text for lazy compilation, a directive to hide the source text from user land code wouldn't, in fact, save memory!

As such, work on the out-of-band memory-saving switch for `Function.prototype.toString` has not progressed. It may pick up again, perhaps if engines change their lazy compilation techniques. And in that case, this appendix will be updated to direct you toward those efforts.
