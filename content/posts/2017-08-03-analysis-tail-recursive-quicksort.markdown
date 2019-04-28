---
layout: single
title:  "Analysis of Tail Recursive Quicksort"
date:   2017-08-03
categories: [analysis, algorithms, notes]
published: true
math: true
psuedocode: true
author: Vaibhav Yenamandra
---

This is a problem that appears as a practice question in Introduction to Algorithms 2rd edition, 3rd Edition by Charles E. Leiserson, Clifford Stein, Ronald Rivest, and Thomas H. Cormen. It's also an interesting exercise / excuse to dig far too deep into quicksort!

## Pseudocode
```python
def tailRecursiveQuickSort(A, startIdx, endIdx):
    while startIdx < endIdx:
        pivotIdx = PARTITION(A, startIdx, endIdx)
        tailRecursiveQuickSort(A, startIdx, pivotIdx - 1)
        startIdx = pivotIdx + 1
```

## Analysis
In the first iteration of `tailRecursiveQuickSort`, the left subarray gets processed i.e. the first call is $\textbf{tailRecursiveQuickSort}(A, 1, q-1)$

After this the p value changes to $p\_{2} = q + 1$ which is the p-value for the second iteration which makes the call $\textbf{tailRecursiveQuickSort}(A, p\_{2}, q\_{2}-1)$ where $q\_{2} = \textbf{PARTITION}(A, p\_{2}, r)$.

This can be generalized to the $i^{th}$ iteration as follows:

* Function call made: $\textbf{tailRecursiveQuickSort}(A, q\_{i-1} + 1, q\_{i} - 1),$
* $p\_{i} = q\_{i-1} + 1,$
* $q\_{i} = \textbf{PARTITION}(A, q\_{i-1} + 1, r)$

Observe that: $q\_{i-1} + 1 \leq q\_{i} \leq r$. Thus during each iteration, $q$ is guaranteed to be incremented by at least $1$.

Additionally, by the $i^{th}$ iteration, the following recursive calls have been excuted on disjoint subarrays of $A$

* $\textbf{tailRecursiveQuickSort}(A, p\_{1}, q\_{1} - 1)$ where $p\_{1} = 1, q\_{1} = q$
* $\textbf{tailRecursiveQuickSort}(A, q\_{1} + 1, q\_{2} - 1)$
*    $\vdots$
* $\textbf{tailRecursiveQuickSort}(A, q\_{i-1} + 1, q\_{i} - 1)$

Since each of these calls acts on a disjoint sub-array with intermediate calls to `PARTIION` which guarantees that each $q\_{i}$ is in it's correct place; these calls can be group together to be seen as a single call to `tailRecursiveQuickSort` which performs the divide.

Therefore calls from $(b)$ through $(c)$ above can be grouped into: $\textbf{tailRecursiveQuickSort}(A, q+1, q\_{i}-1)$ at the end of the $i^{th}$ iteration.

Since $q_{i}$ is guaranteed to increase every by atleast $1$ every iteration, so is $p\_{i} = q\_{i} + 1$. This implies that the while loop must terminate for some value of $p\_{i} > r$, which is guaranteed due to its strictly increasing behavior. Therefore the sequence $q\_{i}$ must eventually cross $r$ for some $i = k$ such that $q\_{k} \geq r$.

So the sequence of calls from the second iteration onwards can be rewritten as the following net result: $\textbf{tailRecursiveQuickSort}(A, q+1, r)$. Therefore the original `tailRecursiveQuickSort` function can be rewritten as follows (after fully unrolling the while loop)

```python
def tailRecursiveQuickSort(A, startIdx, endIdx):
    pivotIdx = PARTITION(A, startIdx,
    tailRecursiveQuickSort(A, startIdx, pivotIdx - 1)
    tailRecursiveQuickSort(A, endIdx, pivotIdx + 1)
```

## Conculsion

The last bit of pseudocode we made by transforming the original code is exactly the same as `QUICKSORT` therefore, `tailRecursiveQuickSort` and `QUICKSORT` are functionally equivalent, proving that `tailRecursiveQuickSort` is able to sort any input sequence, given the correctness of the `PARTITION` routine.

Alternatively, one can also say that the correctness, convergence of `QUICKSORT` guarantees the correctness of this tail recursive form.
