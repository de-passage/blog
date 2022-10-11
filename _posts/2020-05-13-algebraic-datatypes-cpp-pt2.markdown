---
layout: post
title:  "Algebraic data types in C++. Part 2: Function types"
tags: c++ metaprogramming algebraic-datatypes
---

In the [previous post]({% post_url 2020-05-09-algebraic-datatypes-cpp %}), we built a couple utilities allowing us to build sum and product types. In this one, we'll add support for functions and demonstrate some neat tricks to enhance our EDSL. The code in this article has been tested with Clang 9.0.0.

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

## Single argument functions
Before we start, let's think about what the syntax presented above entails. `ev` is the macro we defined in [part 1]({% post_url 2020-05-09-algebraic-datatypes-cpp %}), it expects a value of our type container type: `type_t<T>`.
The rest is simply operator overloading: `operator,` and `operator->`. The former is very straightforward, but the latter has some stringent restrictions attached to its use. It must be a non-static member function that returns either a pointer or a value of a type providing a valid overload of `operator->`. Moreover, and this is very inconvenient in our case, the identifier on the right side must refer to a member of the type returned by the call to `operator->`. This means that the following syntax cannot be defined in C++: 
```cpp
constexpr static inline auto InvalidSyntax = Int -> (Char | Bool);
```
If we want functions returning some of our algebraic data types defined on the spot, we'll have to provide some predefined identifier. The syntax shall look like the following:
``` cpp
constexpr static inline auto ValidSyntax = Int -> return_(Char | Bool);
```
The syntax presented in the introduction also implies that `Bool`, an alias for `type<bool>` is a member of whatever type is returned by our overloaded operator. This is inconvenient because it means we have to somehow know all the aliases we want to use in our return type at the time of the definition of said class, which severely limits our possibilities.  
For now we'll focus on defining a proper `operator->` that supports the more general syntax:
``` cpp
constexpr static inline auto GeneralSyntax = type<int> -> type<bool>;
constexpr static inline auto EquivalentSyntax = Int -> return_(Bool);
```

The first thing we need to do is provide `type_t` with an overloaded `operator->`. 
But what kind of pointer should it return? 
The important part is that it must contain some sort of member named `type<T>`. 
Moreover, the type of this member must be some instance of `type_t` representing a function. 
It should also contains a function named `return_` that accepts values of type `type_t<T>`.  
Let's define such a type [^1]:
``` cpp
template<class ...Ts>
struct adapter_t {
    template <class U> 
    constexpr type_t<std::function<U(Ts...)>>
     return_(type_t<U>) const noexcept {
        return {};
    }

    template <class U>
    constexpr static inline type_t<std::function<U(Ts...)>> type;
};
```
This class is fairly simple, the only thing to note is that `type` is a static member of `adapter_t`. 
Indeed non-static member cannot be templates, and it turns out that it doesn't matter whether `type` is static or not for the purpose of overloading `operator->`.
The type accepts a template parameter list, which will transfer the information of the type of the parameter of our function type.
I've done a bit of future-proofing in the form of a variadic template. Even though we're presently dealing with single parameter functions, we'll soon add support for multiple parameters.

All that's left is to provide an overload for `operator->` to `type_t`.
``` cpp
template<class ...Ts> struct adapter;

template <class T> struct type_t {
  using type = T;
  constexpr const adapter_t<T> *operator->() const noexcept;
};

template<class ...Ts> struct adapter { /* [...] */ };
template <class... Ts> constexpr static inline adapter_t<Ts...> adapter{};

template <class T>
constexpr const adapter_t<T> *type_t<T>::operator->() const noexcept {
  return &adapter<T>;
}
```
The trick here is that the operator doesn't need to return a pointer to a member. 
It can be any pointer, so we define a variable template at namespace scope that we instanciate when `operator->` is called, and return its address.
We pass the wrapped by `type_t` to `adapter_t` at this point so 
This way `type_t` only needs a forward declaration of `adapter_t` to work and we avoid the problem of having types containing instances of each other.

And with this, we have a working preliminary syntax for function types. 
We incidentally meet the requirements for the higher order functions as well because `->` is left associative in C++.  
``` cpp
type<char> -> type<int> -> type<double>
```
 is parsed the same way as 
```cpp
(type<char> -> type<int>) -> type<double>
```
both of which will evaluate to 
```cpp
 type<std::function<int(char)>> -> type<double>
 ```
then to 
``` cpp 
type<std::function<double(std::function<int(char)>)>>
```
This is the opposite of what happens in Haskell, and may be undesirable.
We will leave it at that today, and explore ways to deal with the issue in a following article.

## Handling multiple parameters
With the code of the previous section in place, handling multiple parameters is fairly simple. All we need is to overload `operator,` for `type_t` to produce some type with `operator->` returning the adress of `adapter<Ts...>` where `Ts` is the template parameter pack of the types of the parameters to the function.

Let's start by defining a type with the appropriate `operator->`[^2]:
``` cpp 
template <class... Ts> struct argument_pack_t {
  constexpr const adapter_t<Ts...> *operator->() const noexcept {
      return &adapter<Ts...>;
  }
}; 
```
This is extremely simple because the logic of building the function type is already handled in the `adapter_t` structure.
`adapter_t<Ts...>` neither have nor need the information of what exactly instanciated it, all it has is a parameter list corresponding to the types of the parameters of the function type to build.  
All we have left to do is to create this new structure out of `type_t` using the comma operator.
``` cpp
template<class T, class U>
constexpr argument_pack_t<T, U> operator,(type_t<T>, type_t<U>) noexcept {
    return {};
}
```
`operator,` is left-associative. Therefore, in the case of more than 2 parameters in our pack, we need to overload the operator to accept `argument_pack_t<Ts...>` as the left-hand parameter.
Depending on the syntax we want to allow, we could also overload the operator to accept in right-hand or both positions. Even if we want to disallow them, it is a good idea to disable them explicitely with SFINAE or `static_assert`.
The default behaviour of the comma in C++, in the absence of a matching overload, is to evaluate the left hand side, then discard it silently. 
This can lead to very confusing behaviour for the user if the overload is not properly covering all the possible cases.
``` cpp 
template<class T, class U>
constexpr argument_pack_t<T, U> operator,(type_t<T>, type_t<U>) noexcept {
    return {};
}
template<class ...Ts, class U>
constexpr argument_pack_t<Ts..., U> operator,(argument_pack_t<Ts...>, type_t<U>) noexcept {
    return {};
}

template<class... Ts> struct fail_t : std::false_type {};
template<class... Ts> [[maybe_unused]] // The only case this value is used is to make the build fail
constexpr static inline bool fail = fail_t<Ts...>::value;

// The return type is auto to force evaluation in unevaluated context.
// If spelled out, decltype(<expr using comma>) would incorrectly succeed and 
// produce the return type of the function
template<class T, class ...Us>
constexpr auto operator,(type_t<T>, argument_pack_t<Us...>) noexcept {
    static_assert(fail<T>, "illegal syntax"); // We need the indirection of fail<T>,
                                              // static_assert(false) would fail the build even 
                                              // if the function is never called
}

template<class ...Ts, class ...Us>
constexpr auto operator,(argument_pack_t<Ts...>, argument_pack_t<Us...>) noexcept {
    static_assert(fail<Ts...>, "illegal syntax");
}
```
And that's it. We can now write multi parameter function types with our EDSL. All that's left to do is to make the return types look in line with the rest.

## Cheating our way into nice syntax

As we explained earlier, in order to use our aliases in the return type of our functions, we need to somehow inject them inside of the definition of `adapter_t`.
The obvious way to do this is through public inheritance. We'll bundle our types into a structure then have `adapter_t` publically inherit from that structure.
However, were we to release this code as a library, we would rather not have the user poke around in the header files to configure it.
Analysing what we have so far, we can see that the only type that the user manipulates explictely is `type_t`. Everything else is generated by the operators.
This means we can use `type_t` as a configuration point. 

We'll add a template parameter to `type_t`, which will be the structure containing the aliases. We then propagate that structure throughout `argument_pack_t` if necessary, down to `adapter_t`.
``` cpp
struct default_aliases {
 // ...
};

template <class T, class Aliases = default_aliases> struct type_t {
  using type = T;
  constexpr const adapter_t<Aliases, T> *operator->() const noexcept;
};

template <class Aliases, class First, class... Ts> struct argument_pack_t {
  constexpr const adapter_t<Aliases, First, Ts...> *operator->() const noexcept;
};

template <class Aliases, class... Ts>
struct adapter_t : Aliases { // This won't work
  template <class U>
  constexpr type_t<std::function<U(Ts...)>, Aliases> return_(type_t<U, Aliases>) const noexcept;

  template<class U>
  constexpr static inline type_t<std::function<U(Ts...)>, Aliases> type{}; 
};
```
Of course, all our functions using `type_t`, `adapter_t` and `argument_pack_t` need to be adapted. I won't provide the code here as it is mostly bookkeeping.  
The only this to worry about is the case where the user provides `type_t` instances with different aliases (for example `type<T, alias_list_1> -> type<U, alias_list_2>`, or worse different aliases in a function with multiple parameters). The most obvious (and in my opinion the most sensible) option is to simply disallow this case and force the user to use the same alias type throughout the whole program. It is easy to provide utilities to swap the alias parameter if desired[^3].

This choice has the perverse effect that the above code cannot be used as is. Let's think about what the definition of an alias would look like:
``` cpp
struct my_aliases {
    constexpr static inline type_t</* what comes here ? */, my_aliases> Int{};
};
```
It may be tempting to write `int` in place of the comment, but then the result of `type<T> -> Int` would be `int` rather than `function<int(T)>`!
We need to somehow communicate to our alias structure what the arguments of the function were. 
A first intuition could be to make `my_aliases` a variadic template structure to communicate the parameter list, but then we'd have another problem.
``` cpp
template<class ...Ts>
struct my_aliases {
    constexpr static inline type_t<std::function<int(Ts...)>,
                                   my_aliases</* what then? */>> Int{};
};

constexpr static inline type_t<int, my_aliases</* has to match here too*/>> Int{};
```
We could change all our structures to take in a class template as the alias parameter, and this would indeed work:
``` cpp
template<class T, template<class ...> class Aliases> struct type_t;
template<template<class ...> class Aliases, class T> struct argument_pack_t;
template<template<class ...> class Aliases, class T> struct adapter_t;

template<class ...Ts>
struct my_aliases {
    constexpr static inline type_t<std::function<int(Ts...)>,
                                   my_aliases> Int{};
};

constexpr static inline type_t<int, my_aliases> Int{};
```
However I am not a fan of this solution. First, it involves a lot of modifications and a lot of typing to bring everything in line. Second, and most importantly, it delegates to the user the responsibility of constructing the function types properly[^4].

In order to circumvent those two issues, we will do a bit more work on our side. First, we'll modify our alias declaration to be a template type member of a non-template configuration type. This allow us to keep all the prototypes we have so far.  
Second, the template parameter we receive will be a special type that, when used in combination with a normal type in a magic metafunction that we'll define late, will produce the correct type of our alias.
What all this means is that our final alias declaration will look like this:
``` cpp
struct config {
    template<class T>
    struct aliases {
        constexpr static inline auto Int = alias<int, T>;
        constexpr static inline auto Double = alias<double, T>;
    };
};

static constexpr inline auto Int = type<int, config>;
static constexpr inline auto Double = type<double, config>;
```
In order to achieve our little magic trick, we need to make T a type containing two things: the configuration supertype that we'll need to propagate, and the list of arguments of our function type. This is trivial:
``` cpp
template<class Config, class... Params> struct config_container {};
```
We'll add a simple way to extract the type of configuration, and a way to produce a function type from a return type.
``` cpp
template<class Config, class... Params> struct config_container {
    using config = Config;
    template<class Return>
    using make_function = std::function<Return(Params...)>;
};

template<class Config>
using configuration = typename Config::configuration;

template<class Config, class Return>
using make_function = typename Config::template make_function<Return>;
```
We can now define our magic function `alias`, assuming the `T` we'll get in the `aliases` substructure is in fact some instance of `config_container` (we could also provide an `alias_t` type alias. The logic is identical).
``` cpp
template<class Return, class Config>
constexpr static inline 
    type_t<make_function<Config, Return>, configuration<Config>> 
        alias{};
```
The only thing left to do is to make `adapter_t` inherit from the right type, specialized with the right `config_container`:
```cpp
template<class Config, class ...Parameters>
using make_base_class =
    typename Config::template aliases<config_container<Config, Parameters...>>;

template<class Config, class ...Parameters>
struct adapter_t : make_base_class<Config, Parameters...> {
    /// [...]
};
```
And with this, our EDSL for function types is complete! All we have to do is list the aliases we want to use in the `aliases` substructure and give the `config` one to `type_t`, and we can have whatever return type we want (within the restrictions of the C++ syntax obviously).

## An alternative syntax
The last part was quite a bit of work for such a minor detail, but I have never seen `operator->` exploited in this way so I thought it would be nice to explain how it can be done.

If we're willing to depart a bit more from the Haskell syntax, we could go another route and define a pseudo operator to do the same job. This has the added benefit of enabling normal expressions in the return type.  
We'll use an imaginary operator `-D>`, but anything consisting of a valid C++ identifier surrounded by binary operators could do.
In this case, our pseudo operator will be the identifier `D` between the subtraction and the greater-than comparison operators.

We need to define a special type, a global instance thereof, and some other intermediary type that will contain intermediary information:
``` cpp
struct function_pseudo_operator_core {}
    constexpr static inline D{};
template<class C, class ...Ts>
struct partially_applied_function_pseudo_operator{};
```
Then we define operator overloads accepting `type_t` and `argument_pack` on both sides of our pseudo operator.
Since subtraction has higher precedence than greater-than, we treat this case first. Its return type shall be the intermediary type:
``` cpp
template<class T, class C>
partially_applied_function_pseudo_operator<C, T> 
    operator-(type_t<T, C>, function_pseudo_operator_core) noexcept {
    return {};
}

template<class ...Ts, class C>
constexpr partially_applied_function_pseudo_operator<C, Ts...> 
    operator-(argument_pack_t<C, Ts...>, function_pseudo_operator_core) noexcept {
    return {};
}
```
We then treat the application of this intermediary structure as a `type_t`:
``` cpp
template<class C, class R, class... Ts>
constexpr type_t<std::function<R(Ts...)>, C>
    operator>(partially_applied_function_pseudo_operator<C, Ts...>, type_t<R, C>) noexcept {
    return {};
}
```
And we're done. Mostly. The following works as we expected:
``` cpp 
static_assert(std::is_same<ev(Char -D> Int), std::function<int(char)>>)
static_assert(std::is_same<ev(Char -D> (Double & Int),
                              std::function<std::tuple<double, int>(char)>>)
```
But chaining the pseudo operator doesn't. Indeed, an expression such as `Char -D> Int -D> Double` is parsed as 
``` cpp
((Char - D) > (Int - D)) > Double
```
We need to define an additional overload for this case.
``` cpp
template<class C, class... Ts, class R>
constexpr partially_applied_function_pseudo_operator<C, std::function<R, Ts...>>
    operator>(partially_applied_function_pseudo_operator<C, Ts...>,
              partially_applied_function_pseudo_operator<C, R>) noexcept {
    return {};
}
```
Now, there is something wrong with this definition: what happens if we write the following?
``` cpp
constexpr static inline auto Invalid = Char -D> (Int, Int) -D> Double;
```
We get a compilation error, because the template parameter pack of the right hand type contains two types, while we only handle one in our overload. And it makes sense, what would `Char -D> (Int, Int)` mean? We could cope out by hand coding rules to turn it into a `tuple` or something equivalent, but it wouldn't be very elegant. The best case scenario would be to invert the associativity of our pseudo operators to make them behave like in Haskell, but we have no way to do that easily at the moment.

## Conclusion
We implemented in this article two ways of representing function types, both with their set of problems. We have been using `std::function` as an underlying representation for our function types, but this too, brings some issues. While `std::tuple` and `std::variant` know the types they contain at the point of type declaration, they can be made allocation-free. 
`std::function`, on the other hand, needs to be able to wrap any type matching its interface. This is achieved by generalized type erasure, which implies dynamic memory allocation if the size of the object exceeds the space allocated for small functor optimization.  
On top of this, there are several ways to represent function types in C++. What if function pointers are enough in our case? And what if we'd prefer to do away with the standard library and use our own types.

We will adress all these concerns in the next article, by implementing a proper evaluation procedure for our EDSL.  

---
[^1]: We could use `type_t` directly with some sort of marker indicating that we're building a function type, but that's not how I did it originally and I suspect it would complicate what's coming even more.
[^2]: Here again, we could use `type_t`, but I fear that we would end up in a maze of partial specializations that would turn out to be complicated to make evolve to meet new requirements.
[^3]: None of our types actually need what's contained in our alias structure. It shouldn't make a difference if the result of a computation sees its alias template parameter replaced with another to fit within the context of another computation.
[^4]: And we know better than to trust user input, don't we?
