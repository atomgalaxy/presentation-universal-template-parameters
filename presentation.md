Universal Template Parameters
=============================

P12985R3

By: 

Gašper Ažman,
Bengt Gustafsson,
Corentin Jabot,
Colin MacLean
and Mateusz Pusz



Aim
---

Generalize template parameters to completeness (closure).



Rationale
---------

Metaprogramming libraries routinely encounter limitations.

Remove all artificial ones.



Examples
--------

(they follow)



### Basic pattern matching

P2098R0: `is_specialization_of`

```cpp
auto f(specialization_of<vector> auto&& v) { /*...*/ }
auto f(specialization_of<array> auto&& v) { /*...*/ }
```


(implementation)

```cpp
template <typename T,
          template <__any...> typename Template>
constexpr bool is_specialization_of_v = false;

template <__any... Params,
          template <__any...> typename Template>
constexpr bool is_specialization_of_v<Template<Params...>,
                                      Template>
          = true;

template <typename T,
          template <__any...> typename Template>
concept specialization_of 
          = is_specialization_of_v<T, Template>;
```


##### With Reflection

With P1240R2, the implementation would be like so:

```cpp
template<typename Type,
         template <__any...> typename Templ>
constexpr bool is_specialization_of_v
         = (template_of(^Type) == ^Templ);
```

Alternatively, using a functional interface to avoid instantiations:

```cpp
consteval bool is_specialization_of(
      std::meta::info r_instance,
      std::meta::info r_templ) {
  return template_of(r_instance) == r_templ;
}
```

To define the concept, one still needs universal template parameters.

(Thank you, Daveed)



### Traits

```cpp
template <__any>      constexpr bool is_typename_v    = false;
template <typename T> constexpr bool is_typename_v<T> = true;

template <__any>  constexpr bool is_value_v           = false;
template <auto V> constexpr bool is_value_v<V>        = true;

template <__any> constexpr bool is_class_template_v   = false;
template <template <__any...> typename A>
constexpr bool is_class_template_v<A>                 = true;
```

This suffices for the current language.


#### Proposed Extensions

_variable-template template-parameters_

```cpp
template <__any>
constexpr bool is_variable_template_v                 = false;

template <template <__any...> auto A>
constexpr bool is_variable_template_v<A>              = true;
```


_concept template-parameters_

```cpp
template <__any> constexpr bool is_concept_v          = false;

template <template <__any...> concept A>
constexpr bool is_concept_v<A>                        = true;
```


_universal-template template-parameters_

```cpp
template <__any> constexpr bool is_template_v        = false;

template <template <__any...> __any A>
constexpr bool is_template_v<A>                      = true;
```



### Basic Identifier Usage (`apply`)

```cpp
template <template <__any...> typename F, __any... Args>
using apply = F<Args...>;
```

Usage:

```cpp
// ok, r1 is std::array<int, 3>
using r1 = apply<std::array, int, 3>; 
// ok, r2 is std::vector<int, std::pmr::allocator>
using r2 = apply<std::vector, int, std::pmr::allocator>;
```

<small>(For the theoreticians: `F` functions covariantly here.)</small>


What we actually want to write is

```cpp
template <template <__any...> __any F, __any... Args>
__any apply = F<Args...>;
//~~~~~~~~~ novel: universal alias (but compile-time only)
```

Usage:

```cpp
using r1 = apply<std::array, int, 3>;  // + down with typename
constexpr bool x = apply<std::integral, int>;
constexpr auto pi = apply<std::numbers::pi_v, float>;
```



### Advanced Identifier Usage (`map_reduce`)

```cpp
template <template <__any> __any Map,
          template <__any...> __any Reduce,
          __any... Args>
__any map_reduce = Reduce<Map<Args>...>;
```

Usage:

```cpp
template<auto... xs>
constexpr auto sum = (... + xs);

template <__any... Args>
constexpr int count_types = map_reduce<is_typename_v,
                                       sum,
                                       Args...>;

static_assert(2 == count_types<int, 1, long, std::vector>);
//                             ^^^     ^^^^

```



Proposed Behavior of `__any` Identifiers
------------------------------------------

They are just Dependent Names.


### Dependent names need a fix

Core problem: we assume _parse-token kind_ == _semantic token kind_

```cpp
template<typename T>
constexpr int f() { return sizeof(T); } // f#1 

template<auto v>
constexpr int f() { return v; }         // f#2

template<typename T>
constexpr int h = f<T::name>();  // no way in C++23
```

We either need `typename` or not, there's no way to defer _kind_.

Proposed: we parse as if _constant-expression_, defer kind check to substitution.


Parsing Disambiguation
----------------------

We need to disambiguate when we mean to parse a type or a template.

```cpp
template<template auto U> int caller3() {
  auto u = f<U>();        // Proposed: OK
  auto v = f<U*>();       // Error: U* is not an expression
  auto v = f<typename U*>();  // OK: U* is disambiguated
  using type = X<U>;      // OK

  // Can fail instantiation if argument kind is wrong:
  auto v = U;             // Parses U as expression.
  using t = U*;           // OK: "down with typename"
```


(cont)
```cpp
// template<template auto U> int caller3() ...
  // Error during parse:
  U x;                    // Error: U parsed as expression
  template<typename T>
  using tpl = U<T>;       // Error: U is parsed as a expression.

  template<typename T>
  using tpl = template U<T>; // OK if U is a class template
  typename U x;              // OK if U is a type
  typename template U<int> i;  // OK if U is a class template
  auto vi = template U<int>;   // OK if U is a variable-template
}
```



Splicing in Reflection
----------------------

This is exactly the same thing as reflection needs. We need to solve this
problem anyway.



Fin
===

