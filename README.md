# JS Seeded Random

JS's random-generation methods (`Math.random()`, `crypto.getRandomValues()`, etc) are all "automatically seeded" - each invocation produces a fresh unpredictable random number, not reproducible across runs or realms.  However, there are several use-cases that want a reproducible set of random values, and so want to be able to seed the random generator themselves.

In particular, [[fill in a pointer to the cool rough borders demo]].  This demo uses Math.random() to unpredictably shift the "rough borders", but the Houdini Custom Paint API re-invokes the callback any time the element needs to be "repainted" - whenever it changes size, or is off-screen for a bit, or just generally whenever (UAs are given substantial leeway in this regard). There's no way for the callback to store state for itself (by design), so it can't just pre-generate a list of random numbers and re-use them; instead, it currently just produces a totally fresh set of borders. (You can see this in effect if you move the sliders to change the thickness or roughness, as each change re-invokes the callback.)

Currently, the only way to get around this would be for the callback to implement its own RNG by hand in JS. This seems fairly ridiculous, and this is only the tip of the iceberg in interesting random-using use-cases.  It would be much better to provide the ability to manually seed a generator and get a predictable sequence out.  While we're here, we can lean on JS features to provide a better usability than typical random libs provide for this use-case.

`Math.seededRandoms(seed)`
----------------------------

I propose to add a new method to the `Math` object, provisionally named `seededRandoms()`. It takes a single required argument, the `seed`, which is a JS number (or integer? I dunno what's best).  It is a generator function, and the iterator it returns will yield an infinite sequence of random values seeded by the provided `seed`.

That is, the usage will be something like:

`
for(const [i,r] of enumerate(Math.seededRandoms(0))) {
  // do something with each value
  if(i >= limit) break;
}
`

Why not a `Math.random()` argument?
-----------------------------------

Another possible approach is to add an options object to `Math.random()`, and define a `seed` key that can be provided.  When you do so, it uses that seed to generate the value, rather than its internal seed value.  This approach should be familiar from C/Java/etc.

The downside of this is that you have manually pass the random value back to the generator on the next call as the next `seed`, if you're trying to generate multiple values.  That's fairly clunky when all you want is a predictable sequence of values, and means you have to carry around additional state if the random generation happens across functions or callbacks.

I suspect it's rare to need a single seeded random value; most use-cases for this want a sequence of several values. As such, it's worthwhile to make that use-case easier, and JS's iterator protocol conveniently wraps all the state-management up for you and hides it.
