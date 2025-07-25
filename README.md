# new Global

Stage: To be presented for advancement to stage 1 at a future TC-39 plenary.

Champions:

- Zbyszek Tenerowicz (ZTZ), Consensys, @naugtur
- Kris Kowal (KKL), Agoric, @kriskowal
- Richard Gibson (RGN), Agoric, @gibson042
- Mark S. Miller (MM), Agoric, @erights

## Synopsis

Provide a `globalThis.Global` constructor that produces a new instance of
`globalThis` with a fresh set of evaluators: `eval`, `Function`,
`AsyncFunction`, `GeneratorFunction`, and `AsyncGeneratorFunction` that effect
evaluation with the new `Global` and its associated module map.
The constructor returns the new global with all of the internal slots of
`globalThis` and configurable copies of all the property descriptors from
`globalThis` or just those specified in the array of `keys`.
Dynamic `import` within these new evaluators is bound to the new global.
Dynamic `import` of a `ModuleSource` within these evaluators
instantiates that module in the new global's module map and in the lexical
scope of the new global.

This proposal does not attempt to create a new category of global object, but
creates a mechanism for replicating the existing types such that import and
evaluation behavior can be scoped to different instances.

> This proposal picks up from the previous proposal for
> [Evaluators](https://github.com/tc39/proposal-compartments/blob/7e60fdbce66ef2d97370007afeb807192c653333/3-evaluator.md)
> from the [HardenedJS](https://hardenedjs.org) [`Compartment` proposal][proposal-compartments] and depends upon [proposal-import-hook][],
> [proposal-esm-phase-imports][], and [proposal-source-phase-imports][].

## Interfaces

```ts
interface Global {
  constructor({
    keys?: string[],
    importHook?: ImportHook,
    importMetaHook?: ImportMetaHook,
  })

  Global: typeof Global,
  eval: typeof eval,
  Function: typeof Function,
  // Consequently, internal slots for 
  // AsyncFunction , GeneratorFunction,   AsyncGeneratorFunction 

  // ... and properties copied from globalThis filtered by keys
}
```

<details>
  <summary>proper typescript definition</summary>
  
```ts
interface Global<K extends keyof typeof globalThis = never> extends Pick<typeof globalThis, K> {
  constructor({
    keys?: K[],
    importHook?: ImportHook,
    importMetaHook?: ImportMetaHook,
  })

Global: typeof Global,
eval: typeof eval,
Function: typeof Function
}

````

</details>

_Example_

```js
const newGlobal = new globalThis.Global({
  keys: ['Buffer'],
  importHook,
  importMetaHook,
});

newGlobal.process = { env: process.env }
```

The `Global` constructor copies properties for `keys` (or all properties if
`keys` not specified) from the `globalThis` it originates from, except
`configurable` even if they were not.

Produces a _global_ with fresh:

- `Global` - a new `Global` constructor that will use the new _global_ for
  purposes of duplicating internal slots, the property descriptors of copied
  `keys`, and its `importHook`.
- `Function` and `eval` - evaluators that execute code with the _global_ as the
  global scope and `importHook`,`importMetaHook` used for all imports encountered
  in the evaluated code
- All other function constructors, which can be accessed through `eval` and
  their corresponding, undeniable syntax, like `global.eval('async () =>
{}').constructor`.

The global does not require a fresh `ModuleSource` because
the source is paired with the global by use of dynamic `import` in global evaluation,
as in `new Global().eval('specifier => import(specifier)')(specifier)`.

Invariants:

```js
globalThis.x = {};
const newGlobal = new globalThis.Global({
  keys: Object.keys(globalThis),
});
newGlobal.eval !== thisGlobal.eval;
newGlobal.Global !== thisGlobal.Global;
newGlobal.Function !== thisGlobal.Function;
newGlobal.eval("Object.getPrototypeOf(async () => {})") !==
  Object.getPrototypeOf(async () => {});
newGlobal.eval("Object.getPrototypeOf(function *() {})") !==
  Object.getPrototypeOf(function* () {});
newGlobal.eval("Object.getPrototypeOf(async function *() {})") !==
  Object.getPrototypeOf(async function* () {});
const source = new ModuleSource("export default {}");
(await newGlobal.eval("(...args) => import(...args)")(source)) !==
  (await import(source));
newGlobal.ModuleSource === thisGlobal.ModuleSource;
newGlobal.Array === thisGlobal.Array;
newGlobal.x === thisGlobal.x;
```

## Motivation

### Domain Specific Languages

Tools like Mocha, Jest, and Jasmine install the verbs and nouns of their
domain-specific-language in global scope.

Isolating these changes currently requires creation of a new realm,
and creating new realms comes with the hazard of identity discontinuity.
For example, `array instanceof Array` is not as reliable as `Array.isArray`,
and the hazard is not limited to intrinsics that have anticipated this
problem with work-arounds like `Array.isArray` or thenable `Promise` adoption.

Some of these tools work around this problem by using the platforms existing
facility for creating a new `Global`, albeit an iframe or the Node.js `vm`
module.
Then, they are obliged to graft the intrinsics of one realm over the other,
which leaks for the cases of syntactically undeniable Realm-specific intrinsics
like the `AsyncFunction` constructor and prototype, and requires the
implementer to be vigilant to the extent that they graft every intrinsic from
one realm to another.
We have found such arrangements to be fragile and leaky. Also costly in memory efficiency and developer time.

New `Global` provide an alternate solution: evaluate modules or scripts in a
separate global scope with shared intrinsics.

```js
const dslGlobal = const new Global();
dslGlobal.describe = () => {}
dslGlobal.before = () => {}
dslGlobal.after = () => {};

const source = await import.source(entrypoint);
await dslGlobal.eval('s => import(s)')(source);
```

In this example, only the entrypoint module for the DSL sees additional
globals.
The `source` adopts the import hook associated with `dslGlobal` by
virtue of using the `dslGlobal`'s dynamic `import`.
Current DSLs cannot execute concurrently or depend on dynamic scope to track
the entrypoint that called each DSL verb.

### Enforcing the principle of least authority

On the web, the same origin policy has become sufficiently effective at
preventing cross-site scripting attacks that attackers have been forced to
attack from within the same origin.
Conveniently for attackers, the richness of the JavaScript library ecosystem
has produced ample vectors to enter the same origin.
The vast bulk of a modern web application is its supply chain, including code
that will be eventually incorporated into the scripts that will run in the same
origin, but also the tools that generate those scripts, and the tools that
prepare the developer environment.

The same-origin-policy protects the rapidly deteriorating fiction that
web browsers mediate an interaction between just two parties: the service and
the user.
For modern applications, particularly platforms that mediate interactions among
many parties or simply have a deep supply chain, web application developers
need a mechanism to isolate third-party dependencies and minimize their access
to powerful objects like high resolution timers or network, compute, or storage
capability bearing interfaces.

Some hosts, including a community of embedded systems represented at [ECMA
TC53][tc53], do not have an origin on which to build a same-origin-policy, and
have elected to build their security model on isolated evaluators, through the
high-level Compartment interface.

### Isolating/encapsulating unreliable code

The way modern software is composed has already undermined the validity of the assumption that every author participating has their intentions well aligned for the benefit of the software working correctly. A whole new level of unreliability is now added with the popularity of _coding agents_ and _vibe coding_ where creating syntactically valid but effectively unpredictable JavaScript and integrating it into existing software to check whether it seems to implement the desired functionality is becoming a popular way of building software.

It is not an entirely new concern, as test runners have been concerned with isolating test cases to avoid them relying on global side-effects of other test cases. It is now a concern for a much wider audience with more at stake.

With `Global` constructor comes the ability to isolate fragments of the application in a way that unreliable code cannot rely on shared global state without the maintainer of the software knowing about it.
AI generated sources from independently working agents can come with colliding names for global variables to use and may need separate global scopes to collaborate or coexist. Similarly a misguided attempt at an inline polyfill by an AI or a package author could be prevented by freezing the parts of the new global in which the unreliable code subsequently runs.
Using a new global instead of a new Realm avoids the issues like identity discontinuity impeding the composition of software where function calls need to happen across the isolated and non-isolated code.

The isolation use case depends also on the interaction with `importHook` and `ModuleSource` as described in 

https://github.com/endojs/proposal-import-hook/?tab=readme-ov-file#new-global


### Incremental or in-context execution

There are tools that currently use much more complex and costly mechanisms (similar to the ones described in [Domain Specific Languages](#domain-specific-languages) among other) to provide the ability to execute fragments of JavaScript code in a very specific context of the tool.

That includes REPLs, inline code execution results in editors (eg. [Quokka.js](https://quokkajs.com/)) and various use cases of IDEs in the browser.

Maintaining the global state between executions of user-provided code snippets would benefit from a `Global` constructor.

## Intersection Semantics

### Shared Structs

We expect that the new global, like old globals, would have both its own module
map and also shared struct prototype registry, such that a module executed
within that global would produce its own shared struct prototypes.
This gives platforms a place to stand to ensure that separate globals do not
share any undeniable mutable state.

### Import Hook

The interaction between importHook and Global is described in the importHook proposal
https://github.com/endojs/proposal-import-hook/?tab=readme-ov-file#new-global

### Get Intrinsic

see https://github.com/tc39/proposal-get-intrinsic

A `new Global` object would need to be the source for `Reflect.getIntrinsic` to get the correct evaluators (including `%AsyncFunction%` etc.) from the internal slots and preserve the limited scope of the _global_ if `keys` were set.

`Reflect` would need to be unique own property of a new _global_

## Design Questions

### Prototype chain in the browser

`globalThis` in the browser has a non-trivial prototype chain for some Window
API functionality and events.

```js
let pro = globalThis; 
while (pro = Object.getPrototypeOf(pro)) { 
  console.log(pro.toString())
}
```
```
// browsers
[object Window]
[object WindowProperties]
[object EventTarget]
[object Object]
```
```
// Node.js
[object Object]
[object Object]
```
```
// Deno
[object Window]
[object EventTarget]
[object Object]
```
```
// Hermes
[object Object]
undefined
```


### Backward compatibility and the `constructor` field on a global

`globalThis` already has a constructor in the browser and that constructor is
`Window`, an _Illegal constructor_ as one can inform themselves by attempting
to invoke it.

```js
globalThis.constructor === Window;
const g1 = new Global();

g1.globalThis.constructor === Global; // would need to be true I suppose
g1.globalThis.constructor === g1.globalThis.Global; // definitely not
g1.globalThis.constructor === g1.globalThis.Window; // umm...
g1.globalThis.Window === Global; // maybe that solves it?
```

Meanwhile in Node.js `globalThis.constructor.name === 'Object'`

[proposal-source-phase-imports]: https://github.com/tc39/proposal-source-phase-imports
[proposal-esm-phase-imports]: https://github.com/tc39/proposal-esm-phase-imports
[proposal-compartments]: https://github.com/tc39/proposal-compartments
[proposal-import-hook]: https://github.com/endojs/proposal-import-hook

