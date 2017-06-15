This is one of the original papers that describes precise call-return matching for analysis of procedural languages.

Paper link: http://www.cs.cornell.edu/courses/cs711/2005fa/papers/rhs-popl95.pdf
Reason for reading: want to compare to the work in more recent papers that include precise call-return matching for higher order languages.

## Big ideas

Form an "exploded" graph of the program that draws an intraprocedural dataflow analysis, and then use a summarization-based algorithm to connect only "valid" paths. The setup of the problem is quite nice and I enjoyed reading it. The core of the paper is in section 4, which expands on an algorithm for matching only calls and returns that have the correct nesting structure.

The idea of the algorithm is to insert summary edges between calls and returns. Then, at points where procedures return, you "look back" in the graph to perform precise call-return matching. This seems very similar to the technique used to construct Dyck State Graphs in Might's original approaches to pushdown earlier in the decade. The complication is that this is an algorithm that's slightly different than abstract interpretation, so there's a lot of machinery in Matt's work to hack this stuff into the abstract interpretation framework, since you basically have to introspect back through the graph you build up: this is a lot more natural to do in this context. But perhaps the advantage of that approach is that you always get a sound analysis for free (but not precise).

## Differences from work in functional languages

The difference between this work and the work for higher-order languages is that this paper doesn't allow proper closures. In fact, for simplicty this paper uses a *single* environment shared between all of the program. The authors note that this can be expanded, and I assume it expands to the well known flat environments. This seems like the fundamental difference between this work and that of P4F and the modern functional variants.

I am very interested in trying to understand how this work contrasts with P4F. I suspect once you largely blow away the environment, you end up with an algorithm that just allocates the continuation based on the target expression, which seems to be exactly what this work gives you.

## Praise

This paper is very well written. It sticks with a *single* example, that is very well thought out and expanded upon consistently throughout the text at just the right time. This is something that the modern papers on call-return matching seem to lack. Similarly, good figures explain the more complicated parts of the algorithms, cf. Figure 4.
