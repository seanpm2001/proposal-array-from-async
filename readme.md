# Smart pipelines
ECMAScript Stage-−1 Proposal by J. S. Choi, 2018-02.

This repository contains the formal specification for a proposed “smart pipe operator” `|>` in JavaScript. It is currently not even in Stage 0 of the [TC39 process](https://tc39.github.io/process-document/) but it may eventually be presented to TC39.

## Background
The operator is a backwards-compatible way of chaining nested expressions in a readable, left-to-right manner. Nested transformations become untangled into short steps. It is similar to [Hack’s `|>` and `$$`](https://docs.hhvm.com/hack/operators/pipe-operator), [F#’s `|>`](https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/functions/index#function-composition-and-pipelining), [OCaml’s `|>`](http://blog.shaynefletcher.org/2013/12/pipelining-with-operator-in-ocaml.html), [Elixir/Erlang’s `|>`](https://elixir-lang.org/getting-started/enumerables-and-streams.html), [Elm’s `|>`](http://elm-lang.org/docs/syntax#infix-operators), [Julia’s `|>`](https://docs.julialang.org/en/stable/stdlib/base/#Base.:|>), [LiveScript’s `|>`](http://livescript.net/#operators-piping), [R / magrittr’s `%>%`](https://cran.r-project.org/web/packages/magrittr/index.html), [Perl 6’s `==>`](https://docs.perl6.org/language/operators#infix_==&gt;) and [Unix shells’ and PowerShell’s `|`](https://en.wikipedia.org/wiki/Pipeline_(Unix)). It is conceptually similar but should not be confused with [WHATWG-stream piping](https://streams.spec.whatwg.org/#pipe-chains) and [Node-stream piping](https://nodejs.org/api/stream.html#stream_readable_pipe_destination_options).

The proposal is a variant of a [previous pipe-operator proposal](https://github.com/tc39/proposal-pipeline-operator) championed by [Daniel Ehrenberg of Igalia](https://github.com/littledan). This variant is listed as [Proposal 4: Smart Mix on the pipe-proposal wiki](https://github.com/tc39/proposal-pipeline-operator/wiki#proposal-4-smart-mix). The variant resulted from [previous discussions about pipeline placeholders in the previous pipe-operator proposal](https://github.com/tc39/proposal-pipeline-operator/issues?q=placeholder), which culminated in an [invitation by Daniel Ehrenberg, champion on the current pipe proposal, to try writing a specification draft](https://github.com/tc39/proposal-pipeline-operator/issues/89#issuecomment-363853394). A prototype Babel plugin is also pending.

You can take part in the discussions on the [GitHub issue tracker](https://github.com/tc39/proposal-dynamic-import/issues). When you file an issue, please note in it that you are talking specifically about “Proposal 4: Smart Mix”.

## Motivation
Let these two functions be defined:

```js
function doubleSay (string) {
  return `${string}, ${string}`
}

function capitalize (string) {
  return string[0].toUpperCase() + string.substring(1)
}
```

This nested expression is quite messy. It is spaghetti that requires many levels of indentation. Reading this code requires checking both the left and right of each subexpression to understand the flow of data.

```js
capitalizedString(
  doubledString(
    (await stringPromise)
      ?? throw new TypeError(`Expected string from ${stringPromise}`)
  )
) + '!'
```

## Proposed solution
The code above would become much terser with a binary operator that allows the piping of data through expressions. This terseness would make both reading and writing easier for the JavaScript programmer.

```js
stringPromise
  |> await #
  |> # ?? throw new TypeError()
  |> doubleSay  // a tacit unary function call
  |> capitalize // also a tacit unary function call
  |> # + '!'
```

This same use case appears numerous times in JavaScript code, whenever any value is transformed by expressions of any type: function calls, property calls, method calls, object constructions, arithmetic operations, logical operations, bitwise operations, `typeof`, `instanceof`, `await`, `yield` and `yield *`, and `throw` expressions.

## Nomenclature
The binary operator itself may be referred to as a <dfn>pipe</dfn>, a <dfn>pipe operator</dfn>, or a <dfn>pipeline operator</dfn>; all these names are equivalent. This specification will prefer the term “pipe operator”.

A pipe operator between two expressions forms a <dfn>pipe expression</dfn>. One or more pipe expressions in a chain form a <dfn>pipeline</dfn>. For each pipe expression, the expression before the pipe is the pipeline’s <dfn>left-hand side (LHS)</dfn>; the expression after the pipe is its <dfn>right-hand side (RHS)</dfn>. The pipe operator is said <dfn>to pipeline</dfn> the LHS’s value <dfn>through</dfn> the RHS expression, where “pipeline” here is used as a verb.

The special token `#` is a nullary operator that acts as a special variable. A pipeline’s RHS forms an inner lexical scope—called the pipeline’s <dfn>RHS scope</dfn>—within which `#` is implicitly bound to the value of the LHS.

[TO DO: Define RHS type, tacit expression, and placeholder expression.]

## General semantics
A pipe expression’s semantics are:

1. The LHS is evaluated into the LHS’s value; call this `lhsValue`.

2. The RHS is tested for its type: is it a a tacit function, a tacit constructor, placeholder expression, or an invalid RHS? [TO DO: Link to section on RHS types and tacit expressions.]

  * If the RHS is a tacit function (such as `f` and `M.f`):
    1. The RHS is evaluated (in the current lexical context); call this value the `rhsFunction`.
    2. The entire pipeline’s value is `rhsFunction(lhsValue)`.

  * If the RHS is a tacit constructor (such as `new C` and `new M.C`):
    1. The portion of the RHS after `new` is evaluated (in the current lexical context); call this value the `RHSConstructor`.
    2. The entire pipeline’s value is `new RHSConstructor(lhsValue)`.

  * If the RHS is a placeholder expression (such as `f(#, n)`, `o[s](#)`, `# + n`, and `#.p`):
    1. An inner lexical context is entered.
    2. Within this inner context, `lhsValue` is bound to a variable.
    3. Within this inner context, with the variable binding to `lhsValue`, the RHS is evaluated; call the resulting value `rhsValue`.
    4. The entire pipeline’s value is `rhsValue`.

  * Otherwise, if the RHS is an invalid RHS (such as `f()`, `f(n)`, `o[s]`, `+ n`, and `n`), then throw a syntax error explaining that the placeholder is missing from the pipeline.

The placeholder expression case can be further explained with a nested `do` expression. There are two ways to illustrate this equivalency. The first way is to replace each pipe expression’s placeholders with an autogenerated variable, which must be guaranteed to not conflict with other variables. The alternative way is to use two variables: the placeholder `#` and a dummy variable. [TO DO: Link to both sections.]

### Placeholder semantics by replacing placeholders with autogenerated variables
The first way to illustrate the operator’s semantics is to replace each pipe expression’s placeholders with an autogenerated variable, which must be guaranteed to not conflict with other variables.

Let us say that each pipe expression autogenerates a new variable (`#₀`, `#₁`, `#₂`, `#₃`, …), which in turn replaces the placeholders `#` in each pipe’s RHS. (These `#ₙ` variables are not true syntax; it is merely for illustrative purposes. You cannot actually assign or use `#ₙ` variables.) Let us also group the expressions with left associativity (although this is arbitrary [TO DO: Link to associativity section when written]).

With this notation, each line in this example would be equivalent to the other lines.

```js
1 |> # + 2 |> # * 3

// Static rewriting
(1 |> # + 2) |> # * 3
do { const #₀ = 1; # + 2 } |> # * 3
do { const #₁ = do { const #₀ = 1; # + 2 }; #₁ * 3 }

// Runtime evaluation
do { const #₀ = do { 1 + 2 }; #₀ * 3 }
do { const #₀ = 3; #₀ * 3 }
do { do { 3 * 3 } }
9
```

Consider also the motivating first example above:

```js
stringPromise
  |> await #
  |> # ?? throw new TypeError()
  |> doubleSay // a tacit unary function call
  |> capitalize // also a tacit unary function call
  |> # + '!'
```

With left associativity, this would be statically equivalent to the following:

```js
do {
  const #₃ = do {
    const #₂ = do {
      const #₁ = do {
        const #₀ = await stringPromise;
        #₀ ?? throw new TypeError()
      };
      doubleSay(#₁)
    };
    capitalize(#₂)
  };
  #₃ + '!'
}
```

In general, for each placeholder pipe expression `lhs |> rhs`, assuming that `rhs` is a valid placeholder-containing expression:

* Let `#ₙ` be autogenerated placeholder variable, `#ₙ`, where <var>n</var> is a number that would not conflict with the name of any other autogenerated placeholder variable in the scope of the pipe expression.
* Also let `rhsSubstituted` be `rhs` but with all instances of `#` replaced with `#ₙ`.
* Then the static syntax rewriting (left associative and inside to outside) would simply be: `do { const #ₙ = lhs; rhsSubstituted }`.

### Placeholder semantics by alternating lexical shadowing with dummy variable
The other way to demonstrate placeholder expressions is to use two variables: the placeholder `#` and a dummy variable `•`. It should be noted that `const # = …` is not a valid statement under this proposal’s actual syntax; likewise, `•` is not a part of the proposal’s syntax. Both forms are for illustrative purposes here only.

With this notation, no variable autogeneration is required; instead, the nested `do` expressions will redeclare the same variables `#` and `•`, shadowing the external variables of the same name as needed. The number example above becomes the following. Each line is still equivalent to the other lines.

```js
1 |> # + 2 |> # * 3

// Static rewriting
(1 |> # + 2) |> # * 3
do { const • = 1; do { const # = •; # + 2 } } |> # * 3
do { const • = (do { const • = 1; do { const # = •; # + 2 } }); do { const # = •; # * 3 } }

// Runtime evaluation
do { const • = do { do { const # = 1; # + 2 } }; do { const # = •; # * 3 } }
do { const • = do { do { const 1 + 2 } }; do { const # = •; # * 3 } }
do { const • = 3; do { const # = •; # * 3 } }
do { do { const # = 3; # * 3 } }
do { do { 3 * 3 } }
9
```

Consider also the motivating first example above:
```js
stringPromise
  |> await #
  |> # ?? throw new TypeError()
  |> doubleSay // a tacit unary function call
  |> capitalize // also a tacit unary function call
  |> # + '!'
```

With left associativity, this would be statically equivalent to the following:
```js
do {
  const • = do {
    const • = do {
      const • = do {
        const • = await stringPromise;
        do { const # = •; # ?? throw new TypeError() }
      };
      do { const # = •; doubleSay(#) }
    };
    do { const # = •; capitalize(#) }
  };
  do { const # = •; # + '!' }
}
```

For each pipe expression, evaluated left associatively and inside to outside, the steps of the computation would be:

1. The LHS expression is first evaluated in the current lexical context.
2. The LHS’s result is bound to a hidden special variable `•`.
3. In a new inner lexical context, the value of `•` is bound to the placeholder variable `#`.
4. The pipe’s RHS expression is evaluated within this inner lexical context.
5. The pipe’s result is the result of the RHS.

## Tacit unary function calls
As a further abbreviation, you may omit the placeholder if the operation is just a call of a named unary function. This is called [“tacit” or “point-free style”](https://en.wikipedia.org/wiki/Tacit_programming). This is the “smart” part of these “smart pipeline operators”.

If the RHS expression is merely an identifier (as with `… |> doubleSay` or `… |> capitalize`), possibly in a property chain (<i lang=lt>e.g.</i>, `… |> Symbol.for`), and possibly with a `new` operator (<i lang=lt>e.g.</i>, `… |> new global.Date`), then that identifier is assumed to be a unary function or unary constructor, which is then called with `#` (<i lang=lt>i.e.</i>, `… |> doubleSay(#)` or `… |> capitalize(#)`). For example:

```js
'hello'
  |> await #
  |> # ?? throw new TypeError(`Expected string from ${#}`)
  |> doubleSay
  |> capitalize
  |> # + '!'
```

…is equivalent to:
```js
'hello'
  |> await #
  |> # ?? throw new TypeError(`Expected string from ${#}`)
  |> doubleSay(#)
  |> capitalize(#)
  |> # + '!'
```

Tacit style is not permitted with expressions more complex than single identifiers or simple property chain with no method calls.

If a pipe RHS has *no* placeholder, then it must be a permitted tacit unary function (single identifier or simple property chain). Otherwise, it is a syntax error. In particular, tacit function calls *never* have parentheses. If they need to have parentheses, then they need to have a placeholder.

| Expression                            | Result                          |
| ------------------------------------- | ------------------------------- |
| **`'2018' \|> Date(#)`**              | **`Date('2018')`**              |
| **`'2018' \|> Date`**                 | **`Date('2018')`**              |
|   `'2018' \|> Date()`                 |   syntax error: missing `#`     |
|   `'2018' \|> (Date)`                 |   syntax error: missing `#`     |
| **`'2018' \|> Date.parse(#)`**        | **`Date.parse('2018')`**        |
| **`'2018' \|> Date.parse`**           | **`Date.parse('2018')`**        |
|   `'2018' \|> Date.parse()`           |   syntax error: missing `#`     |
|   `'2018' \|> (Date.parse)`           |   syntax error: missing `#`     |
| **`'2018' \|> global.Date.parse(#)`** | **`global.Date.parse('2018')`** |
| **`'2018' \|> global.Date.parse`**    | **`global.Date.parse('2018')`** |
|   `'2018' \|> global.Date.parse()`    |   syntax error: missing `#`     |
|   `'2018' \|> (global.Date.parse)`    |   syntax error: missing `#`     |
| **`'2018' \|> new global.Date(#)`**   | **`new Date('2018')`**          |
| **`'2018' \|> new global.Date`**      | **`new Date('2018')`**          |
|   `'2018' \|> new global.Date()`      |   syntax error: missing `#`     |
|   `'2018' \|> (new global.Date)`      |   syntax error: missing `#`     |

This rule minimizes the parsing lookahead that the compiler must check before it can distinguish between tacit style and placeholder style. By restricting the space of valid tacit RHS expressions without placeholders, the rule prevents [garden-path syntax](https://en.wikipedia.org/wiki/Garden_path_sentence) that would otherwise be possible: <i lang=lt>e.g.</i>, `… |> compose(f, g, h, i, j, k, #)`.

The rule also statically prevents a writing JavaScript programmer from accidentally omitting a placeholder where they meant to put one. For instance, if `x |> 3` were not a syntax error, then it would be a useless operation and almost certainly not what the writer intended. The JavaScript programmer is encouraged to use placeholders and avoid tacit style, where tacit style may be visually confusing to the reader.

## Multiple placeholders in RHS
The placeholder may be used multiple times in the RHS. Each use refers to the same value. Because it is bound to the result of the LHS, the LHS is still only ever evaluated once.

```js
… |> f(#, #)
// equivalent to:
// do { const # = …; f(#, #) }
```

```js
… |> [#, # * 2, # * 3]
// equivalent to:
// do { const # = …; [#, # * 2, # * 3] }
```

## Loose precedence
The pipe operator’s precedence is quite loose. It is tighter than assignment (`=`, `+=`, …), generator `yield` and `yield *`, and sequence `,`; and it is looser than logical ternary conditional (`… ? … : …`), logical and/or `&&`/`||`, bitwise and/or/xor, `&`/`|`/`^`, equality/inequality `==`/`===`/`!=`/`!==`, and every other type of expression.

Being any tighter than this level would require its RHS to be parenthesized for many frequent types of expressions. However, the result of a pipeline is also expected to often serve as the RHS of a variable assignment `=`.

## Inner functions
Both the LHS and the RHS of a pipe may contain nested inner functions. This works as may be expected:

[TO DO]

## Bidirectional associativity
The pipe operator is presented above as a left-associative operator. However, it is theoretically [bidirectionally associative](https://en.wikipedia.org/wiki/Associative_property): how a pipeline’s expressions are particularly grouped is arbitrary. One could force right associativity by parenthesizing a pipeline, such that it itself becomes the RHS of another, outer pipeline.

[TO DO]

<!--
There are two **open questions** that are equivalent:

* Should the pipe operator be required to be left associative or may it be [bidirectionally associative](https://en.wikipedia.org/wiki/Associative_property)?
* Should placeholders be forbidden in a pipeline’s LHS, unless that placeholder is within the RHS of another pipeline within that LHS?

The pipe operator is currently specified to have left associativity. It is easiest to interpret the pipe operator as left associative; the discussion above interprets the operator using left associativity.

Relatedly, placeholders are *not* allowed in a pipeline’s LHS, unless the placeholders are within the RHS of a pipeline within the LHS. This is true even when there is a surrounding outer pipeline that would have made `#` resolve.

For example, `/*A*/ 1 |> (() => /*B*/ # |> # + 2)` is not allowed, because the inner function’s inner pipe (`/*B*/`)’s LHS’s `#` is not within the RHS, despite . But `(1 |> #) |> # + 2` is allowed.

Theoretically, the pipe operator could be [bidirectionally associative](https://en.wikipedia.org/wiki/Associative_property), in which the grouping of a chained pipeline would be arbitrary. Assuming this hypothetical, one could therefore force right associativity by parenthesizing a pipeline, then placing it in the RHS of another, outer pipeline.

However, the reason why this might be unnatural in JavaScript may be observed by comparing the syntactic expansions of left associativity and right associativity.

Consider the valid `(1 |> # + 2) |> # * 3` versus the invalid `1 |> (# + 2 |> # * 3)`.

```js
// With left associativity.
(1 |> # + 2) |> # * 3
(do { const # = 1; # + 2 }) |> # * 3
do { const # = (do { const # = 1; # + 2 }); # * 3 }
do { const # = (do { 1 + 2 }); # * 3 }
do { const # = 3; # * 3 }
do { 3 * 3 }
9
```

```js
// With right associativity.
1 |> (# + 2 |> # * 3)
1 |> do { const # = # + 2; # * 3 }
do { const # = 1; do { const # = # + 2; # * 3 } }
// ReferenceError: Cannot access uninitialized variable.
```

The reason why the right-associative expansion would be invalid is because variable declarations shadow outer variables of the same name *no matter where they are declared in the inner context*; that’s how the static analysis of variables works in JavaScript. At the point of the inner `do` block’s `const # = # + 2`, that inner block’s `#` has already shadowed the outer block’s `#` with an uninitialized variable. To put it in terms of [IIFEs](https://developer.mozilla.org/en-US/docs/Glossary/IIFE):

```js
// Returns 9.
(function () {
  const $ = (function () {
    const $ = 1;
    return $ + 2
  })();
  return $ * 3
})()
```

```js
// Throws ReferenceError: Cannot access uninitialized variable.
(function () {
  const $ = 1;
  return (function () {
    const $ = $ + 2;
    return $ * 3
  })()
})()
```

It should be noted that the `#` does not have to act this way. Indeed, in other languages such as Clojure, a lexical constant may be redeclared with the same name as a constant from an outer lexical context, yet its assignment may depend on that outer context’s constant. This may be simulated in JavaScript using *double nested `do` expressions* using dummy variables, which in turn would enable the use of placeholders in LHSes.

This may be demonstrated using a dummy placeholder `•`. If the transformation above of `(1 |> # + 2) |> # * 3`:

```js
// With left associativity.
(1 |> # + 2) |> # * 3
(do { const # = 1; # + 2 }) |> # * 3
do { const # = (do { const # = 1; # + 2 }); # * 3 }
do { const # = (do { 1 + 2 }); # * 3 }
do { const # = 3; # * 3 }
do { 3 * 3 }
9
```

…was instead written with double nested `do` expressions with a dummy placeholder `•`, then it would be equivalent:

```js
// With left associativity.
(1 |> # + 2) |> # * 3
(do { const • = 1; do { const # = •; # + 2 } }) |> # * 3
do { const • = (do { const # = 1; # + 2 }); do { const # = •; # * 3 } }
do { const • = (do { 1 + 2 }); do { const # = •; # * 3 } }
do { const • = 3; do { const # = •; # * 3 } }
do { do { const # = 3; # * 3 } }
do { do { 3 * 3 } }
9
```

But if the transformation above of `(1 |> # + 2) |> # * 3`:

```js
// With right associativity.
1 |> (# + 2 |> # * 3)
1 |> do { const # = # + 2; # * 3 }
do { const # = 1; do { const # = # + 2; # * 3 } }
// ReferenceError: Cannot access uninitialized variable.
```

…was similarly rewritten with double nested `do` expressions with a dummy placeholder `•`, then it would become valid (and equivalent):

```js
// With right associativity.
1 |> (# + 2 |> # * 3)
1 |> do { const • = # + 2; do { const # = •; # * 3 } }
do { const • = 1; do { const # = •; do { const • = # + 2; do { const # = •; # * 3 } } } }
do { do { const # = 1; do { const • = # + 2; do { const # = •; # * 3 } } } }
do { do { do { const • = 1 + 2; do { const # = •; # * 3 } } } }
do { do { do { do { const # = 3; # * 3 } } } }
do { do { do { do { 3 * 3 } } } }
9
```
 -->

## Alternative solutions explored
There are a number of other ways of potentially accomplishing the above use cases. However, the authors of this proposal believe that the smart pipe operator may be the best choice. [TO DO]

## Relation to existing work
[TO DO]

