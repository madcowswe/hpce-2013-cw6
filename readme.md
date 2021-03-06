
Bitecoin writeup

We realise that this is quite a long writeup, but we want to share the full story.

We started by analysing the HashReference function, to see how the proofs were generated.
It was clear that this coursework was set up to (to some degree) mimic the way proof of work systems like bitcoin operate.
However, these use cryptographically strong proof of work systems, while this implementation looks like it was designed to be broken.
Just like a CTF (http://en.wikipedia.org/wiki/Capture_the_flag#Computer_security).

What was immediately peculiar with this implementation is that we are free to supply multiple nonces to be included in the hashing, while with implementations like bitcoin, there is only a single 32 bit field to permute.
Digging further we discovered two things:
The hash function seemed very simple and did not look like any strong hash we had seen, and more importantly, the entire hashing operation is done separately for every nonce, and the result is combined with a xor.

Before getting further, some terminology:
* Index (with big I to distinguish it) is used interchangeably for a nonce, as this is the terminology used in the supplied code.
* Point. This is what the code calls an intermediate result of the hash function (originating from a single nonce) before it has been xor-ed to become the final proof.

The expensive operation in this scheme, at least on first sight, appears to be to compute the hash function itself. It requires many iterations (if you can cal ~25 many) of 256 bit arithmetic operations.
In regular crypto currencies you are forced to simply try random nonces, compute the expensive hash, and check the result.

In our case, we can capitalise on the fact that indices are combined after the hash function. This means that if we compute N points using N indices, we can produce O(N^2) point combinations.
In effect we can try O(N^2) very cheap (a single wide xor) combinations for O(N) expensive operations.
If we chose to combine k indices, we can get O(N^k) combinations doing O(kN) expensive operations.

We implemented an Index-pair brute forcing algorithm, and then generalised this to a general recursive k-points brute forcing algorithm.
This is pretty cool, and our plan for about one day was to come up with a very elaborate xor brute forcing engine on the GPU.
The original sketches which involved fancy rotating buffers looked like clockwork, and hence the name, which stuck.

However, as we were brute forcing along trying out this idea on the server using the CPU and using two Indices, we spotted a pattern.
When we printed out the chosen best Index pair, the most significant hex digits would occur as specific pairs.
More specifically we saw 0xa... and 0x1... were always paired, and 0xb.. and 0x0... was always paired, etc.
When one went up, the other went down.
So we set up our local server to do constant 30 second rounds, to give our pairwise brute force algorithm time to converge on the actual optimal solution, and printed out their difference. Jaws were dropped. The difference was 0x94632009. Every single time.

We later found that this constant optimal difference between chosen indices (which we will call GoldenDiff) is a property of numsteps and c.
In fact, in retrospect, we were very lucky to spot this in time, as later the server changed to ask for a random hashsteps, not a constant one, and the local server code asks for a random hashsteps too.

We set up the server to vary different parameters to see what we can change such that the GoldenDiff was still 0x94632009. It turns out that we could change the blockchain data, round ID, and salt, and the GoldenDiff was still the same.
Changing either hashsteps or c changed the optimal Index difference.
So looking at how these parameters apply, it makes sense: hashsteps and c are a property of the hashing function, while the other parameters vary the input data.

So we asked ourselves: why does supplying indices with some specific offset produce really good proofs?
To get a good proof with two Indices, you get it from xor-ing two points. Good proofs have leading zeroes, so good points to xor have the same leading bits.
Since the hash function is based on arithmetic primitives, and since there is a specific significance of the arithmetic difference between the two input Indices, we decided to write a function that finds the arithmetic difference between two points.
We repeatedly picked an arbitrary Index, and paired it with another, offset by GoldenDiff, and checked the difference between the outputs.
Jaws were dropped again, see screenshot in the screenshot folder.
It was constant. All 256 bits.

We suspected that this property wasn't unique to the specific value of GoldenDiff. And indeed, the pattern we saw was: if we stick an Index pair given by (base, base + diff_in), and get the points (pointa, pointb), then if diff_in is constant, then $diff_out = pointb-pointa$ is constant for ANY base.
That is, there is a constant mapping from diff_in -> diff_out.
The significance of GoldenDiff is that it happens to be the the diff_in such that abs(diff_out) is minimised.

This property is, again, independent of input (only dependent on hashsteps and c).
That is, once we have found the GoldenDiff, we can use it on all rounds with the same c and hashsteps.
We therefore implemented a diff-cache which used the pair c and numsteps as the key.


So, we have at this stage established that the name of the game is to find this GoldenDiff as fast as possible, because nothing indicated that the deployed version would get constant hashsteps and c.
Our previous insight was that GoldenDiff = argmin(diff_in)(abs(pointb-pointa)).
The naive algorithm is to simply generate a large amount of index-point pairs and find the absolute difference between all combinations. That is O(n^2) comparisons.
But if we sort the points (and implicitly the index), we can then scan the sorted array for the best adjacent pairs.
As sorting is O(n log n), this provides a massive speedup.

With a point-index set of 2^16 elements, we go from finding the GoldenDiff in several seconds to 30 milliseconds. Sweet!

Now the issue is: We have further algorithmic steps (which will be described later), but which requires knowledge of GoldenDiff. How do we chose how many elements are required to try in order to guarantee to a good enough probability that we will find Golden diff?
We consider the fact that sorting and scanning N elements is equivalent to doing N^2 pair comparisons, as it will, as discussed before, find the same result.
We know that the space to search for GoldenDiff is the 32 bit space.
Trying a combination, we have the probability of finding GoldenDiff is 1 / 2^32. Therefore the probability of not finding GoldenDiff is 1-(1 / 2^32). The probability of not finding GoldenDiff after N^2 combinations is (1-(1/2^32))^(N^2).
We say we want this probability to be 0.01 and ask wolfram alpha to invert this for us, and find N=140600.
This runs in only 60ms. As we plan to implement the diff cache, we only have to pay this penalty on cache misses, so we double the number wolfram alpha gives us just for good measure.
Thus we have concluded that we are pretty much guaranteed to find the GoldenDiff in 120 milliseconds.

So we have the GoldenDiff. Now what?
It seems that over the range of hashsteps and c we tried that the GoldenDiff will produce points that have the first 31-33 bits or so aligned, i.e. their absolute difference is zero as far as the most significant limb is concerned.
This means when xor-ed, the top 31-33 bits are cleared, using two Indices.
We can submit these combined points as the proof.
But we have sometimes up to 16 Indices. What do we do with this opportunity?

We could keep the xor-ed points as a meta-point, and combine meta-points to submit a 4 Index proof.
Again, the naive way would be to generate a big array of meta-points, and try all combinations.
Our goal is to find meta-points with the best aligned most significant bits. Before we used sorting to find the smallest absolute difference. What is neat is that happens to coincide with aligned most significant bits.
So we use sorting to compress the O(N^2) comparisons to O(n log n) again.

So now all we have to do is scan the list, xoring all adjacent pairs, and pick the smallest result.
We could call this our proof, and submit it.
But, we could go further, and instead of picking the best result, we stick all results into an array, sort that, xor adjacent pairs, and now pick the best. This then becomes and 8 index solution. Repeat for 16 index solution.

In summary, this is the flow of the N-levels method:

1. Find golden difference, 120 ms constant
2. Generate point-index pairs, O(N) - slowest part
3. Sort indices, O(N log N) - significant
4. XOR adjacent points to make n-level-meta-points, O(N) - very fast
5. Check maxIndicies (from server) to see if we can go another level deeper if yes, goto step 3
7. Scan for minimum n-level meta point, O(N) - very fast
9. Check if it’s the current best, if so change best solution and proof accordingly

A caveat is that we cannot use the same index twice, so during each pass of step 4 we cull any meta(meta(meta))point-index pair that involves the same index twice (or more).

Each pass expects to clear on average log_2(N^2) zeroes. So with the initial 32 cleared zeroes from GoldenDiff alone, we expect to clear 32+k*log_2(N^2) bits. For example for a work size of N = 2^20 and with 3 passes we expect to clear around 150 bits. On our machine this work size runs in just under one second.


Since the round times are variable, we need a mechanism to decide the work size dynamically to make the most optimal search while staying on time.
As we found that most of our time was spent generating Indices, we scale the work size linearly with the timeBudget.
This became a little problematic as hashSteps became dynamic too. We compensated by making it scale linearly with that too.
However, the sorting time doesn't scale with this, so we simply added a fudgefactor. Given more time, we would've modelled the relative time between hashing and sorting.
A third option could've been ``dynamic bailing'', where each stage needs to stay on a target percentage deadline from the start of the algorithm to the deadline.


At this stage we have a pretty solid algorithm, and we couldn't really find any more improvements or changes we wanted to do. So now we apply parallelisation.
Step 2 and 3 are trivially parallelisable using TBB parallel_for and parallel_sort.
We even parallelise step 4, not because it was a bottle neck, but because it was easy.

Execution times for a work size of N= 2^20 on a 4 core hyper threaded machine:
Generate: 0.42s
3 pass sort and scan: 0.31s

This ends up being our final submission. We explored porting the above algorithm to the GPU, but found that it was not more than twice as quick, and we were out of time by the end.

Nevertheless, some interesting discoveries were made:

Using the Thrust transform primitive, we used the provided wide_mul and wide_add, the GPU was much slower than the CPU in generating the points.
We therefore ported a 256bit multiplication implementation in PTX assembly to use 128 bit inputs, and used a 256 bit addition assembly code [1].
Now the point generation is twice as fast as the CPU with TBB, and according to the profiler is memory bandwidth limited. The major reason is that the write back of the generated points is strided across 8 words. This has poor coalescing performance across the threads.
Changing the point’s storage from array of structures (AoS) to structures of arrays (SoA) should eliminate this problem.

Initially we used the Thrust sort primitive with a custom ordering operator to provide the lexicographic ordering semantics across the multiple limbs.
This, however, was extremely slow. Online sources indicate that when sorting with custom ordering operators the algorithm is different from when sorting on integral types.
Specifically they are merge sort and radix sort, respectively.
Intuitively we would think that mergesort should be fast, especially as the profiler verified that the sort primitive did indeed use shared memory to do the initial passes.
In any case, all we can do is trust the sources that say that this type of sort is simply slow.

Thus we need to convert the sort of the 8 word point with a one word satellite value (the index) to a sort of one limb at a time, and still restricted to a one word satellite value.
To achieve this, we leave the points and the index in their arrays, and use a map to index into these arrays.
We use the following algorithm:

1. Initialise this map to the identity sequence. (1, 2, 3...)
2. for each limb, from least significant to most significant:
  1. Gather into a temporary array (currlimb) the current limb from the points array using the map
  2. Sort the map using the current limb as the key
3. Gather the final points-index pairs to a new array using the map

Execution times on GPU (see attached screenshot):
Generate: 0.2s
1 pass sort: 0.2s

Given more time, optimisations such as SoA, and culling sorting on the lower limbs in the first couple of passes, could have been done.
We can conclude from this discussion that there is no substitute for analysing a problem properly and an algorithm with better inherent complexity. To quote wise old David “Parallelisation is no substitute for a better algorithm”.

Finally, we would like to thank you for such high quality coursework. 
This has been one of, if not the best, programming competition that I (Oskar) have taken part in. Not bad (Thomas).

Looking forward to the oral

Oskar and Thomas



[1] https://devtalk.nvidia.com/default/topic/610914/cuda-programming-and-performance/modular-exponentiation-amp-biginteger/









