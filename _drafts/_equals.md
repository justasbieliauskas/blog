---
layout: post
title: "How I Would Re-design equals()"
date: 2017-06-10
place: Copenhagen, Denmark
tags: java oop
description: |
  ...
keywords:
  - IO java
  - read/write java
  - object-oriented input/output
  - cactoos
  - input/output cactoos
image: /images/2017/06/?
jb_picture:
  caption: xxx
---

I want to rant a bit about Java design, in particular about methods
[`Object.equals()`](https://docs.oracle.com/javase/7/docs/api/java/lang/Object.html#equals%28java.lang.Object%29)
and
[`Comparable.compareTo()`](https://docs.oracle.com/javase/7/docs/api/java/lang/Comparable.html#compareTo%28T%29).
I've been hating them for years, because, no matter how hard
I tried, the code inside them look ugly. Now I know what exactly
is wrong and how I would design this "object-to-object comparing" mechanism
_better_.

<!--more-->

{% jb_picture_body %}

Say, we have a <del>simple</del> primitive class `Weight`, objects of which
represent weight of something, in kilos:

{% highlight java %}
class Weight {
  private int kilos;
  Weight(int k) {
    this.k = kilos;
  }
}
{% endhighlight %}

Next, we want two objects of the same weight to be equal to each other:

{% highlight java %}
new Weight(15).equals(new Weight(15));
{% endhighlight %}

Here is how such a method may look:

{% highlight java %}
class Weight {
  private int kilos;
  Weight(int k) {
    this.k = kilos;
  }
  public boolean equals(Object obj) {
    if (!(obj instanceof Weight)) {
      return false;
    }
    Weight weight = Weight.class.cast(obj);
    return weight.kilos == this.kilos;
  }
}
{% endhighlight %}

The ugly part here is, first of all, the type casting with `instanceof`. The second problem
is that we touch the internals of the incoming object. This design makes
polymorphic behavior of the `Weight` impossible. We simply can't pass
anything else to the `equals()` method, besides an instance of the
class `Weight`. We can't turn it into an interface and introduce
multiple implementations of it:

{% highlight java %}
interface Weight {
  boolean equals(Object obj);
}
{% endhighlight %}

This code will stop to work:

{% highlight java %}
class DefaultWeight {
  // attribute and ctor skipped
  public boolean equals(Object obj) {
    if (!(obj instanceof Weight)) {
      return false;
    }
    Weight weight = Weight.class.cast(obj);
    return weight.kilos == this.kilos; // error here!
  }
}
{% endhighlight %}

The problem is that one object decides for the other whether they are
equal. This inevitably leads to a necessity to touch private attributes,
to do the actual comparison.

What is the solution?

This is what I'm offering. Any comparison, no matter what types we
are talking about, is about comparing two digital values. Either we
compare a weight with a weight, a text with a text, or a user with a user&mdash;our
CPUs can only compare numbers. Thus, we introduce a new interface
`Digitizable`:

{% highlight java %}
interface Digitizable {
  byte[] digits();
}
{% endhighlight %}

Next, we introduce a new object, which does the comparison of
two streams of bytes (I'm not sure the code is perfect, I tested it
[here](),
feel free to improve and contribute with a pull request):

{% highlight java %}
class Comparison<T extends Digitizable> {
  private T lt;
  private T rt;
  Comparison(T left, T right) {
    this.lt = left;
    this.rt = right;
  }
  int value() {
    final byte[] left = this.lt.digits();
    final byte[] right = this.rt.digits();
    int result = 0;
    int idx = 0;
    for (; idx < left.length; ++idx) {
      if (idx >= right.length) {
        result = -1;
        break;
      }
      if (left[idx] < right[idx]) {
        result = -1;
        break;
      }
      if (left[idx] > right[idx]) {
        result = 1;
        break;
      }
    }
    if (result < 1 && idx == left.length
      && idx < right.length) {
      result = 1;
    }
    return result;
  }
}
{% endhighlight %}

Now, we need `Weight` to implement `Digitizable`:

{% highlight java %}
class Weight implements Digitizable {
  private int kilos;
  Weight(int k) {
    this.k = kilos;
  }
  @Override
  public byte[] digits() {
    return ByteBuffer.allocate(4).putInt(this.kilos).array();
  }
}
{% endhighlight %}

Finally, this is how we compare them:

{% highlight java %}
int v = new Comparison<Weight>(
  new Weight(400),
  new Weight(500)
).value();
{% endhighlight %}

This `v` will either be `-1`, `0`, or `1`. In this particular case it will `1`,
because `500` is bigger than `400`.

No more violation of encapsulation, no more type casting, no more
ugly code inside that `equals()` and `compareTo()` methods.
