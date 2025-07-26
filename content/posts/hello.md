+++
date = '2025-07-26T22:23:15+02:00'
draft = true
title = 'Hello'
+++

# Introduction

Welcome to my humble technical blog.

## CPP Code Fence Test

```cpp
template <class Desired, class... Types>
struct TypeIndex;

template <class Desired, class First, class... Types>
struct TypeIndex<Desired, First, Types...> {
    static constexpr std::size_t index = 1 + TypeIndex<Desired, Types...>::index;
};


template <class Desired, class... Types>
struct TypeIndex<Desired, Desired, Types...> {
    static constexpr std::size_t index = 0;
};

```
