% Analyzing Graph Structure via Linear Measurements.
% Reviewed by Nikhil Desai


Introduction
====
Graphs are an integral data structure for many parts of computation. They are highly effective at modeling 
many varied and flexible domains, and are excellent for representing the way humans themselves conceive of 
the world. Nowadays, there is lots of interest in working with large graphs, including social network graphs,
"knowledge" graphs, and large bipartite graphs (for example, the Netflix movie matching graph).

Many of the papers we have covered so far in CS167 have worked with graphs. Karloff et al. (__citation__)
discussed counting triangles with social networks in a MapReduce environment, while Gupta et al. (__citation__) proposed a new way of discovering
"triangle-dense" clusters representing potential communities in 

There are a few similarities in the types of graphs examined in such papers. 

1. First, beyond a doubt, all of them are __large.__ The Facebook graph, for example,
   contains on the order of a billion vertices and nearly one trillion edges. Even the excellent graph primitives we learned in CS161, such as depth-first and breadth-
   first search, would require significant time and space to run on such graphs. 
2. Second, they are all highly __dynamic.__ New structures appear in social network graphs at every moment. In particular, one of the defining features of such graphs is that
   new edges and vertices can be constantly added. The structural changes these induce are of some interest.
2. Third, there is significant interest in the __global structure__ of these graphs, and in particular how that structure changes over time. 
3. Fourth, while we would like an infinite amount of space and time to solve these questions, we don't. Compared to the size of the graphs, __our computing
   power is very limited.__

These conditions, roughly, define the so-called "streaming graph problem," in which the user must analyze a graph presented in the form of incremental updates,
using limited space and time. 

What does "limited space and time" mean? To determine this, we need to review the various techniques available for graph storage. The standard representation we
learned in CS161 was the adjacency matrix, which takes up size $\Theta(V^2).$ In practice, however, the standard is the _adjacency list,_ which takes up space
$\Theta(V+E),$ in the worst-case $O(V^2).$ Another uncommon form is the incidence matrix, which takes up space $\Theta(VE)=O(V^3).$ In general, the space available
to us will be smaller than the worst-case storage necessary for such representations - in particular, it will be something like $O(V\log^k V).$ With regards to time,
streaming models tend to ask that we not look at any part of the graph more than once - in other words, we must make a _single pass_ through the graph and 
then provide an answer to a given query. Technically, this establishes a "runtime" bounded above by $O(V+E)$ or $O(V^2)$; in practice, we want to perform exactly one
action involving the _actual_ graph, and thereafter run in time linear in the size of the underlying sketch.

This may seem like a needlessly constrained model; why should we put arbitrary restrictions on our runtime or the auxiliary space used like this? To get some intuition,
consider a common problem - getting information about the connectivity of a graph. This means not just simple questions like finding the smallest connected component, 
but also the more general question - for any given $u,v\in G,$ are $u$ and $v$ connected?

Let's use connectivity as a central "conceit" for our examination of dynamic graphs, and see what we can do to fix these problems.

Connectivity
====
For a general graph, the problem of connectivity is well-understood and easy to solve. In particular, to find out if $u$ is connected to $v,$ we
need only run a depth-first search from $u$ to expose all the nodes in a graph to which $u$ is connected. This depth-first search takes time $\Theta(V+E)$
(with very small constant factors), and is the best we can do on a static graph. 

Consider what happens when we generalize this problem to dynamic graphs, however. In particular, make two further generalizations: assume that $k$ edges are added,
one by one, to the graph, and assume that after every edge addition we will query for the connectivity of two arbitrary nodes in the graph. 

If we wish to find out the connectivity of two arbitrary nodes in the graph
after every addition to it, we will 

BFS and DFS runtime: $O(V+E)$; auxiliary storage $O(V)$ Algorithms using BFS/DFS repeatedly will traverse same graph _multiple times_! When $V$ is big, this is bad.
When our graphs are big, even $O(V)$ runtime sucks.

Sketching
===
Recall last week's paper introduced the notions of _sketching_ and _streaming_.
Streaming: algorithms that consume their input in "blocks" at a time, holding on to $O(\log n)$ memory.
Sketching: "small" (sublinear, usually logarithmic) representations of a dataset that support queries over that dataset.
_Strongly related notions!_
Streams often build sketches of the data to use as their "memory".
Constructing sketches should happen in a streaming fashion.

Analogy to locality sensitive hashing
Compresses a dataset while retaining a high percentage of its structure
We usually build sketches _for_ particular problems

Compressed sensing: reconstruct sparse signal $\mathbf{x}$ from "small" number of linear measurements
_Functional_ compressed sensing: for some function $f,$ compute $f(\mathbf{x})$
Note: _no need_ to reconstruct $\mathbf{x}$ in this process!
Can perform functional compressed sensing on a vector $\mathbf{x}$ by creating a sketch $\mathcal S(\mathbf{x})$ and asking
this sketch for $f(\mathbf{x})$

1. __How can a sketch work? Don't we need every element of the data structure to get an answer?__
   Yes, we do. Sketches are _lossy_ compressions of a dataset, and preserve only specific types of information.
2. __Is there such a thing as a "canonical" sketch of a dataset?__
   Since sketches are inherently lossy, there will be queries that the sketch cannot answer with accuracy.
   That said, there are some sketches that support many different types of queries and have become the "standard" for many algorithms.

A "summarized" version of the dataset in question, which we can query for salient information
__Information is lost__: we cannot reconstruct the graph. Not all questions can be answered accurately.
We want a $O(N)$ data structure that answers the following question: for $u,v\in G,$ is there a path from $u$ to $v$?
Answer: run Kruskal's algorithm over the graph to generate a _minimum spanning forest._ $u$ and $v$ are connected if they are part of the same MST.
Problem: _what happens if we delete edges?_

Key problem
====

We would like to solve the following problem: 
__how can we sketch an undirected graph $G=(E,V)$ in a way that supports many structural queries?__
Check if two vertices $u,v\in G$ are neighbors.
Compute the number of connected components of the graph
Check if the graph is bipartite.

Matrices and vectors representing graphs have one __key constraint__: their elements are binary!
Adjacency matrix $A\in\{0,1\}^V\times\{0,1\}^V$ 
Vector of neighbors $\mathbf{a}_i$: $\mathbf{a}_i[j] = [(i,j)\in E]$ 

There exist very good algorithms for sketching _linear_ data structures - e.g. vectors
Jowhari et al: "comb" sketching
Input: vector $\mathbf{v}$ with some zero and nonzero elements
Output: index $i$ such that $\mathbf{v}[i]$ is nonzero with high probability
Input like a snaggle-toothed comb; output returns a "tooth" of the comb
Runtime: about $O(\log^2 n)$

Comb sketch
====
__Theorem (Jowhari et al):__ For a vector $x$ of length $n,$ there exists a 
$O(\log^2 n)$ sketch $\mathcal S(x)$ such that
    1. $\mathcal S$ is linear: $\mathcal S(x)+\mathcal S(y)=\mathcal S(x+y)$ for all $y$ of length $n$
    2. Can easily obtain $i \leftarrow \mathcal S(x)$ such that $x_i \neq 0$ 

* TODO: image of comb; highlight tooth of comb

Using comb sketches
===
* Intuition: to approximate connectivity in a graph, comb-sketch matrix representations of the graph.
* Vector of neighbors $\mathbf{a}^i$: $\mathbf{a}^i_j = 1$ if $(i,j)\in E,$ $0$ otherwise
    * $i$th row of adjacency matrix
* Sketch $\mathcal S(\mathbf{a}^i)$ returns neighbor of $i$ with high probability! 

Using comb sketches
===
* Sketches represent linear data structures, so updating is easy - just add the vectors
* Edge _contraction_ is also easy to simulate: add neighbor vectors together
    

Non-sketching algorithm
===
* For each node, find an edge coming off that node. 
* Contract the edge, forming a "supernode."
* Repeat until no edges remain.

You need an animation here
===

Sketching algorithm
===
* Construct a matrix $M$ of size $n\times{n\choose 2}$ 
* $M[i,(j,k)] = 1$ if $i=j$ and $(v_i,v_k)\in E$; $-1$ if $i=k$ and $(v_j,v_i)\in E$; $0$ otherwise.
* Construct $t=O(\log V)$ sketches $\mathcal S_1,\mathcal S_2, \ldots, \mathcal S_t$ of $M,$ of size $O(n\log^2 n)$ each.
* Simulate contraction of edge $(i,j)$ from sketch $\mathcal S_k$ by computing $\mathcal S_{k+1}(\mathbf{a}_i)+\mathcal S_{k+1}(\mathbf{a}_j)$

The matrix $M$
===
* For any $S\subseteq G,$ the size of $E_S$ (the edges crossing the cut $(S,V-S)$) is $\ell_0(\mathbf{x})$ where
  $\mathbf{x}=\sum_{v_i\in S} \mathbf{a}_i.$ 
* $\mathbf{x}_i \in \{-1,0,1\},$ and $\mathbf{x}_i \neq 0$ iff $(v_j,v_k)\in E_S.$ 

The sketches
===
* Contraction affects the distribution of neighbors obtained from the sketch.
* We need to use a "replacement" sketch.
* Number of sketches determines number of "rounds" we can run our algorithm.
* Choose $t=O(\log n)$ so that algorithm has space usage $O(n\log^3 n)$ 

What are the key takeaways?
====


An interesting anecdote
===
"__Acknowledgements.__ The authors would like to thank Piotr Indyk for some useful conversations conducted in the back of a taxi 
somewhere between Orchha and Lucknow. The third author would also like to thank any other passengers for their patience and understanding."
