---
title: "Making Noise"
date: 2019-02-28T19:37:06Z
draft: false
toc: true
images:
tags:
  - genesis
  - golang
---

# The Conceit
I've written about Genesis [before](/posts/genesis-roadmap/), but it's got a lot of complexity attached to it, and the roadmap I originally laid out has shifted a bit. For this post I'm focusing solely on Phase 1, the generation of geography. This is obviously a fundamental starting point, and it has roadblocked my progress on Genesis for quite some time ( somewhere on the order of 8 years or so ).

My [original implementation](https://github.com/therealfakemoot/genesis_retired) was written in Python; this was...servicable, but not ideal. Specifically, visualizing the terrain I generated was impossible at the time. Matplotlib would literally lock up my entire OS if I tried to render a contour plot of my maps if they exceeded 500 units on a side. I had to power cycle my desktop computer many times during testing.

Eventually, I jumped from Python to Go, which was a pretty intuitive transition. I never abandoned Genesis, spiritually, but until this point I was never able to find technology that felt like it was adequate. Between Go's natural performance characteristics and the rise of tools like D3.js, I saw an opportunity to start clean.

It took a year and some change to make real progress

# Making Noise
Noise generation, in the context of "procedural" world or map generation, describes the process of creating random numbers or data in a way that produces useful and sensible representations of geography. To generate values that are sensible, though, there are some specific considerations to keep in mind. Your typical (P)RNG generates values with relatively little correlation to each other. This is fine for cryptography, rolling dice, and so on, but it's not so great for generating maps. When assigning a "height" value to point (X, Y), you may get 39; for (X+1, Y), you might get -21, and so on.

This is problematic because adjacent points on the map plane can vary wildly, leading to inconsistent or impossible geography: sharp peaks directly adjacent to impossible deep valleys with no transition or gradation between. This is where noise functions come in. Noise functions have the property of being "continuous" which means that when given inputs that are close to each other, the outputs change smoothly. A noise function, given (X, Y) as inputs might produce 89; when given (X+1, Y) it might produce 91. (X, Y+1) could yield 87. All these values are close together, and as the inputs vary, the outputs vary *smoothly*.

There seem to be two major candidates for noise generation in amateur projects: [Perlin noise](https://en.wikipedia.org/wiki/Perlin_noise) and [Simplex noise](https://en.wikipedia.org/wiki/Simplex_noise). Perlin noise was popular for a long while, but eventually deemed slow and prone to generating artifacts in the output that disrupted the "natural" feel of its content. Simplex noise is derived from the idea of extruding triangles into higher and higher dimensional space, but beyond that I haven't got a single clue how it works under the hood. I do know that it accepts integers ( in my use case, coordinates on the X,Y plane ) and spits out a floating point value in the range of `[-1,1]`.

# Quantization
This is something I've written about [before](/golang-quantize/), but shockingly, I was entirely wrong about the approach to a solution. At best, I overcomplicated it. Quantization is, using technical terms, transforming inputs in one interval to outputs in another. Specifcally, my noise generation algorithm returns floating point values in the range `[-1, 1]`. Conceptually, this is fine; the values produced for adjacent points in the x,y plane are reasonably similar.

Practically speaking, it's pretty bad. When troubleshooting the noise generation and map rendering, trying to compare `1.253e-64` and `1.254e-64` is problematic; these values aren't super meaningful to a human. When expressed in long-form notation, it's almost impossible to properly track the values in your head. Furthermore, the rendering tools I experimented with would have a lot of trouble dealing with infinitesimally small floating point values, from a configuration perspective if not a mathematical one.

In order to make this noise data comprehensible to humans, you can quantize it using, roughly speaking, three sets of parameters:

1) The value being quantized
2) The maximum and minimum input values
3) The maximum and minimum output values

Given these parameters, the function is `(v - (input.Min) ) * ( output.Max - output.Min ) / ( input.Max - input.Min ) + output.Min`. I won't go into explaining the math, because I didn't create it and probably don't understand it fully. But the important thing is that it works; it's pure math with no conditionals, no processing. As long as you provide these five parameters, it will work for all positive and negative inputs.

Now, with the ability to scale my simplex noise into ranges that are useful for humans to look at, I was ready to start generating visualizations of the "maps" produced by this noise function. At long last, I was going to see the worlds I had been creating.

# Until Next Time

This is where I'll close off this post and continue with the [solutions](/posts/unfolding-the-map/).
