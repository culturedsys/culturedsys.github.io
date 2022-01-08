---
layout: post
title: If there's a concern, it's your concern
summary: Aspect-oriented programming attempts to handle "cross-cutting concerns"
  separately from business logic. The problem is that these cross-cutting 
  concerns are vanishingly rare.
---

Aspect-oriented programming is still a fairly popular programming tool. The idea
is to reduce the coupling, and increase the modularity, of programs by providing
"aspects" to handle "cross-cutting concerns" separately, rather than
intermingling these concerns with each other and with a program's specific
business logic. The problem with this approach is that cross-cutting concerns,
in the way required by aspect-oriented programming, are vanishingly rare. 
<!-- more -->

To see this, it's instructive to look at [one of the earliest papers on
AOP](https://www.cs.ubc.ca/~gregor/papers/kiczales-ECOOP1997-AOP.pdf). The
motivating example is an image processing program, where an operation on an
image is built up from a series of filters. In a (non-aspect oriented)
implementation, each filter could be a function which takes in an image and
returns the new, filtered image. However, this has the problem that it creates
an intermediate image for each filter, which uses more memory than necessary.
This could be avoided by fusing some of the filters. For example, if one filter
lightens each pixel, and another filter sets each pixel to black or white
depending on a cutoff, we don't need to create an intermediate image from the
operation of the first filter and pass it to the second, instead, we could apply
both pixel-based operations at once, and produce only one resulting image.

However, if we were to implement this fusion manually, we would need to manually
interleave the functionality of the filters, which is less re-usable and harder
to understand than a system built out of independent filters. The authors
propose, instead, an aspect-oriented system in which the program is expressed as
a graph of independent filters, but is then processed to fuse operations where
it is possible to do so. The cross-cutting concern of efficiency is thus handled
separately from the transformation which the program implements.

This is indeed a compelling example of the advantages of aspect-oriented
programming, but its worth looking into the specifics of this case that make it
work. The problem is that there is a clear why of expressing _what_ the program
does, but that clear expression does not provide the best (most efficent)
description of _how_ that should be done; aspect-oriented programming provides a
way to specify the what and the how separately. The "what", handled by the
conventional program, is also known as the functional requirements; the "how",
the cross-cutting concern handled by aspect-oriented programming, by contrast,
are called non-functional requirements. This specific case is more generally
true - aspect-oriented programming is useful for non-functional requirements,
but inappropriate for functional requirements.

That aspects should be used to handle non-functional requirements is implied by
the authors' description of aspect-oriented programming as a procedure to handle
cases which aren't well handled by functional decomposition. By this they
primarily mean, cases that aren't handled well by decomposition into
functions/procedures; but the point of functional decomposition is that a
program is decomposed along functional lines, that is, decomposing programs into
functions is effective because what the program does can be described by a
composition of what individual functions do. Functional decomposition is
fundamentally about functional requirements. But this connection between
functional decomposition and functional requirements makes me skeptical of the
authors' claim that:

> Aspects tend not to be units of the system’s functional decomposition, but
> rather to be properties that affect the performance or semantics of the
> components in systemic ways.

More specifically, I have a problem with the idea that aspects are an
appropriate way of modifying the semantics of a component.