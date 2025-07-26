+++
date = '2025-07-26T23:34:30+02:00'
draft = false
title = 'C++ Variant From Scratch'
+++

# C++ Variant From Scratch


> **NOTE**: As of 26. july, this post is still incomplete. I'm sure I'll get
around to it eventually. Don't expect any kind of mind-blowing writing prowess to shine through just yet.

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

*Note*: In retrospect, I should have used `std::byte` instead of `char`. The two are close to equivalent (`std::byte` is unsigned), but `std::byte` is more descriptive.

#### Initialization

Every constructor and assignment is templated, and receives a specific type. If we want to be able to associated a unique identifier
with each type of the variant, we have to ascertain its position in the top-level parameter pack.

The templated utility class below will help us do so. It iteratively pops one type off the front of a pack until it encounters the
desired type, at which point the value, or index, will be computed.

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

In addition, we are going to need the following utility as well:

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
the contained type in the variant. There is one piece of functionality here that
isn't clearly explained yet -- the call to `destroy()`. Let's get into it.

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

Notice the last ellipsis at the end of the lambda function. The pack expansion
will ensure that the `Indices` pack, which remains unexpanded inside the
`TypeAt` selection, is expaned over the entirety of the lambda definition. As
such, we are left with as many lambda functions as there are indices in the
`Indices` pack.


Now that we have type-safe destruction, we can safely instantiate, and destroy
a variant. This on its own isn't too interesting. Let's see how we can get
around to actually *using* the data, shall we?

#### Value and Move Semantics

Variants should be possible to both copy (if its held types permits), and move.
The implementation herein uses many of the tricks that apply to generating
destructor tables and visitation dispatch tables as can be seen both above and below.

For brevity's sake, take a look at the copy and move constructors and
assignment operators in the [code repo](https://github.com/thaugdahl/variant-v2/blob/main/main.cpp).


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


## What's missing

- Strong Exception Guarantees
- Proper SFINAE Handling




## Final Code

By now, our variant can hold any type, and invoke the visitor in as few
indirections as we can get without digging into the compiler's internals.
In real-world projects, use `std::variant` instead of our bastardized variant,
as you will most certainly get an implementation that is battle-tested and has
compiler support.

The complete repository can be found on [GitHub](https://github.com/thaugdahl/variant-v2).
