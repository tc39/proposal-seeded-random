# Seeded Pseudo-Random Numbers

**Stage: 2**

**Champion: Tab Atkins-Bittner**

**Spec Draft: <https://tc39.github.io/proposal-seeded-random/>** (out of date, README is current source of truth)

------

JS's PRNG methods (`Math.random()`, `crypto.getRandomValues()`, etc) are all "automatically seeded" - each invocation produces a fresh unpredictable random number, not reproducible across runs or realms.  However, there are several use-cases that want a reproducible set of random values, and so want to be able to seed the random generator themselves.

1. New APIs like the CSS Custom Paint, which can't store state but can be invoked arbitrarily often, want to be able to produce the same set of pseudo-random numbers each time they're invoked.

    Demo: <https://lab.iamvdo.me/houdini/rough-boxes/> This demo uses Math.random() to unpredictably shift the "rough borders", but the Houdini Custom Paint API re-invokes the callback any time the element needs to be "repainted" - whenever it changes size, or is off-screen for a bit, or just generally whenever (UAs are given substantial leeway in this regard). There's no way for the callback to store state for itself (by design), so it can't just pre-generate a list of random numbers and re-use them; instead, it currently just produces a totally fresh set of borders. (You can see this in effect if you zoom the page in or out, as each change repaints the element and re-invokes the callback.)

2. Testing frameworks that utilize randomness for some purpose, and want to be able to use the same pseudo-random numbers on both local and CI, and to be able to reproduce a given interesting test run.

3. Games that use randomness and want to avoid "save-scumming", where players save and repeatedly reload the game until an event comes out the way they want.

Currently, the only way to achieve these goals is to implement your own PRNG by hand in JS. Simple PRNGs like an LCG aren't hard to code, but they don't produce good pseudo-random numbers; better PRNGs are harder to implement correctly. It would be much better to provide the ability to manually seed a generator and get a predictable sequence out.  While we're here, we can lean on JS features to provide a better usability than typical random libs provide for this use-case.

Expected Usage
--------------

A `Random.Seeded` object, once constructed, has a `.random()` method. This works identically to `Math.random()`, so literally any existing usage of `Math.random()` can be replaced by initializing a `Random.Seeded` and then calling `prng.random()` instead.

It could even be installed onto the global object, like:

```js
let globalPRNG = Random.Seeded.fromFixed(0);
Math.random = globalPRNG.random.bind(globalPRNG);

// now `Math.random()` produces the same sequence of values on every page load.
```

Security libraries that currently "poison" `Math.random()` to avoid its possible use as a communications channel could also do the above, to allow it to still operate roughly as intended but without the communications possibility.

API
------

Here's a quick summary of the API proposal:

```js
const Random = { // new global namespace object
  random(): Number {
    return TheInternalBrowserPRNG.random();
  },
  seed(): Uint8Array {
    return TheInternalBrowserPRNG.seed();
  },
};

Random.Seeded = class SeededRandom {
  #state: Uint8Array;

  constructor(seed: Uint8Array) {
    if(!seed instanceof Uint8Array) throw new TypeError();
    if(seed.length > 32) throw new RangeError();
    
    // Prefix with 0 bytes if needed.
    const paddedSeed = new Uint8Array(32);
    paddedSeed.set(32 - seed.length, seed);

    this.#state = stateFromSeed(paddedSeed);
    return this;
  }

  static fromSeed(seed: Uint8Array): Random.Seeded {
    if(!seed instanceof Uint8Array) throw new TypeError();
    if(seed.length != 32) throw new RangeError();
    const prng = InternalMakeFreshSeededRandom();
    prng.setState(stateFromSeed(seed));
    return prng;
  }

  static fromState(state: Uint8Array): Random.Seeded {
    if(!state instanceof Uint8Array) throw new TypeError();
    if(state.length != 112) throw new RangeError();
    const prng = InternalMakeFreshSeededRandom();
    prng.setState(state);
    return prng;
  }

  static fromFixed(byte: Number): Random.Seeded {
    if(typeof byte != "number") throw new TypeError();
    if(!Number.isInteger(byte) || byte < 0 || byte > 255) throw new RangeError();
    const prng = InternalMakeFreshSeededRandom();
    const seed = new Uint8Array(32);
    seed[31] = byte;
    prng.setState(stateFromSeed(seed));
    return prng;
  }

  random(): Number {
    let [val, this.#state] = randomVal(this.#state); // Number in [0,1)
    return val;
  }

  seed(): Uint8Array {
    let [seed, this.#state] = randomSeed(this.#state); // Uint8Array that's a valid seed.
    return seed;
  }

  getState(): Uint8Array {
    return copy(this.#state);
  }

  setState(state: Uint8Array): Seeded.Random {
    if(!state instanceof Uint8Array) throw new TypeError();
    if(state.length != 112) throw new RangeError();
    this.#state = copy(state);
    return this;
  }
}
```

### The `Random` Namespace Object ###

In conjunction with the [More Random Methods proposal](https://github.com/tc39/proposal-random-functions),
this proposal adds a new `Random` namespace object,
used to hold the various new randomness-related operations.

The `Random` namespace object, in addition to holding the `Random.Seeded` class (described below),
holds methods matching that of the `Random.Seeded` class,
which simply call the matching method on a *user-agent internal* `SeededPRNG` instance.
This internal instance is initialized from a random seed by the user agent.


#### The `Random.random()` and `Random.seed()` functions ####

This proposal defines two methods on the `Random` namespace object,
`.random()` and `.seed()`,
matching the two methods of the same names defined by `Random.Seeded`.
"More Random Methods" will define more.

Each simply returns the result of calling `.random()` or `.seed()`
on the UA-internal `Random.Seeded` object.

`Random.random()`, thus, is identical to `Math.random()`, except that it uses a higher-quality prng algorithm.

`Random.seed()` is intended to be used to initialize a `Random.Seeded` to an unpredictable starting value,
like `new Random.Seeded(Random.seed())`.



### Creating a PRNG: the `new Random.Seeded(Uint8Array)` constructor ###

This proposal adds a new class, `Random.Seeded`, which lives on the `Random` namespace object as `Random.Seeded`.

The constructor takes a single `seed` argument, which is a Uint8Array of length 32 or less.

If `seed` is less than 32 bytes, it's prefixed with enough 0 bytes to make it 32 bytes long. If it's more than 32 bytes, a `RangeError` is thrown.

`seed` is then used to create a state vector, and a fresh `Random.Seeded` object with that state vector is returned.

> [!NOTE]
> Note that `TypedArray` objects have a `.slice()`, same as `Array`, so if you want to seed a `Random.Seeded` from a value with an unpredictable byte size, you can use `new Random.Seeded(tarr.slice(-32))` (or `.slice(0, 32)`, etc, depending on how you want to slice too-large seeds). Too-small seeds will be automatically padded for you.

> [!NOTE]
> [Issue 26](https://github.com/tc39/proposal-seeded-random/issues/26) - Should we allow other buffer/view types with a stable byte ordering (not dependent on system endianness)? Or all buffer/view types, matching general DOM practices? This API would be forming precedent across ES.

### Factory Methods: `.fromSeed()`, `.fromState()`, `.fromFixed()` ###

In addition to the constructor, three static factory methods exist on the `Random.Seeded` class.

* `Random.Seeded.fromSeed(seed)` has the same behavior as the constructor, but requires a correct length (32 bytes) seed value. (Useful if you want to make sure your seed source doesn't accidentally regress and start passing too little entropy.)
* `Random.Seeded.fromState(state)` takes a state vector (a `Uint8Array` of length 112) and returns a `Random.Seeded` initialized directly to that state. (This is more convenient/efficient than creating a `Seeded.Random` from a junk value and calling `.setState()` immediately.)
* `Random.Seeded.fromFixed(num)` takes a Number byte (an integer 0-255), and treats this as the lowest byte of an otherwise-zero seed value. (In other words, equivalent to `new Random.Seeded(Uint8Array.of(num))`.)


### Getting a Random Number: the `.random()` method ###

To obtain a random number from a PRNG object, the object has a `.random()` method. On each invocation, it will output an appropriate pseudo-random number in the range `[0,1)` based on its state, and then update its state for the next invocation.

To generate this value:

1. Obtain 64 random bits from the PRNG.
2. Shift right by 11, obtaining a 53-bit integer.
3. Convert this integer to an equivalent float64.
4. Multiply this float64 by `1/(2**53)`.
5. Return the result.

Using the prng object is thus basically identical to using `Math.random()`:

```js
const prng = new Random.Seeded(0);
for(let i = 0; i < limit; i++) {
  const r = prng.random();
  // do something with each value
}
```

> [!NOTE]
> This specific generation algo is used by Rust and numpy.
> See <https://github.com/rust-random/rand/blob/7aa25d577e2df84a5156f824077bb7f6bdf28d97/src/distributions/float.rs#L111-L117>

### Getting a Random Seed: the `.seed()` method ###

There are reasonable use-cases for generating *multiple, distinct* PRNGs on a page;
for example, a game might want to use one for terrain generation, one for cloud generation, one for AI, etc.
Using a single `Random.Seeded` object can be hacked into doing this
(for example, by saying that the first of every three values is for terrain, the second is for clouds, etc),
but that's hacky and wasteful.

Instead, you can initialize multiple `Random.Seeded` object with random seeds from an *existing* `Random.Seeded` object,
like so:

```js
const parent = Random.Seeded.fromFixed(0);
const child1 = new Random.Seeded(parent.seed());
const child2 = new Random.Seeded(parent.seed());
// child1.random() != child2.random()
```

This ensures that your "child" PRNGs will always produce the same sequence of values
if you start from the same "parent" PRNG,
while leveraging the full possible seed entropy.

To generate this value:

1. Obtain 256 random bits from the PRNG.
2. Return a (length 32) Uint8Array containing those bits.

### Serializing/Restoring/Cloning a PRNG: the `.getState()` and `.setState()` methods ###

The `.getState()` method returns a fresh `Uint8Array` containing the PRNG's current state.
(Note: the state is different and larger than a seed; 112 bytes vs 32.)

The `.setState()` method takes a `Uint8Array` containing a PRNG state,
verifies that it's a valid state for the PRNG
(correct size, and any other constraints)
and replaces its own state with that data
(copying from the argument,
not using the object directly).

You can then clone a PRNG like:

```js
const prng = Random.Seeded.fromFixed(0);
for(let i = 0; i < 10; i++) prng.random(); // advance the state a bit
const clone = Random.Seeded.fromState(prng.getState());
// prng.random() === clone.random()
```

For example, a game can store the current state of the prng in a save file,
ensuring that upon loading it will generate the same sequence of random numbers
as it would have if the player had continued playing.






Algorithm Choice
----------------

This proposal specifies that the ChaCha12 algorithm is used as the PRNG algorithm.
This ensures two things:

1. The range of possible seeds is knowable and stable, so if you're generating a random seed you can take full advantage of the possible entropy.
2. The numbers produced are identical across (a) user agents, and (b) versions of the same user agent.  This is important for, say, using a seeded sequence to simulate a trial, and getting the same results across different computers.


FAQ
----

### Why not a `Math.random()` argument? ###

Another possible approach is to add an options object to `Math.random()`, and define a `state` key that can be provided.  When you do so, it uses that seed to generate the value, rather than its internal seed value.  This approach should be familiar from C/Java/etc.

The downside of this is that you have manually pass the random value back to the generator on the next call as the next `state`, if you're trying to generate multiple values.  That's fairly clunky when all you want is a predictable sequence of values, and means you have to carry around additional state if the random generation happens across functions or callbacks.

It also requires either that the produced value is *suitable* as a state, which isn't always the case (for many algos, states contain much more than 64 bits), or requires `Math.random()`, when invoked with a state, to produce a `{val, nextState}` pair, instead of just producing the value directly like normal.

### We should add `randInt()`/etc as well ###

This proposal is focused specifically on making a seeded PRNG, and intentionally matches the signature/behavior of the current unseeded `Math.random()`. I don't intend to explore additional random methods here, as they should exist in both seeded and unseeded forms.

Instead, <https://github.com/tc39-transfer/proposal-random-functions> is a separate proposal for adding more random functions to the existing unseeded functionality. The intention is that the `Random.Seeded` object from this proposal will grow all the same methods, so if we added `Math.randomInt()`, we'd also get `Random.Seeded.randomInt()`, etc.

Whichever proposal advances first will just concern itself with itself, and whichever advances second will carry the burden of defining that overlap. (That is, if this proposal goes first, then `proposal-random-functions` will define that all its methods also exist on `Random.Seeded`; if it goes first, then this proposal will define that all the new random functions also exist as `Random.Seeded` methods.)
