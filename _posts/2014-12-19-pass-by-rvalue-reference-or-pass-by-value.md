---
layout: post
category: blog
tags: [c++, c++11, optimization]
date: 2014-12-19 18:00:00 -0500
---
{% include JB/setup %}

Passing by rvalue reference is typically an optimization for the case when data can be "stolen" from the parameter instead of copying from it.

{% highlight c++ %}
struct S
{
    void init(const SomeType& param); // copy from param
    void init(SomeType&& param);      // steal from param
};
{% endhighlight %}

People noticed that if `SomeType` is cheaply movable, then we don't have to write two overloads and can get away with a single pass-by-value:

{% highlight c++ %}
struct S
{
    void initByVal(SomeType param); // can now steal from param
};

SomeType t;
s.initByVal(std::move(t)); // move data to param, then can steal from it
{% endhighlight %}

Compared to the first version, this will result in one more invocation of SomeType's move constructor to construct `param`, but as it is cheap to move it does not matter (much).

It is often mentioned that the first approach would result in `2^N` overloads in case of `N` arguments, and passing by value magically solves this issue. However, passing by reference solves it just as well (see below). Also, passing by value [can easily lead to unspecified behavior](http://stackoverflow.com/questions/24814696/move-semantics-and-function-order-evaluation) and, in my opinion, this alone makes it worth considering alternative approaches.

<!-- more -->

First, note that similarly to _only passing by value_ we might equally well _only pass by rvalue reference_:

{% highlight c++ %}
struct S
{
    void initByRef(SomeType&& param);
};

SomeType t;
s.initByRef(std::move(t)); // this works, and even better (less moves)
s.initByRef(SomeType()); // this also works
{% endhighlight %}

The only case when it "does not work" is when the argument passed to `initByRef()` is an lvalue. Note, however, that in this case the pass-by-value approach will do no magic neither - it will copy-construct (not move!) `param` and this copy cannot be elided. The question of the day is: do you actually want such copies to be created implicitly? If the code above was `initByVal(t);`, did the author actually mean to copy the object or did they simply forget the `std::move()`?

Anyway, how to deal with non-movable lvalues? Apart from `std::move()`, there is a second simple way of turning an lvalue into an rvalue: build a temporary object. Actually, we are already doing this implicitly when passing by value, and of course nothing stops us from doing this explicitly now:

{% highlight c++ %}
template<typename T>
T copy_to_temp(const T& t) { return t; }

SomeType t; // I'll need t later, so cannot std::move(t)
s.initByRef(copy_to_temp(t)); // but can copy

s.initByVal(t); // note that here t is also copied, it just happens implicitly
{% endhighlight %}

 While `copy_to_temp` may not be a great name, I picked it just to show how exactly the same idea can be applied to move-only types (such as `unique_ptr<T>`). Obviously, we cannot `copy_to_temp` something that is not copyable, but then we can `move_to_temp` just as simply:

{% highlight c++ %}
template<typename T>
T move_to_temp(T& t) { return std::move(t); }

void foo(unique_ptr<SomeType>&& param);

unique_ptr<SomeType> p;
foo(move_to_temp(p)); // note: this does move, p will become empty
{% endhighlight %}

But wait a second - why moving to a temporary when we can simply turn it into rvalue `foo(std::move(p));`? Does not it do the same thing but just less efficiently? Well, it turns out there is a subtle difference: `foo(move_to_temp(p))` guarantees that after the operation `p` will be empty (because it was "moved from"). Herb Sutter called this "the killer argument" for passing by value vs passing by reference in the comments to Scott Meyers' [Should move-only types ever be passed by value?](http://scottmeyers.blogspot.ca/2014/07/should-move-only-types-ever-be-passed.html). Well, here is a way to provide the same guarantee when passing by reference *if this actually is a requirement*: `move_to_temp`.

On a side note, this is a very weak guarantee. All you get is that data was "moved from" the object that was passed to the function. The data might be copied somewhere, but it might as well be just ignored and destroyed when the temporary goes out of scope. My expectation is that in most cases `foo(std::move(p))` will be just as good as `foo(move_to_temp(p))` because *no one will notice the difference*.

To sum up, here is what _only passing by reference_ does compared to _only passing by value_:

1. Saves one move construction (modulo move elisions)
2. Avoids the mentioned [unspecified behavior situation](http://stackoverflow.com/questions/24814696/move-semantics-and-function-order-evaluation)
3. Makes copies as explicit as moves. When you pass an lvalue to the function accepting rvalue references only, you get a compilation error. Is it a bad thing? In my opinion it is actually good - it forces you to decide whether you mean to *move* or to *copy* and indicate it explicitly for better maintainability.
4. Turns adding a `const T&` overload into an *optimization for the case when you cannot move*.