---
layout: post
category: blog
tags: [c++, optimization]
date: 2014-11-21 15:00:00 -0500
---
{% include JB/setup %}

Following this interesting [StackOverflow question about RVO failure](http://stackoverflow.com/questions/25963685/why-does-visual-studio-not-perform-return-value-optimization-rvo-in-this-case/), I've experimented a little to see in which cases copies are elided by the compiler. The question is about RVO failing in case when NRVO is performed properly; it is hard to explain *why* this happens without knowing the internals of the MSVC compiler, but here are a few observations on *when* this happens.

<!-- more -->

The same behavior can be reproduced with simple classes instead of the rather complex `std::vector`:

{% highlight c++ %}
#include <iostream>

struct C
{
    C(const C& in) { std::cout << "C(&) <-- copy\n";}
    C(C&& in) {}

    C() {}
    ~C() {}
};

struct D
{
    C c;

    D(C _c) : c(std::move(_c)) { std::cout << "D(C)\n";}
};

D test_RVO()
{
    C c;
    return D(std::move(c)); 
}

D test_NRVO()
{
    C c;
    D d(std::move(c));
    return d;
}

int main()
{
    std::cout << "RVO\n";
    test_RVO(); // or auto val = test_RVO();
    
    std::cout << "\nNRVO\n";
    test_NRVO();
}
{% endhighlight %}

When built and run with MSVC 2013 with optimizations enabled, it produces the following output:

{% highlight bash %}
RVO
D(C)
C(&) <-- copy

NRVO
D(C)
{% endhighlight %}

[try online](http://rextester.com/YJL85582) (as long as their compiler is not upgraded to 2014+)

In `test_NRVO()`, `D` is not copied due to the NRVO optimization, but the equivalent RVO optimization fails for `test_RVO()`.  Note that in debug mode (`/Od`) the results might be different as some optimizations are disabled and not all copies are elided. This has been changed in MSVC 2014/15 where even in debug mode all copies seem to be elided (this RVO behavior is fixed too). Gcc and clang elide all copies including the non-optimized builds (there is `-fno-elide-constructors` option though).

### Taking a closer look

First, note that the same issue occurs when instead of `std::move` the object is passed directly to the D's constructor:

{% highlight c++ %}
D test_RVO()
{
    return D(C());
}
{% endhighlight %}

Outputs

{% highlight bash %}
D(C)
C(&) <-- copy
{% endhighlight %}

Next, adding traces to move constructors

{% highlight c++ %}
//...
    C(C&& in) { std::cout << "C(&&)\n"; }
//...
    D(D&& in) : c(in.c) { std::cout << "D(&&)\n"; }
//...
{% endhighlight %}

produces the following ouput for `D(C())`

{% highlight bash %}
RVO
C(&&)
D(C)
C(&) <-- copy
D(&&)
{% endhighlight %}

`C(&&)` is printed from the `D(C _c)` constructor as per `c(std::move(_c))`, which is then followed by the constructor's `D(C)` trace. Then, however, the `D` instance is being moved when returning from `test_RVO()`. The move in its turn calls `c(in.c)` and this invokes `C`'s copy constructor. This `C` copy could be turned into a move by doing `c(std::move(in.c))` in the `D(D&&)` constructor, but this is somewhat irrelevant: the question is why the whole `D`'s move is not elided by RVO in the first place. NRVO does elide it (there is no `D(&&)`):

{% highlight bash %}
NRVO
C(&&)
C(&&)
D(C)
{% endhighlight %}

### Constructor parameter type matters

It turns out that the reason of the RVO failure is linked to the properties of the type that is being passed in the constructor `D(C _c)`. For example, using a different type in the constructor magically "fixes" RVO:

{% highlight c++ %}
struct X
{
};

struct D
{
    //...
    D(X _x) : c() { std::cout << "D(X)\n"; }
};

D test_RVO()
{
    return D(X());
}
{% endhighlight %}

Outputs

{% highlight bash %}
RVO
D(X)
{% endhighlight %}

There is no `D(&&)` meaning no move is performed, and RVO works. How is `D(X _x)` different from `D(C _c)`? Let's add a few things to `X`:

{% highlight c++ %}
struct X
{
    X() {}
    ~X() {}

    X(X const&) {}
};
{% endhighlight %}

And now RVO suddenly fails:

{% highlight bash %}
RVO
D(X)
C(&) <-- copy
D(&&)
{% endhighlight %}

Wait. What?

It turns out that RVO is performed correctly with the `D(X)` constructor, but it starts failing when `(-a- and (-b- or -c-))` are uncommented below:

{% highlight c++ %}
struct X
{
    X() {}
    //~X() {}           -a-
    
    //X(X const&) {}    -b-
    //X(X&&) {}         -c-
};
{% endhighlight %}

In other words, **a constructor of type `D` with signature `D(T)` leads to RVO failure when used as above, when the type `T` has a non-default destructor and a non-default copy (or move) constructor**. Certainly, this applies to that SO example where `T` is `std::vector` implementing all of those methods.

A couple of final notes:

* This is not specific to values returned from functions. For example, the same applies to expressions like `D d = D(T());`, where MSVC <2014 does not elide the copy/move (under the same conditions).

* There is no RVO failure when the constructor is replaced with a static method returning by value:

{% highlight c++ %}
struct D
{
    //...
    static D D::construct(X _x)
    {
        return D();
    }
};

D test_RVO()
{
    return D::construct(X()); // <-- RVO is performed correctly

    // return D(X());         // <-- but not this way
}
{% endhighlight %}