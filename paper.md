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

Consider what happens when we generalize this problem to dynamic graphs, however. In particular, make two further generalizations: assume that $k$ edges or vertices 
are added, one by one, to the graph, and assume that after every edge addition we will query for the connectivity of two arbitrary nodes in the graph. This moves 
the graph into the realm of the _streaming_ model as well - observe that we can build up a graph by simply starting with an empty graph, then adding its $V+E$ edges,.
making $V+E$ queries. 

With no additional storage allowed, the straightforward algorithm to solve this problem will have to simply rerun DFS over the graph after _every single query,_
leading to an effective runtime of $O(kV+kE).$ In the case where $k=V+E,$ this will degenerate to $O(E^2)=O(V^4)$ in the worst case. Clearly, this performance
is poor; we would like to know how to do better. It turns out the solution to this involves something called "sketching," which we shall see more of below.

Sketching
===
When solving a problem on a static input, there's no advantage to holding on to information after the algorithm is completed; in fact, it would be an obvious waste
to do so. However, when working with _dynamic_ input, especially one which changes "incrementally," it becomes advantageous to retain at least _some_ information
about it. Usually, a dynamic dataset is so big that one cannot simply store it in memory; instead, we must store some smaller representation of it, which gives
us an "approximation" of the data it contains. This, vaguely, is the notion of sketching.

More specifically, given a dynamic data structure $T,$ a __sketch__ $\mathcal S(T)$ is a substantially sublinear dynamic data structure that "approximates" the contents of $T.$ 
We will break this down further.

* By __sublinear,__ we mean that if $|D|$ denotes the memory necessary to store data structure $D,$ then $|\mathcal S(T)|=o(|T|).$ In practice, we will
  want to achieve a "win" of roughly a linear factor; thus, if $|T|=\Theta(n),$ we will seek for $|\mathcal S(T)|=\Theta(\log^k n)$ for some $k,$
  and likewise a data structure of size $|T|=O(n^2)$ (such as a general graph) will get space $O(n\log^k n).$ 
* By __dynamic,__ we mean that $\mathcal S(T)$ should handle updates. To be fair, we will allow $\mathcal S(T)$ to _choose_ which types of updates it supports, and
  refuse to support any others. More on this below.
* By __"approximation,"__ we mean a few things. As a prelude, observe that a sketch $\mathcal S(T)$ _cannot_ perfectly preserve the 
  contents of $T$, because $|\mathcal S(T)|$ must be strictly _smaller_ than $|T|,$ and information theory tells us perfect compression in this form is impossible.
  There are two ways we can deal with this problem. First, we could decide to preserve only certain types information about $T$, letting queries relying on other 
  information fail. Second, we could decide to relax the requirement of accuracy, and instead let the sketch only _approximate_ the answers to specific questions
  about $T.$ In practice, we shall do both. 

This may seem a little obscure, so we will give a few examples of sketches. 

1. Consider the __Bloom filter.__ As covered in CS 161, the Bloom filter is a way to approximate the notion of a set, supporting queries such as
   \textsc{IsElement} and \textsc{AddElement} (but no others). It has space strictly smaller than that necessary to store an actual set (in fact, depending on the
   error parameters, it can be as small or as large as the user needs!), and because it supports the \textsc{AddElement} operation, it supports the dynamic updating of
   the set. This sounds an awful lot like a sketch!
2. Recall the paper from two weeks ago in which we discussed locality-sensitive hashing. For a set $T,$ define $\mathcal S(T)=\{h(x)\mid x\in T\},$ where $h$
   is drawn from some locality-sensitive family $\mathcal H$ of hash functions. In this case, $\mathcal S(T)$ supports $\textsc{AreNeighbors}(x,y)$ for any
   $x,y\in T,$ and supports any queries that rely solely upon discerning the "families" of its member.

At this point, sketches may seem like an unnecessarily limited form of computation - why on earth would we ever want to use them? In practice, however, forcing these
restrictions allows sketches to have immense general-purpose applications. 

* If we have a large dataset $T,$ very often we will be unable to hold all of $T$ in memory
  at once. Every time we want to obtain information about $T,$ we will need to rerun an at-best-linear computation over $T$; if we have many such queries, this will
  quickly become large. Sketches provide a way to "cache" the information we know about $T.$
* Many times, we will receive information as a "stream" of updates rather than as a single monolithic input. In the Facebook social graph, for example, new members and
  new friendships occur all the time. We want our cache to be able to incorporate these updates and record them over time.

To re-emphasize a few points, recall that sketches will _not_ preserve all information about a dataset; they can be thought of as "lossy" compressions. 
As a consequence, only certain questions can be asked of sketches, and even those may not be accurately answered. For many algorithms, though, these
restrictions lead to powerful gains in space and time.

Solving connectivity with sketches
====
To recap: we want a $O(V\log^k V)$ data structure that answers questions about connectivity. In particular: for $u,v\in G,$ is there a path from $u$ to $v$?
And what happens after we add or delete an arbitrary edge $(x,y)$? 

We already tried one naive approach, namely rerunning the depth-first search algorithm after each addition of an edge, but saw quickly that it was too slow.

We propose a simple "sketch" that will solve the problem, as long as only edges and vertices are added to the graph. In particular, 
run Kruskal's algorithm over the graph to generate a _minimum spanning forest_ of the graph. Then each connected component of the graph will be represented
by _a unique_ MST, and $u$ and $v$ will be connected if and only if they are part of the same MST. Since the maximum number of edges in a minimum spanning forest
is $V-1,$ the total size of this data structure is $O(V).$ Furthermore, we store the connected components using the union-find
disjoint-set data structure covered in CS 161, which takes up space $O(V).$ The union-find structure supports testing for set membership and "merging" subsets
with high effectiveness, but does not support "splitting" subsets up.

Obviously, this sketch supports the query $\textsc{Is-Connected}(x,y).$ It also supports $\textsc{Add-Edge}(x,y)$ - to connect the nodes $u$ and $v,$ we 
check if they are already in the same connected component. If they are, we ignore the addition; if they are not, we connect them and record 
of $u$ and $v$ 
draw
an edge connecting the two vertices in the graph. 

Here's a problem, though: _what happens if we delete edges?_

Key problem
====
We've now finally arrived at the problem the McGregor et al. paper aims to solve. In particular, we want to find a sketch $\mathcal S(G)$ of size
$O(V\log^k V)$ for a graph $G$ 
that solves the problem of dynamic graph connectivity, supporting the high-level queries of whether two vertices $u,v\in G$ are connected, and
adding or deleting an edge $(u,v)$ from $G.$ 

Intuition for the sketch
====
It turns out that the most effective way to sketch graphs will rely on being able to sketch some kind of _representation_ of them. Observe that most of
the ways we represent graphs rely on regular data structures - in particular, matrices. The problem of sketching matrices is well-understood, and our graphs enforce
a constraint that makes sketching them even easier. In particular, objects such as the 
adjacency matrix representing a graph have a __key constraint__: their elements are binary! This means that we only need the relatively coarse constraint
of looking for nonzero elements in the matrix in order to generate an efficient sketch of a graph. 

Conveniently, the problem of sketching vectors of numbers is well-understood in many different applications. In particular, the authors of the paper use

Jowhari et al: "comb" sketching
Input: vector $\mathbf{v}$ with some zero and nonzero elements
Output: index $i$ such that $\mathbf{v}[i]$ is nonzero with high probability
Input like a snaggle-toothed comb; output returns a "tooth" of the comb
Runtime: about $O(\log^2 n)$

Comb sketch
====
In particular, the following theorem holds from previous work in the field.
__Theorem (Jowhari et al):__ For a vector $x$ of length $n,$ there exists a $O(\log^2 n)$-sized sketch $\mathcal S(x)$ such that:
1. $\mathcal S$ is _linear_ - that is, there exists some operation $\oplus$ such that 
   $\mathcal S(x)\oplus\mathcal S(y)=\mathcal S(x+y)$ for all $y$ of length $n$, and
2. We can "easily" (in roughly constant time) obtain $i \leftarrow \mathcal S(x)$ such that $x_i \neq 0$

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
