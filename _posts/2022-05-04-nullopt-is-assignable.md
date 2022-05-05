---
layout: post
title: C++ | std::nullopt is assignable (Clickbait title)
---

Yesterday I've written code as such:

```cpp
class Task {
    std::optional<MyStruct> prev_run_value;
}
void Task::Task(){
    ...
    if (has_ever_run_before()){
        *prev_run_value = get_value_from_previous_run();
    }
    ...
}

void Task::run(){
    if (!prev_run_value){
        // Okay actually run for the very first time
    }else{
        // Only run if enough time elapsed since last run
    }
}
```
And today the first slack message I had to read was a coworker telling me he reverted my code from master :(

Can you spot the bug? It took me quite some time to figure it out. Hint: `!prev_run_value` always evaluates to true, so the task always runs for the "very first" time.

The problem lies in the following line:
```
*prev_run_value = get_value_from_previous_run();
```
Look close. The bug is 1 character long. Look again. The line doesn't assign to the optional `prev_run_value` but to the value encapsulated by `prev_run_value`. As `prev_run_value` was default constructed up to this point, the placeholder value `nullopt` is stored by the optional. But why is `nullopt` a value of type `std::nullopt_t` a value of type `MyStruct` assignable? (I dislike cliffhanger so here is the tldr: std::optional's deref operator *always* returns T& (or uninitialized/stack memory; std::optional stores its data as a union internally) and as c++ is all about speed, no sanity check before access is done.)

# libstdc++ implementation of std::optional
Reading through the [implementation](https://code.woboq.org/gcc/libstdc++-v3/include/std/optional.html#std::_Optional_payload) of std::optional in libstdc++, a couple of things emerge:

1. The `nullopt_t` type is rather trivial and easy to understand. It does not implement a generic assign operator:
```cpp
/// Tag type to disengage optional objects.
struct nullopt_t
{
  // Do not user-declare default constructor at all for
  // optional_value = {} syntax to work.
  // nullopt_t() = delete;
  // Used for constructing nullopt.
  enum class _Construct { _Token };
  // Must be constexpr for nullopt_t to be literal.
  explicit constexpr nullopt_t(_Construct) { }
};
/// Tag to disengage optional objects.
inline constexpr nullopt_t nullopt { nullopt_t::_Construct::_Token };
```

2. Diving deeper into std::optional, the class `_Optional_payload` comes into play, which holds the data. The data is laid out as follows.
```cpp
struct _Empty_byte { };
union {
    _Empty_byte _M_empty;    // Author note: Thats correct
    _Stored_type _M_payload; // sizeof(_Empty_byte) == 1 :)
};
bool _M_engaged = false;
```
[This SO-post](https://stackoverflow.com/a/52338325) explains well why there is a union with an empty payload. Basically std::optional<T> has to be default constructible even if T is not default constructible.

3. Cow-says tells us:
```text
   *--------------------------------------------------------*
   | Cpp was not designed to stop its users from doing      |
   | stupid things, as that would also stop them from doing |
   | clever things.                                         |
   *--------------------------------------------------------*
          o
           o   ^__^
            o  (oo)\_______
               (__)\       )\/\
                   ||----w |
                   ||     ||
```
As c++ is all about speed (and "clever" things) - even on the cost of correctness - accessing the internal data via `operator*` does not check whether the optional contains a value. It may return uninitialized/stack memory.
```cpp
// The _M_get operations have _M_engaged as a precondition.
constexpr _Tp&
_M_get() noexcept
{
  __glibcxx_assert(_M_is_engaged()); // AFAIK __glibcxx_assert is only compiled for libstdc++ tests
  return this->_M_payload._M_payload;
}
```

# Takeaways

1. Read the docs! It would have prevented to raise a stupid question like "But why is `nullopt` a value of type `std::nullopt_t` a value of type `MyStruct` assignable?", as `nullopt` is never returned and also can never be returned in a strongly typed language (functions have exactly one return type).

But as we all know: 1 Minute of reading the docs can safe you hours of debugging. So why bother ¯\_(ツ)_/¯.

This missconception also stemmed from my mental model of std::optional: It contains either a value or nothing (std::nullopt). That is wrong. Optional always contains (the memory of) a value, but it might not be initialized.

2. Stop using operators - they often do not include sanity checks. Instead use the proper functions. In this case `emplace`.
