# new Global

Stage: To be presented for advancement to stage 1 at a future TC-39 plenary.

Champions:
- Zbyszek Tenerowicz (ZB), Consensys, @naugtur 
- Kris Kowal (KKL),  Agoric, @kriskowal
- Richard Gibson (RGN), Agoric @gibson042 
- Mark S. Miller (MM), Agoric @erights 

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

> This proposal picks up from the previous proposal for
> [Evaluators]
> (https://github.com/tc39/proposal-compartments/blob/7e60fdbce66ef2d97370007afeb807192c653333/3-evaluator.md)
> from the [HardenedJS](https://hardenedjs.org) [`Compartment`
> proposal][proposal-compartment] and depends upon [proposal-import-hook][],
> [proposal-esm-phase-imports][], and [proposal-source-phase-imports][].

## Interfaces

```ts
interface Global {
  constructor({
    keys?: string[],
    importHook?: ImportHook,
    importMeta?: Object,
  })

  Global: typeof Global,
  Function: typeof Function,
  eval: typeof eval,
  
  // and ...globalThis[...keys]
}
```

```js
new globalThis.Global({
    keys: Reflect.ownKeys(globalThis), // default behavior equivalent
    importHook,
    importMeta,
});
```

The `Global` constructor copies properties for `keys` (or all properties if
`keys` not specified) from the `globalThis` it originates from, except
`configurable` even if they were not.

Produces a _global_ with fresh:

- `Global` - a new `Global` constructor that will use the new _global_ for
  purposes of duplicating internal slots, the property descriptors of copied
  `keys`, and its `importHook`.
- `Function` and `eval` - evaluators that execute code with the _global_ as the
  global scope and `importHook`,`importMeta` used for all imports encountered
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
const thatGlobal = new globalThis.Global({
  keys: Object.keys(globalThis),
});
thatGlobal.eval !== thisGlobal.eval;
thatGlobal.Global !== thisGlobal.Global;
thatGlobal.Function !== thisGlobal.Function;
thatGlobal.eval('Object.getPrototypeOf(async () => {})') !== Object.getPrototypeOf(async () => {});
thatGlobal.eval('Object.getPrototypeOf(function *() {})') !== Object.getPrototypeOf(function *() {});
thatGlobal.eval('Object.getPrototypeOf(async function *() {})') !== Object.getPrototypeOf(async function *() {});
const source = new ModuleSource('export default {}');
(await thatGlobal.eval('(...args) => import(...args)')(source)) !== (await import(source));
thatGlobal.ModuleSource === thisGlobal.ModuleSource;
thatGlobal.x === thisGlobal.x;
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
We have found such arrangements to be fragile and leaky.

New `Global` provide an alternate solution: evaluate modules or scripts in a
separate global scope with shared intrinsics.

```js
const dslGlobal = const new Global({
  globalThis: {
    __proto__: globalThis,
    describe,
    before,
    after,
  }
});
dslGlobal.describe = () => {}
dslGlobal.before = () => {}
dslGlobal.after = () => {};

const source = await import.source(entrypoint);
await dslGlobal.eval('s => import(s)')(source);
```

In this example, only the entrypoint module for the DSL sees additional
globals.
The `source` adopts the import hook associatd with `dslGlobal` by
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

## Intersection Semantics

### Shared Structs

We expect that the new global, like old globals, would have both its own module
map and also shared struct prototype registry, such that a module executed
within that global would produce its own shared struct prototypes.
This gives platforms a place to stand to ensure that separate globals do not
share any undeniable mutable state.

## Design Questions

### prototype chain in the browser

`globalThis` in the browser has a non-trivial prototype chain for some Window
API functionality and events.

### Backward compatibility and the `constructor` field on a global

`globalThis` already has a constructor in the browser and that constructor is
`Window`, an _Illegal constructor_ as one can inform themselves by attempting
to invoke it.

```js
globalThis.constructor === Window
const g1 = new Global()

g1.globalThis.constructor === Global // would need to be true I suppose
g1.globalThis.constructor === g1.globalThis.Global // definitely not
g1.globalThis.constructor === g1.globalThis.Window // umm...
g1.globalThis.Window === Global // maybe that solves it?
```

Meanwhile in Node.js `globalThis.constructor.name === 'Object'`

[proposal-source-phase-imports]: https://github.com/tc39/proposal-source-phase-imports
[proposal-esm-phase-imports]: https://github.com/tc39/proposal-esm-phase-imports
[proposal-import-hook]: https://github.com/endojs/proposal-import-hook
