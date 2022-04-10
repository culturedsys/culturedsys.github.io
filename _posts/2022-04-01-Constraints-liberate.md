---
layout: post
title: Constraints liberate, TypeScript edition
---

A little TypeScript puzzle I encountered recently:

```typescript
interface Valid {
  readonly isValid: true;
}

interface Invalid {
  readonly isValid: false;
  message: string;
}

type Validation = Valid | Invalid;

function report(validation: Validation) {
  if (!validation.isValid) {
    console.error("Invalid", validation.message);
  }

  console.info("Valid");
}
```

Looks fine right? And [the TypeScript playground has no problem with
it](https://www.typescriptlang.org/play?#code/JYOwLgpgTgZghgYwgAgGpwDbACbIN4BQyyUEc2A9iBgJ7LADO6W2AXMmFAK4QDcBAXwIFQkWIhQBJEADdMOfERJlK1Oo2Y528DAz5KAthAYM4AcwjsGnUGf5CCYGgAcUm7HDDAqyALxp5XAAfZGk5Fn4CGC4QBC8fcJwACgBKdndFYlIwLigQfHomQPZOHmQBe2Fo2Pj80ETsJKMTc0tkayhbNNDZQMzlHLyCjWLkHT0AGmRm0wtyyqiYuO980mcKKDAkhs8V9MDdqhT+4BhkJIBCHdqAOhGWY8JiYgQqBgoMCBvoKA2kgCIwoF-lNrisbjNWil+MQHC83h8vqAYBQAe5-tDBEA).
But in my local environment, it was complaining that the `message` property
didn't exist on `validation`, despite the fact that `validation` should be
narrowed to `Invalid` in the `if` branch. <!-- more --> 

Eventually, I figured out that the problem was due to the local environment not
having `strictNullChecks` set. And, if you make the effects of that explicit, by making the `isValid` fields optional:

```typescript
interface Valid {
  readonly isValid?: true;
}

interface Invalid {
  readonly isValid?: false;
  message: string;
}
```

you can [reproduce the
error](https://www.typescriptlang.org/play?#code/JYOwLgpgTgZghgYwgAgGpwDbACbIN4BQyyUEc2A9iBgJ7LADO6W2A-AFzJhQCuEA3AQC+BAqEixEKAJIgAbphz4iJMpWp1GzHB2TwMDASoC2EBgzgBzCJwbdQlwSIJgaABxTbscMMCrIAXjRFXAAfZFkFFkECGB4QBF9-KJwACgBKTi9lYlIwHigQfHomEM5uPmQhJ1E4hKSi0BTsVNNzKxtkOygHTIj5EJzVfMLirTK9TEMAGmQ2i2sqmtj4xL8i0jcKKDBU5p91rJCDqnSh4BhkVIBCfYaAOnGWM8JiYgQqBgoMCHvoKG2qQARJEQkDZnd1vd5h10oJiM53p9vr9QDAKMCvEC4cIgA)
in the TypeScript playground. The problem is, if `isValid` can be undefined in
`Valid`, then `!isValid` might be true for a `Valid` object, and so that
possibility can't be excluded in the `if` branch.

It's counterintuitive, though, that making the compiler *less* strict can make
your program stop compiling. I suppose this is an example of how constraints
liberate, liberties constrain. Making the compiler stricter means it knows more
about what your program is doing, so it can prove more code is correct.