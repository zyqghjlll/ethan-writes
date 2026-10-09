# From Two Sum to 3Sum: Learning to Compare Tradeoffs

Revisiting Two Sum led me to 3Sum—and a question I’m still learning to answer: how do I compare two approaches when both are O(n²)?

I started with brute force: fix one element, then search for the others.

For Two Sum, the key improvement was replacing repeated linear searches with hash-table lookups.

For 3Sum, I carried that idea forward: fix one element, then solve a Two Sum problem on the remaining suffix.

But the requirements had changed. I needed all matching pairs, not just one, and the final triplets had to be unique by value. My approach used a consistent ordering for each triplet and a set to remove duplicates.

Then I asked ChatGPT about alternatives and learned about sorting followed by a two-pointer search.

I wanted to understand what sorting actually made possible:

- Equal values become adjacent, making duplicates easier to skip.
- Sorted order lets us safely eliminate candidates as the pointers move.
- The inner search no longer needs a new hash table for each fixed element.

The hash-based approach takes expected O(n²) time, while the sorted two-pointer approach takes O(n²) time overall. The pointer search uses constant auxiliary space after sorting, excluding the output and any memory used by sorting.

That raised a more interesting question for me:

If both have quadratic time complexity, what should guide my choice?

I initially thought I needed an exact count of every operation. I’m now learning to separate several questions:

- How many times does the main work happen?
- What does each iteration actually do?
- How much memory is needed at one time?
- What costs come from sorting, allocation, and deduplication?

I’m not yet confident deriving all those counts independently. That’s something I want to practice, starting with small examples and building toward formulas.

My next algorithm question is 4Sum. But before rushing toward a general k-Sum solution, I want to get better at explaining the costs and tradeoffs of the approaches I already know.

One problem leads to another. Writing about the journey helps me see both what I understand and what I still need to work on.

How did you learn to compare algorithms beyond Big-O?

#Algorithms #ProblemSolving #LearningInPublic