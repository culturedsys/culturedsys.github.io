---
layout: post
title: Extensible collection processing in Java with transducers
tags:
  - java
  - functional programming
  - barefunc
---

Streams were one of the big features of post-8 Java, and they remain a very
convenient tool for applying transformations to collections. One limitation,
though, is that they're not really extensible; you're largely stuck with the
stream operations provided by the standard library. In Java 8, for example,
there was no `takeWhile` operation. It is of course possible to write this
yourself, so you can do something like:

```java
StreamUtils.takeWhile(Stream.of(1, 2, 3)
    .map(x -> x + 1), 
  x -> x < 2);
```

The comparatively superficial problem with this is that custom stream operations
look very different from the stream methods, making them harder to read when
used as part of a stream pipeline The more fundamental problem is that they're
fairly cumbersome to write; generally, the implementation involves creating a
new `Stream` implementation which itself uses a new `Spliterator`
implementation, which wraps the `Spliterator` from the source `Stream` (for
instance, here is [the implementation of `takeWhile` from the *protonpack*
library](https://github.com/poetix/protonpack/blob/master/src/main/java/com/codepoetics/protonpack/TakeWhileSpliterator.java)).

Would it be possible to provide something with the ergonomics of streams, but
with more extensibility? I happened to run across the
[*transducist*](https://github.com/dphilipson/transducist) TypeScript library,
which suggests maybe a more extensible alternative to streams could be created
based on transducers. <!-- more -->

## If I wanted to go there, I wouldn't start from here

What are transducers? To explain that, it helps to start by talking about
*re*ducers. Reducers are functions that can be passed to a `reduce` function;
that is, they take an accumulator and an element of a collection, and return a
modified accumulator:

```java
interface Reducer<A, T> extends BiFunction<A, T, A> { }
```

Then you could use a Reducer in something like:

```java
var sum = Reduce.over(List.of(1, 2, 3))
  .reduce((a, t) -> a + t, 0);
```

Reducers can be used to implement a lot of operations on collections. In
particular, because the accumulator can itself be a collection, a reducer can
perform a transformation on the collection - for instance map:

```java
static <T, U> Reducer<List<U>, T> map(Function<T, U> f) {
  return (acc, t) -> {
    var result = new ArrayList<>(acc);
    result.add(f.apply(t));
    
    return result;
  };
}
```

or filter:

```java
static <T> Reducer<List<T>, T> filter(Predicate<T> p) {
  return (acc, t) -> {
    if (p.test(t)) {
      var result = new ArrayList<>(acc);
      result.add(t);

      return result;
    }

    return acc;
  };
}
```

There are a couple of issues with this (leaving aside the fact that it copies
the list on each iteration; we'll deal with that eventually): the resulting
collection type is hardcoded to be a `List`, and the code to produce that list
is duplicated between the reducers. We can solve the second problem by factoring
out the list creation code:

```java
static <T> List<T> addToList(List<T> list, T t) {
  var result = new ArrayList<>(list);
  result.add(t);
  
  return result;
}

static <T, U> Reducer<List<U>, T> map(Function<T, U> f) {
  return (acc, t) -> {
    return addToList(acc, f.apply(t));
  };
}

static <T> Reducer<List<T>, T> filter(Predicate<T> p) {
  return (acc, t) -> {
    if (p.test(t)) {
      return addToList(acc, t);
    }

    return acc;
  };
}
```

This also suggests a solution to the first problem - rather than hardcoding 
`addToList`, we make it a parameter:

```java
static <T, U, A> Reducer<A, T> map(Function<T, U> f, BiFunction<A, U, A> add) {
  return (acc, t) -> {
    return add.apply(acc, f.apply(t));
  };
}

static <T, A> Reducer<A, T> filter(Predicate<T> p, BiFunction<A, T, A> add) {
  return (acc, t) -> {
    if (p.test(t)) {
      return add.apply(acc, t);
    }

    return acc;
  };
}
```

The interesting thing here is that the parameter is a function taking some kind
of accumulator, and an element, and returning an updated accumulator: that is,
it is itself a `Reducer`. So we can apply further reductions without producing
an intermediate collection by supplying a downstream reducer. To filter and
then sum, we could do:

```java
var sumOfEvens = Reduce.over(List.of(1, 2, 3, 4))
  .reduce(Reducers.filter(x -> x % 2 == 0),
    (sum, x) -> sum + x);
```

## Composing reductions with transformers

What our implementation of map and filter now have in common is that they take a
`Reducer`, and produce another `Reducer`. Whe can formalise this as a
`Transformer` (note that even if this were an interface rather than an abstract
class, it wouldn't be a *functional* interface, because the abstract method is
generic):

```java
public abstract class Transformer<I, O> { 
  public abstract <A> Reducer<A, I> apply(Reducer<A, O> downstream);
}
```

Then we can re-write `map` and `filter` as methods that produce `Transformer`s,
like:

```java
static <T, U> Transformer<T, U> map(Function<T, U> f) {
  return new Transformer<T, U>() {
    @Override
    public <A> Reducer<A, T> apply(Reducer<A, U> downstream) {
      return (acc, t) -> downstream.apply(acc, f.apply(t));
    }
  };
}
```

Because these transformers have a uniform interface, we can also compose them
together, to build up a chain of reducing functions without (yet) specifying the
final reducer:

```java
public final <P> Transformer<I, P> compose(Transformer<O, P> other) {
  return new Transformer<I, P>() {
    @Override
    public <A> Reducer<A, I> apply(Reducer<A, P> downstream) {
      return Transformer.this.apply(other.apply(downstream));
    }      
  };
}
```

And this allows us to create a somewhat stream-like, fluent, interface for
reductions:

```java
var result = Reduce.over(List.of(1, 2, 3, 4, 5))
  .compose(Transformers.filter((Integer x) -> x % 2 == 0))
  .compose(Transformers.map((Integer x) -> x * 2))
  .reduce(Transformers::addToList, Collections.emptyList());
```

## Short-circuiting operations

To return to the initial motivating example, `takeWhile` can also be implemented
as a transformer, one which passes the value to the downstream reducer as long as
the condition holds, and once it doesn't hold, it just returns the existing
accumulator unchanged:

```java
public static <T> Supplier<Transformer<T, T>> takeWhile(Predicate<T> p) {
  return () -> new Transformer<T,T>() {
    private boolean conditionHolds = true;

    @Override
    public <A> Reducer<A, T> apply(Reducer<A, T> downstream) {
      return (acc, t) -> {
        conditionHolds = conditionHolds && p.test(t);
        if (!conditionHolds) {
          return acc;
        }  
        return downstream.apply(acc, t);
      };
    }      
  };
}
```

There are a couple of issues here. One is that the reducer is stateful (it needs
to know, not just whether the current value satisfied the predicate, but whether
past values did), so you want to avoid re-using the same instance for multiple
reductions. I handle that by having the `takeWhile` function return a
`Supplier`, which produces a fresh instance (and adding an overload to the
`compose` method which takes a supplier, maintaining a consistent interfance). 

The other problem is efficiency - the implementation above returns the
accumulator unchanged once the predicate doesn't hold, but it still gets called,
even when it isn't going to be doing anything. That is, it has no way to
indicate that processing of the source should finish. This can be accomplished
by changing the return value to indicate whether or not further processing is
required:

```java
sealed interface Result<T> { }
record Continue<T>(T acc) implements Result<T> { }
record Stop<T>() implements Result<T> { }

public static <T> Transformer<T, T> takeWhile(Predicate<T> p) {
  return new Transformer<T,T>() {
    @Override
    public <A> Reducer<A, T> apply(Reducer<A, T> downstream) {
      return (acc, t) -> {
        if (!p.test(t)) {
          return new Reducer.Stop<>();
        }  
        return downstream.apply(acc, t);
      };
    }      
  };
}
```

(this also removes the need for `takeWhile` to be stateful, but there are other
reducers, like `take`, which do need to be stateful).

A mirror image of this problem occurs for transformers which need to know when
the whole collection has been processed, for example, a replacement for
`Stream`'s `sorted` method. You can't be sure a collection is in order until
you've seen every element, which also means a transformer which is intended to
sort its input can't pass any elements to the downstream reducer until it knows
that the upstream has finished. We can deal with this by adding a `finish`
method to the `Reducer` interface (with a default implementation which does
nothing, and just returns the accumulater it is passed). Then we can implement a
`sorted` transformer like:

```java
public static <T> Transformer<T, T> sorted(Comparator<? super T> comparator) {
  return new Transformer<T, T>() {
    @Override
    public <A> Reducer<A, T> apply(Reducer<A, T> downstream) {
      return new Reducer<A, T>() {
        private final PriorityQueue<T> buffer = new PriorityQueue<>(comparator);

        public Reducer.Result<A> apply(A acc, T t) {
          buffer.add(t);
          return new Reducer.Continue<>(acc);
        }

        @Override
        public A finish(A acc) {
          while(!buffer.isEmpty()) {
            var r = downstream.apply(acc, buffer.remove());
            if (r instanceof Reducer.Continue<A> c) {
              acc = c.value();
            } else {
              break;
            }
          }

          return downstream.finish(acc);
        }
      };
    }
  };
}
```

The `apply` method here doesn't directly send its input downstream, or add it
to the accumulator. Instead, it adds it to a `PriorityQueue` (so that we can
later retrieve the elements in order); only when `finish` is called are the
elements from upstream passed to the downstream reducer, in sorted order.

## Collectors

Finally, to return to something I mentioned near the beginning of the article,
the inefficient `addToList` reducer:

```java
public static <T> Reducer.Result<List<T>> addToList(List<T> list, T t) {
  var result = new ArrayList<>(list);
  result.add(t);
  
  return new Reducer.Continue<>(result);
}
```

As a reducer, this takes a list and returns a new list with the element added to
it; to maintain that property while using a standard Java `ArrayList`, it copies
the list every time. It would be more efficient to modify the list in place, and
indeed the reducing function could do this, but that could be confusing and lead
to errors, as the expectation is that reducers don't modify the accumulator.
Streams deal with this by distinguishing reduction from collection, where a
`Collector` assembles items into a mutable result, and we could do the same. A
simple `Collector` is a combination of a `Consumer<T>` (accepting each element)
and a `Supplier<R>` (supplying the final result). So a `toList` collector could
be implemented as:

```java
class ToList<T> implements Supplier<List<T>>, Consumer<T> {
  private final List<T> result = new ArrayList<>();
  @Override
  public List<T> get() {
    return result;
  }

  @Override
  public void accept(T t) {
    result.add(t);
  }
} 
```

The problem is in using this with our transformers. We build up a transformer
chain which passes its result items to a `Reducer`; but the collector defined
above isn't a `Reducer`. However, with a little tinkering, we can start to see
it in a way which looks more like a reducer. The first thing to note is that we
could implement `Reducer`'s `apply` method in terms of accept:

```java
Reducer.Result<A> apply(A acc, T t) {
  accept(t);
  return Reducer.Continue<>(acc);
}
```

`accept` doesn't use the accumulator, so we don't care what `A` is; we don't
care what the value is, and we don't care what the type is, either. To implement
the `Reducer` interface, we do need to know what the type is, but luckily
there's a name for the type where we don't care what the value is: `Unit`. So a
collector could be a `Reducer<Unit, T>`, and we can define a `Collector`
interface that implements a `Reducer<Unit, T>`, like so:

```java
interface Collector<R, T> implements 
    Supplier<R>, Consumer<T>, Reducer<Unit, T> {
  default Reducer.Result<Unit> apply(Unit unit, T t) {
    accept(t);
    return Reducer.Continue<>(unit);
  }
}
```

and then our `ToList` collector above can also implement `Collector<List<T>,
T>`. Then all we need to do is implement a `collect` method, like:

```java
public <R> R collect(Supplier<Collector<R, U>> supplier) {
  var collector = supplier.get();

  reduce(collector, Unit.unit());

  return collector.get();
}
```

This covers quite a bit of the functionality of `Stream`s, in a fairly ergonomic
and extensible way. `Stream`s have further features, particularly allowing
parallel processing. These could potentially also be implemented in terms of
transducers, but that's an extension for another day.

An initial implementation of transducers, containing the code in this post and a
bit more, is available as part of [my Barely Functional library on
GitHub](https://github.com/culturedsys/barefunc).