---
title: Pushdown Control-Flow Analysis for Free
tags:
 - scheme
 - "programming-languages"
 - "program-analysis"
 - "pushdown"
 - "control-flow-analysis"
---

Today I'm summarizing the results of a paper I've read a few times: [Pushdown for Free](https://michaeldadams.org/papers/p4f/pushdown-for-free.pdf). This paper is one of a few in Tom Gilray's dissertation. These notes are in very rough form and I'm sure I made some typos (or even outright logical errors) so feel free to correct me.

This paper has a really nice and useful result, if you're used to writing abstract interpreters in the AAM style. It:
 - Gives perfect call-return ("pushdown") precision with of no greater (big-O) complexity than without.
 - Outlines a simple implementation strategy for doing so that uses off-the-shelf technology from AAM.

This paper relies on understanding the story of [Abstracting Abstract Machines](http://matt.might.net/papers/vanhorn2010abstract.pdf). I think this paper is fairly simple to read if you understand that story, but if you don't I think it would be confusing. (Indeed, the first time I read this paper I was still first beginning to understand how AAM really worked).

I think the core idea in this paper is a bit of a bait-and-switch: it looks so simple that it's easy to nod your head. The intuition for the core idea is interesting---and I think the authors did a good job at presenting it---but it's still very challenging. I would like to write up a more comprehensive post drawing some more elaborate diagrams and explaining the core idea in the paper, which the authors probably understood from having read a paper I did not (the "Abstracting Abstract Control" paper).

# Notes

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

    alloc(o, state) = state.pc

This allocator will only ever use a finite number of addresses, because there are only a finite number of locations in the program. If `m` were to be invoked again, `m` would reuse the same location, join more stuff into that location, and eventually reach a fixed point. In this case I am also relying on the fact that the abstraction of the base types (integers) is finite. For example, I could pick a sign domain here and that would reach a fixed point.

Note that this is a fairly coarse abstraction. It says "conflate every value that is allocated at the same program point, without any regard to the stack, etc..." This lacks any notion of context sensitivity, so values from different stacks will get merged. This doesn't really mean so much here because this program is pretty nonsensical. A better example would be this:

    let f x = (\g. g x) (\x. x) in
    assert(f (-1) < 0);
    assert(f (+1) > 0);

Using this allocator, and assuming we model integers with the sign domain, this allocator would result in saying that f (-1) could be both positive *and* negative, breaking the assertion. By the way, these examples with assertions are really helpful for me to understand how this stuff works.

#### Concrete evaluation 

To recover concrete evaluation, you would keep adding more stuff into the heap. The way to do this is to have addresses be the natural numbers and to allocate based on the number of things in the heap (which emulates getting the next index in an arbitrarily long array):

    alloc(o, state) = |state.store|

Building abstract machines in AAM is all about playing around with this allocation function. By the way, the codomain of the allocation function defines the type of locations in the heap.

#### Call--Return precision

Our previous abstract interpretation example conflated calls and returns. For example, there is no way that a negative value could flow back into the point f (+1), since +1 is (obviously) positive. However, typical semantics for AAM do just this. Why do they do this? Because they heap allocate contexts (or "continuations" or "stacks" in my terms). Originally, AAM did this because stacks must be finite: an infinte stack would mean a machine that ran for infinite time, and that would be obviosly bad. The first approaches to AAM used a CPS converted semantics and used the same allocator for continuations as they did for the value allocator. But this is bad, because it means we lose precision for continuations in the same way that we lose precision for values.

The crux of the issue is in identifying that we need to add more precision to continuations, but not values. In fact, we want *perfect* precision for continuations. This will happen through the tricky use of an allocation function I'll describe. The genius observation is that we can cook up a finite address to use for calls and returns such that we still get perfect call return precision. It's really surprising to me.

#### Univariance / Monovariance / Polyvariance

These papers use some insular terms in ways that I don't think are fully explained. "Polyvariance" was an opaque term to me for a long time, and I really had no idea what it meant. But it's really just a super simple concept: how much precision do you give various things in your abstract interpretation. There are three terms that are used a bunch. They are all instances of the allocator framework given here.

- "univariance" means that there is only ever *one* address that your allocator is given. This is confusing, because it basically means that everything in your allocator is going to be *supppper* imprecise. This term confused me because I thought to myself, "why would anyone want an allocator that did that!? What is its technical purpose." I think it's just one very simple instance used to explain things, but there are some potential applications of it. For exmaple, in other papers Tom notes this can be used for dead code eliminiation.

- "monovariance" means that everthing gets allocated based on its name. For example, the allocator up above that allocates based on pc is a form of a univariant allocator. Actually that might not be quite right. I think the real "monovariant" allocator is just this:

    alloc(x,state) = x

This is strange to me. Because I might think to myself "but what if the program uses x twice in two different ways? Won't they be conflated!?" And the answer is that yes, they will. So I think monovariant allocators are by convention assumed to take a program point's label as well:

     alloc(x^l, state) = l

Tom notes to me that even alpha renaming variables does not solve the problem because of a subtle interaction with P4F's allocation strategy I describe below.

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

#### Store Widening

Global store widening is a trick that Tom tells me he uses a lot in these analyses. This trick threads a global store through an analysis. The problem is basically that the naïve analysis without global store widening ends up doing a bunch of redundant work that ends up getting merged together in the final solution. 

I don't have a crisp explanation of this, except to pull a quote from the paper:

> The crux of the issue is that, in exploring a naïve state-space
> (where each state is specific to a whole store), we may explore both
> sides of every diamond in the store lattices. All combinations of
> possible bindings in a store may need to be explored, including
> every alternate path up the store lattice. For example, along one
> explored path we might extend an address a˜1 with clof1 before
> extending it with clof2, and along another path we might add these
> closures in the reverse order (i.e., clof2 before clof1). We might also
> extend another address a˜2 with clof1 either before or after either
> of these cases, and so forth. This potential for exponential blowup
> is unavoidable without further widening or coarser structural
> abstraction.

Sets are unordered, so building up a set of `{a,b}` as a first then `b`, or `b` first then `a`, means you're wasting a lot of time (exponential when you could have polynomial). Store widening is way to combat this, by pulling the stores out of the states. Instead, it represents every state as a "configuration" (which is everything except a store) and then pairs it with a global store.

#### PDCFA

The first pushdown control flow analysis shown is PDCFA. This is the original---and complicated---version of pushdown analysis that uses a Dyck State Graph to represent a pushdown system. This part of the paper is a real slog. PDFCFA builds up a graph of vertices and labeled edges. Vertices are powersets of configurations (think states) and edges are labeled edges between configurations with either a push or a pop of a frame.

Dyck State Graphs are constructed in a kind of roundabout way, by giving a propositional definition of what makes a DSG correct for an infinite stack machine. This is a little confusing, because you can't actually compute the infinte-stack machine. Still, it helps give the intuition about what a DSG is.

## P4F

The big reveal in P4F is that you need only pay attention to two things when you allocate a continuation address:
- The target control expression (or program counter / method name)
- The environemnt passed to the invocation

Why these two things are sufficient to get call--return precision is totally unclear to me at first. It does not feel at all obvious to me why that should be the case.

The most helpful idea in this part of the paper was the following: think about when it is possible to have return-flow merging in a finite-state analysis. This happens when you get to a return point configuration `(return v, rho, ak)`, and the address `ak` is less precise than the set of configurations that transitioned to the entry point at the beginning of the function.

For example, consider you have the same example over again:

    let f x = (\g. g x) (\x. x) in
    assert(f (-1) < 0);
    assert(f (+1) > 0);

In a monovariant analysis, when you return to `g` from the body of `\x. x`, the first time we return -1 to that point, and the second time we return +1 to that point. We want to avoid this.

### Things that confused me

**I think this is the most important part of this post**.

This quote is the crux of the paper.

> The primary insight behind our technique is that a set of intraprocedural configurations (like c˜0 through c˜5) necessarily share the exact same set of genuine continuations (in this example, the incoming call-sites for c˜0)

I really toiled over this for a long time. Why was this the case? I kept trying to think of counterexamples, and eventually pulled out some intuition?

At first I wondered: what did it mean to share a set of genuine continuations? Doesn't any particular intraprocedural configuration just have one *unique* continuation? And then I realized that no, this was not the case. There are many potential places that you could flow back to, because we're in abstract interpretation land. This seems extremely subtle to me: P4F isn't trying to make each continution precisely unique, since that would be imposible. It's just trying to get the correct precision for call--return matching. For example, consider a program that recursively loops. It's not that P4F will keep a unique configuration for each invocation of a function as it calls itself arbitrarily many times. It will just ensure that---if that function is called somewhere *else*, it will not be conflated with the invocation.

I also thought this: "but wait! I can easily imagine a case where I call `f` from `g`, and also from `h`, with the same environment, and expect different results." It turns out that no, I actually can't. Because in the pure lambda calculus those things will just be identified. However, once you add state things start to get trickier. I'm not quite sure how state interacts with higher order environments fully in this strategy, but I suspect it's complicated.

