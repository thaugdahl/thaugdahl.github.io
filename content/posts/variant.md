+++
date = '2025-07-26T23:34:30+02:00'
draft = false
title = 'C++ Variant From Scratch'
+++

# C++ Variant From Scratch

For quite some time now, I've been wanting to do a full-scale implementation
of `std::variant` from scratch. Why would you want to put yourself through
the torment, you ask? Here's what I say:

1. Your understanding of the STL deepens.
2. Your template metaprogramming chops get their much needed exercise.
3. You get your chance to make your sacrifice to the ever-benevolent standards library devs.

By the end of this exercise you're likely going to fall into one of two categories (insert there are 10 kinds of people joke here):

1. You ripped your hair out long ago, and swear never to utter another C++ incantation ever again.
2. You realise C++ is beautifully complex, and you can do more or less what ever you want to it, and it won't complain. It might amputate your feet while you're at it. A nicely symbiotic relationship.

## The anatomy of the variant

1. **Storage:** A variant type is, in the algebraic datatype world referred to as a tagged sum type. It can hold any of the constituent member types.
2. **Utilities**: To be able to interrogate the held type of the variant, we will add some simple functions to get information about the state of the variant.
2. **Visitation:** The infrastructure surrounding the variant type should enable the user to *visit* the variant with a sufficient [overload set](/extras/overload-set).

We will go through both of these.

### Storage

The top-level declaration of the variant looks like the following:

```cpp
template <class... Types>
class Variant;
```

The variant must be able to hold any of the types in the variadic pack. To this
end, we must ensure that the storage we allocate is *at least* as large as the
largest of the types in the pack, and maintains the maximum alignment.

```cpp
template <class... Types>
class Variant {
    //...
    static constexpr std::size_t MaxSize = std::max({sizeof(Types)...});
    static constexpr std::size_t MaxAlign = std::max({alignof(Types)...});

    // (!) Alignment and size taken care of
    alignas(MaxAlign) char storage[MaxSize];

    std::size_t active_index;
    //...
};
```

*Note*: In retrospect, I should have used `std::byte` instead of `char`. The two are close to equivalent (`std::byte` is unsigned), but `std::byte` is more descriptive and opaque.

#### Initialization

Every constructor and assignment is templated, and receives a specific type. If
we want to be able to associated a unique identifier with each type of the
variant, we have to ascertain its position in the top-level parameter pack.
Look at the [TypeIndex](#type-index) utility. In addition to getting a type's
index in a parameter pack, we also need to subscript -- or extract -- single
types by their position in the parameter pack. This is achieved with the
[TypeAt](#type-pack-subscripting) utility.



Now there are some more things to contend with. The first type of a variant should
be default constructible. If we instantiate a variant without an initializer,
we default back to default-instantiating the first type in the variant's type pack.

```cpp
Variant() {
    using FirstType = typename TypeAt<0, Types...>::type;
    new (storage) FirstType{};
    active_index = 0;
}

template <class T>
Variant(T &&value) {
    static constexpr std::size_t type_index = TypeIndex<T, Types...>::value;
    new (&storage) T{std::forward<T>(value)};
    active_index = type_index;
}

template <class T>
void set(T &&value) {
    static constexpr std::size_t type_index = TypeIndex<T, Types...>::value;
    // Clean up previously held resource
    destroy();

    new (&storage) T{std::forward<T>(value)};
    active_index = type_index;
}
```

Assignment will now ensure that the variant's active index correctly identifies
the contained type in the variant. There is one piece of functionality here we
haven't covered yet -- the call to `destroy()`. Let's get into it.

#### Destruction

Our variant type should be able to correctly free its held resources. If the
active type of the variant holds resources, we want to make sure we're respecting
its destructor as well.

The following code makes sure to create a dispatch table that ensures minimal
indirection to invoke the held type's destructor. Since we already have the
`active_index`, we can use it to directly index into a table of destructor
handlers.

Each handler will make sure to cast the storage pointer to a pointer of the
actual held object, and thereby invoke the correct destructor.

*Note*: `IndexSequence` and `MakeIndexSequence` are unnecesarily hand-rolled
implementations of `std::index_sequence`. See [this file](https://github.com/thaugdahl/variant-v2/blob/main/type_utils.hpp) for its
implementation.

```cpp
void destroy() {
    static constexpr auto destructor_table = create_destructor_table();
    if ( is_valueless() ) return;
    destructor_table[active_index](storage);
    active_index = sizeof...(Types);
}

static constexpr auto create_destructor_table() {
    return create_destructor_table_impl(MakeIndexSequence<sizeof...(Types)>{});
}

template <std::size_t... Indices>
static constexpr auto create_destructor_table_impl(IndexSequence<Indices...>) {
    return std::array<void (*)(char *), sizeof...(Indices)> {
        [](char *data) {
            using ActiveType = typename TypeAt<Indices, Types...>::type;
            reinterpret_cast<ActiveType*>(data)->~ActiveType();
        }...
    };
}

~Variant() { destroy(); }
```

The *magic*, if you want to call it that, happens in the `create_destructor_table_impl` function.
It uses a clever *index sequence* to properly index into the variadic type pack.
There are two key features of this function that drives the whole dispatch machinery:

1. An array of indexed lambda functions, where
2. each function casts the storage to the correct type


Notice the last ellipsis at the end of the lambda funtion. The `Indices` pack
is unexpanded inside the lambda, and is more crucially used as a template
argument for `TypeAt`. The effect of the ellipsis applied to the lambda is that
the resulting pack expansion gives us one lambda for each constituent of the
`Indices` pack. In each lambda, the `Indices` is thus expanded to yield the index of the
type for which the *currently expanded* lambda should apply.


Now that we have type-safe destruction, we can safely instantiate, and destroy
a variant. This on its own isn't too interesting. Let's see how we can get
around to actually *using* the data, shall we?

#### Value and Move Semantics

Variants should be possible to both copy (if its held types permits), and move.
The implementation herein uses many of the tricks that apply to generating
destructor tables and visitation dispatch tables as can be seen both above and below.

For brevity's sake, take a look at the copy and move constructors and
assignment operators in the [code repo](https://github.com/thaugdahl/variant-v2/blob/main/main.cpp).


### Utilities

Given an initialized variant, you may want to know whether the variant holds
a specific type, or the index of the type in the variant's template pack. If
your variant is moved-from, the variant may also be *valueless*. Observe the
utilities below:

```cpp
const bool is_valueless() const {
    return active_index == sizeof...(Types);
}

std::size_t index() const {
    return active_index;
}

template <class TypeT>
bool holds_alternative() const {
    return active_index == TypeIndex<TypeT, Types...>::value;
}
```

These utilities may be used for the free function formulation suggested
in the next section.



### Visitation

Cppreference has the following notes for `std::visit`

> Let n be (1 * ... * `std::variant_size_v<std::remove_reference_t<VariantBases>>`), implementations usually generate a table equivalent to an (possibly multidimensional) array of n function pointers for every specialization of std::visit, which is similar to the implementation of virtual functions.
>
> Implementations may also generate a switch statement with n branches for std::visit (e.g., the MSVC STL implementation uses a switch statement when n is not greater than 256).
>
> On typical implementations, the time complexity of the invocation of v can be considered equal to that of access to an element in an (possibly multidimensional) array or execution of a switch statement.

Here is the crux of our manual implementation, we are not implementing core
compiler support for a variant, we are merely implementing a faximile of the
real deal. The compiler may do a myriad transformations to the original
visitation to the point where it boils down to a simple switch statement. We
will mimic this behavior using a compile-time generated dispatch table.

#### Dispatch Table

If you want to be completely cv- and ref-qualified for your visitation
dispatch, you're going to need multiple dispatch tables. Here are the needed
cases:

1. Const lvalue (`const A &`)
2. Const rvalue (`const A &&`)
3. Non-const lvalue (`A &`)
4. Non-const rvalue (`A &&`)

In each case you should carefully consider how visitor functors (the callable
visitor) and the variant's value are forwarded.

The C++ standard added a member function for visitation to `std::variant` in
C++26. Prior to this addition, visitation has been done with the free function
`std::visit`. This implementation opts for a member function. If you opt for a
free function, the process is mostly the same.

We will consider *one* of the cv- and ref-qualified visitations.

```cpp
template <class Visitor>
static constexpr auto make_visitor_dispatch_table_rvalue() {
    return make_visitor_dispatch_table_rvalue_impl<Visitor>(
        MakeIndexSequence<sizeof...(Types)>{});
}

template <class Visitor, std::size_t... Indices>
static constexpr auto make_visitor_dispatch_table_rvalue_impl(IndexSequence<Indices...>) {
    using RetType =
    typename FunctorReturnType<Visitor,
    typename TypeAt<0, Types...>::type>::type;

    return std::array<RetType (*)(char *, Visitor &&), sizeof...(Indices)>{
        [](char *data, Visitor &&v) {
            using ActiveType = typename TypeAt<Indices, Types...>::type;
            return std::forward<Visitor>(v)(std::move(*reinterpret_cast<ActiveType *>(data)));
        }...};
}

template <class Visitor>
decltype(auto) visit(Visitor &&v) && {

    if ( is_valueless() ) {
        throw std::runtime_error{"Invalid variant access. Valueless."};
    }

    static constexpr auto dispatch_table_rvalue =
        make_visitor_dispatch_table_rvalue<Visitor>();
    return dispatch_table_rvalue[active_index](storage,
                                               std::forward<Visitor>(v));
}
```

Now we're getting into some deeper uses of template metaprogramming. Let's go
for a walk through the code.

**`visit()`**

The last function, `visit` is ref-qualified for an rvalue-instance of
`variant`. This means that at the use-site of the visitation, the variant is
either expiring as an `xvalue`, or it is materialized as a pure rvalue (through
a copy-elided return or an otherwise immediately initialized variant). An example
of a case where this overload is selected can be seen here:

```cpp
Variant<A, B> get_variant() {
    return {A{}}; // Copy-list initialization. The returned variant holds an A.
}

void use_variant() {
    // get_variant() yields a prvalue
    get_variant().visit([](auto x) { /* Use variant value */ }
}
```

**make\_visitor\_dispatch\_table\_rvalue()**

This function instantiates an index sequence for parameter-pack indexing. It
immediately delegates actual construction  of the dispatch table to an
implementation function.

**make\_visitor\_dispatch\_table\_rvalue\_impl()**

If we want our visitation to be able to return a value, we should figure out
the return type of the visitor functor. No matter the overload set, the return type
must be consistent over all the possible ways the visitor functor can be invoked over
the variant. In other words, if one visitor returns `int`, then all visitors in the same
`visit` call must also return `int`. In this case, we use `decltype` to figure out
the return type of the visitor invoked on the first type of the `Variant` pack, and assume
the rest of the invocations also return the same type. The `FunctorReturnType` definition can be seen here:

```cpp
template <class Functor, class... Params>
struct FunctorReturnType {
    using type = decltype(declval<Functor>()(declval<Params>()...));
};
```

`declval` is a compile-time only function whose return type is a reference to
an instance of the passed template parameter. The first parameter is the
functor itself, so we invoke the functor on the parameters using the same
`declval` trick. The parameter pack expansion will more or less yield something
like `decltype(f(a,b,c,d,...))`, which is going to yield the return type of the
function itself.

Now, using the same method as was used in the destructor table, we create a
table of lambdas that correctly cast the type of the held value, and invokes
the visitor on a **moved** (or rvalue-cast) value. This should coincide
correctly with the move-semantics of the use-site. If you recall, we are
invoking `visit` on an `xvalue` or `prvalue` -- the held value is likely going
to be destroyed anyway.

If you want your variant to cover all const and ref-qualifiers, you need to set up
the visitation infrastructure three more times over. There are probably more succinct
ways of doing this, such as a compile-time overload selection in the `visit` function with
a *deducing this*. In this case however, we aren't concerning ourselves with the luxuries
of C++20 and beyond.




## Type and Metaprogramming Utilities

The following metaprogramming utilities are used throughout this project. They
are hand-rolled, and there exist STL counterparts. I recommend using the STL counterparts.
The hand-rolled versions are here for your reading (dis-)pleasure

### Index Sequence

We want to be able to hold a sequence of numbers for use with template argument
matching. `IndexSequence` is a simple templated struct which can be specialized for
any order and count of numbers.

```cpp
template <std::size_t... Indices>
struct IndexSequence {};

template <std::size_t I1, std::size_t... Rest>
struct IndexSequenceImpl {
    using type = typename IndexSequenceImpl<I1 - 1, I1 - 1, Rest...>::type;
};

template <std::size_t... Rest>
struct IndexSequenceImpl<0, Rest...> {
    using type = IndexSequence<Rest...>;
};

template <std::size_t I>
using MakeIndexSequence = typename IndexSequenceImpl<I>::type;
```

The machinery is quite simple.

1. Recursive case: The entire pack of `size_t` is split into a head and a tail.
   We reproduce the tail in its entirety, and add two singly decremented heads.
2. Base case: When the head of the parameter pack is `0`, we terminate the
   recursion, and yield only the tail.


Here is a simple illustration of how it works:

- *Step 1*: `5`
- *Step 2*: `4 4 5`
- *Step 3*: `3 3 4 5`
- *Step 4*: `2 2 3 4 5`
- *Step 5*: `1 1 2 3 4 5`
- *Step 6*: `0 0 1 2 3 4 5`
- *Step 7*: `0 1 2 3 4 5`

The reproduction of the head twice-over allows us to discard the leading `0` in
the base case, and be left with a neatly organized sequence. The base case also
forwards the final tail pack to `IndexSequence` and yields the final wrapped
sequence.

`IndexSequence` and its parameter pack is used multiple times over in the implementation
of a `Variant`. It allows for proper indexing of the variant's parameter pack.

### Type Index

Given a parameter pack of types, we may want to know at which position a
specific type occurs. This is achieved by the `TypeIndex` utility.

```cpp
template <class Desired, class... Types>
struct TypeIndex;

template <class Desired, class... Types>
struct TypeIndex<Desired, Desired, Types...> {
    static constexpr std::size_t value = 0;
};

template <class Desired, class First, class... Types>
struct TypeIndex<Desired, First, Types...> {
    static constexpr std::size_t value = 1 + TypeIndex<Desired, Types...>::value;
};

template <class Desired>
struct TypeIndex<Desired> {
    static_assert(!is_same<Desired, Desired>::value, "Can't find type in types pack");
};
```

If you inspect the implementation, you will see that we're stripping off the
types in the variadic pack one by one until the leading parameter and the
subsequent parameter match. At this point we've incrementally built up the
index until the base-case is encountered. As a result `TypeIndex<...>::value`
will yield the correct index.

### Type Pack Subscripting

In absence of built-in parameter pack indexing support from C++26,
we're going to have to resort to this nifty utility.

In this utility we're decrementing the `Index` parameter and stripping types
one-by-one off the parameter pack until the `Index` is `0`.

```cpp
template <std::size_t Index, class... Types>
struct TypeAt;

template <class First, class... Types>
struct TypeAt<0, First, Types...> {
    using type = First;
};

template <std::size_t Index, class First, class... Types>
struct TypeAt<Index, First, Types...> {
    using type = typename TypeAt<Index-1, Types...>::type;
};
```

## What's missing

This variant implementation does not give strong exception guarantees. I may at
some point make a follow-up article covering strong exception guarantees and how
they may be added to this `Variant` implementation.

The hand-rolled metaprogramming utilities could also benefit from stronger
SFINAE-handling. They may yield some painfully esoteric error messages if used
incorrectly.

## Key Takeaways

In this post, we've covered a feature-rich `Variant` implementation that tries
its best to mimic the STL `variant`. We've covered the reason for the purpose of
the variant type, and how it may be implemented in an efficient manner.

I want to pay my respects to Klaus Iglberger for his fantastic coverage of the
Visitor design pattern at CppCon 2022. It is likely the video that sparked my
interest for `std::variant` back in 2022. I am happy to finally have gotten around
to my medium to large-scale implementation. I enthusiastically encourage people
to watch [this video](https://www.youtube.com/watch?v=PEcy1vYHb8A).

On a more critical note, if you ever feel like reinventing the wheel, avoid the
Ixian hubris of putting yourself above the STL. Being condemned to the Ixian
wheel because of critical issues in production is rarely worth it [^1].

## Final Code

By now, our variant can hold any type, and invoke the visitor in as few
indirections as we can get without digging into the compiler's internals.
In real-world projects, use `std::variant` instead of our bastardized variant,
as you will most certainly get an implementation that is battle-tested and has
compiler support.

The complete repository can be found on [GitHub](https://github.com/thaugdahl/variant-v2).

[^1]: Ixion—condemned in Greek myth to spin on an eternal wheel for his transgressions—serves as a warning against overreaching
