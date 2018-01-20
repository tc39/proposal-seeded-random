# JS Seeded Random

JS's random-generation methods (`Math.random()`, `crypto.getRandomValues()`, etc) are all "automatically seeded" - each invocation produces a fresh unpredictable random number, not reproducible across runs or realms.  However, there are several use-cases that want a reproducible set of random values, and so want to be able to seed the random generator themselves.

In particular, <https://lab.iamvdo.me/houdini/rough-boxes/> (currently needs Chrome with the Experimental Web Platform Features flag).  This demo uses Math.random() to unpredictably shift the "rough borders", but the Houdini Custom Paint API re-invokes the callback any time the element needs to be "repainted" - whenever it changes size, or is off-screen for a bit, or just generally whenever (UAs are given substantial leeway in this regard). There's no way for the callback to store state for itself (by design), so it can't just pre-generate a list of random numbers and re-use them; instead, it currently just produces a totally fresh set of borders. (You can see this in effect if you move the sliders to change the thickness or roughness, as each change re-invokes the callback.)

Currently, the only way to get around this would be for the callback to implement its own RNG by hand in JS. This seems fairly ridiculous, and this is only the tip of the iceberg in interesting random-using use-cases.  It would be much better to provide the ability to manually seed a generator and get a predictable sequence out.  While we're here, we can lean on JS features to provide a better usability than typical random libs provide for this use-case.

`Math.seededRandoms(seed)`
----------------------------

I propose to add a new method to the `Math` object, provisionally named `seededRandoms()`. It takes a single required argument, the `seed`, which is either a JS integer or BigInt.  

It is a generator function, and the iterator it returns will yield an infinite sequence of random values seeded by the provided `seed`.

That is, the usage will be something like:

```
for(const [i,r] of enumerate(Math.seededRandoms(0))) {
  // do something with each value
  if(i >= limit) break;
}
```

The specification will also define a *specific* random-number generator for this purpose.  (I do not have one in mind; happy to let more informed people make this decision.)  This ensures two things:

1. The range of possible seeds is knowable and stable, so if you're generating a random seed you can take full advantage of the possible entropy.
2. The numbers produced are identical across (a) user agents, and (b) versions of the same user agent.  This is important for, say, using a seeded sequence to simulate a trial, and getting the same results across different computers.

The algorithm used is not, in this proposal, intended to be configurable.

Issue: Resuming a seeded sequence later
---------------------------------------

`Math.random()` returns a value between 0 and 1.  `Math.seededRandoms()` wants a seed value that is an integer (standard for random algos).  If `Math.seededRandoms()`'s output value is also between 0 and 1, then it's not appropriate to use for a seed value directly.  This is problematic if you want to save the current state of the generator so you can resume from the same point later, and object references will expire (such as storing itinto localStorage, or sending over the network).

Possible solutions:

1. `Math.seededRandoms()` returns an integer in whatever range the random-number algo produces.  It can then be fed directly back in as a seed value.  Con is that the behavior now differs from `Math.random()`.
2. `Math.seededRandoms()` returns a value between 0 and 1.  The spec defines how to turn this number back into a seed value (presumably just `Math.floor(val * maxRandomValue)`?), and authors do that themselves when required.  (This presumes that the generator *actually* generates an integer, and returns a `[0,1]` value simply by dividing the integer by the maximum possible integer. Is this a valid assumption?)

Issue: Why not a `Math.random()` argument?
------------------------------------------

Another possible approach is to add an options object to `Math.random()`, and define a `seed` key that can be provided.  When you do so, it uses that seed to generate the value, rather than its internal seed value.  This approach should be familiar from C/Java/etc.

The downside of this is that you have manually pass the random value back to the generator on the next call as the next `seed`, if you're trying to generate multiple values.  That's fairly clunky when all you want is a predictable sequence of values, and means you have to carry around additional state if the random generation happens across functions or callbacks.

I suspect it's rare to need a single seeded random value; most use-cases for this want a sequence of several values. As such, it's worthwhile to make that use-case easier, and JS's iterator protocol conveniently wraps all the state-management up for you and hides it.
