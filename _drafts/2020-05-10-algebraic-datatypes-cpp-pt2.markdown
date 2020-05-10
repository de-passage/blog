---
layout: post
title:  "Algebraic data types in C++. Part 2: Function types"
tags: c++ metaprogramming algebraic-datatypes
---

In the [previous post]({% post_url 2020-05-09-algebraic-datatypes-cpp %}), we built a couple utilities allowing us to build sum and product types. In this one, we'll add support for functions and demonstrate some neat tricks to enhance our EDSL.

## Taking inspiration from Haskell

In Haskell, function types can be expressed very simply:
``` haskell
type OneParameter = Int -> Bool -- type of a function taking an Int and returning a Bool
type TwoParameters = Char -> Int -> Bool -- takes a Char and an Int and return a Bool
type HigherOrder = (Char -> Int) -> Bool -- takes as input a function from Char to Int, and return a Bool
```
As you may infer from the example above, `Char -> Int -> Bool` is really more a chain of functions than a single one. It is in fact identical to `Char -> (Int -> Bool)`, a function taking a `Char` and returning a function from `Int` to `Bool`. This makes perfect sense in Haskell where functions are automatically curried: functions taking multiple arguments are converted into a succession of function taking one argument and returning a function. In C++ however, there is not much incentive to curry functions as the resulting syntax is rather clunky. We'll instead provide syntax for uncurried function types, which will look like this:
``` cpp
using OneParameter = ev(Int -> Bool);
using TwoParameter = ev((Char, Int) -> Bool);
using HigherOrder = ev((Char -> Int) -> Bool);
```

## Implementation

1 argument -> adapter & modif 
2 arguments -> argument_pack

## Cheating our way into nice syntax

how to use Int and others

## Conclusion
think about the implementation of functions (std function? pointer?), hint at policies

