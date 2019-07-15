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

`Math.seededPRNG({seed})`
-------------------------

I propose to add a new method to the `Math` object, provisionally named `seededPRNG()`. It takes a single options-bag argument, with a required property `seed`, whose value must be either a JS Number or BigInt. *\[And/or a TypedArray?]*

`seededPRNG()` returns a PRNG object, which has a `.random()` member function.  On each invocation, it will output an appropriate pseudo-random number based on its seed, and then update its seed for the next invocation.  These values must approximate a uniform distribution over the range \[0,1), same as `Math.random()`.

That is, the usage will be something like:

```
const prng = Math.seededPRNG({seed:0});
for(let i = 0; i < limit; i++) {
  const r = prng.random();
  // do something with each value
}
```

Serializing/Restoring/Cloning a PRNG
------------------------------------

The current state of the algorithm, suitable for feeding as the `seed` of another invocation of `seededPRNG()` that will produce identical numbers from that point forward, is accessible via a `.seed()` method on the PRNG object.  *\[Should we guarantee what type it's returned as?]*

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

