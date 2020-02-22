Intro
-----

This document describes a stable non-recursive adaptive merge sort named quadsort.


The quad swap
-------------

At the core of quadsort is the quad swap. Traditionally most sorting
algorithms have been designed using the binary swap where two variables
are sorted using a third temporary variable. This typically looks as
following.

    if (val[0] > val[1])
    {
        tmp[0] = val[0];
        val[0] = val[1];
        val[1] = tmp[0];
    }

Instead the quad swap sorts four variables using four temporary variables.
During the first stage the four variables are partially sorted in the four
temporary variables, in the second stage they are fully sorted back to the
original four variables.
```
            [A]       [S] ╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴                  [A]🠆
               ＼   ／   ＼                   ＼             ／
                 (?)        ╴                    ╴         ╴
               ／   ＼        ＼                   ＼   ／
            [A]       [S]╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴        (?)╴╴╴╴╴╴╴[A]🠆
                         ＼        ＼         ＼   ／
                           (?)🠆      (?)🠆      ╴
                         ／        ／         ／   ＼
            [A]       [S]╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴        (?)╴╴╴╴╴╴╴[A]🠆
               ＼   ／        ／                   ／   ＼
                 (?)        ╴                    ╴         ╴
               ／   ＼   ／                   ／             ＼
            [A]       [S]╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴╴                  [A]🠆
```


This process is visualized in the diagram above.

After the first round of sorting a single if check determines if the four
temporary variables are sorted in order, if that's the case the swap
finishes up immediately. Next it checks if the temporary variables
are sorted in reverse-order, if that's the case the swap finishes up
immediately. If both checks fail the final arrangement is known and
two checks remain to determine the final order.

This eliminates 1 wasteful comparison for in-order sequences while creating
1 additional comparison for random sequences. However, in the real world we
are rarely comparing truly random data, so in any instance where data is
more likely to be orderly than disorderly this shift in probability will give
an advantage.

There should also be a slight overall performance increase due to the
elimination of wasteful swapping. In C the basic quad swap looks as
following:

    if (val[0] > val[1])
    {
        tmp[0] = val[1];
        tmp[1] = val[0];
    }
    else
    {
        tmp[0] = val[0];
        tmp[1] = val[1];
    }

    if (val[2] > val[3])
    {
        tmp[2] = val[3];
        tmp[3] = val[2];
    }
    else
    {
        tmp[2] = val[2];
        tmp[3] = val[3];
    }

    if (tmp[1] <= tmp[2])
    {
        val[0] = tmp[0];
        val[1] = tmp[1];
        val[2] = tmp[2];
        val[3] = tmp[3];
    }
    else if (tmp[0] > tmp[3])
    {
        val[0] = tmp[2];
        val[1] = tmp[3];
        val[2] = tmp[0];
        val[3] = tmp[1];
    }
    else
    {
       if (tmp[0] <= tmp[2])
       {
           val[0] = tmp[0];
           val[1] = tmp[2];
       }
       else
       {
           val[0] = tmp[2];
           val[1] = tmp[0];
       }

       if (tmp[1] <= tmp[3])
       {
           val[2] = tmp[1];
           val[3] = tmp[3];
       }
       else
       {
           val[2] = tmp[3];
           val[3] = tmp[1];
       }
    }

In the case the array cannot be perfectly divided by 4 the tail, existing
of 1-3 elements, is sorted using the traditional swap.

quadsort
--------

In the first stage of quadsort the quad swap is used to pre-sort the
array into sorted 4-element blocks as described above.

The second stage uses an approach similar to the quad swap to detect
in-order and reverse-order arrangements, but as it's sorting blocks of 4 or
more elements the final step needs to be handled like the traditional merge
sort.

This can be visualized as following:

    main memory: AAAA BBBB CCCC DDDD

    swap memory: ABABABAB  CDCDCDCD

    main memory: ABCDABCDABCDABCD

In the first row quadswap has been used to create 4 blocks of 4 sorted
elements each. In the second row quadsort has been used to merge the blocks
into 2 blocks of 8 sorted elements each in swap memory. In the last row the
blocks are merged back to main memory and we're left with 1 block of 16
sorted elements. The following is a visualization.

![quadsort visualization](https://github.com/scandum/quadsort/blob/master/quadsort.gif)

These operations do require doubling the memory overhead for the swap space.

Skipping
--------

Another difference is that due to the increased cost of merge operations it
is beneficial to check whether the 4 blocks are in order or in reverse-order.
In the case of the 4 blocks being in order the merge operation is skipped,
as this would be pointless. This does however require an extra if check, and
for randomly sorted data this if check becomes increasingly unlikely to be
true as the block size increases. Fortunately the frequency of this if check
is quartered each loop, while the potential benefit is quadrupled each loop.

In the case only 2 out of 4 blocks are in order or in reverse-order the
comparisons in the merge itself are unnecessary and subsequently omitted.
The data still needs to be swapped but this is a less computational
intensive procedure.

This allows quadsort to sort in order and reverse-order sequences using
n + log n comparisons instead of n * log n comparisons.

Boundary checks
---------------

Another issue with the traditional merge sort is that it performs wasteful
boundary checks. This looks as following:

    if (a < a_max && b < b_max)
        if (a < b)
            [insert a]
        else
            [insert b]

To optimize this quadsort compares the last element of sequence A against
the last element of sequence B. If the last element of sequence A is smaller
than the last element of sequence B we know that the (b < b_max) if check
will always be false because sequence A will be fully merged first.

Similarly if the last element of sequence A is greater than the last element
of sequence B we know that the (a < a_max) if check will always be false.

Alignment
---------
When sorting an array of 65 elements you end up with a sorted array of 64
elements and a sorted array of 1 element in the end. Due to the ability to
skip this will result in no additional swap operation if the entire sequence
is in order. Regardless, if a program sorts in intervals it should pick an
optimal array size to do so.

A suboptimal array size is not disastrous for quadsort and it's outside of
the scope of this document to provide a solution.

Big O
-----

| Name | Best | Average | Worst | Stable | memory |
| ---- | ---- | ------- | ----- | ------ | ------ |
| quadsort | n | n log n | n log n | yes | n |

Quadsort makes n comparisons when the data is already sorted or reverse sorted.

Benchmark: quadsort vs qsort
----------------------------
The following benchmark was on WSL gcc version 7.4.0 (Ubuntu 7.4.0-1ubuntu1~18.04.1).
The source code was compiled using gcc -O3 quadsort.c. Each test was ran 100 times
and only the best run is reported.
```
         quadsort: sorted 1000000 ints in 0.091492 seconds. O(19287679) (random order)
            qsort: sorted 1000000 ints in 0.102719 seconds. O(18674792) (random order)

         quadsort: sorted 1000000 ints in 0.001882 seconds. O(  999999) (forward order)
            qsort: sorted 1000000 ints in 0.026813 seconds. O( 9884992) (forward order)

         quadsort: sorted 1000000 ints in 0.001907 seconds. O(  999999) (uniform order)
            qsort: sorted 1000000 ints in 0.026840 seconds. O( 9884992) (uniform order)

         quadsort: sorted 1000000 ints in 0.004256 seconds. O( 1416672) (reverse order)
            qsort: sorted 1000000 ints in 0.026216 seconds. O(10066432) (reverse order)

         quadsort: sorted 1000000 ints in 0.024379 seconds. O(15371381) (wave order)
            qsort: sorted 1000000 ints in 0.035232 seconds. O(14656080) (wave order)

         quadsort: sorted 1000000 ints in 0.023598 seconds. O( 6229390) (random tail)
            qsort: sorted 1000000 ints in 0.043767 seconds. O(11832535) (random tail)

         quadsort: sorted    1024 ints in 0.015332 seconds. O( 4624249) (random range)
            qsort: sorted    1024 ints in 0.027505 seconds. O( 4217883) (random range)
```
In this benchmark quadsort is compared against glibc qsort() using the same general
purpose interface and without any known unfair advantage.

     random order: array size of 1000000, filled with random numbers.
    forward order: numbers range from 1 to 1000000.
    reverse order: numbers range from 1000000 to 1.
      random tail: numbers range from 1 to 750000, last 250000 numbers are random.
     random range: array sizes range from 1 to 1024, filled with random numbers.

If you want to quickly run an independent benchmark yourself you can do so at this link. 

https://rextester.com/MBSRF85640

Benchmark: quadsort vs std::stable_sort
--------------------------------
The following benchmark was on WSL gcc version 7.4.0 (Ubuntu 7.4.0-1ubuntu1~18.04.1).
The source code was compiled using g++ -O3 quadsort.cpp. Each test was ran 100 times
and only the best run is reported.
```
         quadsort: sorted 1000000 ints in 0.070530 seconds. O(       0) (random order)
            qsort: sorted 1000000 ints in 0.073672 seconds. O(       0) (random order)

         quadsort: sorted 1000000 ints in 0.000758 seconds. O(       0) (forward order)
            qsort: sorted 1000000 ints in 0.010011 seconds. O(       0) (forward order)

         quadsort: sorted 1000000 ints in 0.000765 seconds. O(       0) (uniform order)
            qsort: sorted 1000000 ints in 0.010010 seconds. O(       0) (uniform order)

         quadsort: sorted 1000000 ints in 0.002657 seconds. O(       0) (reverse order)
            qsort: sorted 1000000 ints in 0.010340 seconds. O(       0) (reverse order)

         quadsort: sorted 1000000 ints in 0.010093 seconds. O(       0) (wave order)
            qsort: sorted 1000000 ints in 0.013558 seconds. O(       0) (wave order)

         quadsort: sorted 1000000 ints in 0.017181 seconds. O(       0) (random tail)
            qsort: sorted 1000000 ints in 0.024212 seconds. O(       0) (random tail)

         quadsort: sorted    1024 ints in 0.010446 seconds. O(       0) (random range)
            qsort: sorted    1024 ints in 0.016880 seconds. O(       0) (random range)
```
In this benchmark quadsort is compared against the c++ std::stable_sort without any
known advantages.

If you want to quickly run an independent benchmark yourself you can do so at this link.

https://rextester.com/WCGH74486

![quadsort benchmark](https://github.com/scandum/quadsort/blob/master/benchmark.gif)
