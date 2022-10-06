---
layout: post
title: Implementing the IO monad in Java for questionable fun and less profit
---

Why would you use the IO monad in Java? Well, the IO monad allows you to
represent side-effecting actions as values, separating IO from logic which you
can express as pure functions thereby.... Yes, yes, yes, but why would you use
the IO monad *in Java*? In languages like Haskell, the IO monad is the only way
to perform IO, so you can be certain any function that does not reference the IO
monad is a pure function; even Scala, which doesn't prevent you from doing IO
outside of the IO monad, provides a number of useful tools for working with
monads. Java doesn't provide either of these advantages, and indeed I doubt it's
particular useful to *use* the IO monad in Java. But that doesn't mean it's not
interesting to implement the IO monad in Java. In surgery, there's an adage that
to master a procedure, you "see one, do one, teach one". Analogously, I've used
the IO monad when writing in Haskell and Scala, but I thought I might get a
deeper understanding by attempting to implement it; and I suppose by writing
this blog post explaining the implementation, I'm now completing the "teach one"
step (the reason for choosing to write the implementation in Java specfically
was that I was sketching out an approach to something in Java for work, when I
realised I would end up with half a buggy version of the IO monad, at which
point I scrapped the plan and went with something simpler; but it did get me
thinking).

<!-- more -->

## A simple implementation

IO is a monad, and as such has two main operations. The first, `pure`, takes a
value and creates an IO action that does nothing and produces the specified
value. The second operation, which is officially called `bind` but is often
called `flatMap`, takes an IO action, and creates a new IO action which executes
the first action and then calls a function which takes the value produced by the
first action and returns a new IO action. So a simple implementation of the IO
monad could be:

```java
public sealed interface Io<T> {
  static <U> Io<U> pure(U value) {
    return new Pure<>(value);
  }

  default <U> Io<U> flatMap(Function<T, Io<U>> f) {
    return new FlatMap<>(this, f);
  }
}

record Pure<T>(T value) implements Io<T> { }

record FlatMap<S, T>(Io<S> previous, Function<S, Io<T>> f) 
  implements Io<T> { }
```

Wait, that doesn't seem very useful; it doesn't do anything? That's a key
feature of monadic IO -- IO actions are represented as values, and so the
operations on IO actions don't perform any actions, instead they create new
values, representing new actions. So our IO implementation needs something else:
a way to actually perform IO. This has two parts; first a way to wrap up code
with side effects as an IO value:

```java
static <U> Io<U> suspend(Supplier<U> action) {
  return new Suspend<>(action);
}

record Suspend<T>(Supplier<U> action) 
  implements Io<T> { }
```

and second, the real meat: a method to actually execute the actions represented
by an IO value. The details depend on the type of the IO action. `pure`
represents a value (with no IO), and `suspend` represents some code that
performs IO and produces a value, so performing these actions is very
straightforward:

```java
default T run() {
  return switch(this) {
    case Pure<T> pure -> pure.value()
    case Suspend<T> suspend -> suspend.action().get();
    case FlatMap<?, T> flatMap -> ...
  }
}
```

`flatMap` is only slightly more complicated. `flatMap` takes the result of
performing an IO action, and applies a function to it to produce another action.
So, to run a `flatMap` action, you can run the first action, apply the function
to that result, and then run the resulting action, i.e.:

```java
case FlatMap<?, T> flatMap -> 
  flatMap.f().apply(flatMap.previous().run()).run()
```

Unfortunately, if you actually type this, Java will complain that "The method
`apply(capture#1-of ?)` in the type `Function<capture#1-of ?,Io<T>>` is not
applicable for the arguments (capture#2-of ?)". The problem here is accessing
multiple properties of a `FlatMap<?, T>`, with a wildcard generic. `flatMap.f()`
has the type `Function<?, Io<T>>`, and `flatMap.previous().run()` has the type
`?`; Java always treats `?`s as representing different types, even though in
this case they obviously represent the same type. However, Java will perform the
relevant type checking for the generic parameters of a method, so can you work
around the problem by defining a separate method:

```java
  case FlatMap<?, T> flatMap -> flattenFlatMap(flatMap).run()

  ...

  private static <S, T> Io<T> flattenFlatMap(FlatMap<S, T> flatMap) {
    return flatMap.f().apply(flatMap.previous().run());
  }
```

## Making it stack safe

Unfortunately, there's another problem with this implementation. Consider this
method:

```java
@Test
public void flatMapIsStackSafe() {
    Io<Integer> io = Io.pure(0);

    for(int i = 0; i < 100000; i++) {
        io = io.flatMap(v -> Io.pure(v + 1));
    }

    assertThat(io.run(), equalTo(100000));
}
```

Here, we build up a computation from a large number of simple steps, with each
step passing its result to the next through `flatMap`. When we call `run` on the
resulting `flatMap`, that calls run on the previous `flatMap`, which calls `run`
again, and so on, until the stack overflows. 

The traditional functional programming solution to this kind of problem is to
rewrite the recursive function so that it is *tail*-recursive, that is, it
directly returns the result of the recursive call (this is called being in "tail
position", i.e., at the end). The direct recursive call to `run` in `run` is
already in tail position, but at the moment, the `run` method makes another
recursive call to `run` (via `flattenFlatMap`), and then further processes this
result. This means that the stack frame of each function needs to be kept
around, so that the flow of control can roll back up the stack to complete the
last part of each call.

When I first tried to rewrite `run` in a tail recursive form, I was stumped. The
problem is that `run` recusively calls `run`, and then does something else; but
the whole point of `flatMap` is that it performs the previous action, and then
does something else. How do your write "do something then do something else", in
the form "do something"? I eventually found an answer in [Pierre-Yves Saumant's
*Functional Programming in
Java*](https://www.manning.com/books/functional-programming-in-java). The key
lies in something about the IO monad I haven't mentioned yet -- the monad laws.

A monad doesn't just have to implement `pure` and `flatMap`; the implementations
also have to maintain certain properties. The relevant property here is the
associative law. If you have a monad `m` and two functions `f` and `g`:

```
m.flatMap(f).flatMap(g) ≡ m.flatMap(x -> f.apply(x).flatMap(g))
```

So you can convert a chain of `flatMap`s to a single `flatMap` with another
`flatMap` inside it. If you flatten a chain of `flatMap`s down to a single
`flatMap`, then you know that `flatMap.previous()` is *not* itself a `flatMap`,
so calling `flatMap.previous().run()` will not cause a further level of
recursion. As this flattening only applies when a `flatMap` is itself
`flatMap`ped, in order to apply it, we need to inspect the `previous` value of
our `flatMap` when running it, i.e.:

```java
private static <S, T> Io<T> flattenFlatMap(FlatMap<S, T> flatMap) {
  var f = flatMap.f();

  if (flatMap.previous() instanceof FlatMap<?, S> previousFlatMap) {
    return flattenNestedFlatMap(previousFlatMap, f);
  }

  return f.apply(flatMap.previous().run());
}

private static <R, S, T> Io<T> flattenNestedFlatMap(FlatMap<R, S> flatMap, 
    Function<S, Io<T>> f) {
  return flatMap.previous()
      .flatMap(r -> flatMap.f().apply(r).flatMap(f));
}
```

## Making it actually stack safe

Unfortunately, if you run the test again, the stack will still overflow. When a
function is called in tail position, the stack frame of the current function
doesn't *need* to be kept around, but that doesn't mean it *won't* be kept
around. It takes some effort for the compiler to detect the tail call and make
the optimisation; compilers for functional languages, where recursive algorithms
are commonplace, usually implement this optimisation, but recursive alorithms
have historically been uncommon in Java, so the Java compiler doesn't implement
the optimisation (at least, the OpenJDK and derived compilers don't; I'm not
sure if the standard permits compilers to makes the optimisation). 

However, it's possible to explicitly express the tail-call optimisation in your
code. Rather than actually performing the tail call, you can return an object
representing the fact that you want to perform a tail call. And, as you will
presumably want your recursion to stop at some point, you also need to represent
the fact that you *don't* want to perform a tail call (and instead return a
specific value). So you have a type with two cases:

```java
public sealed interface TailCall<T> {
  static <T> TailCall<T> done(T value) {
    return new Done<>(value);
  }

  static <T> TailCall<T> call(Supplier<TailCall<T>> supplier) {
    return new Call<>(supplier);
  }
}

record Done<T>(T value) implements TailCall<T> {  }
record Call<T>(Supplier<TailCall<T>> supplier) implements TailCall<T> { }
```

If you return this `TailCall` object, the stack frame will be cleaned up, so
this will avoid stack overflow; then you can evaluate the `TailCall` object
iteratively, which will also avoiding stack overflow:

```java
default T eval() {
  var current = this;
  while (current instanceof Call<T> call) {
    current = call.supplier().get();
  }

  return ((Done<T>) current).value();
}
```

To use this in our IO implementation, we can replace every use of `run` which
might get called on a `flatMap`, with a version that returns a `TailCall`
instead of directly making a tail call:

```java
private TailCall<T> runToTailCall() {
  return switch(this) {
    Pure<T> pure -> TailCall.done(pure.value())
    Suspend<T> suspend -> TailCall.done(suspend.supplier().get())
    FlatMap<?, T> flatMap -> 
        TailCall.call(() -> flattenFlatMap(flatMap).runToTailCall())
  };
}
```

Note here how the recursive call in `runToTailCall` is now wrapped in
`TailCall.call`, so will be evaluated in a stack-safe way when the `TailCall` is
evaluated.

So, [here is an implementation of the IO monad in
Java](https://github.com/culturedsys/barefunc/blob/io/core/src/main/java/systems/cultured/barefunc/io/Io.java)
-- on a branch that will never be merged, because it's probably useless. But
maybe it's interesting.