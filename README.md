# Restricting subclassing support in built-in methods

Champions: Shu-yu Guo (Google), Yulia Startsev (Mozilla)

Stage: 1

Last presentation: [Notes](https://github.com/tc39/notes/blob/master/meetings/2020-06/june-3.md#restrict-subclassing-support-for-built-in-methods-stage-1), [slides](https://docs.google.com/presentation/d/1vJeJFueDwrj8ebXFdGsEO1J_Q-DzfU01dLEGVd26A9o/edit#slide=id.p)

Subclass instance creation via @@species in built-in methods of `Array`, `RegExp`, `Promise`, and _TypedArray_, as well as property lookups of e.g. `"exec"` on the receiver in built-in methods of `RegExp`, are considered by many implementers and some committee members to be one of TC39’s greatest mistakes for the language. It imposes great complexity on the implementation and mental model of the language, the cost which has resulted in many security vulnerabilities.

This proposal seeks to remove subclassing support via @@species and related machinery, such as property lookups of `"flags"` and `"exec"` in certain `RegExp` built-ins.

This is a significant, fill-or-kill, backwards incompatible change that seeks to remove all built-in subclassing machinery. Doing finer-grained removal for certain methods or constructors will result in more confusion and does not remedy the complexity and maintenance burden costs.

# Motivation

Supporting subclassing in built-ins via @@species and related machinery incurs significant burden:

- Increased implementation complexity and maintainability
- Performance cliffs
- Security bugs

[Natalie Silvanoich](https://github.com/natashenka) from Project Zero gave a [talk](https://docs.google.com/presentation/d/11fkQeEisoszNGF8SrautVT1ltSnsQBWRxJ4usoc-g_o/edit#slide=id.g2b34aaab4a_0_10) to TC39 in 2018 that called out the effect of @@species on security vulnerabilities. Below is a list of security bugs caused by @@species:

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

Supporting built-in subclassing has also negatively affected authoring of new built-ins. Current and future spec authors, often in deference to consistency, propagate the @@species machinery.

@@species has also negatively affected other specifications that interact with JavaScript, like WebAssembly, that were not aware of this corner of the specification.

# Taxonomy of subclassing

@domenic has an excellent taxonomy for subclassing built-ins in JS, which I adopt here and modified slightly. JS currently supports all of the following types of subclassing of built-ins.

## Type I: minimal support

Type I is supported if creating subclasses of built-ins is possible. For example, if derived class can call `super()`. Support for this is provided via `new.target`.

### Example

```js
class A extends Array {
  constructor(a,b,c) {
    super(a,b,c);
  }
}
new A(1,2,3);      // return type: A
```

Without Type I support, and doesn't support `new.target` the following would be true

```js
class A extends Array {
  constructor(a,b,c) {
    super(a,b,c);
  }
}
new A(1,2,3);      // return type: Array
```


### Cost benefit

㊟ **There is crucial dependence on Type I subclassing and it is worth the implementation and language cost.**

Type I is used by user libraries, as well as by the web platform in WebIDL and DOM.

## Type II: subclass instance creation in built-in methods

Type II is supported if built-in methods create new instances of the subclass. For example, if `Array.prototype.map` or `Array.from` returns instances of subclasses of `Array`. Support for this is _subsumed_ by the support for Type III below via `this.constructor[`@@species`]`.

### Example

```js
class A extends Array { }

A.from([1,2,3])    // return type: A
 .map(x => x + 1); // return type: A
```

### Cost benefit

㊟ **Beneficial, but at cost.**

Type II is the intuition enjoyed by many developers. It enables user libraries to subclass built-ins like `Array` without also having to maintain overrides of all instance-creating methods.

However, it incurs implementation complexity in that built-in methods have overrideable behavior that results in arbitrary code being executed via `this.constructor`. In implementations, this results in a proliferation of slow-paths and invariants that cause JIT code to deoptimize. Failure to do so may and have resulted in serious security vulnerabilities in browsers. In the language, `this.constructor` resulting in possible arbitrary code execution in some built-ins increases difficulty of reasoning.

## Type III: customizable subclass instance creation in built-in methods

Type III is supported if built-in methods create new instances of the subclass's choosing. For example, if `Array.prototype.map` or `Array.from` returns instances of subclasses of `Array` via `SubclassConstructor[`@@species`]`. Support for this is provided by delegating to `this.constructor[`@@species`]` inside built-in methods with custom values for the @@species property.

The main difference between Type II and Type III is user expectation, not implementation. Type II is the user expectation that built-in methods, when called on instances of subclasses, have some way of querying the class of those instances and create instances of the subclass. Type III is the addition that subclasses themselves can override Type II behavior programmatically.

### Example

```js
class A extends Array {
  static [Symbol.species] = Array;
}

A.from([1,2,3])    // return type: A
 .map(x => x + 1); // return type: Array
```

### Cost benefit

㊟ **Not useful, and at great cost.**

Type III gives subclasses expressivity to actually _opt out_ of Type II support. If `NodeList.prototype.map`, as inherited from `Array.prototype.map`, actually wanted to return an `Array` instead of a `NodeList`, it would opt out via setting `NodeList[`@@species`]` to `Array`. In more complex cases, each instance-creating method may have its own consideration, and a single @@species value on the constructor is insufficient.

Supporting @@species compounds the cost already incurred by Type II by making the paths even more complex and the invariants more brittle. In the language, the fact that developers have to think at all about @@species, which is not generally useful, is harmful.

There are no known compelling use cases that are worth this cost.

## Type IV: delegation to property lookups in built-in methods

Type IV is supported if built-in methods consult properties on instances instead of internal slots. For example, if `RegExp.prototype[`@@match`]` calls `this.exec` instead of the built-in RegExp exec. Support for this is provided by, well, delegating to property lookups.

Note that `RegExp`'s @@match, @@matchAll, @@replace, @@search, and @@split symbols themselves aren't strictly only for subclassing support, as they are not used in `RegExp` methods themselves. Instead, they are used as a protocol for `String` so that completely custom `RegExp` instances may be consumed.

### Example

I hope this example demonstrates that one cannot subclass `RegExp` piecemeal and have a good time.

```js
function R() { }
Object.setPrototypeOf(R, RegExp);
Object.setPrototypeOf(R.prototype, RegExp.prototype);
R.prototype.exec = function() {
  console.log("overridden");
  return null;
};
// Define a new .global since RegExp#global throws on
// non-RegExp-branded `this`
Object.defineProperty(R.prototype, "global", { value: false });
console.log("some string".match(new R("foo")))     // logs "overridden"
```

### Cost benefit

㊟ **Harmful, and at great cost.**

Type IV is harmful expressivity. It is very difficult for implementations to provide robust fast paths at all for `RegExp`, which users have high performance expectations of. The cost of this, depending on the number of overrideable properties, is the cost of Type II and III combined, and then some.

It also makes the language significantly more complex to reason about for built-ins that support it. Users should not be subclassing `RegExp`s piecemeal, and overriding subsets of behaviors via `exec` or one of the flag properties like `global` and expect to have a good time. And similarly for `Promise`s. It also makes the spec very hard to understand (cf PromiseCapabilities).

(Since `RegExp`'s symbols aren't for subclassing, they are not considered harmful in this context.)

# Proposed new old semantics

We propose to remove support for Type II, Type III, and Type IV subclassing, and only keep Type I.

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

Before this change:

```javascript
class MyArray extends Array { /* ... */ }
let ma = (new MyArray(42)).map((x) => x);
console.log(ma instanceof MyArray) // true
```

After this change:

```javascript
class MyArray extends Array { /* ... */ }
let ma = (new MyArray(42)).map((x) => x);
console.log(ma instanceof MyArray) // false
// Result is an Array, not a MyArray.
```

### Constructor methods

The following methods on `Array` will create and return an `Array` exotic object in the current Realm. They will no longer conditionally use the `this` value as a constructor if IsConstructor(`this`) is true.

- `Array.from`
- `Array.fromAsync`
- `Array.of`

This means any subclass calling these methods on the subclass constructor will always get `Array` instances back.

Before this change:

```javascript
class MyArray extends Array { /* ... */ }
let ma = MyArray.from([1,2,3]);
console.log(ma instanceof MyArray) // true
```

After this change:

```javascript
class MyArray extends Array { /* ... */ }
let ma = MyArray.from([1,2,3]);
console.log(ma instanceof MyArray) // false
// Result is an Array, not a MyArray.
```

### Removing @@species

`Array[`@@species`]` will be removed. This means that the following will no longer be possible using
the `@@species` symbol like this:

```js
class A extends Array {
  static [Symbol.species] = OtherArray;
}

A.from([1,2,3])     // return type: A
  .map(x => x + 1); // return type: OtherArray
```


## `RegExp`

RegExp subclassing machinery involves both @@species and dynamic property lookups on the `this` value in various prototype methods. Property lookups will be removed in favor of internal slots. @@species will be removed in favor of creating `RegExp` objects in the current Realm.

Notably, the protocol for `RegExp`-likes (e.g. @@match) is not removed as part of this proposal since they are not used by `RegExp` instance or constructor methods themselves.

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

Notably, the `String` prototype methods are not proposed to be modified.

### Removing @@species

`RegExp[`@@species`]` will be removed.


### Example

Before this change:

```js
class R extends RegExp {
  exec() {
    return "overridden";
  }
}
console.log("some string".match(new R("foo")))     // "overridden"
```

After this change:

```js
class R extends RegExp {
  exec() {
    return "overridden";
  }
}
console.log("some string".match(new R("foo")))     // null
```

## `Promise`

### Prototype methods

The following methods on `Promise.prototype` will create and return a `Promise` object in the current Realm. They will no longer consult `this.constructor[`@@species`]`.

- `Promise.prototype.finally`
- `Promise.prototype.then`

This means a subclass calling these methods on subclass instances will always get `Promise` instances back.

Before this change:

```javascript
class MyPromise extends Promise { /* ... */ }
let mp = (new MyPromise(executor)).then(() => {});
console.log(mp instanceof MyPromise) // true
```

After this change:

```javascript
class MyPromise extends Promise { /* ... */ }
let mp = (new MyPromise(executor)).then(() => {});
console.log(mp instanceof MyPromise) // false
// Result is a Promise, not a MyPromise.
```

### Constructor methods

The following methods on `Promise` will create and return a `Promise` object in the current Realm. They will ignore the `this` value.

- `Promise.all`
- `Promise.allSettled`
- `Promise.any`
- `Promise.race`
- `Promise.reject`
- `Promise.resolve`

This means any subclass calling these methods on the subclass constructor will always get `Promise` instances back.


Before this change:

```javascript
class MyPromise extends Promise { /* ... */ }
let mp = MyPromise.resolve(() => {});
console.log(mp instanceof MyPromise) // true
```

After this change:

```javascript
class MyPromise extends Promise { /* ... */ }
let mp = MyPromise.resolve(() => {});
console.log(mp instanceof MyPromise) // false
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

This means a subclass calling these methods on subclass instances will always get _TypedArray_ instances back.

Before this change:

```javascript
class MyBuffer extends Uint8Array { /* ... */ }
let mb = (new MyBuffer(42)).filter((x) => true);
console.log(mb instanceof MyBuffer) // true
```

After this change:

```javascript
class MyBuffer extends Uint8Array { /* ... */ }
let mb = (new MyBuffer(42)).filter((x) => true);
console.log(mb instanceof MyBuffer) // false
// Result is a Uint8Array, not a MyBuffer.
```

### Constructor methods

The following methods on _TypedArray_ will create and return an _TypedArray_ exotic object in the current Realm. They will ignore the `this` value.

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

`ArrayBuffer.prototype.slice` will create and return an %ArrayBuffer% object in the current Realm. It will no longer consult `this.constructor[`@@species`]`.

### Removing @@species

`ArrayBuffer[`@@species`]` will be removed.

## `SharedArrayBuffer`

### Prototype methods

`SharedArrayBuffer.prototype.slice` will create and return an %SharedArrayBuffer% object in the current Realm. It will no longer consult `this.constructor[`@@species`]`.

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

# Web compatibility

Built-in subclassing was added as part of ES6. All major browsers have shipped support for it for years: Chrome since 51, Firefox since 41, and Safari since 10. The compatibility risk of unshipping @@species is very real.

There is a cross-vendor concerted effort to assess compatibility risk. Current efforts include, but are not limited to:

1. Using Chrome UseCounter data to get a conservative picture of usage of subclassing mechanisms, namely @@species and `.constructor`.
1. Using queries on HTTP Archive.
1. Using a crawler to check more in depth URLs from queries on HTTP Archive.
1. Using instrumented builds of browsers to manually check for breakage.

Very preliminary numbers suggest that there is significant number of occurrences (up to 2% of all page visits in Chrome) of modifying `.constructor` or @@species on `Array`, `RegExp`, and `Promise`, and much less so in _TypedArray_ constructors (up to 0.04% of all page visits in Chrome). Manual inspection of the web sites that do such modifications reveal that they are not real uses of built-in subclassing but are instead of an [outdated core-js shim](https://github.com/zloirock/core-js/blob/9ed55e682ff0a9814b6bb0c15c8c058bdfb8d954/src/%24.species.js) unconditionally installing a `function() { return this; }` as the @@species getter.

Thus, the working hypothesis is that most of the real uses are false positives due to outdated shims, and this change is by and large web compatible.

## Hunch by subclassing type

- Removing Type II has the biggest compatibility risk
- Removing Type III is likely to be compatible
- Removing Type IV is likely to be compatible

## Notable libraries that break

### Node.js `Buffer`s and the `Buffer` [polyfill](https://github.com/feross/buffer)

`Buffer` is a subclass of `Uint8Array`. Uses of `Uint8Array.prototype.map`, `Uint8Array.prototype.filter`, `Uint8Array.prototype.subarray`, and `Uint8Array.prototype.slice` will produce `Uint8Array`s in the proposed semantics instead of `Buffer`s. Note that `Buffer` overrides `slice`, but inherits the other three methods.

# Exit criteria

If removing Type III and Type IV is not web compatible, this proposal shall be withdrawn.

If removing Type II is not web compatible (or if there is renewed consensus in TC39 to uphold developer intuition) but removing Type III and Type IV subclassing is web compatible, then this proposal shall explore alternative ways to support only Type II with less implementation and security burden. If no good alternative arises, and implementers deem the benefits of removing Type III and Type IV alone does not justify changing behavior, this proposal shall be withdrawn.
