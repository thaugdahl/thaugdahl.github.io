+++
date = '2025-07-26T23:43:19+02:00'
draft = true
title = 'Overload Set'
+++

# Overload Set

An overload set is, essentially, a candidate set of functions that can be invoked
on a given argument. See [overload resolution](https://en.cppreference.com/w/cpp/language/overload_resolution.html).

## Simple definition

Note that this simplified example doesn't consider inherited types, and low-cost implicit conversions. Refer to the rules of [overload resolution](https://en.cppreference.com/w/cpp/language/overload_resolution.html) for specific rules motivating overload selection.

```cpp
struct my_specific_overload_set {
    // (1)
    int operator()(const A &a) { /* Do something with a const A*/ }

    // (2)
    int operator()(B &b) { /* Do something with a B*/ }

    // (3)
    int operator()(C c) { /* Do something with a copied or pure A*/ }

    // (4) Sink function, accepts everything else
    template <class T>
    int operator()(T &&other) { /* Do something with a perfect-forwarded T */ }
};

int main() {
    A a;
    B b;
    C c;

    D d;


    my_specific_overload_set f{};

    f(a); // (1)
    f(b); // (2)
    f(c); // (3)
    f(d); // (4)
    f(A{}); // (4)
}
```

## Automatic overload set (≥ C++17)

Since C++17 bestowed variadic using statements on us, an automatic overload set
is very simply defined as:

```cpp
template <class... Fs>
struct overload_set : public Fs... {
    using Fs::operator()...;

    overload_set(Fs &&...fs) : Fs{std::forward<Fs>(fs)}... {}
};
```

You may have to explicitate a deduction guide for this, but this is, in
essence, an overload set that can be instantiated with any number of
non-conflicting lambdas and function pointers.

You could also use the older style explicit function to aid deduction:

```cpp
template <class... Fs>
overload_set<Fs...> make_overload(Fs &&...fs)
{
    return overload_set<Fs...>(std::forward<Fs>(fs)...);
}
```

## Automatic overload set (< C++17)


Observe the following code for a recursive definition that yields the same
functionality as the C++17 example.

```cpp
template <class... Fs>
struct overload_set;

template <class F1, class... Fs>
struct overload_set<F1, Fs...> : public F1, public overload_set<Fs...>
{

    using F1::operator();
    using overload_set<Fs...>::operator();

    overload_set(F1 &&f1, Fs && ...fs)
        noexcept(std::is_nothrow_move_constructible<F1>::value &&
                std::is_nothrow_move_constructible<overload_set<Fs...>>::value)
        : F1(std::forward<F1>(f1)), overload_set<Fs...>(std::forward<Fs>(fs)...)
    {}
};

template <class F1>
struct overload_set<F1> : public F1
{

    using F1::operator();

    overload_set(F1 &&f1)
        noexcept(std::is_nothrow_move_constructible<F1>::value)
        : F1(std::forward<F1>(f1))
    {}
};


template <class... Fs>
overload_set<Fs...> make_overload(Fs &&...fs)
{
    return overload_set<Fs...>(std::forward<Fs>(fs)...);
}
```

The "trick", if you will, that the using declaration for the nested overload
set will pull in all the constituent function call operator overloads from it.
In total, the topmost instance will thus hold all function call operators and
form a full overload set.




