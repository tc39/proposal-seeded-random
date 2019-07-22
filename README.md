# Seeded Pseudo-Random Numbers

**Stage: 1**

**Champion: Tab Atkins-Bittner**

------

JS's PRNG methods (`Math.random()`, `crypto.getRandomValues()`, etc) are all "automatically seeded" - each invocation produces a fresh unpredictable random number, not reproducible across runs or realms.  However, there are several use-cases that want a reproducible set of random values, and so want to be able to seed the random generator themselves.

1. New APIs like the CSS Custom Paint, which can't store state but can be invoked arbitrarily often, want to be able to produce the same set of pseudo-random numbers each time they're invoked.

    Demo: <https://lab.iamvdo.me/houdini/rough-boxes/> (currently needs Chrome with the Experimental Web Platform Features flag).  This demo uses Math.random() to unpredictably shift the "rough borders", but the Houdini Custom Paint API re-invokes the callback any time the element needs to be "repainted" - whenever it changes size, or is off-screen for a bit, or just generally whenever (UAs are given substantial leeway in this regard). There's no way for the callback to store state for itself (by design), so it can't just pre-generate a list of random numbers and re-use them; instead, it currently just produces a totally fresh set of borders. (You can see this in effect if you zoom the page in or out, as each change repaints the element and re-invokes the callback.)

2. Testing frameworks that utilize randomness for some purpose, and want to be able to use the same pseudo-random numbers on both local and CI, and to be able to reproduce a given interesting test run.

3. Games that use randomness and want to avoid "save-scumming", where players save and repeatedly reload the game until an event comes out the way they want.

Currently, the only way to achieve these goals is to implement your own PRNG by hand in JS. Simple PRNGS like an LCG aren't hard to code, but they don't produce good pseudo-random numbers; better PRNGs are harder to implement correctly. It would be much better to provide the ability to manually seed a generator and get a predictable sequence out.  While we're here, we can lean on JS features to provide a better usability than typical random libs provide for this use-case.

Creating a PRNG: the `Math.seededPRNG({seed})` function
------------------------------------------

I propose to add a new method to the `Math` object, provisionally named `seededPRNG()`. It takes a single options-bag argument, with a required property `seed`, whose value must be either a JS Number or BigInt. *\[And/or a TypedArray?]*

It returns a PRNG object, the usage of which is described below.

Getting a Random Number: the `.random()` method
-----------------------------------------------

To obtain a random number from a PRNG object, the object has a `.random()` method. On each invocation, it will output an appropriate pseudo-random number based on its seed, and then update its seed for the next invocation.  These values must approximate a uniform distribution over the range \[0,1), same as `Math.random()`.

Using the prng object is thus basically identical to using `Math.random()`:

```
const prng = Math.seededPRNG({seed:0});
for(let i = 0; i < limit; i++) {
  const r = prng.random();
  // do something with each value
}
```

Serializing/Restoring/Cloning a PRNG: the `.seed` getter
--------------------------------------------------------

The current state of the algorithm, suitable for feeding as the `seed` of another invocation of `seededPRNG()` that will produce identical numbers from that point forward, is accessible via a `.seed` getter method on the PRNG object.  *\[Should we guarantee what type it's returned as?]*

You can then clone a PRNG like:

```js
const prng = Math.seededPRNG({seed:0});
for(let i = 0; i < 10; i++) prng.random(); // advance the state a bit
const clone = Math.seededPRNG({seed:prng.seed});
// prng.random() === clone.random()
```

Since the seed is publicly accessible, it can be stored in a file/etc for later revival.
For example, a game can store the current state of the seed in a save file,
ensuring that upon loading it will generate the same sequence of random numbers as before.

Making "Child" PRNGs: the `.randomSeed()` method
------------------------------------------------

There are reasonable use-cases for generating *multiple, distinct* PRNGs on a page;
for example, a game might want to use one for terrain generation, one for cloud generation, one for AI, etc.
Using a single PRNG source can be hacked into doing this
(for example, by saying that the first of every three values is for terrain, the second is for clouds, etc),
but that's hacky and wasteful.

One *could* manually generate several sub-PRNGs by using a master PRNG to generate several random values,
and seeding each with the result,
like:

```js
const MAX_SEED = ????;
const parent = Math.seededPRNG({seed:0});
const child = Math.seededPRNG({seed: parent.random() * MAX_SEED});
```

But this limits the entropy of the seeds to the numerical precision of the JS number type.

To avoid all of this and provide a robust way to generate sub-PRNGs,
the PRNG object must have a `.randomSeed()` method.
It's identical to `.random()`,
but rather than producing a JS number that is uniform over the range [0,1),
it produces a pseudo-random BigInt *\[or TypedArray?]* that is uniform over the range of valid seeds.
It then advances the PRNG's internal state,
same as `.random()`.

You can then produce sub-PRNGs like:

```js
const parent = Math.seededPRNG({seed:0});
const child1 = Math.seededPRNG({seed:parent.randomSeed()});
const child2 = Math.seededPRNG({seed:parent.randomSeed()});
// child1.random() != child2.random()
```


Algorithm Choice
----------------

The specification will also define a *specific* random-number generator for this purpose.  *\[Which one?]*  This ensures two things:

1. The range of possible seeds is knowable and stable, so if you're generating a random seed you can take full advantage of the possible entropy.
2. The numbers produced are identical across (a) user agents, and (b) versions of the same user agent.  This is important for, say, using a seeded sequence to simulate a trial, and getting the same results across different computers.

The algorithm used is not, in this proposal, intended to be configurable.

Issue: Why not a `Math.random()` argument?
------------------------------------------

Another possible approach is to add an options object to `Math.random()`, and define a `seed` key that can be provided.  When you do so, it uses that seed to generate the value, rather than its internal seed value.  This approach should be familiar from C/Java/etc.

The downside of this is that you have manually pass the random value back to the generator on the next call as the next `seed`, if you're trying to generate multiple values.  That's fairly clunky when all you want is a predictable sequence of values, and means you have to carry around additional state if the random generation happens across functions or callbacks.

It also requires either that the produced value is *suitable* as a seed, which isn't always the case (for many algos, seeds can have 64+ bits), or requires `Math.random()`, when invoked with a seed, to produce a `{val, nextSeed}` pair, instead of just producing the value directly like normal.

