<pre class='metadata'>
Title: Deducing this
Status: D
ED: wg21.tartanllama.xyz/deducing-this
Shortname: dXXXX
Level: 0
Editor: Barry Revzin, barry.revzin@gmail.com
Editor: Simon Brand, simon@codeplay.com
Abstract: Put abstract here
Group: wg21
Audience: EWG
Markup Shorthands: markdown yes
Default Highlight: C++
Line Numbers: yes
</pre>

# Motivation

In C++03, member functions could have cv-qualifications, so it was possible to have scenarios where a particular class would want both a `const` and non-`const` overload of a particular member (Of course it was possible to also want `volatile` overloads, but those are less common). In these cases, both overloads do the same thing - the only difference is in the types accessed and used. This was handled by either simply duplicating the function, adjusting types and qualifications as necessary, or having one delegate to the other. An example of the latter can be found in Scott Meyers' "Effective C++", Item 3:

```
class TextBlock {
public:
    const char& operator[](std::size_t position) const {
        // ...
        return text[position];
    }

    char& operator[](std::size_t position) {
        return const_cast<char&>(
            static_cast<const TextBlock&>(this)[position]
            );
    }
    // ...
};
```

Neither the duplication or the delegation via const_cast are arguably great solutions, but they work.

In C++11, member functions acquired a new axis to specialize on: ref-qualifiers. Now, instead of potentially needing two overloads of a single member function, we might need four: &, const&, &&, or const&&. We have three approaches to deal with this: we implement the same member four times, we can have three of the overloads delegate to the fourth, or we can have all four delegate to a helper, private static member function. One example might be the overload set for optional<T>::value(). The way to implement it would be something like:

## Quadruplication

```
template <typename T>
class optional {
    // ...
    constexpr T& value() & {
        if (has_value()) {
            return this->m_value;
        }
        throw bad_optional_access();
    }

    constexpr const T& value() const& {
        if (has_value()) {
            return this->m_value;
        }
        throw bad_optional_access();
    }

    constexpr T&& value() && {
        if (has_value()) {
            return std::move(this->m_value);
        }
        throw bad_optional_access();
    }

    constexpr const T&& value() const&& {
        if (has_value()) {
            return std::move(this->m_value);
        }
        throw bad_optional_access();
    }
    // ...
};
```

## Delegate to 4th

```
template <typename T>
class optional {
    // ...
    constexpr T& value() & {
        return const_cast<T&>(
            static_cast<optional const&>(
                *this).value());
    }

    constexpr const T& value() const& {
        if (has_value()) {
            return this->m_value;
        }
        throw bad_optional_access();
    }

    constexpr T&& value() && {
        return const_cast<T&&>(
            static_cast<optional const&>(
                *this).value());
    }

    constexpr const T&& value() const&& {
        return static_cast<const T&&>(
            value());
    }
    // ...
};
```

## Delegate to helper

```
template <typename T>
class optional {
    // ...
    constexpr T& value() & {
        return value_impl(*this);
    }

    constexpr const T& value() const& {
        return value_impl(*this);
    }

    constexpr T&& value() && {
        return value_impl(std::move(*this));
    }

    constexpr const T&& value() const&& {
        return value_impl(std::move(*this));
    }

private:
    template <typename Opt>
    static decltype(auto)
    value_impl(Opt&& opt) {
        if (!opt.has_value()) {
            throw bad_optional_access();
        }
        return std::forward<Opt>(opt).m_value;
    }


    // ...
};
```

It's not like this is a complicated function. Far from. But more or less repeating the same code four times, or artificial delgation to avoid doing so, is the kind of thing that begs for a rewrite. Except we can't really. We _have_ to implement it this way. It seems like we should be able to abstract away the qualifiers. And we can... sort of. As a non-member function, we simply don't have this problem:

```
template <typename T>
class optional {
    // ...
    template <typename Opt>
    friend decltype(auto) value(Opt&& o) {
        if (o.has_value()) {
            return std::forward<Opt>(o).m_value;
        }
        throw bad_optional_access();
    }
    // ...
};
```

This is great - it's just one function, that handles all four cases for us. Except it's a non-member function, not a member function. Different semantics, different syntax, doesn't help.

There are many, many cases in code-bases where we need two or four overloads of the same member function for different `const`- or ref-qualifiers. More than that, there are likely many cases that a class should have four overloads of a particular member function, but doesn't simply due to laziness by the developer. We think that there are sufficiently many such cases that they merit a better solution than simply: write it, then write it again, then write it two more times.

# Proposal
    
This paper proposes a new way of declaring a member function that allows for deducing the type and value category of the instance parameter, while still being invokable as a member function.

We propose allowing the naming of the first parameter of a cv/ref-unqualified member function `this`, which shall be of reference type. The this parameter will be the explicit instance of class type, and can be deduced based on the qualification and value category of the class instance object on which the member function is invoked.

```
struct X {
    template <typename This>
    void foo(This&& this);

    template <typename This, typename T>
    void bar(This&& this, T&& );

    template <typename This>
    void quux(This& this);
};

void demo(X x, const X* px) {
    X{}.foo();    // invokes X::foo<X>
    x.bar(4);     // invokes X::bar<X&, int>
    px->bar(2.0); // invokes X::bar<const X&, double>

    X{}.quux();   // ill-formed
    x.quux();     // invokes X::quux<X>
    px->quux();   // invokes X::quux<const X>
}
```

The type of this member function would be qualified based on the deduced qualification of the instance parameter. That is, `decltype(&X::foo<X>)` is `void (X::*)() &&` and `decltype(&X::bar<const X&, int>)` is `void (X::*)(int&& ) const&`. Similarly, `decltype(&X::quux<X>)` is void `(X::*)() &`.

While the type of `this` is deduced, it will always be some qualified form of the class type in which the member function is declared, never a derived type:

```
struct B {
    template <typename This>
    void do_stuff(This&& this);
};

struct D : B { };

D d;
d.do_stuff();        // invokes B::do_stuff<B&>, not B::do_stuff<D&>
```

Within these member functions, the keyword `this` will be used as a reference, not as a pointer. While inconsistent with usage in normal member functions, it is more consistent with its declaration as a parameter of reference type, and the difference will be a signal to users that this is a different kind of member function. Hence, accessing members would be done via `this.` and not `this->`. Due to these different access rules, all member access must be qualified. There will no longer be an implicit `this`:

```
template <typename T>
class Z {
    T value;
public:
    template <typename Object>
    decltype(auto) get(Object&& this) {
        return value;                            // error
        return std::forward<Object>(this).value; // ok
    }
};
```

While the examples so far all have this as a parameter whose type is deduced, this proposal does not enforce that. We will also allow simply naming the class type as the parameter type, as long as it's a reference type We do not expect this form to be used, but we likewise do not see the reason to limit the rule.

```
struct A {
    int i;

    // C++11
    void foo() & { std::cout << i; }

    // this proposal: ok, if unlikely to be used
    void foo(A& this) { std::cout << this.i; }

    // error: this as a parameter must have reference type
    void foo(A* this) { std::cout << this->i; }
};
```

Since in many ways member functions act as if they accepted an instance of class type as their first parameter (for instance, in `INVOKE` and all the functional objects that rely on this), we believe this is a logical extension of the language rules to solve a common and growing source of frustration. This sort of code deduplication is, after all, precisely what templates are for.

Overload resolution between new-style and old-style member functions would compare the explicit this parameter of the new functions with the implicit this parameter of the old functions:

```
struct C {
    template <typename This>
    void foo(This& this); // #1
    void foo() const;     // #2
};

void demo(C* c, C const* d) {
    c->foo(); // calls #1, better match
    d->foo(); // calls #2, non-template preferred to template
}
```

As `this` cannot be used as a parameter name today, this proposal is purely a language extension. All current syntax remains valid.

# Alternative syntax

Rather than naming the first parameter `this`, we can also consider introducing a dummy template parameter where the qualifications normally reside. This syntax is also ill-formed today, and is purely a language extension:

```
template <typename T>
struct X {
    T value;

    // as proposed
    template <typename This>
    decltype(auto) foo(This&& this) {
        return std::forward<This>(this).value;
    }

    // alternative
    template <typename This>
    decltype(auto) foo() This&& {
        return std::forward<This>(*this).value;
    }
};
```

# Examples

This proposal can de-duplicate and de-quadruplicate a large amount of code. In each case, the single function is only slightly more complex than the initial two or four, which makes for a huge win. What follows are a few examples of how repeated code can be reduced.

The particular implementation of optional is Simon's, and can be viewed on [GitHub](https://github.com/TartanLlama/optional), and this example includes some functions that are proposed in P0798, with minor changes to better suit this format:

## C++17

```
class TextBlock {
public:

    const char& operator[](std::size_t position) const {
        // ...
        return text[position];
    }

    char& operator[](std::size_t position) {
        return const_cast<char&>(
            static_cast<const TextBlock&>(this)[position]
            );
    }
    // ...
};
```

## This proposal

```
class TextBlock {
public:
    template <typename This>
    auto& operator[](This& this, std::size_t position) {
        // ...
        return this.text[position];
    }
    // ...
};
```

## C++17

```
template <typename T>
class optional {
    // ...
    constexpr T* operator->() {
        return std::addressof(this->m_value);
    }

    constexpr const T* operator->() const {
        return std::addressof(this->m_value);
    }
    // ...
};
```

## This proposal

```
template <typename T>
class optional {
    // ...
    template <typename This>
    constexpr auto operator->(This& this) {
        return std::addressof(this.m_value);
    }
    // ...
};
```

## C++17

```
template <typename T>
class optional {
    // ...
    constexpr T& operator*() & {
        return this->m_value;
    }

    constexpr const T& operator*() const& {
        return this->m_value;
    }

    constexpr T&& operator*() && {
        return std::move(this->m_value);
    }

    constexpr const T&& operator*() const&& {
        return std::move(this->m_value);
    }

    constexpr T& value() & {
        if (has_value()) {
            return this->m_value;
        }
        throw bad_optional_access();
    }

    constexpr const T& value() const& {
        if (has_value()) {
            return this->m_value;
        }
        throw bad_optional_access();
    }

    constexpr T&& value() && {
        if (has_value()) {
            return std::move(this->m_value);
        }
        throw bad_optional_access();
    }

    constexpr const T&& value() const&& {
        if (has_value()) {
            return std::move(this->m_value);
        }
        throw bad_optional_access();
    }
    // ...
};
```

## This proposal

```
template <typename T>
class optional {
    // ...
    template <typename This>
    constexpr auto&& operator*(This&& this) {
        return std::forward<This>(this).m_value;
    }

    template <typename This>
    constexpr auto&& value(This&& this) {
        if (this.has_value()) {
            return std::forward<This>(this).m_value;
        }
        throw bad_optional_access();
    }
    // ...
};
```

## C++17

```
template <typename T>
class optional {
    // ...
    template <typename F>
    constexpr auto and_then(F&& f) & {
        using result = invoke_result_t<F, T&>;
        static_assert(is_optional<result>::value,
                      "F must return an optional");

        return has_value()
            ? invoke(std::forward<F>(f), **this)
            : nullopt;
    }

    template <typename F>
    constexpr auto and_then(F&& f) && {
        using result = invoke_result_t<F, T&&>;
        static_assert(is_optional<result>::value,
                      "F must return an optional");

        return has_value()
            ? invoke(std::forward<F>(f), std::move(**this))
            : nullopt;
    }

    template <typename F>
    constexpr auto and_then(F&& f) const& {
        using result = invoke_result_t<F, const T&>;
        static_assert(is_optional<result>::value,
                      "F must return an optional");

        return has_value()
            ? invoke(std::forward<F>(f), **this)
            : nullopt;
    }

    template <typename F>
    constexpr auto and_then(F&& f) const&& {
        using result = invoke_result_t<F, const T&&>;
        static_assert(is_optional<result>::value,
                      "F must return an optional");

        return has_value()
            ? invoke(std::forward<F>(f), std::move(**this))
            : nullopt;
    }
    // ...
};
```

## This proposal

```
template <typename T>
class optional {
    // ...
    template <typename This, typename F>
    constexpr auto and_then(This&& this, F&& f) & {
        using result = invoke_result_t<F, decltype((
                                                       std::forward<This>(this).m_value))>;
        static_assert(is_optional<result>::value,
                      "F must return an optional");

        return this.has_value()
            ? invoke(std::forward<F>(f),
                     std::forward<This>(this).m_value)
            : nullopt;
    }
    // ...
};
```

Keep in mind that there are a few more functions in P0798 that have this lead to this explosion of overloads, so the code difference and clarity is dramatic.

For those that dislike returning auto in these cases, it is very easy to write a metafunction that matches the appropriate qualifiers from a type. Certainly simpler than copying and pasting code and hoping that the minor changes were made correctly in every case.

# SFINAE-friendly callables

Another seemingly unrelated problem is that of writing these numerous overloads for function wrappers, as demonstrated in P0826. `std::not_fn` is specified to behave as if:

```
template <typename F>
class call_wrapper {
    F f;
public:
    // ...
    template <typename... Args>
    auto operator()(Args&&... ) &
        -> decltype(!declval<invoke_result_t<F&, Args...>>());

    template <typename... Args>
    auto operator()(Args&&... ) const&
        -> decltype(!declval<invoke_result_t<const F&, Args...>>());

    // ...
};
```

which has the surprise result of incorrectly propagating the cv-qualifiers of the call wrapper:

```
struct fun {
    template <typename... Args>
    void operator()(Args&&...) = delete;

    template <typename... Args>
    bool operator()(Args&&...) const { return true; }
};

std::not_fn(fun{})(); // ok? Returns false
```

`fun` shouldn't be invocable unless it's `const`, but the simple approach of quadruplicating the overloads led to a situation where we can fallback to the const overload of the call wrapper (the `&&`-qualified overload led to a substitution failure so we were able to fall back to the `const&&`-qualified one, which succeeded). Implementing this correctly, while still preserving SFINAE-friendliness, is decidedly non-trivial.

This proposal, however, offers a very simple solution to this problem: deduce `this`:

```
template <typename F>
class call_wrapper {
    F f;
public:
    // ...
    template <typename This, typename... Args>
    auto operator()(This&& this, Args&&... )
        -> decltype(!declval<invoke_result_t<match_qual_t<This, F>, Args...>>());
    // ...
};

std::not_fn(fun{})(); // error
```

Here, there is only one overload with everything deduced together, with This = fun. As this overload is not viable (because fun is not invocable unless it's const), and there is no other overload, overload resolution is complete without a viable candidate. As desired.
