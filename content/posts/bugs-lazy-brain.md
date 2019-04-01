---
title: "Bugs & Lazy Brain"
date: 2019-03-31T14:31:46+02:00
---
Why is it a problem in software engineering?
--------------------------------------------

Imagine how would be a world where cars, dishwashers, any electronics would
have as many bugs as we have in software? It would be ridiculously bad: a car
would refuse to start when it encounters a division by zero, your fridge would
just stop working because it lost connection to Internet, etc... We do not
tolerate bugs in those devices, yet, in software engineering, we have learned
to accept them and cope with them in our planning.

Where bugs come from?
---------------------

Usually when we debug we realize the mistakes of others. But it is a humbling
experience when the mistakes actually comes from ourselves. It is only then
that we realize and ask ourselves: "Why did I write this?". Most often it is
due to the lack of experience at the time, time constraint, things we thought
were harmless, etc... And most of the time we can only blame the human factor:
the machine did operate exactly as we asked, but the human who wrote the code
is actually flawed.

A fight against the lazy brain
------------------------------

Sometimes the comfort in some programming languages is only encouraging the
laziness of the developer. Example with this piece of code in JavaScript where
you just "forget" to handle the error case:

```javascript
function get_the_nth_element_and_add_1(arr, n) {
    return arr[n] + 1;
}
```

When you write it, it looks like it's done and complete and your lazy brain
just want to test the happy case scenario on it to feel the sense of
accomplishment. While in fact, you just started introducing a potential bug:
`arr` might not have the element `n` and you're just *not* handling that case.

Using an option type you can actually avoid that mistake by being explicit on
what is going to be needed to handle all case scenarios:

```rust
fn get_the_nth_element_and_add_1(arr: Vec<u32>, n: usize) -> u32 {
    let element = arr.get(n);

    if element.is_some() {
        return element.unwrap() + 1;
    } else
        // TODO: what do I do?
    }
}
```

Of course this looks cumbersome to use in an everyday basis but this is
actually a verbose example to show *how* you can achieve a better code quality
by fighting your lazy brain using better design patterns. Obviously your brain
won't like it for the first couple of uses. But at some point it will become
natural and even developing in programming languages that don't have option
types will trigger a compilation error in the back of your head.

What needs to be learned from this experience is that sometimes you need to
force yourself to learn a new thing to actually discover better ways of doing
things and improve yourself at the same time. We all want to be good developers
but we often forget how you achieve this: by keeping learning new things and
new ways. Not everything you will learn will be good, but at least it will give
you more understanding and more choices in the future.

Is it the perfect solution to avoid bugs?
-----------------------------------------

First things first, you need to accept that you won't be able to produce
perfect code all the time. Of course you are doing your best to produce
bug-free code but remember that this won't happen in real life.

The second important point is that you need to acknowledge your own ignorance
to put yourself in a constantly learning mindset. When you are in a learning
mindset you are continuously seeking to understand and know what you are doing
before committing and pushing code. Of course learning has a cost in
development time but one of the best consequences is that you are going to save
the time to the person who is going to debug your code.

Last and not least: the lazy brain is only one part of the bug problem. There
are many other aspects that are actually involved in the high number of bugs
and knowledge and better design patterns are no silver bullets. Actually, in
some cases, it even helps introducing new problems like overengineering.
