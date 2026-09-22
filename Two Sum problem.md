# What Two Sum Taught Me About Turning Search Into Lookup

Today I revisited **Two Sum**. Although it looks like a simple problem, the reasoning process taught me something more reusable than the final solution: how to separate unavoidable work from reducible work and transform an open-ended search into an exact lookup.

## The problem

Given an integer array `nums` and an integer `target`, return the indices of two numbers whose sum equals `target`.

You may assume that exactly one valid answer exists. You cannot use the same array element twice, and you may return the two indices in any order.

For example:

```text
Input: nums = [2, 7, 11, 15], target = 9
Output: [0, 1]
```

The target complexity is:

- Expected time: `O(n)`
- Extra space: `O(n)`

## Starting with brute force

I began with the most direct approach.

I could select one index `i` and then examine every index after it, from `i + 1` to `n - 1`. For each pair, I would calculate the sum and compare it with the target.

This examines every valid pair exactly once. The number of possible pairs is:

```text
n(n - 1) / 2
```

Therefore, the brute-force approach takes `O(n²)` time and `O(1)` extra space.

At this point, I knew the approach worked. The next question was not simply, “How can I remove the nested loops?” Instead, I wanted to understand what the algorithm was actually paying for.

## What work can actually be reduced?

Any solution still needs to identify two values, perform a calculation, and verify that the result satisfies the target.

Whether I use addition:

```text
a + b == target
```

or subtraction:

```text
b == target - a
```

some calculation and comparison still need to happen.

The main opportunity for optimization was therefore not eliminating arithmetic or comparison. It was reducing the work required to find the right pair.

In the brute-force approach, after selecting one element, I repeatedly search the remaining array for another element that completes the target.

That repeated search is the reducible work.

## Turning pair search into exact lookup

Once I select one value `x`, the second value is no longer unknown.

It must be:

```text
target - x
```

This changed the way I viewed the problem.

My question was no longer:

> Which pair should I try next?

It became:

> I already know the exact value I need. How can I quickly find its index?

This reformulation turned a combination-search problem into a lookup problem.

A hash table became useful because it could map each value to its index and support expected constant-time lookup.

## My first idea: a two-pass approach

My initial hash-table design used two traversals.

First, I would store every value and its index:

```text
value → index
```

Then I would traverse the array again. For each current value, I would calculate its complement and check whether that complement existed in the table.

The process was:

```text
Build the complete hash table
→ select the first value
→ calculate the required complement
→ look up its index
```

This reduced the expected running time to `O(n)` by replacing each repeated linear search with a hash-table lookup.

However, this design introduced another question: what happens when duplicate values appear?

## Investigating duplicate values

Consider:

```text
nums = [3, 3]
target = 6
```

The two values are equal, but their indices are different. The solution must allow equal values while preventing the same array element from being used twice.

Another useful example is:

```text
nums = [3, 2, 2, 2]
target = 4
```

If the complete array is loaded into a table that stores only one index for each value, later occurrences of `2` overwrite earlier ones.

A two-pass approach can still return one valid pair, provided that it explicitly verifies:

```text
stored_index != current_index
```

However, this made me notice that the complete table contained information from both earlier and later positions. The algorithm therefore needed an additional check to prevent an element from matching itself.

## Refining the idea into one pass

The one-pass approach organizes the stored information differently.

Instead of preparing information about the complete array, the hash table represents only the elements that have already been visited.

For each current value:

1. Calculate the required complement.
2. Check whether that complement exists in the table.
3. If it exists, return its stored index and the current index.
4. Otherwise, store the current value and its index.

In pseudocode:

```text
create an empty hash table named seen

for each index i and value x:
    needed = target - x

    if needed exists in seen:
        return [seen[needed], i]

    store x → i in seen
```

The order is important:

> Check before inserting.

Before processing the current index, the table contains values from earlier indices only. Therefore, any matching index retrieved from the table must be different from the current index.

This also handles duplicate values naturally.

For example:

```text
nums = [3, 3]
target = 6
```

The first `3` is stored. When the second `3` is processed, the required complement is already present at an earlier index.

## Why the pair cannot be missed

Suppose the valid pair appears at indices:

```text
p < q
```

When the traversal reaches index `q`, the value at index `p` has already been processed and stored.

Because the two values form the target:

```text
nums[p] = target - nums[q]
```

the complement lookup succeeds when `q` is processed.

The hash table is therefore not merely a collection of values. It represents the history of the processed portion of the array.

## One pass versus two passes

Both approaches use stored information to eliminate repeated linear searches, but they prepare and reuse that information differently.

### Two-pass processing

- Builds the complete lookup table first.
- Searches the table during a second traversal.
- Always pays the cost of constructing the complete table.
- Must explicitly prevent the current index from matching itself.
- Duplicate values may overwrite earlier indices.

### One-pass processing

- Builds the lookup table incrementally.
- Searches previously processed values while traversing.
- May return before the complete array has been stored.
- May use less space in practice when the answer appears early.
- Prevents self-matching through the order of operations.

Both methods have:

- Expected time: `O(n)`
- Worst-case extra space: `O(n)`

The one-pass version does not improve the asymptotic complexity over the two-pass version. Its advantage is that the stored state matches the exact information needed at each moment.

## My main takeaway

My optimization did not begin with choosing a hash table.

It began by examining the brute-force method and separating two kinds of work:

- Work that could not be completely avoided, such as calculation and verification
- Work that could be reduced, such as repeatedly searching for possible partners

Once I fixed one value, the other value became known:

```text
needed = target - current
```

That transformed the problem:

```text
enumerate every possible pair
→ find a pair more intelligently
→ fix one value
→ derive the exact value needed
→ look up that value directly
→ store only the history needed for future decisions
```

The most reusable lesson for me is:

> Do not optimize an algorithm as one indivisible block. Decompose its work, identify what is repeated, use known conditions to narrow the search target, and then choose a data structure that supports the resulting operation efficiently.

#Algorithms #DataStructures #ProblemSolving #SoftwareEngineering #LearningInPublic
