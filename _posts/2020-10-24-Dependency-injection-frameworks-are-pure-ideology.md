---
layout: post
title: Dependency injections frameworks are pure ideology
---

> Mingus believes now that it got too far away from jazz -- spontaneity -- since
> almost all of the music was written. He remembers one rehearsal at which Teddy
> had left several bars open for blowing and everyone jumped on him with 'Man,
> are you lazy? Write it out!'
>
> <footer>Diane Dorr-Dorynek, from 
> <a href="http://aln2.albumlinernotes.com/Mingus_Ah_Um.html">the liner notes to 
> <cite>Mingus Ah Um</cite></a></footer>

Dependency injection is a great idea. Supplying dependencies to a class, rather
than having the class create them themselves, is a practical illustration of
the benefits of the single-responsibility principle. Creating dependencies is a
different responsibility from using them, and the class that uses dependencies
probably doesn't (or shouldn't) have the information required to create the
dependencies; so you can isolate this responsibility by in a separate class (or
multiple classes, for different environments or for testing) by injecting the
dependencies. But dependency injection doesn't require a framework, and indeed
dependency injection frameworks are useless if they're not actively harmful;
their continued prevelance is the zombie-like persistence of a dead
ideology.<!-- more -->

Dependency injection frameworks developed as a lightweight alternative to
Enterprise Java Beans, which were the microservices of the late 90s - a
technology for creating applications by wiring up independent components. The
EJB specification included lots of parts (distributed objects, persistence), and
mandated, or at least strongly encouraged, applications to be built using all
these specific EJB technologies. Dependency injection frameworks were more
lightweight in the sense that they focussed just on this wiring up of
independent components. Indeed, because they were more lightweight they
made this kind of componentisation easier, and so more widespread.

The classic dependency injection framework was implemented by early versions of
Spring, where each component is defined as an independent Java class, with the
injections of one component into another coordinated by XML files. The idea is
that the connections between components are configuration, and so can be
externalised; new systems can be constructed without any code changes, just by
re-assembling the components. This turned out to be a bad idea. Connecting
components to one another is programming, and should be treated as such: it
needs to be tested, and benefits from compile-time type-checking and IDE
support. So coordinating dependency injection via externalised XML is now
discouraged; modern dependency injection frameworks express dependency
requirements with in-code metadata annotations.

The problem with these modern dependency injection frameworks is that they're
pointless. We already have a way to declare and provide dependencies in code.
The Spring developers already implicitly recognise this, as the current
recommended practice is to declare required dependencies simply by having them
as constructor parameters (rather than by annotating fields); I hope at some
point they'll notice that it's also possible to provide dependencies by passing
them to constructors. Wiring dependencies by writing constructor calls is better
than delegating the job to a dependency injection framework because it's more
explicit. *What* gets injected *where* is visible to developers and, perhaps
even more importantly, to tooling, so the presence of required dependencies can
be checked at compile time, and the specific implementation of particular
dependencies can be navigated through standard IDE functionality.

So, when I see people using dependency injection frameworks, I'm reminded of the
Charles Mingus anecdote quoted above -- I think, 'Are you lazy? Just write the
constructor calls.' But, actually, the continued existence of dependency
injection frameworks isn't due to individual developer laziness; it's the
lingering remnant of the componentized dream which reached its full realization
in the original Spring's externalised configuration. This is one form of a
longer-term industry dream, that of programs without programmers, which inspired
fourth-generation languages in the 70s, and which we see in today's low-code
platform. And perhaps one day this dream will come true, but it didn't come true
through dependency injection frameworks; and we don't need to keep using these
frameworks through loyalty to this half-remembered dream.

(Some people will say, if you don't use a dependency injection framwork, how
will you manage the generation of proxies that implement aspect-oriented
programming? The answer here is that [AOP is also bad, and you shouldn't use
it](/2022/01/08/Concern-is-your-concern/).)