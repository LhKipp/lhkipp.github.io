---
layout: post
title: C++ | Pass by const& + && vs pass by value for functions which need a copy of its argument
---

The [CppCoreGuidelines](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#f15-prefer-simple-and-conventional-ways-of-passing-information) recommend passing 'In & retain "copy"' parameters by const reference for structs (or other non cheap / non possible to copy types). If you have asserted the need to go faster, the guidelines recommend adding an overload which allows passing by rvalue reference (&&).

Lets consider a simple example:
```cpp
void append_or_update(std::vector<Object>& v, const Object& elem){
    if(auto it = ranges::find(c, elem); it != end(c)){
        *it = elem; // Copy assignment called
    }else{
        c.emplace_back(elem); // Copy constructor called
    }
}
```
Please note, that `elem` is always copied within the function.

The implementation above follows the CppCoreGuidelines. As the codebase is all about inserting and updating values in a vector, the function is used heavily. We have identified the need for performance, and are now allowed to optimize. According to the guidelines, we should add a rvalue-overload
```cpp
void insert_or_update(vector<Object>& v, Object&& elem){
    if(auto it = ranges::find(c, elem); it != end(c)){
        *it = std::move(elem); // Move assignment called
    }else{
        c.emplace_back(std::move(elem)); // Move constructor called
    }
}
```

Coding according to the guidelines will leave one with the runtime-optimal code. The caller can now choose to copy `elem` or pass it as an rvalue. However functions which take a copy of their argument have to be implemented twice. 

Another option (which comes at the cost of one additional (cheap) move operation) is writing the function once, where it accepts `elem` (the argument to copy) by value.
```cpp
// `elem` will be either constructed by a call to Object's copy constructor or move constructor.
void insert_or_update(vector<Object>& v, Object elem){
    if(auto it = ranges::find(c, elem); it != end(c)){
        *it = std::move(elem); // Move assignment called
    }else{
        c.push_back(std::move(elem)); // Move constructor called
    }
}
```
Going down this route is considered an "Exception", which should only be used for types which are move-only and cheap-to-move.

At least the codebases I have seen, do not suffer from performance problems, because of one additional (possibly cheap) move operation - but because of code-bloat and other suboptimal patterns. So here is my guideline:

If a function has to copy its argument, consider taking the argument by value, to give the caller the option to 
* either move-construct the argument
* or to copy-construct the argument


Please note: Whichever route you'll decide to go down - in both cases the caller needs to be aware, that he needs to call `std::move`. It wouldn't be c++, if there would be no gotcha.
