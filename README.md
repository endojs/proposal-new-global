# new Global

## Synopsis

Provide a `Global` constructor that produces a new instance of globalThis with  `Global`, `eval`, `Function` constructor, and `ModuleSource` (or equivalent) constructor such that execution contexts
generated from the evaluators refer back to this global, and with
 virtualized host behavior for dynamic import in script
contexts based on the `importHook` and `importMeta` provided.
Based on the `keys` option (that defaults to all) further fields are copied from the `globalThis` in which the `Global` constructor lives.

All options are optional.


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
  ModuleSource: typeof ModuleSource,
  
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

The `Global` constructor copies values for `keys` (or all entries if `keys` not specified) from the globalThis it originates from.

Produces a _global_ with fresh:
- `Global` - the same Global constructor but copying values from the new _global_
- `Function` and `eval` - evaluators that execute code with the _global_ as the global scope and `importHook`,`importMeta` used for all imports encountered in the evaluated code
- `ModuleSource` - (tentatively, but we need some way to execute modules with that _global_) TBD

Invariants
```js
globalThis.x = {};
const thatGlobal = new globalThis.Global({
  keys: Object.keys(globalThis)
});
thatGlobal.eval !== thisGlobal.eval;
thatGlobal.Global !== thisGlobal.Global;
thatGlobal.ModuleSource !== thisGlobal.ModuleSource;
thatGlobal.Function !== thisGlobal.Function;
thatGlobal.eval('Object.getPrototypeOf(async () => {})') !== Object.getPrototypeOf(async () => {});
thatGlobal.x === thisGlobal.x;
```

## Motivation

## Design Questions

### `keys` default

- what about non-enumerable keys?
- what about symbol keys?
- what copying semantics is used? 
  - would getters be invoked or copied?

structuredClone() is not part of ECMAScript, sadly

### prototype chain in the browser

`globalThis` in the browser has a non-trivial prototype chain for some Window API functionality and events


### Backward compatibility and the `constructor` field on a global

`globalThis` already has a constructor in the browser and that constructor is `Window`, an _Illegal constructor_ as one can inform themselves by attempting to invoke it
```js
globalThis.constructor === Window
const g1 = new Global()

g1.globalThis.constructor === Global // would need to be true I suppose
g1.globalThis.constructor === g1.globalThis.Global // definitely not
g1.globalThis.constructor === g1.globalThis.Window // umm...
g1.globalThis.Window === Global // maybe that solves it?
```

Meanwhile in Node.js `globalThis.constructor.name === 'Object'`
