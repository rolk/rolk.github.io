---
title: "Type-less iteration in C++"
layout: post
date: 2018-10-29 11:00 +0200
tags:
- c++
---

Say that you have declared a variable like this:

{% highlight c++ %}
auto& v = foo();
{% endhighlight %}

because the return-type of foo is obscured thanks to heavily use of
templates.

However, this type is "container-like" and you would very much like to
retrieve individual items from this container through an iterator. As
is convention, there is a typedef inside the type of v which defines
the type of such an iterator, but how do we get to this iterator, if
we never specified the type of v, but just glossed over it with auto?

Fear not, dear reader! Together with auto is a counterpart called
decltype, which can be used to get the type of a variable. Since v was
even a reference, we need some extra template magic to unpack it
further:

{% highlight c++ %}
typedef typename std::remove_reference<decltype(v)>::type v_type;
typedef typename v_type::iterator v_it;
{% endhighlight %}

Now we are free to use this newly-found type in our iterations:

{% highlight c++ %}
for(v_it it = v.begin(); it != v.end(); ++it) {
   // ...
}
{% endhighlight %}

But what if this v variable is "matrix-like" and have another level of
iterators inside each of the iterators again? Well, now that we are
already inside the rabbit hole, we can just continue to dig deeper:

{% highlight c++ %}
typedef typename v_it::value_type::iterator v_itit;
for(v_itit inner = (*it).begin(); inner != (*it).end(); ++inner) {
   // ...
}
{% endhighlight %}
