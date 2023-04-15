---
layout: post
title: Functional programming is infectious
summary: You learn about clever purely-functional abstractions and you think, neat, but
    not really relevant to everyday programming; then the next thing you know,
    you're implementing a fairly boring bit of business logic and you think, dammit,
    what I really need here are profunctor optics. 
tags: 
    - java
    - functional programming
    - barefunc
---

You learn about clever purely-functional abstractions and you think, neat, but
not really relevant to everyday programming; then the next thing you know,
you're implementing a fairly boring bit of business logic and you think, dammit,
what I really need here are profunctor optics. 

Mainstream programming languages now generally include some support for
functional idioms: first-class functions, map, maybe fold. The idea seems to be
to introduce functional features to be used when they are convenient, but to
avoid the more complicated functional abstractions. This usually turns out to
mean writing functional code mostly at a small scale. <!-- more --> For
instance, this is nice to see in a Java code base:

```java
var revenue = orders.stream()
    .mapToInt(Order::getPrice)
    .sum()
```

but I'd be a bit more dubious about something like:

```java
var discountedOrders = orders.stream()
    .filter(order -> {
        var discount = discountRepository.find(order.getDiscount());
        return discountValidator.discountIsValid(discount, order);
    }).map(order -> {
        var discount = discountRepository.find(order.getDiscount());
        return order.toBuilder()
            .setPrice(discount.calculateDiscountedPrice())
            .build();
    }).collect(Collectors.toList());
```

When either the pipeline or the elements get more complicated, functional
programming constructs in Java can become uwieldy. Clearly there are ways to
improve the code above (leaving aside the fact that it's kind of a weird example
because I thought it up in two minutes), but it might well be the clearer to
refactor to a non-functional style

```java
for(Order order: orders) {
    var discount = discountRepository.find(order.getDiscount());
    
    if (!discountValidator.discountIsValid(discount, order)) {
        continue;
    }

    order.setPrice(discount.calculateDiscountedPrice());
}
```

But if you initially wrote a nice short functional version, to which you
gradually add complexity, it's not necessarily obvious at what point to refactor
to an imperative style, and by the time it's obvious it can be quite a big
change (especially if you go the whole way from immutability to mutability, as
in the example above, although to be fair you probably shouldn't do that).

The thing is, functional programming is infectious, in the sense that once you
begin to write in a functional style, it's generally convenient to continue
doing so, and that's when the more complicated functional abstractions become
useful. Case in point: I have [struggled to understand traversable like many
others](https://blog.plover.com/prog/haskell/traversable.html); but when I
eventually figured it out (with the help of that linked blog post), I began to
see uses for it all over the place. I was recently writing something which used
a fairly straightforward result type (`Result<T, E>`, which can contain either
the value of a successful operation, or information about an error that
occurred), and, unsurprisingly, I had to do some bulk operations and ended up
with a list of these results; just the job for traversables, but unfortunately
there is no traverse in the Java standard library.

So my initial thought was to write a `traverse` function for lists and results,
(or, actually, `sequence`, which is the related fuction you need when you
already have a list of results; you would use `traverse` when you have a list
and a function that returns a result). But then it occurred to me that there's
already a Java idiom to specify how to put together a collection: `Collector`.
So I came up with a `Flatteners` class (I thought that was a more obvious name
than "sequencer" if you're not familiar with its traversable origin) that
provides `Collector`s for `Result`s, and allows you to write things like:

 ```java
 // Produces Ok([1, 2, 3])
Stream.of(Result.ok(1), Result.ok(2), Result.ok(3))
        .collect(Flatteners.result());

// Produces Error(2)
Stream.of(Result.ok(1), Result.error(2), Result.error(3))
        .collect(Flatteners.result());
 ```

This isn't actually an implementation of traversable. The key feature of
traversable is that it is structure-preserving. This is a sufficiently obvious
property with lists that it's easy not to notice - you would naturally expect
that sequencing a list would give you the result back in the same order as the
list. A full implementation of traversable would also work with trees or any
other data structure, and preserve their structure as well. `Collector`s work on
`Stream`s which have *at most* a linear structure (they might have no inherent
structure at all), so you can't use them to preserve a structure more
complicated than a list (although my `Flatteners` implementation does let you
specify a further collector so you can specify a `Set` or something else instead
of a `List`).

But I think the flattening collector is still reasonably useful, despite not
being a full traversable implementation. This did get me thinking about how far
one can venture into the world of functional abstraction in Java while still
staying reasonably idiomatic, so I started [a little
library](https://github.com/culturedsys/barefunc) where I'll put some Java
experiments along these lines.