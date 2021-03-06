# This proposal has merged into [dynamic-code-brand-checks](https://github.com/mikesamuel/dynamic-code-brand-checks)

Relax the requirement that the argument to eval be a string in a non-breaking way.

## Background

Currently, `typeof x === 'string' || eval(x) === x`.

For example:

```js
console.log(eval(123) === 123);

const array = [ 'alert(1)' ];
console.log(eval(array) === array);  // Does not alert

const touchy = { toString() { throw new Error(); } };
console.log(eval(touchy) === touchy);  // Does not throw.
```

This follows from step 2 of the definition of [PerformEval](https://tc39.github.io/ecma262/#sec-performeval):

> ### Runtime Semantics: PerformEval ( *x*, *evalRealm*, *strictCaller*, *direct* )
> <p>The abstract operation PerformEval with arguments *x*, *evalRealm*, *strictCaller*, and *direct* performs the following steps:</p>
>
> 1. Assert: If *direct* is **false**, then *strictCaller* is also **false**.
> 1. If Type(*x*) **is not String**, return *x*.

## Goal

Encourage *source to sink* checking of arguments to `eval` to reduce the risk of XSS.

Source to sink checking means we do something at the source of a value so that, when the value
reaches a "sensitive sink" (like `eval`), we can double-check that it is safe.

The [Trusted Types][] proposal brings source-to-sink checks to browser built-ins
like `.innerHTML`.

> 1. Introduce a number of types that correspond to the XSS sinks we wish to protect.
>
> **TrustedScript**: This type would be used to represent a trusted JavaScript code block i.e.
> something that is trusted by the author to be executed by ... passing to an `eval` function family.
>
> 2. Enumerate all the XSS sinks we wish to protect. ...
>
> 3. Introduce a mechanism for disabling the raw string version of each of the sinks identified. ...
>
> ----
>
> As valid trusted type objects must originate from a policy, those policies alone form the trusted
> codebase in regards to DOM XSS.

So by double-checking at *sinks* like `eval` we can ensure that 
only carefully reviewed code that is the *source* of *TrustedScript*
values will load alongside trusted code.

This won't work today because, as explained above, `eval(myTrustedScript)` does not actually
evaluated the textual content of *TrustedScript* values.

## The Larger Context

This proposal is useful in the context of other planned changes:

1.  The [Trusted Types][] proposal as explained.
1.  A proposed [tweak to the HostEnsureCanCompileStrings][host callout proposal]
    that would allow browsers to take more context into account when deciding
    whether to allow a particular call to `eval`.

## Backwards compatibility constraints

To avoid breaking the web, `eval(x)` should do nothing different when x is a
non-string value that existing applications might create.

This should be the case regardless of whether HostEnsureCanCompileStrings says
yes or no as demonstrated in node:

```sh
$ npx node@10 --disallow_code_generation_from_strings -e 'console.log(eval(() => {}))'
[Function]

$ npx node@10 --disallow_code_generation_from_strings -e 'console.log(eval("() => {}"))'
[eval]:1
console.log(eval("() => {}"))
        ^

EvalError: Code generation from strings disallowed for this context
```

When `eval` is passed a non-string value, even though the Host callback rejects all
attempts to `eval`, it doesn't throw when given the value `() => {}`, but does when
given the string `"() => {}"`.

## Possible solutions

You can browse the [ecmarkup output](https://mikesamuel.github.io/evalable/)
or browse the [source](https://github.com/mikesamuel/evalable/blob/master/spec.emu).

All the approaches below do the following

*  Define IsCodeLike(*x*), a spec abstraction that returns true for strings
   so is backwards compatible, and returns true for some other values.
   (See details of specific proposals below.)
*  Tweak PerformEval, which is called by both direct and indirect `eval`, to
   use IsCodeLike(*x*) instead of the existing (Type(*x*) is String).
*  Use ToString(*x*) for the code to parse.

### Internal slot

*  IsCodeLike(*x*) additionally returns true when *x* is an object that
   has an internal slot \[\[CodeLike\]\].

#### Pros

*  IsCodeLike has no observable side effect for values so is backwards
   compatible when a program produces no code like values.

### Well-known Symbol

From [ecma262 issue #938](https://github.com/tc39/ecma262/issues/938#issuecomment-457352474):

> # Include the string to be compiled in the call to
> `HostEnsureCanCompileStrings` Perhaps we could tweak the `eval` algo
> so that if the input is a string or has a particular symbol property
> then it is stringified to avoid breaking code that depends on the
> current behavior where `eval` is identity except for strings.

*  Define a well-known symbol, *\@\@evalable*.
*  IsCodeLike(*x*) additionally returns true when *x* is an object and
   `x[Symbol.evalable]` is truthy.
   Note: using a new symbol makes IsCodeLike backwards comparible for programs
   that do not use `Symbol.evalable`.

#### Pros

*  Code-like values are proxyable.  (TODO: is this a feature?)
*  User code can define code-like values.  (TODO: is this a feature?)
*  Can be tested in test262 via user code.


## Testing

TODO: Updates to

*  https://github.com/tc39/test262/blob/master/test/language/eval-code/direct/non-string-object.js
*  https://github.com/tc39/test262/blob/master/test/language/eval-code/indirect/non-string-object.js


## Related

https://github.com/tc39/ecma262/issues/1495


[Trusted Types]: (https://wicg.github.io/trusted-types)
[host callout proposal]: https://github.com/mikesamuel/proposal-hostensurecancompilestrings-passthru
