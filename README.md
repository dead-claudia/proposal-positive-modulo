# Divisor-dependent Modulo

A proposal for adding a divisor-dependent modulo operator to ECMAScript.

## Introduction

The existing dividend-dependent modulo operator returns the sign of the dividend (hence, "dividend-dependent"). However, this sometimes leads to unintuitive behavior. Let's take the simple case of checking for odd numbers:

```js
function isOdd(x) {
    return x % 2 === 1
}

function isEven(x) {
    return x % 2 === 0
}
```

It all here seems working at first glance:

|  `x`   | `isOdd(x)` | `isEven(x)` |
|:------:|:----------:|:-----------:|
| `1`    | `true`     | `false`     |
| `3`    | `true`     | `false`     |
| `2`    | `false`    | `true`      |
| `4`    | `false`    | `true`      |
| `0`    | `false`    | `true`      |
| `-0`   | `false`    | `true`      |
| `-1`   | `true`     | `false`     |
| `-3`   | `true`     | `false`     |
| `-2`   | `false`    | `true`      |
| `-4`   | `false`    | `true`      |
| `1.5`  | `false`    | `false`     |
| `-3.5` | `false`    | `false`     |

But if you forget to test any of these, you'll never notice `isOdd` is buggy:

|  `x`   |         `isOdd(x)`         | `isEven(x)` |
|:------:|:--------------------------:|:-----------:|
| `-1`   | `false` (should be `true`) | `false`     |
| `-3`   | `false` (should be `true`) | `false`     |

To fix this, you'll need to do one of the following:

```js
// Option 1: check both signs
function isOdd(x) {
    return x % 2 === 1 || x % 2 === -1
}

// Option 2: check the absolute value
function isOdd(x) {
    return Math.abs(x) % 2 === 1
}
```

Now it works for everything.

|  `x`   |    `isOdd(x)`    | `isEven(x)` |
|:------:|:----------------:|:-----------:|
| `1`    | `true`           | `false`     |
| `3`    | `true`           | `false`     |
| `2`    | `false`          | `true`      |
| `4`    | `false`          | `true`      |
| `0`    | `false`          | `true`      |
| `-0`   | `false`          | `true`      |
| `-1`   | `true` (correct) | `false`     |
| `-3`   | `true` (correct) | `false`     |
| `-2`   | `false`          | `true`      |
| `-4`   | `false`          | `true`      |
| `1.5`  | `false`          | `false`     |
| `-3.5` | `false`          | `false`     |

Do I expect this to be caught in a code review? Not really.

There's also other issues and glitches people have run into with a dividend-dependent modulo being the default, like in [this StackOverflow question about an issue in C](https://stackoverflow.com/questions/14997165/fastest-way-to-get-a-positive-modulo-in-c-c) where they wanted to allow wraparound on an array where the index might be negative *and* less than `-array.length`. The usual workaround is to do `(a % b + b) % b`.

## Proposal

Now, imagine if we had a `x %% y` that was like `x % y`, but instead of returning the sign of `x`, returned the sign of `y`. Let's look at those again, but swap in the new operators:

```js
function isOdd(x) {
    return x %% 2 === 1
}

function isEven(x) {
    return x %% 2 === 0
}
```

|  `x`   | `isOdd(x)` | `isEven(x)` |
|:------:|:----------:|:-----------:|
| `1`    | `true`     | `false`     |
| `3`    | `true`     | `false`     |
| `2`    | `false`    | `true`      |
| `4`    | `false`    | `true`      |
| `0`    | `false`    | `true`      |
| `-0`   | `false`    | `true`      |
| `-1`   | `true`     | `false`     |
| `-3`   | `true`     | `false`     |
| `-2`   | `false`    | `true`      |
| `-4`   | `false`    | `true`      |
| `1.5`  | `false`    | `false`     |
| `-3.5` | `false`    | `false`     |

## Math properties and optimizability

Suppose those two functions are only called with integers. With the old definition, there's the peculiar case of `-1 % 2 === -1`, so the engine would have extra work to do to desugar. So let's start out at looking at the versions using the current dividend-dependent `x % y`:

```js
function isOddBuggy(x) {
    return x % 2 === 1
}

function isOdd(x) {
    return Math.abs(x) % 2 === 1
}

function isEven(x) {
    return x % 2 === 0
}
```

But a JIT can't just desugar to just `x & 1` because `x % Math.pow(2, y)` for integer `x` and `y >= 0` is *not* equivalent to `x & (Math.pow(2, y) - 1)` for those same `x` and `y`. JS returns `-0` if `-x` is a positive multiple of `y`, and when `x` is both negative and *not* a positive multiple of `y`, the sign is still negative. For example:

| `x`  | `y` | `x % Math.pow(2, y)` | `x & (Math.pow(2, y) - 1)` |
|:----:|:---:|:--------------------:|:--------------------------:|
| `-3` | `1` | `-3 % 2` &rarr; `-1` | `-3 & 1` &rarr; `1`        |
| `-4` | `2` | `-4 % 4` &rarr; `-0` | `-4 & 3` &rarr; `0`        |

This means two things:

1. If `x` is negative and the result is 0, you have to always promote.
2. The code gen ends up about 5x as long.

So if the engine is to optimize those to raw assembly, it'd end up looking something along the lines of this:

```js
// Each line roughly corresponds to a single CPU instruction
function isOddBuggy(x) {
    let t, u

    t = x < 0
    x = x & 1
    t = ToInteger(x === 1)
    u = ToInteger(x === -1)
    x = t | u
    x = Boolean(x)
    return x
}

// Note: engines IIUC don't optimize for `Math.abs(x) % y`
function isOdd(x) {
    let t, u

    t = -x
    if (x < 0) x = t
    t = x < 0
    x = x & 1
    t = ToInteger(x === 1)
    u = ToInteger(x === -1)
    x = t | u
    x = Boolean(x)
    return x
}

function isEven(x) {
    x = x & 1
    x = Boolean(x)
    return x
}
```

Let's compare that with the dividend-dependent `x %% y`:

```js
function isOdd(x) {
    return x %% 2 === 1
}

function isEven(x) {
    return x %% 2 === 0
}
```

A JIT *can* just desugar those to just use `x & 1` because `x %% Math.pow(2, y)` for integer `x` and `y >= 0 && y <= 32` *is* equivalent to `x & (Math.pow(2, y) - 1)` for those same `x` and `y`. Even if `x` is negative or `-0`, the sign of `y` is what's kept, and in this case, is also always positive. For example:

|      `x`      |     `y`      |          `x %% Math.pow(2, y)`          |       `x & (Math.pow(2, y) - 1)`      |
|:-------------:|:------------:|:---------------------------------------:|:-------------------------------------:|
| `-3`          | `1`          | `-3 %% 2` &rarr; `1`                    | `-3 & 1` &rarr; `1`                   |
| `-4`          | `2`          | `-4 %% 4` &rarr; `0`                    | `-4 & 3` &rarr; `0`                   |
| `-2147483648` | `2147483648` | `-2147483648 %% 2147483648` &rarr; `0`  | `-2147483648 & 2147483648` &rarr; `0` |

So those would end up JIT-compiled to much simpler and more performant code, and they wouldn't even have to necessarily optimize for the pattern `x %% y === z` or even `x %% y === 0` for integers (floats would have to check for `NaN`s also, but nothing else):

```js
// Each line roughly corresponds to a single CPU instruction
function isOdd(x) {
    x = x & 1
    x = ToInteger(x === 1)
    x = Boolean(x)
    return x
}

function isEven(x) {
    x = x & 1
    x = Boolean(x)
    return x
}
```

And of course, other positive power-of-2 indices can be optimized for similarly.

## Spec

`x %% y` would have the same precedence and rules for its operands as `x % y` (as in, be a multiplicative operator), and [12.7.3](https://tc39.es/ecma262/#sec-multiplicative-operators-runtime-semantics-evaluation) would be altered accordingly using *T*::modulo as explained below.

The following abstract operations would be added to the BigInt and Numeric types:

BigInt::modulo(*x*, *y*)

1. Let *result* be ? BigInt::remainder(*x*, *y*).
1. If *y* < 0, then set *result* to BigInt::unaryMinus(*result*).
1. Return *result*.

Number::modulo(*x*, *y*)

1. Let *remainder* be ? Number::remainder(*x*, *y*).
1. Let *result* be the Number with the sign of *y* and the magnitude of abs(*remainder*).
1. Return *result*.

> Note: I don't use the `(x % y + y) % y` as the spec here because of three reasons:
>
> 1. For bigints, it just complicates the spec and clouds the intent.
> 2. For floats, it's imprecise compared to `copysign(fmod(x, y), y)`.
> 3. It's meant to model how a native implementation might implement it, not how a transpiler would.
>
> Transpilers usually optimize for code size over mathematical precision, so I'd expect them to do `(x % y + y) % y`.
