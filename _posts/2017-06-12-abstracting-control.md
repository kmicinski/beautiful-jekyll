---
title: Abstracting control
tags: racket, scheme, control
---

Reason for reading: I saw this paper referenced a few places, and I've never read it. I just wanted to skim over it and understand what it was about. Notably, I think this was the paper that introduces shift and reset. I wanted to practice bending my mind to continuations more.

# Helpful continuation motto

I always find these continuation papers to be hard to read, because I forget they basically invert control flow. Holding the rest of the whole computation in your hand feels like a scary thing to me, since I'm so used to programming under a stack. In these instances before reading these papers I remember the CPS mantra:

> "Don't call us, we'll call you."

To illustrate that nothing in CPS is ever really calls something, it invokes it with a continuation to call it back.

# Big idea

If you translate terms to CPS, you can naturally express control operators easily by using a translation. That is: rather than having to, say, implement control operators as something down in your semantics, you can use a normal semantics and just CPS convert.

Think about how normal CPS conversion works. It threads continuations through functions by adding them as an extra parameter. In the normal lambda calculus without higher-order control, this means that things reduce naturally and in the reduction (beta) order as you would expect. Now think about what would happen if you started messing with those continuations--say by dropping one on the ground:

For example, a normal (not-CPS-converted) term:

    (\x. x) 2

Would naively CPS convert to (I got this wrong a few times, illustrating how complicated it is even for simple terms...):

    (\k. (\k'. k' (\x. (\k''. k'' x))) (\f. (\k'. k' 2) (\a. f a k))
    -->
    (\k. (\f. (\k'. k' 2) (\a. f a k)) (\x. k''. k'' x)
    -->*
    (\k. (\k'. k' 2) (\a. (\x. k''. k'' x) a k))
    -->*
    (\k. (\a. (\x. k''. k'' x) a k))) 2)
    -->*
    (\k''. k'' 2) k
    -->*
    k 2

(Note that the translation of `2` isn't in the paper..)

One general observation is that at each step of the way, you could imagine that you might drop k, aborting you to the top of the program, because `k` holds the rest of the execution in its hands. Instead say we replaced 2 with something we wanted to cause an exception, like `throw 1`. We could translate that as:

    translate((\x. x) (throw 1)) = (\k. (\k'. k' (\x. (\k''. k'' x))) (\f. (\k'. 1) (\a. f a k))
    -->
    (\k. (\k'. k' (\x. (\k''. k'' x))) (\f. (\k'. 1) (\a. f a k))
    --> 
    (\k. (\f. (\k'. 1) (\a. f a k)) (\x. (\k''. k'' x))
    --> 
    (\k. (\k'. 1) (\a. (\x. (\k''. k'' x) a k))
    -->
    (\k. 1)

This is simple enough, since you hold a continuation in your hands you can just drop it and return 1 at any time. This illustrates that by rearranging the continuations, you can jump around in the program (obviously).

The main idea is that you can implement some higher order control operators by setting up the CPS conversion right. The idea of `shift` is that it reifies the current continuation as a function, which is not necessary in the semantics because it happens via CPS. The idea of `reset` is to delimit the continuation block.

# Terms I didn't know

- "applicative order" -- call by value. See this page: https://mitpress.mit.edu/sicp/full-text/sicp/book/node85.html

# Things that confused me

- Original definition of "escape", which I suppose is call/cc? The notation is confusing to me.
- It took me a few tries to read the definition of substitution correctly.

# Notes

## Extended CPS

#### Defining shift

```
translate(shift k in e) = \k. translate(e) (\x. x) [ k |-> \a k'. k' (k a) ]
```

Read in better english this rule says:

"Translate `shift k in e` as: "Give me a continuation you want me to run after my result. Using the identity continuation, run the following expression: e, where every occurrence of k in e has been replaced by \a k'. k' (k a)."

Note that, because the identity is used as the continuation passed into the translated expression, if (for example) `e` doesn't contain any occurrences of `k`, then the continuation "jumps" in a non local way back up to the top. This is where reset comes in, which I discuss next.

#### Defining reset

tba..
