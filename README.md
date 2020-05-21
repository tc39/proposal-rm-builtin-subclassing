# Subclassing support in built-in methods delenda est

Champions: Shu-yu Guo (Google), Yulia Startsev (Mozilla)

Subclass instance creation via @@species in built-in methods of `Array`, `RegExp`, `Promise`, and _TypedArray_, as well as property lookups of e.g. `"exec"` on the receiver in built-in methods of `RegExp`, are considered by many implementers and some committee members to be one of TC39’s greatest mistakes for the language. It imposes great complexity on the implementation and mental model of the language, the cost which has resulted in many security vulnerabilities.

This proposal seeks to remove subclassing support via @@species and related machinery, such as property lookups of `"flags"` and `"exec"` in certain `RegExp` built-ins.

This is a significant, fill-or-kill, backwards incompatible change that seeks to remove all built-in subclassing machinery. Doing finer-grained removal for certain methods or constructors will result in more confusion and does not remedy the complexity and maintenance burden costs.

# Motivation

Supporting subclassing in built-ins via @@species and related machinery incurs
significant burden:

- Increased implementation complexity and maintainability
- Performance cliffs
- Security bugs

Natalie Silvanoich gave a [talk](https://docs.google.com/presentation/d/11fkQeEisoszNGF8SrautVT1ltSnsQBWRxJ4usoc-g_o/edit#slide=id.g2b34aaab4a_0_10) to TC39 in 2018 that called out the effect of @@species on security vulnerabilities. Below is a list of security bugs caused by @@species:

Chrome:
- https://bugs.chromium.org/p/chromium/issues/detail?id=920491
- https://bugs.chromium.org/p/chromium/issues/detail?id=840106
- https://bugs.chromium.org/p/chromium/issues/detail?id=804971
- https://bugs.chromium.org/p/chromium/issues/detail?id=800356
- https://bugs.chromium.org/p/chromium/issues/detail?id=726622
- https://bugs.chromium.org/p/chromium/issues/detail?id=726636
- https://bugs.chromium.org/p/chromium/issues/detail?id=799952 (no security impact in release build)

Firefox

- https://bugzilla.mozilla.org/show_bug.cgi?id=1537924

Supporting built-in subclassing has also negatively affected authoring of new
built-ins. Current and future spec authors, often in deference to consistency,
propagate the @@species machinery.

@@species has also negatively affected other specifications that interact with
JavaScript, like WebAssembly, that were not aware of this corner of the
specification.

# Proposed New Old Semantics

## `Array`

### Prototype methods

The following methods on `Array.prototype` will create and return an `Array` exotic object in the current Realm. They will no longer consult `this.constructor[`@@species`]`.

- `Array.prototype.concat`
- `Array.prototype.filter`
- `Array.prototype.flat`
- `Array.prototype.flatMap`
- `Array.prototype.map`
- `Array.prototype.slice`
- `Array.prototype.splice`

This means a subclass calling these methods on subclass instances will always get `Array` instances back.

```javascript
class MyArray extends Array { /* ... * / }
let ma = (new MyArray(42)).map((x) => x);
// Result is an Array, not a MyArray.
```

### Constructor methods

The following methods on `Array` will create and return an `Array` exotic object in the current Realm. They will no longer conditionally use the `this` value as a constructor if IsConstructor(`this`) is true.

- `Array.from`
- `Array.of`

This means any subclass calling these methods on the subclass constructor will always get `Array` instances back.

```javascript
class MyArray extends Array { /* ... * / }
let ma = MyArray.from([1,2,3]);
// Result is an Array, not a MyArray.
```

### Removing @@species

`Array[`@@species`]` will be removed.

## `RegExp`

RegExp subclassing machinery involves both @@species and dynamic property lookups on the `this` value in various prototype methods. Property lookups will be removed in favor of internal slots. @@species will be removed in favor of creating `RegExp` objects in the current Realm.

### Constructor

- The `RegExp(` _pattern_, _flags_ `)` constructor will no longer check for `IsRegExp(` _pattern_ `)`.
- Step 2.b will be changed to check for [[RegExpMatcher]].
- Step 5 will be removed.

### Prototype methods

Methods on `RegExp.prototype` will have the following changes where applicable.

- If `this` does not have [[RegExpMatcher]], throw a TypeError.
- When creating new `RegExp` instances, a %RegExp% instance will be created in the current Realm instead of consulting `this.constructor[`@@species`]`.
- `this.`[[OriginalFlags]] will be consulted instead of `this.flags`.
- `this.`[[OriginalFlags]] will be consulted instead of `this.dotAll`.
- `this.`[[OriginalFlags]] will be consulted instead of `this.global`.
- `this.`[[OriginalFlags]] will be consulted instead of `this.ignoreCase`.
- `this.`[[OriginalFlags]] will be consulted instead of `this.sticky`.
- `this.`[[OriginalFlags]] will be used instead of `this.unicode`.
- `this.`[[OriginalSource]] will be used instead of `this.source`.
- RegExpBuiltinExec will be used instead of `this.exec`.

These changes apply to the following methods.

- `RegExp.prototype[`@@match`]`
- `RegExp.prototype[`@@matchAll`]`
- `RegExp.prototype[`@@replace`]`
- `RegExp.prototype[`@@search`]`
- `RegExp.prototype[`@@split`]`
- `RegExp.prototype.test`

### `String` prototype methods

The following methods on `String.prototype` will be modified to check for [[RegExpMatcher]] where they are currently checking IsRegExp.

- `String.prototype.startsWith`
- `String.prototype.endsWith`
- `String.prototype.includes`
- `String.prototype.matchAll`

### Removing @@species

`RegExp[`@@species`]` will be removed.

## `Promise`

### Prototype methods

The following methods on `Promise.prototype` will create and return a `Promise` object in the current Realm. They will no longer consult `this.constructor[`@@species`]`.

- `Promise.prototype.finally`
- `Promise.prototype.then`

This means a subclass calling these methods on subclass instances will always get `Promise` instances back.

```javascript
class MyPromise extends Promise { /* ... * / }
let mp = (new MyPromise(executor)).then(() => {});
// Result is a Promise, not a MyPromise.
```

### Constructor methods

(This may be decoupled from the rest of the proposal.)

The following methods on `Promise` will create and return a `Promise` object in the current Realm. They will require `this` to be the `Promise` constructor.

- `Promise.all`
- `Promise.allSettled`
- `Promise.any`
- `Promise.race`
- `Promise.reject`
- `Promise.resolve`

This means any subclass calling these methods on the subclass constructor will always get `Promise` instances back, or throw if not called with `Promise` as the receiver.

```javascript
class MyPromise extends Promise { /* ... * / }
let mp = MyPromise.resolve(() => {});
// Result is an Promise, not a MyPromise.
```

### Removing @@species

`Promise[`@@species`]` will be removed.

## _TypedArray_

### Constructor

- The _TypedArray_`(` _typedArray_ `)` constructor will no longer use SpeciesConstructor in step 16 and will use %ArrayBuffer%.
- Step 17 becomes unnecessary and is removed.

### Prototype methods

The following methods on _TypedArray_`.prototype` will create and return a _TypedArray_ object in the current Realm. They will no longer consult `this.constructor[`@@species`]`.

- _TypedArray_`.prototype.filter`
- _TypedArray_`.prototype.map`
- _TypedArray_`.prototype.slice`
- _TypedArray_`.prototype.subarray`

This means a subclass calling these methods on subclass instances will always get `Promise` instances back.

```javascript
class MyBuffer extends Uint8Array { /* ... * / }
let mb = (new MyBuffer(42)).filter((x) => true);
// Result is a Uint8Array, not a MyBuffer.
```

### Constructor methods

The following methods on _TypedArray_ will create and return an _TypedArray_ exotic object in the current Realm. They will require `this` to be the _TypedArray_ constructor.

- _TypedArray_`.from`
- _TypedArray_`.of`

This means any subclass calling these methods on the subclass constructor will always get _TypedArray_ instances back.

```javascript
class MyBuffer extends Uint8Array { /* ... * / }
let mb = MyBuffer.from([1,2,3]);
// Result is an Uint8Array, not a MyBuffer.
```

### Removing @@species

_TypedArray_`[`@@species`]` will be removed.

## `ArrayBuffer`

### Prototype methods

`ArrayBuffer.prototype.slice` will create and return an %ArrayBuffer% object in the current Realm. It will no longer consule `this.constructor[`@@species`]`.

### Removing @@species

`ArrayBuffer[`@@species`]` will be removed.

## `SharedArrayBuffer`

### Prototype methods

`SharedArrayBuffer.prototype.slice` will create and return an %SharedArrayBuffer% object in the current Realm. It will no longer consule `this.constructor[`@@species`]`.

### Removing @@species

`SharedArrayBuffer[`@@species`]` will be removed.

## `Map`

### Removing @@species

`Map[`@@species`]` will be removed. It is currently not used.

## `Set`

### Removing @@species

`Set[`@@species`]` will be removed. It is currently not used.

## Meta

`Symbol.species` will remain as a vestigial symbol if any user code wants to use it in its own subclassing protocol.

# Web Compatibility Challenge

Built-in subclassing was added as part of ES6. All major browsers have shipped support for it for years: Chrome since 51, Firefox since 41, and Safari since 10. The compatibility risk of unshipping @@species is very real.

There is a cross-vendor concerted effort to assess compatibility risk. Current efforts include, but are not limited to:

1. Using Chrome UseCounter data to get a conservative picture of usage of subclassing mechanisms, namely @@species and `.constructor`.
1. Using queries on HTTP Archive.
1. Using a crawler to check more in depth URLs from queries on HTTP Archive.
1. Using instrumented builds of browsers to manually check for breakage.

Very preliminary numbers suggest that there is significant number of occurrences (up to 2% of all page visits in Chrome) of modifying `.constructor` or @@species on `Array`, `RegExp`, and `Promise`, and much less so in _TypedArray_ constructors (up to 0.04% of all page visits in Chrome). Manual inspection of the web sites that do such modifications reveal that they are not real uses of built-in subclassing but are instead of an [outdated core-js shim](https://github.com/zloirock/core-js/blob/9ed55e682ff0a9814b6bb0c15c8c058bdfb8d954/src/%24.species.js) unconditionally installing a `function() { return this; }` as the @@species getter.

Thus, the working hypothesis is that most of the real uses are false positives due to outdated shims, and this change is by and large web compatible.

## Notable libraries that break

### Node.js `Buffer`s and the `Buffer` [polyfill](https://github.com/feross/buffer)

`Buffer` is a subclass of `Uint8Array`. Uses of `Uint8Array.prototype.map`, `Uint8Array.prototype.filter`, `Uint8Array.prototype.subarray`, and `Uint8Array.prototype.slice` will produce `Uint8Array`s in the proposed semantics instead of `Buffer`s. Note that `Buffer` overrides `slice`, but inherits the other three methods.

