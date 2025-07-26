+++
date = '2025-07-26T23:34:30+02:00'
draft = true
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

#### Assignment



#### Destruction



### Visitation

Cppreference has the following notes for `std::visit`

> Let n be (1 * ... * `std::variant_size_v<std::remove_reference_t<VariantBases>>`), implementations usually generate a table equivalent to an (possibly multidimensional) array of n function pointers for every specialization of std::visit, which is similar to the implementation of virtual functions.
>
> Implementations may also generate a switch statement with n branches for std::visit (e.g., the MSVC STL implementation uses a switch statement when n is not greater than 256).
>
> On typical implementations, the time complexity of the invocation of v can be considered equal to that of access to an element in an (possibly multidimensional) array or execution of a switch statement.

Here is the crux of our manual implementation, we are not implementing core compiler support for a variant, we are merely implementing a faximile of the real deal. The compiler may do a myriad transformations to the original visitation to the point where it boils down to a simple switch statement. We will mimic this behavior using a compile-time generated dispatch table.

#### Dispatch Table

If you want to be completely cv- and ref-qualified for your visitation dispatch, you're going to need multiple dispatch tables. Here are the needed cases:

1. Const lvalue (`const A &`)
2. Const rvalue (`const A &&`)
3. Non-const lvalue (`A &`)
4. Non-const rvalue (`A &&`)

In each case you should carefully consider




## Final Code

The complete repository can be found on [GitHub](https://github.com/thaugdahl/variant-v2).

```cpp
template <class... Types>
class Variant {

    using Self = Variant<Types...>;

    static constexpr std::size_t MaxSize = std::max({sizeof(Types)...});
    static constexpr std::size_t MaxAlign = std::max({alignof(Types)...});

    alignas(MaxAlign) char storage[MaxSize];

    std::size_t active_index;

    void destroy() {
        static constexpr auto destructor_table = create_destructor_table();
        if ( is_valueless() ) return;
        destructor_table[active_index](storage);
        active_index = sizeof...(Types);
    }


public:

    ~Variant() { destroy(); }

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


private:

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


    static constexpr auto create_mover_dispatch_table() {
        return create_mover_dispatch_table_impl(MakeIndexSequence<sizeof...(Types)>{});
    }

    template <std::size_t... Indices>
    static constexpr auto create_mover_dispatch_table_impl(IndexSequence<Indices...>) {
        return std::array<void (*)(char *, char *), sizeof...(Indices)> {
            [] (char *other, char *own) {
                using ActiveType = typename TypeAt<Indices, Types...>::type;
                ActiveType &other_ref = *reinterpret_cast<ActiveType *>(other);

                // Placement new is necessary. Our storage will be in an uninitialized state.
                new (own) ActiveType{std::move(other_ref)};
            }...
        };
    }

    static constexpr auto create_copier_dispatch_table() {
        return create_copier_dispatch_table_impl(MakeIndexSequence<sizeof...(Types)>{});
    }

    template <std::size_t... Indices>
    static constexpr auto create_copier_dispatch_table_impl(IndexSequence<Indices...>) {
        return std::array<void (*)(const char *, char *), sizeof...(Indices)> {
            [] (const char *other, char *own) {
                using ActiveType = typename TypeAt<Indices, Types...>::type;
                const ActiveType &other_ref = *reinterpret_cast<const ActiveType *>(other);

                // Placement new is necessary. Our storage will be in an uninitialized state.
                new (own) ActiveType{other_ref};
            }...
        };
    }

    static constexpr auto mover_dispatch_table = create_mover_dispatch_table();
    static constexpr auto copier_dispatch_table = create_copier_dispatch_table();

private:


  template <class Visitor>
  static constexpr auto make_visitor_dispatch_table_lvalue() {
    return make_visitor_dispatch_table_lvalue_impl<Visitor>(
        MakeIndexSequence<sizeof...(Types)>{});
  }

  template <class Visitor, std::size_t... Indices>
  static constexpr auto make_visitor_dispatch_table_lvalue_impl(IndexSequence<Indices...>) {
    using RetType =
        typename FunctorReturnType<Visitor,
                                   typename TypeAt<0, Types...>::type>::type;

    return std::array<RetType (*)(char *, Visitor &&), sizeof...(Indices)>{
        [](char *data, Visitor &&v) {
          using ActiveType = typename TypeAt<Indices, Types...>::type;
          return std::forward<Visitor>(v)(*reinterpret_cast<ActiveType *>(data));
        }...};
  }

  template <class Visitor>
  static constexpr auto make_visitor_dispatch_table_const_lvalue() {
    return make_visitor_dispatch_table_const_lvalue_impl<Visitor>(
        MakeIndexSequence<sizeof...(Types)>{});
  }

  template <class Visitor, std::size_t... Indices>
  static constexpr auto
  make_visitor_dispatch_table_const_lvalue_impl(IndexSequence<Indices...>) {

    using RetType =
        typename FunctorReturnType<Visitor,
                                   typename TypeAt<0, Types...>::type>::type;

    return std::array<RetType (*)(const char *, Visitor &&), sizeof...(Indices)>{
        [](const char *data, Visitor &&v) {
          using ActiveType = typename TypeAt<Indices, Types...>::type;
          return std::forward<Visitor>(v)(*reinterpret_cast<const ActiveType *>(data));
        }...};
  }

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
  static constexpr auto make_visitor_dispatch_table_const_rvalue() {
    return make_visitor_dispatch_table_const_rvalue_impl<Visitor>(
        MakeIndexSequence<sizeof...(Types)>{});
  }

  template <class Visitor, std::size_t... Indices>
  static constexpr auto
  make_visitor_dispatch_table_const_rvalue_impl(IndexSequence<Indices...>) {

    using RetType =
        typename FunctorReturnType<Visitor,
                                   typename TypeAt<0, Types...>::type>::type;

    return std::array<RetType (*)(const char *, Visitor &&), sizeof...(Indices)>{
        [](const char *data, Visitor &&v) {
          using ActiveType = typename TypeAt<Indices, Types...>::type;
          return std::forward<Visitor>(v)(std::move(*reinterpret_cast<const ActiveType *>(data)));
        }...};
  }

public:

    template <class Visitor>
    decltype(auto)
    visit(Visitor &&v) & {

        if ( is_valueless() ) {
            throw std::runtime_error{"Invalid variant access. Valueless."};
        }

        static constexpr auto dispatch_table_lvalue =
            make_visitor_dispatch_table_lvalue<Visitor>();
        return dispatch_table_lvalue[active_index](storage,
                                                   std::forward<Visitor>(v));
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

    template <class Visitor>
    typename FunctorReturnType<Visitor, typename TypeAt<0, Types...>::type>::type
    visit(Visitor &&v) const & {

        if ( is_valueless() ) {
            throw std::runtime_error{"Invalid variant access. Valueless."};
        }

        static constexpr auto dispatch_table_const_lvalue =
            make_visitor_dispatch_table_const_lvalue<Visitor>();
        return dispatch_table_const_lvalue[active_index](
            storage, std::forward<Visitor>(v));
    }

    template <class Visitor>
    typename FunctorReturnType<Visitor, typename TypeAt<0, Types...>::type>::type
    visit(Visitor &&v) const && {

        if ( is_valueless() ) {
            throw std::runtime_error{"Invalid variant access. Valueless."};
        }

        static constexpr auto dispatch_table_const_rvalue =
            make_visitor_dispatch_table_const_rvalue<Visitor>();
        return dispatch_table_const_rvalue[active_index](
            storage, std::forward<Visitor>(v));
    }



    Variant(Variant<Types...> &&other) {
        mover_dispatch_table[other.active_index](other.storage, storage);
        active_index = other.active_index;
        other.destroy();
    }

    Self &operator=(Self &&other) {
        destroy();
        mover_dispatch_table[other.active_index](other.storage, storage);
        active_index = other.active_index;
        other.destroy();

        return *this;
    }

    Variant(const Self &other) {
        copier_dispatch_table[other.active_index](other.storage, storage);
        active_index = other.active_index;
    }

    Self &operator=(const Self &other) {
        destroy();
        copier_dispatch_table[other.active_index](other.storage, storage);
        active_index = other.active_index;
        return *this;
    }

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


};
```
