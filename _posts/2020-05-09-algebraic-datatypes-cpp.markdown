---
layout: post
title:  "Algebraic data types in C++. Part 1"
date:   2020-05-09 12:11:28 +0800
tags: c++ metaprogramming algebraic-datatypes
---

While playing with Haskell, I have been intrigued by how easy it is to define and use [algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type). Although the template system makes it fairly straightforward to define this kind of types in C++ using `std::tuple` and `std::variant`, I wanted to experiment with the idea of providing a simple way of manipulating algrebraic data types, free of the clutter of templates.

## Design goals

In Haskell, we can express a product type (that is to say a data structure combining two types) like this:
~~~ haskell
data Pair = Pair Int Int

pair :: Pair
pair = Pair 0 0
~~~
This code defines a new type `Pair` containing two integers and a value `pair` of this type containing 0 twice. An equivalent in C++ could be: 

~~~ cpp
using Pair = std::pair<int, int>;

Pair pair{0, 0};
~~~
It is worth noting that the Haskell code produces a completely new type in the process, while the C++ version produces a type alias. We won't concern ourselves with the difference in this article. Unfortunately creating new types in C++ is not as straightforward as we would like it to be, and we'll save the problem for a future post.

Haskell also has built-in support for sum types (disjoint unions of two types). 
~~~ haskell 
data Either = Left Int | Right Char

left :: Either
left = Left 0

right :: Either
right = Right 'a'
~~~
The equivalent C++ code (with the same caveat) would be:
~~~ cpp
using Either = std::variant<int, char>;

Either left{0};
Either right{'a'};
~~~

We can also combine the above in the following way: 
~~~ haskell 
data Combination
    = One Int 
    | Two Int Char
    | Three Int Char Double

one :: Combination
one = One 0

two :: Combination
two = Two 0 'a'

three :: Combination
three = Three 0 'a' 0.0
~~~
~~~ cpp
using Combination = 
    std::variant<int,
                 std::pair<int, char>,
                 std::tuple<int, char, double>>;

Combination one{0};
Combination two{std::pair{0, 'a'}};
Combination three{std::tuple{0, 'a', 0.0}};
~~~
As you can see, as the complexity of the types increase, it also become harder to read and understand 

In this post, we'll use simple metaprogramming techniques to write the expressions above in the following way:
~~~ cpp 
using Pair = ev(Int & Int);
using Either = ev(Int | Char);
using Combination = ev( Int
                      | Int & Char
                      | Int & Char & Double);
~~~
Admittedly it is neither very impressive nor very useful, but it will lay the foundations for more interesting applications that we'll explore in subsequent articles.

`std::variant` is a C++17 addition to the standard library, so the code will use C++17. In my understanding of the inner workings of `std::variant`, there is nothing preventing one to implement an equivalent class in C++11. It should be fairly straightfoward to translate the code to C++11[^1].

## Type to value, value to type

The first thing to notice in the code above is that we use operators to manipulate the "types". As operators only work on values, it follows that we need to somehow represent our types as values. If you have ever seen the Boost.Hana library, the concept should be familiar to you. The trick is simply to define an empty structure with a single template parameter and instanciate it. 
``` cpp
template <class T>
struct type_t {
    using type = T;
};
 
template <class T>
constexpr inline static type_t<T> type{};
```
With this, we can now use `type<int>` as a value representing the type `int`. I also embedded the type itself as a using declaration to be able to extract it from the value. 

With our new variable template in hand, we can define a few aliases for convenience.
``` cpp
constexpr inline static auto Int = type<int>;
constexpr inline static auto Char = type<char>;
constexpr inline static auto Double = type<double>;
```

We can also very simply define equality for our instances of type_t. Since they have no state, for any given T, any instance of `type_t<T>` is equal to all other instances of `type_t<T>` and different from all instances of `type_t<U>` where U is a different type from T.  
Implementing the corresponding operators is trivial using `<type_traits>` header of the standard library.
``` cpp
template <class U, class V>
constexpr bool operator==(type_t<U>, type_t<V>) noexcept {
  return std::is_same_v<U, V>;
}

template <class U, class V>
constexpr bool operator!=(type_t<U> u, type_t<V> v) noexcept {
  return !(u == v);
}
```

Converting back from a value to a type is not very complicated either. The only hurdle is that we cannot write `type<int>::type` since the double colon syntax can only be applied to types and namespaces.
Converting a value to a type is done using the `decltype` operator.
``` cpp
static_assert(std::is_same_v<decltype(Int)::type, int>);
```
We'll also define a macro to automate the boilerplate away. 
``` cpp
#define ev(...) decltype(__VA_ARGS__)::type

static_assert(std::is_same_v<ev(Int), int>);
```
I am using a variadic macro as a bit of future-proofing. C++ types sometimes contain comma, which would break a single argument macro. For example `ev(type<tuple<int, int>>)` would fail to compile.  
If you're using C++20, you can do away with the macro and use a template alias to extract the type.
``` cpp 
template<auto T>
using ev = decltype(T)::type;

static_assert(std::is_same_v<ev<Int>, int>);
```

## Combining types

Now that we can turn types into values and back, all we need is a way to combine those values into values representing combinations of `std::tuple` and `std::variant`. Since we've determined earlier that we want to use operators to do this, we can already write a stub of the appropriate function. 
``` cpp
template<class T, class U>
type_t<std::tuple<T, U>> operator&(type_t<T>, type_t<U>) {
    return {};
}

static_assert(std::is_same_v<ev(Int & Char), std::tuple<int, char>>);
```
If you are not used to this kind of metaprogramming, it may be surprising to find out that this code does compile. The body of the function is mostly empty because, as `type_t` is an empty type, there is nothing meaningful that we can do with its instances. We simply create a new one to be returned, and the magic happens in the template deduction process.  
As the compiler encounters the expression `Int & Char`, which is short for `type<int> & type<char>`, it will look up the declaration that matches the type of the expression, and instanciate the template function above, deducing T to be `int` and U to be `char`. The return type mechanically becomes `type_t<std::tuple<int, char>>`, which is almost what we want.

Almost, because we have a problem with chaining operator calls. The operator `&` is left associative, so the result of the expression `ev(Int & Char & Double)` would be of type `std::tuple<std::tuple<int, char>, double>`, rather than the desired `std::tuple<int, char, double>`. We need to somehow merge the new value with the tuple in the return type. 

C++ doesn't allow us to have partially specialized template functions, but fortunately we can overload a function with more restrictive versions. In order to solve the problem we have, we need a total 4 different functions to deal with all the cases:
``` cpp
// General case
template<class T, class U>
type_t<std::tuple<T, U>> operator&(type_t<T>, type_t<U>) {
    return {};
}

// Left hand value is a tuple
template<class U, class... Ts>
type_t<std::tuple<Ts..., U>> operator&(type_t<std::tuple<Ts...>>, type_t<U>) {
    return {};
}

// Right hand value is a tuple
template<class T, class ...Us>
type_t<std::tuple<T, Us...>> operator&(type_t<T>, type_t<std::tuple<Us...>>) {
    return {};
}

// Both values are tuples
template<class ...Ts, class ...Us>
type_t<std::tuple<Ts..., Us...>> operator&(type_t<std::tuple<Ts...>>, type_t<std::tuple<Us...>>) {
    return {};
}

static_assert(std::is_same_v<ev(Int / Char), std::tuple<int, char>>);
static_assert(std::is_same_v<ev(Int / Char / Double), std::tuple<int, char, double>>);
static_assert(std::is_same_v<ev(Int / (Char / Double)), std::tuple<int, char, double>>);
static_assert(std::is_same_v<ev((Int / Double) / (Char / Double)), std::tuple<int, double, char, double>>);
```
And with this, the code works for sum types! We could simply copy paste the code and replace tuple by variant, and be done with it. But for extra style points, we'll factor the merging logic in its own bit of template metaprogramming.
``` cpp
// General case
template<template<class...> class S, class T, class U> 
struct merge { using type = S<T, U>; };

// Left hand type is a S (some sort of meta container, a.k.a. a template type)
template<template<class...> class S, class ...Ts, class U> 
struct merge<S, S<Ts...>, U> { using type = S<Ts..., U>; };

// Right hand type is a S
template<template<class...> class S, class T, class ...Us> 
struct merge<S, T, S<Us...>> { using type = S<T, Us...>; };

// Both types are S
template<template<class...> class S, class ...Ts, class ...Us> 
struct merge<S, S<Ts...>, S<Us...>> { using type = S<Ts..., Us...>; };

template<template<class...>class S, class T, class U>
using merge_t = typename merge<S, T, U>::type;

template<class T, class U>
type_t<merge_t<std::tuple, T, U>> operator&(type_t<T>, type_t<U>) {
    return {};
}

template<class T, class U>
type_t<merge_t<std::variant, T, U>> operator|(type_t<T>, type_t<U>) {
    return {};
}
```
All we're doing is enumerating our four cases in the various partial specializations of `merge` and using them in the return type (through `merge_t` for convenience). We've parameterized the "container" so that we can use the same metafunction in both operators.  
When presented with a call to one of the operators, the compiler will try to instanciate its return type. In order to do so, it must instanciate `merge<T, U>` and will select the most restrictive version possible. It will then fill in the body of the class, creating the `type` alias refering to the tuple we want. Then, the `merge_t` alias will extract that inner type and complete the return type.

Incidentally, the priority of the C++ operators takes care of ordering the calls and we are already in possession of a small library fitting the initial requirement [^2].
~~~ cpp
using Combination = 
    std::variant<int,
                 std::tuple<int, char>,
                 std::tuple<int, char, double>>;

constexpr static inline auto 
    combination = Int 
                | Int & Char
                | Int & Char & Double;

static_assert(combination == type<Combination>);
~~~

## Conclusion

Our little experiment is a good example of what you can achieve with a little bit of metaprogramming, but although we met our initial goal, the result is somewhat unsatisfactory. There is no support for nested structures, function types or recursive types. We also didn't pay attention at what actually instanciating these types looks like (hint: it can get hairy), or how to use them.

We'll address those concerns, and probably more, in future articles. In the meantime, you can get the code [on my Github](https://github.com/de-passage/algebraic-datatypes.cpp).  
  

---

[^1]: To the extent that the implementation of `std::variant` can be called "straightforward".
[^2]: We're using `std::tuple` rather than `std::pair` but it is almost the same thing. It is possible to force the use of `pair` in the case where there are only two values, but the benefit is small compared to the hassle. 