---
title: Abstracting control
tags: racket, scheme, control
---

Paper link: http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.43.8753

Reason for reading: I saw this paper referenced a few places, and I've never read it. I just wanted to skim over it and understand what it was about. Notably, I think this was the paper that introduces shift and reset. I wanted to practice bending my mind to continuations more.

# Helpful continuation motto

I always find these continuation papers to be hard to read, because I forget they basically invert control flow. Holding the rest of the whole computation in your hand feels like a scary thing to me, since I'm so used to programming under a stack. In these instances before reading these papers I remember the CPS mantra:

> "Don't call us, we'll call you."

To illustrate that nothing in CPS is ever really calls something, it invokes it with a continuation to call it back.

# Big ideas

### Shift and reset are different than prompts because they are static rather than dynamic

This was made as an offhand note in the paper, but it is definitely an important one. It seems to subtly relate to their implementation as you work implement using continuation marks as well. I do recall some of this from the continuation marks papers. It might have been nice to see a clear illustrated example here, but I think I could imagine what was going on.

### Recover control as functions rather than in the semantics

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

### Use metacircularity to efficiently generate a CPS-translated term

The authors make the observation that using the naïve CPS translation means that you build up a lot of terms that you don't really need. Typically, there would be another "partial evaluation" cleanup pass that would take care of these unnecessary terms. However, it turns out that if you are willing to make the term builder generate thunks, you can simplify this process a lot.

#### First trick

Make the CPS-converter *itself* a CPS-style function. This is at the entry of section 2 and allows efficiently converting lambda terms with shift and reset to the lambda calculus in an efficient way. They make generous use of the quasiquote and unquote facilities that clearly separate where the built terms end and the scheme code begins. One observation I had while reading this was that I never really "got" quasi quotes until I thouht about them as Ruby string interpolation...

#### Second trick

Once you write this CPS-converter in a CPS style, the authors make this kind of genius observation that you can write the analysis using shift / return. This is a really neat idea and I liked it a lot, it made definitions of these things a ton simpler.

# Terms I didn't know

- "applicative order" -- call by value. See this page: https://mitpress.mit.edu/sicp/full-text/sicp/book/node85.html

# Things that confused me / I didn't read

- Original definition of "escape", which I suppose is call/cc? The notation is confusing to me.
- It took me a few tries to read the definition of substitution correctly.

## Sections 4 and beyond

Sections 4 and beyond introduce a generalization of the meta-continuation semantics introduced in section 1. I didn't really understand this section very well. I think I understood its motivation, but I was confused for the following reasons:

- The authors claim that the translation to CPS results in an evaluation order for terms that is not call-by-value. But I can't see why. I suspect I could produce a long example detailing why, but I didn't think about it for very long.

- The authors claim that the fix to this was to CPS covert the converted result again. This is not **at all** obvious to me, and I wish the authors had explained it a bit more. Probably they did, if I had been more careful to read. Perhaps you would care to talk to me about it?

I don't really get section four, except for in very rough spirit. I sort of generally get the idea that you can use some metacircular facilitites to instrument the semantics at various levels, and that having a CPS converter gives you a direct hand on the continuations for doing this. That makes sense. But this section was weak in form of concrete examples or motivatio for why I would even want to do this, except for giving a laundry list of things you could add to a langaue in sequence, which seems like the right motivation. I did generally feel like this part was well written, I just didn't take the time to slog through it.

# Notes

## Extended CPS

#### Defining shift

```
translate(shift k in e) = \k. translate(e) (\x. x) [ k |-> \a k'. k' (k a) ]
```

Read in better english this rule says:

"Translate `shift k in e` as: "Give me a continuation you want me to run after my result. Using the identity continuation, run the following expression: e, where every occurrence of k in e has been replaced by `\a k'. k' (k a).`"

Note that, because the identity is used as the continuation passed into the translated expression, if (for example) `e` doesn't contain any occurrences of `k`, then the continuation "jumps" in a non local way back up to the top. This is where reset comes in, which I discuss next.

#### Defining reset

```
translate(reset e) = \k. k (translate(e) (\x. x))
```

This *delimits* the use of continuations in `e`. Passing the identity block `\x. x` to the translation of `e` means that--at worst--e could escape up to the `reset`. Consider what would happen in the `raise` example above, you would never be able to access the continuation `k` since it gets walled off by the identity continuation being passed in.

# Final thoughts

This was a fairly nicely written paper, and I think I got most of the core ideas before section four, which I will remember for a while and come back to if necessary. It was interesting to read the original paper on shift and reset.
