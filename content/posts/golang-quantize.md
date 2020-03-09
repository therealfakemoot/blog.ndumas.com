---
title: "Golang Quantize"
date: 2018-04-22T17:30:51Z

tags: ["golang","math"]
series: ''
draft: false
---

# The Goal

Before going too deep into the implementation details of Genesis, I'll touch on the high level aspect of quantization. Quantization is a technique used to map arbitrary inputs into a well defined output space. This is, practically speaking, a hash function. When the term 'quantization' is used, however, it's typically numeric in nature. Quantization is typically used in audio/image processing to compress inputs for storage or transmission.

# Quantizing OpenSimplex
My use case is a little less straightforward. The OpenSimplex implementation I'm using as a default noisemap generator produces values in the [interval](https://en.wikipedia.org/wiki/Interval_(mathematics)#Including_or_excluding_endpoints) [-1,1]. The 3d simplex function produces continuous values suitable for a noisemap, but the values are by nature infinitesimally small and diverge from their neighbors in similarly small quantities. Here's an example of a small sampling of 25 points:

```
[1.9052595476929043e-65 0.23584641815494023 -0.15725758120580122 -0.16181229773462788 -0.2109552918614408 -0.24547524871149487 0.4641016420951697 0.08090614886731387 -0.3720484238283594 -0.5035758520116665 -0.14958647968356706 -0.22653721682847863 0.4359742698469777 -0.6589156578369094 -1.1984697154842467e-66 0.2524271844660192 -0.3132366454912306 -0.38147748611610527 5.131908781688952e-66 0.3814774861161053 0.07543249830197025 0.513284589875744 -1.4965506447200717e-65 0.031883015701786095 0.392504694554317]
```

As you can see, there are some reasonably comprehensible values, like `-0.50357585`, `-0.222`, `0.075432`, but there's also values like `-1.1984697154842467e-66` and `1.9052595476929043e-65`. Mathematically, these values end up being continous and suitable for generating a noisemap but for a human being doing development work and examining raw data, it's almost impossible to have any intuitive grasp of the numbers I'm seeing. Furthermore, when I pass these values to a visualization tool or serialize them to a storage format, I want them to be meaningful and contextually "sane". The noisemap values describe the absolute height of terrain at a given (X,Y) coordinate pair. If we assume that terrain hight is measured in meters, a world whose total height ranges between -1 meter and 1 meter isn't very sensible. A good visualization tool can accomodate this data, but it's not good enough for my purposes.

To that end, I'm working on implementing a quantization function to scale the [-1,1] float values to arbitrary user defined output spaces. For example, a user might desire a world with very deep oceans, but relatively short mountain features. They should be able to request from the map generator a range of [-7500, 1000], and Quantize() should evenly distribute inputs between those desired outputs.

In this way, I'll kill two birds with one stone. The first bird has been fudging coefficients in the noise generation algorithm, and at the "edges" of the `Eval3` function to modify the "scale" of the output. This has been an extremely troublesome process because I do not have enough higher maths education to full grasp the entirety of the simplex noise function, and because there's so many coefficients and "magic numbers" involved in the process that mapping each of their effects on each other and the output simultaneously is a very daunting task. The second bird is a longer term goal of Genesis involving detailed customizability of terrain output.

By virtue of having an effective quantization function, a user will be able to customize terrain to their liking in a well-defined manner, uniformly scaling the map however they desire.

# The problem
Unfortunately, my hand-rolled implementation of Quantize is not yet fully functional. Open source/free documentation on quantization of floating point values to integer domains is very sparse. I was only able to find one StackOverflow post where someone posted their MATLAB implementation which was marginally useful but largely incomprehensible, as I currently do not know or own MATLAB.

With some trial and error, however, I was able to get very close to a working *and* correct implementation:

```
Output Domain: {Min:-5 Max:5 Step:1}
[1.9052595476929043e-65 0.23584641815494023 -0.15725758120580122 -0.16181229773462788 -0.2109552918614408 -0.24547524871149487 0.4641016420951697 0.08090614886731387 -0.3720484238283594 -0.5035758520116665 -0.14958647968356706 -0.22653721682847863 0.4359742698469777 -0.6589156578369094 -1.1984697154842467e-66 0.2524271844660192 -0.3132366454912306 -0.38147748611610527 5.131908781688952e-66 0.3814774861161053 0.07543249830197025 0.513284589875744 -1.4965506447200717e-65 0.031883015701786095 0.392504694554317]
[-1 1 -2 -2 -3 -3 3 0 -4 -6 -2 -3 3 -7 -1 1 -4 -4 -1 2 0 5 -1 0 2]
```

Unfortunately, my current implementation is outputting values outside the provided domain. it almost looks like the output is twice what it should be, but I know that doing things incorrectly and just dividing by 2 afterwards isn't sufficient. I've got relatively few leads at the moment, but I'm not giving up.

# The Code

{{< highlight go "linenos=table">}}
package main

import (
        "fmt"
        noise "github.com/therealfakemoot/genesis/noise"
        // "math"
        // "math/rand"
)

// Domain describes the integer space to which float values must be mapped.
type Domain struct {
        Min  float64
        Max  float64
        Step float64
}

// func quantize(delta float64, i float64) float64 {
// return delta * math.Floor((i/delta)+.5)
// }

func quantize(steps float64, x float64) int {
        if x >= 0.5 {
                return int(x*steps + 0)
        }
        return int(x*(steps-1) - 1)
}

// Quantize normalizes a given set of arbitrary inputs into the provided output Domain.
func Quantize(d Domain, fs []float64) []int {
        var ret []int
        var steps []float64

        for i := d.Min; i <= d.Max; i += d.Step {
                steps = append(steps, i)
        }

        stepFloat := float64(len(steps))
        // quantaSize := (d.Max - d.Min) / (math.Pow(2.0, stepFloat) - 1.0)

        for _, f := range fs {
                ret = append(ret, quantize(stepFloat, f))
        }

        fmt.Printf("Steps: %v\n", steps)
        // fmt.Printf("Quanta size: %f\n", quantaSize)

        return ret
}

func main() {
        d := Domain{
                Min:  -5.0,
                Max:  5.0,
                Step: 1.0,
        }

        n := noise.NewWithSeed(8675309)

        var fs []float64
        for x := 0.0; x < 10.0; x++ {
                for y := 0.0; y < 10.0; y++ {
                        fs = append(fs, n.Eval3(x, y, 0))
                }
        }

        // for i := 0; i < 20; i++ {
        // fs = append(fs, rand.Float64())
        // }

        v := Quantize(d, fs)

        fmt.Printf("%v\n", fs)
        fmt.Printf("%v\n", v)
}
{{< / highlight >}}

