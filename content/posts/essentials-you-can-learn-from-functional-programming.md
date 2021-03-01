+++
title = "Essentials you can learn from Functional Programming"
date = 2021-02-26
description = "Improve your code with functional programming principles"
[taxonomies]
authors = ["Pratik Chaudhary"]
+++

Learning Functional Programming (FP) paradigm has shown me a whole new exciting world and has had a net positive effect on the quality of code I write today, even in languages of different paradigms. So I wanted to share some of what I have learned from FP. They can be applicable to varying degrees depending on the programming languages—more so in high level languages (like JS, Java, C#, etc) and less so in low level languages (like C). This will be about general principles of functional programming so you will not be seeing highly abstract concepts such as [category theory or monads](https://en.wikipedia.org/wiki/Monad_(functional_programming)).

I have used typescript for all the examples except one because javascript is known by the majority, and I have used simple typescript annotation for clarity of types of data being passed around. Even if you don't know them, you should still be able to understand examples if you have written code before (which you should have anyway, otherwise this post might not make any sense to you 😛). I have also used rust as an example for a section because it has good support for the feature I wanted to demonstrate, unlike typescript. Again, you don't need to know rust to understand it.

## Referential transparency
I believe this is the most significant or probably even the defining property of functional programming. So what is this property that made functional programming so attractive?
> An expression in a program is said to show referential transparency if that expression can be swapped with another of equal value without changing the behavior of the program.

Consider a simple example. An expression `2+1` can be swapped with an equivalent value `3` and the program would be semantically equal, so the expression `2+1` can be said to show referential transparency.

Let me demonstrate this concept in the context of code. Consider two examples A and B.

```typescript
// A
function attribute(book: string, author: string): string {
  return book + " by " + author;
}
attribute("LOTR", "Tolkien");
```

```typescript
// B
let author = "Tolkien";
let someCounter = 0;
function attribute(book: string): string {
  someCounter++;
  return book + " by " + author;
}
attribute("LOTR");
```
Which of the two examples above do you think maintains referential transparency?

Spoiler Alert! It's A. Because `attribute` in A entirely depends on its two parameters and nothing else and produces no side effects. The expression `attribute("LOTR", "Tolkein")` could be replaced by its corresponding output which, if you work it out, is `"LOTR by Tolkien"`, and you would still get an equivalent program. This type of function, which maintains referential transparency, is also often called **pure function**.

But for B, it is not the same case, because `attribute` has a dependency on things other than its parameters and also produces a side effect (which in this case is a mutation of non-local state). Not only does an output of `attribute` become unpredictable now but also it affects other parts of code.

You might have already started to develop an intuition for why referential transparency is significant. Allow me to help cement your intuition even more.

### Easier to reason about and debug code
When you are dealing with a pure function you can forget the rest of the code and just focus solely on the body of the function, simply because you know the function only depends on its arguments and no other things in the code can affect it. This can reduce your cognitive load by large while you are reading a source code, as you have a much narrower scope to spend your attention on. Consequently, this also makes it easier to debug code, as narrower scope means you would find the cause of bugs sooner.

### Encourages good abstraction
A pure function is guaranteed to give you the same output for a particular input every time, so you are only concerned with an input and an output of a pure function. This encourages you to write proper abstraction, because now you have to think about how different functions should communicate without one function knowing the implementation of another. Sure, this is not a surefire way to design a good abstraction but thinking under the terms of it should give you a good headstart.

### Refactoring without fear
It is easier to change the implementation of a pure function as long as you can keep its interface constant, without fear of breaking other parts of code. That means you can write all your dirty logic inside a function initially provided that it does not do any side effects and produce a correct output and then can come back later to refactor the internal logic with confidence.

### High portability and testability
A pure function does not assume anything beyond inputs given to it, so it can be applied anywhere in code regardless of its surrounding context. This just makes functions incredibly easier to reuse in different parts of the code.

It also makes testing super simple and fast, because all you have to do to test a pure function is to give it some input values and assert its corresponding output. No mocking. No initialization of context. Just some pure assertion.


## Establish boundary between logic and effects
We have just learned the irrefutable benefits of referential transparency but unfortunately, you can not write any useful program without performing side effects (network calls, writing to database, etc.), which breaks referential transparency. Does this mean all those ramblings of writing pure function were for nothing? Of course not. Just because we have to introduce side effects does not mean they should pervade our entire codebase. In fact, such a codebase would be hard to test, error-prone, and highly brittle. The key to writing maintainable and robust code that still does useful things is to establishing a discipline in how pure units (logic) communicate with impure units (effects).

Even though this has been in the back of my mind for a long time, I never ventured out to put this idea out concretely in words or figures or any palpable form for that matter. And when I did try to do so like I am doing here, I felt it much harder to do so, partly because I never did have a concrete idea myself and partly because I did not have good terminologies for components involved in this idea. Then I tried looking on the internet what other people have to say about this to help cement my own understanding, and the most that clicked with what I was thinking along the line of was ["Boundaries" talk by Gary Bernhardt](https://www.destroyallsoftware.com/talks/boundaries). (If you have 30 minutes to spare, I highly recommend you watch it too.) To simplify, he talks about how a program can be architectured as *functional core* (logic) wrapped by *imperative shell* (effects).

To give a gist of how this concept can be leveraged, I have taken a function for an example out of one of the real codebases I worked on.  The function is inside a class so you can see all the `this` accesses, but I have only included the function here to reduce noise. I have also slightly modified it by removing things that are not relevant here. (For the record, I did not write this abomination. 🙃)

```ts
scanBox(scanCode: string, boxes: Box[]) {
  let boxExistence = 0;
  for (let b = 0; b < boxes.length; b++) {
    if (scanCode === boxes[b].code) {
      --boxExistence;
      let currentBox = {
        code: boxes[b].code,
        id: boxes[b].id
      }
      this.modalCtrl.dismiss(currentBox);
    } else {
      ++boxExistence;
      if (boxExistence == boxes.length) {
        this.beepService.unloadAlert('badAlert');
        this.alertFailedMessage('Scanned box did not exist.');
      }
    }
  }
}
```
*Sigh.* Pretty messy, eh? This function as it is now is pretty complicated. To give you a little context of what it is doing, it is checking for a `box` with a particular code in the `boxes` array and dismisses the modal with the box if found. As you can tell, the function is very hard to reason about. It mixes logic and effects and makes them tightly coupled; and to make matters worse, it uses a mutable counter variable—an extra thing to keep tabs on while you're figuring out what path the code follows in every iteration.

Let's try to make it better. Forget about that code and try to focus on what the main intent of the function is. It is to find a `box` by a given code in the `boxes` array, right? Everything else such as dismissing modal, beeping a sound, alerting a message, etc. is not important as far as the core logic goes. So let's just try to extract that.

```ts
findBox(scanCode: string, boxes: Box[]): Box | null {
  for (let box of boxes) {
    if (box.code === scanCode) return box;
  }
  return null;
}
```
Now let's use it in the actual thing.
```ts
scanBox(scanCode: string, boxes: Box[]) {
  const box = findBox(scanCode, boxes);
  if (box) {
    const currentBox = {
      code: box.code,
      id: box.id
    }
    this.modalCtrl.dismiss(currentBox);
  } else {
    this.beepService.unloadAlert('badAlert');
    this.alertFailedMessage('Scanned box did not exist.');
  }
}
```
So you see, the core logic that is `findBox` is completely isolated now. The effectful function that is `scanBox` is now only integrating the data and performing side effects. There remains only a single decision-making in `scanBox` so there are very few things that can go wrong with it now, also making it much simpler to read and reason about. However, `findBox` does have all the nitty-gritty details now so we do have to test it. But the thing about pure functions is they are much easier and cheaper to test.

I have shown you how we can achieve segregation between pure logic and impure effects and doing so makes the code much more readable and maintainable. But although the code I used for an example here is a real-world code and I could separate side effects and logic easily on it, not every code can be as easy and natural to be refactored like that. Extracting logic (decision making) to pure function is not always that simple. Sometimes, you can have impure logic dependent on full of conditional branching or completely on the side effects (say database query), in which case you will have no such easy path for separation. There exist ways but not without the possiblity of complicating code so much so that they might not even be worth it in the end. You might better be off leaving them impure and test them with integration tests for such cases.

But that's not to say there's no value in what we have just learned here. Even if you could not extract components as pure functions, extracting smaller components that perform side effects can still be a big payoff, as long as they don't share a mutable state and are not too tightly coupled. But don't overdo it, or you can have problems like [this](https://overreacted.io/the-wet-codebase/). There are always trade-offs, and there's no straight answer to finding the perfect balance amidst the trade-off. You get better at this as you accumulate experience. Frankly speaking, even I don't feel adequate in this. I am still learning too!

### Bonus refactoring
You remember `findBox` from above. The algorithm I wrote there is such a common pattern that most programming languages have such function baked into their standard library. For example in javascript, it's called `find`. So let's just use that now.

```ts
scanBox(scanCode: string, boxes: Box[]) {
  const box = boxes.find(box => box.code === scanCode);
  if (box) {
    const currentBox = {
      code: box.code,
      id: box.id
    }
    this.modalCtrl.dismiss(currentBox);
  } else {
    this.beepService.unloadAlert('badAlert');
    this.alertFailedMessage('Scanned box did not exist.');
  }
}
```
Now you don't even have to do a unit test on that logic as `find` is already a well-tested function. There are lots of other functions like that: `filter`, `every`, `some`, etc. Different languages may have different names for them but they work the same way. Try to leverage them as much as possible so long as they don't complicate the code.


## Steer clear of Shared Mutability
You probably heard before that global variables are bad. Yup, that's one example of shared mutability! If you know that they are bad and still insist on using them without giving it a second thought, you are probably underestimating how dangerous shared mutability can be.

> Shared mutable state is data referenced by two or more components and at least one of them is modifying the data.

There are different types of side-effects, such as talking to a database, logging, network calls, writing to file, etc. Out of all of them, the most contagious and the one you want to steer clear of most is shared mutability. Other side effects like network calls can at least be contained and controlled easily if you isolate them. Sure, it would still be an impure function but so long as you handle all the possible cases (success and failure), the other parts of the code would remain predictable. Shared mutation, on the other hand, makes data flow hidden, making it almost impossible to isolate any component it touches. Mutation is so contagious that if you pass mutable data down the stack of many function calls, it plagues all the calls. if you are not careful enough, you will have a hard time tracking where it is mutated and what value it is at a particular point, causing more mental overhead of following the code. Not only can this cause unexpected bugs but also will they be hard to debug. Such is the nature of this heinous shared mutability.

(To avoid any potential confusion, I am only talking about mutability that is shared here. If mutability is localized like with local variables, then it's not the problem I am talking about here.)

So how do we avoid them or at least try our best to? I thought about it and jotted down some common places where you can make a mistake of introducing mutability and common patterns employed to avoid them.

### Return from function instead of reassigning external variables
Reassigning non-local variables as a way for data flow is common among beginners, especially in OOP languages. They know that global variables are bad but for some reason, they somehow have the impression that class variables are not global, while, in fact,  when your class gets big enough, they are no better than global variables.

```ts
class Example {
  someData: Data[];

  getData() {
    someData = /* get data somehow */
  }

  updateData() {
    someData = /* new updated data */
  }

  doSomethingWithData() {
    /* someData gets used here *.
  }
}
```
I have had to work on a class like this—using class variables as a way for data flow—but with way more methods and class variables and nested method calls entangled with each other. It gets really hard to predict what the data contains at a particular point of code because it's difficult to know what changed the data before that point; the method that might have changed it is hidden beneath some nested calls. Trust me, working on this kind of code is not a pleasant experience.

The solution to this problem is simple. Instead of mutating class variables, write functions that take data as arguments and return the data. Just a simple data transformation pipeline. The code would then look something like this:

```ts
class Example {
  getData(): Data[] {
    return data/* get data somehow and return it */
  }

  updateData(data: Data[]): Data[] {
    /* make a new data with changed contents and return it */
    return updateData
  }

  doSomethingWithData(data: Data[]) {
    /* Notice how data is parametrized so you can exactly know
    where it passed from, making code predictable */
  }
}
```
Depending on languages, you don't really need `class` for this. I believe a simple module works better for this. Regardless, the key point here is that explicitly passing around data and returning from functions make the data flow much more predictable and clear.

### Don't mutate parameters
Mutating parameters in the function has the same problem as mutating global variables does, as it obfuscates the data flow in the code.

```ts
function updateData(someData: Data[]) {
  /* Mutate someData directly here */
}
```
This function changes the data passed to it in place. Other functions that hold the reference to the original data might not get the data they were expecting because now it has been changed magically. This problem is worse when concurrency is involved because the order of function calls can not be predicted, but even in synchronous single-threaded code, it can still be a problem because ["it's effectively threaded"](https://manishearth.github.io/blog/2015/05/17/the-problem-with-shared-mutability/#its-effectively-threaded).
> My intuition is that code far away from my code _might as well be_ in another thread, for all I can reason about what it will do to shared mutable state.    —[*u/mozilla_kmc*](https://www.reddit.com/r/rust/comments/2x0h17/whats_your_killer_rust_feature/cow3zod/)

The solution to this problem is again the same—return a new, updated data from a function without mutating arguments.

```ts
function updateData(someData: Data[]): Data[] {
  /* Make a new updated data and return */
  return updatedData;
}
```
### Use immutable data classes or records
The pattern of updating data without mutating the original object to avoid mistakes mentioned above have found to be so good an idea that mainstream languages have started to have immutable [data classes](https://kotlinlang.org/docs/data-classes.html) or [records](https://devblogs.microsoft.com/dotnet/c-9-0-on-the-record/) baked in. They are not only easier to work with updating than plain traditional classes but also guarantee safety from accidental mutability. So if the language you use supports the idea of immutable record in any form, definitely consider using it over plain classes for data.

### Limit ways to talk to shared mutable state
Even though we have talked till now about ways to avoid mutable state, sometimes you really need a shared mutable state, say, for performance-critical code. In that case, to not let that state go rampant unleashing its wickedness across all your code, you might want to isolate that state and define a limited set of behavior to interact with it. If there are multiple states, some combinations might not even make sense, so defining specified operations to modify or access them can prevent inconsistent states. Also, instead of referencing the state directly by other code, pass it as an argument to functions or classes so that they still remain easy to test in isolation.


## Total function
Total function is probably the easiest to get right, given a good type system, but can improve your resiliency of code by far.
> Total function is a function that has a valid output for every input value in its set of domain.
```ts
function identity(values: any): any[] {
  return values.map(x => x)
}
```
This function returns a valid output if an array is passed as an argument, but as soon as anything other than an array is passed, it blows up in runtime. The `any` type argument has a large domain—includes all the possible values in the language. But identity function only returns valid output for arrays. This makes `identity` not total. Contrast that with another function that is total.
```ts
function identityTotal(values: string[]): string[] {
  return values.map(x => x)
}
```
Why should we care about the totality of function? To minimize possible ways a program can crash in runtime. In fact, if all of the functions are total, the program can never crash in runtime, except by external factors. But in order to make writing total functions easier and natural, you need a type system, particularly that supports tagged union. This is why I strongly believe a good type system is crucial in writing robust and resilient software.

### Enter Tagged Union
Tagged Union (or Discriminant Union) lets us encode different possible types on a single type. It is similar to [enumerated type](https://en.wikipedia.org/wiki/Enumerated_type) as you know in most languages, but each type can carry extra information other than its own tag. Consider the difference in the following examples: enumerated type and tagged union respectively.
```rust
// enumerated type
enum Color { Red, Blue, Green }
```
```rust
// Tagged Union
enum Command {
  // Notice how Copy encodes extra information
  Copy { source: String, dest: String },
  Delete { source: String },
  Open {source: String },
}
```

Tagged union gives us a powerful way to model data in the real world because things in the real world also happen to have cases. Referencing the example above, a command can either be `copy`, `delete`, or `open`. A value can be something or nothing. An I/O operation can either be a success or a failure. Even the failure can be due to either of many cases. Notice the pattern here?

The cases add complexity to the code. Nesting cases can exponentially increase the total number of cases. And they are inevitable in a sufficiently complex program. Trying to cover all cases without guarantees that you did not miss any can be very difficult and error-prone. That's where tagged union has a real value to add. If you can encode all the possible cases in tagged union and with an exhaustive pattern matching it gets impossible to forget about handling any case because a compiler can verify at compile-time that you covered all the cases.

Because of this, most languages that support tagged union use it to model the value that can be possibly absent, instead of a billion-dollar mistake—null. This way the absence of value can be encoded in the types and can be verified to be handled in compile-time.

Also, the infamous unchecked runtime exception is one of the biggest offenders of breaking the totality of the function, because the function that can throw exception can crash in runtime. Sure, if exceptions are caught and handled properly, it can prevent crashes. But like nulls, they usually get around the type system, so it rests entirely on the programmer to do it right. Which, I have seen, never works well in practice. Be honest with yourself, think about how many times software you helped write has crashed with an exception thrown in your face because you forgot to catch it properly. Fortunately, tagged union can model errors intuitively without the disadvantages of exceptions. Strongly typed languages like Haskell, Rust, OCaml, etc. exactly do this. Here's an example of tagged union used to model errors in rust to better your intuition:

```rust
// This tagged union models the success and failure case for almost anything
enum Result<T, E> {
  Ok(T),
  Err(E)
}

// File::open returns a `Result` so you can't really use the value inside it
// without pattern matching all the cases i.e Ok and Err
// so you are forced to handle the error case.
let ff = File::open("hello.txt");
let f = match ff {
    Ok(file) => file,

    /* Opening file can fail due to many reasons.
       Enum defined for this error encodes all the possible cases 
       you might want to handle.

       pub enum ErrorKind {
           NotFound,
           PermissionDenied,
           ConnectionRefused,
           // there are more
       }
    */
    Err(error) => match error.kind() {
        ErrorKind::NotFound =>
        ErrorKind::PermissionDenied =>
        ErrorKind::ConnectionRefused =>
        // If you leave out any case here, compiler will refuse to compile 
        // because of exhaustive checking. Sure you can use a wildcard 
        // variable to catch all the cases if you want to but you
        // still have to be explicit about it.
    },
};
```
(This does not mean exceptions should never be used. I believe they are still the right tool to use for unrecoverable errors, but for recoverable errors which most errors are, tagged union way is an easy win over exceptions.)

Tagged union is an invaluable tool in helping us write total functions. Unfortunately, not all mainstream languages have this feature available, although they might have been catching up gradually. But the key point here is that you should strive to write more total functions than not with whatever features you can leverage in the language you write. Perhaps with error codes instead of exceptions. Mind you, they might not be as ergonomic as tagged union, however.


## First class function
First class function is a concept of treating function as a first class value. That means functions can be assigned to variables and passed around just like other values.
```ts
// a function is assigned to variable fnValue
const fnValue = (x: number, y: number) => x + y

// Notice the type of f
function hof(x: number, y: number, f: (number, number) => number) {
  return f(x, y) * 2
}

// passing fnValue, a function, as argument to another function, hof
hof(2, 3, fnValue)
```

This ability of passing around functions as values gives us a powerful tool for abstracting logic in a way you can not otherwise.
```ts
function dbOp1() {
  const t = transactioStart()
  t.updateSomething();
  t.updateAnotherThing();
  t.commit();
}

function dbOp2() {
  const t = transactionStart();
  t.deleteSomething();
  t.updateSomething();
  t.commit();
}
```
See how both functions have repeated logic of starting and committing a transaction. The only thing that changes is in between. This is not just about repeated logic though; starting and committing transactions manually can be error-prone because you may forget to write commit. So we need a way to abstract those transaction bits so that we don't have to worry about that every time we want to run some database queries transactionally. This is where first class function comes in.
```ts
function transaction(body: (Transaction) -> ()) {
  const t = transactionStart();
  body(t)
  t.commit();
}

function dbOp1() {
  transaction(t => {
    t.updateSomething();
    t.updateAnotherThing();
  });
}

function dbOp2() {
  transaction(t => {
    t.deleteSomething();
    t.updateSomething();
  });
}
```
(I intentionally left out error handling and rolling back of transaction to not stray away from the main point here. But they can be easily handled in the transaction function too.)

So yes, first class function is useful for building abstraction like this which can make your code cleaner and safer. There are still lots of cool patterns and things you can do with first class function, but I am not going to talk about them here. My goal was to introduce you to first class function if you were not familiar with it before, and that I think I have accomplished here. Be sure to look at commonly used functions that leverage this feature like `map`, `filter`, `find`, etc. Most languages have them in the standard library, sometimes under different names.


## Wrapping Up
All of these principles or concepts that I learned because I stumbled into functional programming, I believe, have indeed helped me write better code in my day-to-day programming, even if they are not strictly functional programming. But again everything has trade-offs, and it may vary depending on the language and the implementation. So just because I said something to do here, you might not want to directly go and apply it without assessing and weighing in your situation. Experiment in your own environment, play around, make your own judgement on whether what you are going to do is right in that environment.

I guess I have met my goal in sharing and familiarizing you with concepts that might have been foreign before. Hopefully, it has convinced you to at least keep them in mind, if not to try them, the next time you write code, and maybe even will have helped you improve your code, as a result.

And a secret (not-so-secret) hope that this all has convinced you to dip your feet in any of the functional programming languages like OCaml, F#, Haskell. 😉
