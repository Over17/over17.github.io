---
layout: post
title:  "SIMD and Cooking"
date:   2024-12-10 18:15:00 +0300
categories: performance simd cooking
---
# Why cooking?

At Unity I was blessed to work for several years together with Andreas Fredriksson on some cool SIMD stuff. It's easy - you hear Andreas Fredriksson, you think SIMD and intrinsics. Thank you Andreas, I learned a lot from you! (passion for SIMD included)

In one of his brilliant talks, he compared SIMD operations to cutting sellerie - and this is a fantastic analogy - even more than you can imagine.

## Cutting sellerie

Imagine your knife is your CPU, and cutting is doing whatever calculation you want. Sellerie stick is your data that you want to apply the operation to.

An easy comparison is cutting one sellerie stick (scalar operation) to cutting four sellerie sticks at once (vector operation). Hmm, that's four times faster! What else can we think of here?

1. Data layout. To be able to cut four sticks at once (process vector data), you have to spent more time by picking four sticks from your pile instead of one (load and ensure your data is laid out in SoA fashion).

![pick_four_sticks](/assets/images/2024-12-10-simd-cooking-1.png)

2. Alignment. Vector data must be aligned (to 16 bytes in simple cases), and you have to align your four sellerie sticks to make sure that you cut to accurate pieces from the very beginning of the sticks.

![align_sticks](/assets/images/2024-12-10-simd-cooking-2.png)

3. Data size. Now, you can start your cutting. One cut produces four pieces, and takes the same time as if you were cutting a single stick. The longer the sticks (the more data), the more speedup you get. It doesn't make much sense to do the preparation and alignment if all you need is a single cut - in this case, scalar operations may win speed-wise.

![cut_four_at_once](/assets/images/2024-12-10-simd-cooking-3.png)

4. Processing tails. If your data length doesn't divide by vector length, then you have to process the "tail" - perform scalar operations on `Length mod VectorSize` elements/bytes. in SIMD code, it's the scalar ops right after the vector loop. Back to sellerie: if your sticks are not of the same length, then you'll have to realign the remaining after the shortest is finished or even cut the remainder one by one, in a scalar way.

![tail_left](/assets/images/2024-12-10-simd-cooking-4.png)

5. Memory bandwidth. It came to my mind once that the width of your cutting board is equivalent to your memory bandwidth; if you are cutting one sellerie stick (scalar), you are wasting most of your board width (mem bandwidth).

![mem_bandwidth_wasted](/assets/images/2024-12-10-simd-cooking-5.png)

6. Power consumption. Although the latency of your vector vs. scalar operations (how long it takes to do the cut) is more or less the same, your hand gets tired more quickly if you cut four sticks vs. one - same applies to the CPU - vector operations consume more power than scalar ones, but not 4x.

Disclaimer: I'm mentioning four sticks as if it was 4 singles packed into a 128-bit vector. Of course, it applies to wider vectors in the same way, just replace 4 with 8, 16 or whatever large your vector is.

It's a curious analogy, isn't it - also shows there are more facets than just "do four at once". Many people don't even realize they can't utilize full memory bandwidth without SIMD operations!