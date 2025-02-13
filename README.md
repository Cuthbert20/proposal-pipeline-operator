# ESNext Proposal: The Pipeline Operator

This proposal introduces a new operator `|>` similar to
  [F#](https://en.wikibooks.org/wiki/F_Sharp_Programming/Higher_Order_Functions#The_.7C.3E_Operator),
  [OCaml](http://caml.inria.fr/pub/docs/manual-ocaml/libref/Pervasives.html#VAL%28|%3E%29),
  [Elixir](https://hexdocs.pm/elixir/Kernel.html#%7C%3E/2),
  [Elm](https://edmz.org/design/2015/07/29/elm-lang-notes.html),
  [Julia](https://docs.julialang.org/en/v1/base/base/#Base.:|%3E),
  [Hack](https://docs.hhvm.com/hack/expressions-and-operators/pipe),
  and [LiveScript](http://livescript.net/#piping),
  as well as UNIX pipes. It's a backwards-compatible way of streamlining chained function calls in a readable, functional manner, and provides a practical alternative to extending built-in prototypes.

***

**⚠ Warning**: The details of the pipeline syntax are **currently unsettled**. There are [**two competing proposals**](https://github.com/tc39/proposal-pipeline-operator/wiki) under consideration. This readme is a minimal proposal, which covers the basic features of the pipeline operator. It functions as a strawman for comparing the tradeoffs of the competing proposals.

Those proposals are as follows:

* **F# Pipelines**: [**Explainer**](https://github.com/valtech-nyc/proposal-fsharp-pipelines/blob/master/README.md) + [**Specification**](https://valtech-nyc.github.io/proposal-fsharp-pipelines/)
* **Smart Pipelines**: [**Explainer**](https://github.com/js-choi/proposal-smart-pipelines/blob/master/readme.md) + [**Specification**](https://jschoi.org/18/es-smart-pipelines/spec)

[Babel plugins](https://github.com/tc39/proposal-pipeline-operator/issues/89#issuecomment-363853394) for are already underway to gather feedback.

See also the [**latest presentation to TC39**](https://docs.google.com/presentation/d/1eFFRK1wLIazIuK0F6fY974OIDvvWXS890XAMB59PUBA/edit#slide=id.p) as well as [**recent GitHub issues**](https://github.com/tc39/proposal-pipeline-operator/issues?utf8=✓&q=is%3Aissue+sort%3Aupdated-desc+) for more information.

***

## Introduction

The pipeline operator is essentially a useful syntactic sugar on a function call with a single argument. In other words, `sqrt(64)` is equivalent to `64 |> sqrt`.

This allows for greater readability when chaining several functions together. For example, given the following functions:

```js
function doubleSay (str) {
  return str + ", " + str;
}
function capitalize (str) {
  return str[0].toUpperCase() + str.substring(1);
}
function exclaim (str) {
  return str + '!';
}
```

...the following invocations are equivalent:

```js
let result = exclaim(capitalize(doubleSay("hello")));
result //=> "Hello, hello!"

let result = "hello"
  |> doubleSay
  |> capitalize
  |> exclaim;

result //=> "Hello, hello!"
```

### Functions with Multiple Arguments

The pipeline operator does not need any special rules for functions with multiple arguments; JavaScript already has ways to handle such cases.

For example, given the following functions:

```js
function double (x) { return x + x; }
function add (x, y) { return x + y; }

function boundScore (min, max, score) {
  return Math.max(min, Math.min(max, score));
}
```

...you can use an arrow function to handle multi-argument functions (such as `add`):

```js
let person = { score: 25 };

let newScore = person.score
  |> double
  |> (_ => add(7, _))
  |> (_ => boundScore(0, 100, _));

newScore //=> 57

// As opposed to: let newScore = boundScore( 0, 100, add(7, double(person.score)) )
```

*Note: The use of underscore `_` is not required; it's just an arrow function, so you can use any parameter name you like.*

As you can see, because the pipe operator always pipes a single result value, it plays very nicely with the single-argument arrow function syntax. Also, because the pipe operator's semantics are pure and simple, it could be possible for JavaScript engines to optimize away the arrow function.

### Use of `await`

The current minimal proposal makes `|> await f` an early error, so there is no support currently for `await` in the pipeline. Each proposal has a different solution to `await` in a pipeline, so support is planned. Please see the respective proposals for their solutions.

### Usage with `?` partial application syntax

If the [partial application proposal](https://github.com/rbuckton/proposal-partial-application) (currently a [stage 1 proposal](https://github.com/rbuckton/proposal-partial-application)) gets accepted, the pipeline operator would be even easier to use. We would then be able to rewrite the previous example like so:

```js
let person = { score: 25 };

let newScore = person.score
  |> double
  |> add(7, ?)
  |> boundScore(0, 100, ?);

newScore //=> 57
```

## Motivating Examples

### Transforming Streams/Iterables/AsyncIterables/Observables

There is native syntax for creating Iterables/AsyncIterables:

```js
function* numbers() {
  yield 1
  yield 2
  yield 3
}
```

and for consuming them:

```js
for (const number of numbers()) {
  console.log(number)
}
```

But no syntax yet for transforming an Iterable or stream-like object to another. The pipeline operator allows processing of any Iterable or stream-like object in an easy-to-read multi-step pipeline:

```js
const filter = predicate => function* (iterable) {
  for (const value of iterable) {
    if (predicate(value)) {
      yield value
    }
  }
}

const map = project => function* (iterable) {
  for (const value of iterable) {
    yield project(value)
  }
}

numbers()
  |> filter(x => x % 2 === 1)
  |> map(x => x + x)
```

Popular libraries like [RxJS](https://github.com/ReactiveX/rxjs) currently simulate this through the [`.pipe()` method](https://rxjs-dev.firebaseapp.com/api/index/function/pipe):

```js
Observable.from([1, 2, 3]).pipe(
  filter(x => x % 2 === 1),
  map(x => x + x)
)
```

This works quite nicely with RxJS, but native stream objects like [`Iterable`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols#The_iterable_protocol), [`AsyncIterable`](https://github.com/tc39/proposal-async-iteration#async-iterators-and-async-iterables), [`Observable`](https://github.com/tc39/proposal-observable) etc. do not have a `.pipe()` method and therefore need to be wrapped with a library like RxJS or IxJS first.

The pipeline operator makes it possible to easily transform these stream representations in a functional way without having to depend on a library like RxJS or IxJS.

For users of these libraries, it saves boilerplate code, especially when having to interop with native stream representations or other libraries: In the example above, `map` and `filter` can come from a library like IxJS and be applied on `numbers()` without having to wrap `numbers()` in a library-specific object first like `Ix.Iterable.from(numbers()).pipe(...)`.

For WHATWG and Node streams (which have a pipe method) it allows transformation with simple functions instead of through more complicated `TransformStream` objects, with easier error handling (Node's `pipe()` does not forward errors). Since Node 10 `ReadableStream` also implements `AsyncIterable` so any transformation function written for AsyncIterables can directly work with NodeJS streams, just like any transformation function written for Iterables can work with arrays or Sets.

### Object Decorators

Mixins via `Object.assign` are great, but sometimes you need something more advanced. A **decorator function** is a function that receives an existing object, adds to it (mutative or not), and then returns the result.

Decorator functions are useful when you want to share behavior across multiple kinds of objects. For example, given the following decorators:

```js
function greets (person) {
  person.greet = () => `${person.name} says hi!`;
  return person;
}
function ages (age) {
  return function (person) {
    person.age = age;
    person.birthday = function () { person.age += 1; };
    return person;
  }
}
function programs (favLang) {
  return function (person) {
    person.favLang = favLang;
    person.program = () => `${person.name} starts to write ${person.favLang}!`;
    return person;
  }
}
```

...you can create multiple "classes" that share one or more behaviors:

```js
function Person (name, age) {
  return { name: name } |> greets |> ages(age);
}
function Programmer (name, age) {
  return { name: name }
    |> greets
    |> ages(age)
    |> programs('javascript');
}
```

### Validation

Validation is a great use case for pipelining functions. For example, given the following validators:

```js
function bounded (prop, min, max) {
  return function (obj) {
    if ( obj[prop] < min || obj[prop] > max ) throw Error('out of bounds');
    return obj;
  };
}
function format (prop, regex) {
  return function (obj) {
    if ( ! regex.test(obj[prop]) ) throw Error('invalid format');
    return obj;
  };
}
```

...we can use the pipeline operator to validate objects quite pleasantly:

```js
function createPerson (attrs) {
  attrs
    |> bounded('age', 1, 100)
    |> format('name', /^[a-z]$/i)
    |> Person.insertIntoDatabase;
}
```

### Usage with Prototypes

Although the pipe operator operates well with functions that don't use `this`, it can still integrate nicely into current workflows:

```js
import Lazy from 'lazy.js'

getAllPlayers()
  .filter( p => p.score > 100 )
  .sort()
|> (_ => Lazy(_)
  .map( p => p.name )
  .take(5))
|> (_ => renderLeaderboard('#my-div', _));
```

### Mixins

["Real" Mixins](http://justinfagnani.com/2015/12/21/real-mixins-with-javascript-classes/) have some syntax problems, but the pipeline operator cleans them up quite nicely. For example, given the following classes and mixins:

```js
class Model {
  // ...
}
let Editable = superclass => class extends superclass {
  // ...
};
let Sharable = superclass => class extends superclass {
  // ...
};
```

... we can use the pipeline operator to create a new class that extends `Model` and mixes `Editable` and `Sharable`, with a more readable syntax:

```js
// Before:
class Comment extends Sharable(Editable(Model)) {
  // ...
}
// After:
class Comment extends Model |> Editable |> Sharable {
  // ...
}
```

### Real-world Use Cases

Check out the [Example Use Cases](https://github.com/mindeavor/es-pipeline-operator/wiki/Example-Use-Cases) wiki page to see more possibilities.

## Implementations

### Browser

* Firefox 58+ has pipeline support behind the `--enable-pipeline-operator` compile flag

### Build Tools

[@babel/plugin-proposal-pipeline-operator](https://github.com/babel/babel/tree/master/packages/babel-plugin-proposal-pipeline-operator)

## Related proposals

If you like this proposal, you will certainly like the [proposal for easier partial application](https://github.com/rbuckton/proposal-partial-application). Take a look and star if you like it!

You may also be interested in these separate proposals for a function composition operator:

- Operator: [TheNavigateur/proposal-pipeline-operator-for-function-composition](https://github.com/TheNavigateur/proposal-pipeline-operator-for-function-composition)
- Operator (generalized): [isiahmeadows/lifted-pipeline-strawman](https://github.com/isiahmeadows/lifted-pipeline-strawman)
- Method: [simonstaton/Function.prototype.compose-TC39-Proposal](https://github.com/simonstaton/Function.prototype.compose-TC39-Proposal)
- Function: [fantasyland/ECMAScript-proposals (issue #1 comment)](https://github.com/fantasyland/ECMAScript-proposals/issues/1#issuecomment-306243513)
