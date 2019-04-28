---
layout: post
title:  "Bitwise AND of numbers in a range"
date:   2017-07-08
categories: [interviews, algorithms, notes, bit-hacks]
math: true
published: true
author: Vaibhav Yenamandra
---
# Bitwise AND of numbers in a range

Given a range $[m, n]$ where $0 \leq m \leq n \leq 2^{31} - 1$, return the bitwise AND of all numbers in this range, inclusive.

## Discussion
The problem is trivial to solve with a loop but there's an interesting idea lurking around the corner, so we will avoid doing this via a straightforward loop.

Take a number, $m = 14$. Lets examine what happens in binary as we move forward from 14

```
(14) 0000 0000 0000 0000 0000 0000 0000 1110
(15) 0000 0000 0000 0000 0000 0000 0000 1111
(16) 0000 0000 0000 0000 0000 0000 0001 0000 <-
(17) 0000 0000 0000 0000 0000 0000 0001 0001
(18) 0000 0000 0000 0000 0000 0000 0001 0010
(19) 0000 0000 0000 0000 0000 0000 0001 0011
(20) 0000 0000 0000 0000 0000 0000 0001 0100
(21) 0000 0000 0000 0000 0000 0000 0001 0101
(22) 0000 0000 0000 0000 0000 0000 0001 0110
(23) 0000 0000 0000 0000 0000 0000 0001 0111
(24) 0000 0000 0000 0000 0000 0000 0001 1000
(25) 0000 0000 0000 0000 0000 0000 0001 1001
(26) 0000 0000 0000 0000 0000 0000 0001 1010
(27) 0000 0000 0000 0000 0000 0000 0001 1011
(28) 0000 0000 0000 0000 0000 0000 0001 1100
(29) 0000 0000 0000 0000 0000 0000 0001 1101
```
Looking from the right hand side, here's some properties to keep in mind:
1. $n$ is likely to have more significant bits set than $m$. So the AND of all contiguous elements in this range can only be as large as $m$
2. If $m < 2^k < n$, then the result is always a $0$, since the power of $2$'s leftmost set bit will be further right than $m$'s leftmost set bit
3. If $n > m + -1$, the rightmost bit for this range is $0$

Property $3$ can be used with a little more insight:

Notice what happens if we replace the rightmost bit from the sequence with $X$ for don't care.
```
(14) 0000 0000 0000 0000 0000 0000 0000 111X
(15) 0000 0000 0000 0000 0000 0000 0000 111X
(16) 0000 0000 0000 0000 0000 0000 0001 000X
(17) 0000 0000 0000 0000 0000 0000 0001 000X
(18) 0000 0000 0000 0000 0000 0000 0001 001X
(19) 0000 0000 0000 0000 0000 0000 0001 001X
(20) 0000 0000 0000 0000 0000 0000 0001 010X
(21) 0000 0000 0000 0000 0000 0000 0001 010X
(22) 0000 0000 0000 0000 0000 0000 0001 011X
(23) 0000 0000 0000 0000 0000 0000 0001 011X
(24) 0000 0000 0000 0000 0000 0000 0001 100X
(25) 0000 0000 0000 0000 0000 0000 0001 100X
(26) 0000 0000 0000 0000 0000 0000 0001 101X
(27) 0000 0000 0000 0000 0000 0000 0001 101X
(28) 0000 0000 0000 0000 0000 0000 0001 110X
(29) 0000 0000 0000 0000 0000 0000 0001 110X
```
Ignoring the $X$, alternating numbers now form an increasing sequence, how long can we do this? We ignored the right most bit because this changed constantly, so we'll stop ignoring it when it stops changing. When it does stop changing, no bit to it's left will change because contiguous binary sequences bits change left to right, one at a time.

So AND-ing the entire sequence is the same as find the common left bits of all numbers in that range, and doesn't in fact depend on all the numbers, but just the endpoints.

This is what the method looks like:
```
(17) 0000 0000 0000 0000 0000 0000 0001 0001
(26) 0000 0000 0000 0000 0000 0000 0001 1010
---
(08) 0000 0000 0000 0000 0000 0000 0001 000X
(13) 0000 0000 0000 0000 0000 0000 0001 101X
---
(04) 0000 0000 0000 0000 0000 0000 0001 00XX
(06) 0000 0000 0000 0000 0000 0000 0001 10XX
---
(02) 0000 0000 0000 0000 0000 0000 0001 0XXX
(03) 0000 0000 0000 0000 0000 0000 0001 1XXX
---
(01) 0000 0000 0000 0000 0000 0000 0001 XXXX
(01) 0000 0000 0000 0000 0000 0000 0001 XXXX
```

It doesn't necessarily have to be $1$, but it has to end when right shifting the endpoints causes them to become equal.

## Code
This here is an iterative `C++` implementation of the idea above
```cpp
class Solution
{
    public:
    int rangeBitwiseAnd(int m, int n)
    {
        int rightMask = 1;
        while(n != m)
        {
            m >>= 1;
            n >>= 1;
            rightMask <<= 1;
        }

        // Bottom k-bits are cleared
        return m * rightMask;
    }
};
```
