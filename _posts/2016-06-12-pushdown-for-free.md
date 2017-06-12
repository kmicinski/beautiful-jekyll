---
title: Pushdown Control-Flow Analysis for Free
date: 06-12-17
tags:
 - scheme
 - "programming-languages"
 - "program-analysis"
 - "pushdown"
 - "control-flow-analysis"
---

Today I'm summarizing the results of a paper I've read a few times: [Pushdown for Free](https://michaeldadams.org/papers/p4f/pushdown-for-free.pdf). This paper is one of a few in Tom Gilray's dissertation.

This paper has a really nice and useful result, if you're used to writing abstract interpreters in the AAM style. It:
 - Gives perfect call-return ("pushdown") precision with of no greater (big-O) complexity than without.
 - Outlines a simple implementation strategy for doing so that uses off-the-shelf technology from AAM.

This paper relies on understanding the story of [Abstracting Abstract Machines](http://matt.might.net/papers/vanhorn2010abstract.pdf). I think this paper is fairly simple to read if you understand that story, but if you don't I think it would be confusing. (Indeed, the first time I read this paper I was still first beginning to understand how AAM really worked).

## A few important prerequisite facts from AAM

A few important reminders about AAM this paper uses..

#### Allocators

In the following I leave a lot of things hazy: I don't give an actual semantics I'm talking about, but I refer to something like Java. I don't give a formal definition of machine state, the join operator, etc...

The story in AAM is that you can massage a traditional abstract machine semantics so that:
  1. Everything redirects through the a store (or heap).
  2. That heap is made finite (by simply arbitrarily choosing its range to be a finite set of addresses)
    - For exmaple, a typical heap address space will be a bunch of object IDs or pointers in the case of C. You allocate more and more as the program runs. In AAM you would instead use something like the program label to create new objects.
  3. When updating the heap, values are joined.
  4. Evaluation of the transition system for the abstract machine's semantics is lifted to work with a nondeterministic set of states.

You need to know this story because this paper is all about picking the right allocator to get call-return precision. Precision and scalability in AAM is all about choosing this allocator to be right. The allocator says when to conflate things or not. Note that you can choose any allocator you want and get a correct result, but if you want the abstract interpreter to terminiate you need a finite number of addresses.

Just as a regular semantics for a programming language (like Java) will allocate new pointers in the heap as it generates new objects, the AAM semantics generate new pointers too. The difference is that there will only ever be a finite number of them, so things get "smushed together." This happens by means of a join operator on abstract values. One example for objects might be:

    l1: method void m(x) {
    l2:   Integer o = x;
    l3:   m(x+1);
    l4: }

If we ran this program in Java, it would run forever. Clearly, an abstract analyzer of this program should not run forever. Instead, it will begin to execute `m` and will allocate `o` at some point. To do so it will use an allocator, which has signature `alloc : state -> addr`. Note that I haven't said what `state` is, but let's assume it's a record containing at least the field `pc` for the current program counter. One simple strategy the machine could use is to allocate the object `o` based on the label it was created:

    alloc(state) = state.pc

This allocator will only ever use a finite number of addresses, because there are only a finite number of locations in the program. If `m` were to be invoked again, `m` would reuse the same location, join more stuff into that location, and eventually reach a fixed point. In this case I am also relying on the fact that the abstraction of the base types (integers) is finite. For example, I could pick a sign domain here and that would reach a fixed point.

Note that this is a fairly coarse abstraction. It says "conflate every value that is allocated at the same program point, without any regard to the stack, etc..." This lacks any notion of context sensitivity, so values from different stacks will get merged. This doesn't really mean so much here because this program is pretty nonsensical. A better example would be this:

let f x = (\g. x) (\x. x) in
assert(f (-1) < 0);
assert(f (+1) > 0);

Using this allocator, and assuming we model integers with the sign domain, this allocator would result in saying that f (-1) could be both positive *and* negative, breakign the assertion. By the way, these examples with assertions are really helpful for me to understand how this stuff works.

#### Concrete evaluation 

To recover concrete evaluation, you would keep adding more stuff into the heap. The way to do this is to have addresses be the natural numbers and to allocate based on the number of things in the heap (which emulates getting the next index in an arbitrarily long array):

    alloc(state) = |state.store|

Building abstract machines in AAM is all about playing around with this allocation function. By the way, the codomain of the allocation function defines the type of locations in the heap.

#### Call--Return precision

Our previous abstract interpretation example conflated calls and returns. For example, there is no way that a negative value could flow back into the point f (+1), since +1 is (obviously) positive. However, typical semantics for AAM do just this. Why do they do this? Because they heap allocate contexts (or "continuations" or "stacks" in my terms). Originally, AAM did this because stacks must be finite: an infinte stack would mean a machine that ran for infinite time, and that would be obviosly bad. The first approaches to AAM used a CPS converted semantics and used the same allocator for continuations as they did for the value allocator. But this is bad, because it means we lose precision for continuations in the same way that we lose precision for values.

The crux of the issue is in identifying that we need to add more precision to continuations, but not values. In fact, we want *perfect* precision for continuations. This will happen through the tricky use of an allocation function I'll describe. The genius observation is that we can cook up a finite address to use for calls and returns such that we still get perfect call return precision. It's really surprising to me.

# Notes

## Section 2

Section two introduces normal (concrete) evaluation of programs in ANF. ANF is a fairly straigtforward translation. I always think of it as bytecode. For example, in ANF:

    (x1 + x2) * (x1 - x3)

would be written as:

    let r1 = x1 + x2 in
    let r2 = x1 - x3 in
    r1 * r2

#### Trick: allocate values and continuations separately

Here's one trick that abstract interpretation people use I didn't know about: if you want to add precision to one part of your semantics, allocate for it separately. This could mean that you have a separate store for it, or you could take your store as the union of two kinds of addresses. The observation here is that we *know* we need to add more precision for continuations, because they are the thing that conflates calls and returns.

The paper presents this pretty plainly, but I think this is a key part of the issue: we need to have more precision for continuations, so change the type of addresses for continuations. Then, allocate for them in a way that lets us add precision.

As is typical, values are allocated using a "value allocator" with the signature:

    alloc : Var x Store^ -> Addr^

(I write `Sigma^` to indicate it's an abstract store.)
And continuations are allocated using a special continuation allocator:

    allock : Sigma^ x Exp x Env^ x Store^ 

Frankly, I never would have thought of this, but it mirrors a trick I've seen used in this [excellent paper on object sensitivity](https://yanniss.github.io/typesens-popl11.pdf). There, they realize that you need to allocate for *objects* in a special way. Here, the authors realize you need to allocate *continuations* in a special way.

(I've just said what the type of the allocator is, not what its implementation is yet.)

## Terms
- "administrative normal form"
- "direct style" vs "continuation-passing style".
