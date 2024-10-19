


# Investigating Multiplication on the Gameboy Advance
The Gameboy Advance has a pretty neat CPU - the ARM7TDMI. And by neat, I mean a chaotic and
sadistic bundle of questionable design decisions. Seriously, they decided that the program counter should
be a _general purpose register_. Why??? Insert simile here. I'm not even joking, you can use the program
counter as the output to, say, an XOR instruction. Or an AND instruction.

Or a multiply instruction.

The ARM7TDMI's multiplication instruction has a pretty interesting side effect. Here the manual says that
after a multiplication instruction executes, the carry and overflow flags are `UNPREDICTABLE`.

![An image of the ARM7TDMI manual explaining that the carry and overflow flags are `UNPREDICTABLE` after a multiply instruction.](/manual.png)

As if anything else in this god forsaken CPU was predictable. What this means is that software cannot and
should not rely on the value of the carry flag after multiplication executes. It can be set to anything. Any
value. 0, 1, a horse, whatever. This has been a source of memes in the emudev community for a bit -
people would frequently joke about how the implementation of the carry flag may as well be `cpu.flags.c =
rand() & 1;`. And they had a point - the carry flag seemed to defy all patterns; nobody understood why it
behaves the way it does. But the one thing we did know, was that the carry flag seemed to be
_deterministic_. That is, under the same set of inputs to a multiply instruction, the flag would be set to the
same value. This was big news, because it meant that understanding the carry flag could give us key
insight into how this CPU performs multiplication.

And just to get this out of the way, the carry flag's behavior after multiplication isn't an important detail to
emulate at all. Software doesn't rely on it. And if software _did_ rely on it, then screw that software, those
developers got what was coming to them. But the carry flag is a meme, and it's a really tough puzzle, and
that was motivation enough for me to give it a go. Little did I know it'd take _3 years_ of on and off work.

## Standard Algorithm
What's the simplest, most basic multiplication algorithm you can think of to multiply a <span style="color:#3a7dc9"> **multiplier**</span> with a <span style="color:#DC6A76"> **multiplicand**</span>? One really easy way is to
leverage the distributive property of multiplication like so:

$$
\color{#3a7dc9}123\color{#4A4358}  \cdot  \color{#DC6A76}4 \color{#4A4358}=
\color{#3a7dc9}{100 \color{#4A4358} \cdot  \color{#DC6A76}4} \color{#4A4358} + \color{#3a7dc9}{20 \color{#4A4358} \cdot  \color{#DC6A76}4} \color{#4A4358} + \color{#3a7dc9}{3 \color{#4A4358} \cdot  \color{#DC6A76}4}
$$

There's two steps here - first compute the addends, then sum them. This
is the basic two-step process you'll find in lots of multiplication algorithms - most of them simply differ in
how they compute the addends, or how they add the addends together. We can generalize this algorithm
to binary pretty easily too:
$$
\color{#3a7dc9}1101 \color{#4A4358} \cdot \color{#DC6A76}11\color{#4A4358} =
\color{#3a7dc9}{1000 \color{#4A4358} \cdot\color{#DC6A76} 11} \color{#4A4358} + \color{#3a7dc9}{100 \color{#4A4358} \cdot \color{#DC6A76} 11} \color{#4A4358} + \color{#3a7dc9}{0 \color{#4A4358} \cdot\color{#DC6A76} 11} \color{#4A4358} + \color{#3a7dc9}{1 \color{#4A4358} \cdot \color{#DC6A76}11}
$$
The convenient thing about binary is that it's all ones and zeros, meaning the addends are only ever `0`, or
the <span style="color:#DC6A76"> **multiplicand**</span> left shifted by some factor. This makes the addends easy to compute, and means that for
an `N-bit` number, we need to produce `N` different addends, and add them all up to get the result.
That's slow. We can do better.

## Modified Booth's Algorithm
The main slowness of the Standard Algorithm is that it requires you to add a  _lot_ of numbers together.
Modified Booth's algorithm is an improvement on the Standard Algorithm that cuts the number of addends in two. Let's start with the standard definition for multiplication, written as a summation. Note that `m[i]` is defined as the bit at index `i` of `m` when `0 <= i < n`.
$$
\begin{aligned}
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=0}^{n-1} (2^i \cdot \color{#3a7dc9}{m[i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) \cr
\end{aligned}
$$

Now we apply the following transformations. Yes I know this looks scary, you could skip to the final equation if you want.
$$
\begin{align}
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=0}^{n-1} (2^i \cdot \color{#3a7dc9}{m[i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} )\cr\cr
   &\quad\text{Separate the summation into even and odd elements:}\cr \cr
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=0}^{\frac{n}{2}-1} (2^{2i} \cdot \color{#3a7dc9}{m[2i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + \sum_{i=0}^{\frac{n}{2}-1}  (2^{2i + 1} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) \cr
\cr&     \quad\text{                        Split the second summation into two more summations:}\cr \cr
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=0}^{\frac{n}{2}-1} (2^{2i} \cdot \color{#3a7dc9}{m[2i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + (2 - 1) \cdot \sum_{i=0}^{\frac{n}{2}-1}  (2^{2i + 1} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) \cr 
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=0}^{\frac{n}{2}-1} (2^{2i} \cdot \color{#3a7dc9}{m[2i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + 2 \sum_{i=0}^{\frac{n}{2}-1}  (2^{2i + 1} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) - \sum_{i=0}^{\frac{n}{2}-1}  (2^{2i + 1} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) \cr
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=0}^{\frac{n}{2}-1} (2^{2i} \cdot \color{#3a7dc9}{m[2i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + \sum_{i=0}^{\frac{n}{2}-1}  (2^{2i + 2} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) - \sum_{i=0}^{\frac{n}{2}-1}  (2^{2i + 1} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) \cr \cr&     \quad\text{                        Pull out a single element from each summation, one at a time:}\cr \cr
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=1}^{\frac{n}{2}-1} (2^{2i} \cdot \color{#3a7dc9}{m[2i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + \sum_{i=0}^{\frac{n}{2}-1}  (2^{2i + 2} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) - \sum_{i=0}^{\frac{n}{2}-1}  (2^{2i + 1} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + (\color{#3a7dc9}{m[0]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358})\cr
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=1}^{\frac{n}{2}-1} (2^{2i} \cdot \color{#3a7dc9}{m[2i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + \sum_{i=0}^{\frac{n}{2}-2}  (2^{2i + 2} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) - \sum_{i=0}^{\frac{n}{2}-1}  (2^{2i + 1} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + (\color{#3a7dc9}{m[0]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} + 2^{n} \cdot \color{#3a7dc9}{m[n - 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} )\cr
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=1}^{\frac{n}{2}-1} (2^{2i} \cdot \color{#3a7dc9}{m[2i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + \sum_{i=0}^{\frac{n}{2}-2}  (2^{2i + 2} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) - \sum_{i=1}^{\frac{n}{2}-1}  (2^{2i + 1} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + (\color{#3a7dc9}{m[0]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} + 2^{n} \cdot \color{#3a7dc9}{m[n - 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}  + 2 \cdot \color{#3a7dc9}{m[1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} )\cr
\cr&     \quad\text{                        Manipulate the range of the second summation to match the ranges of the other two:}\cr \cr
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=1}^{\frac{n}{2}-1} (2^{2i} \cdot \color{#3a7dc9}{m[2i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + \sum_{i=1}^{\frac{n}{2}-1}  (2^{2i} \cdot \color{#3a7dc9}{m[2i - 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) - \sum_{i=1}^{\frac{n}{2}-1}  (2^{2i + 1} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + (\color{#3a7dc9}{m[0]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} + 2^{n} \cdot \color{#3a7dc9}{m[n - 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}  + 2 \cdot \color{#3a7dc9}{m[1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} )\cr
\cr&     \quad\text{                        So that we can combine the summations now:}\cr \cr
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=1}^{\frac{n}{2}-1} (2^{2i} \cdot \color{#3a7dc9}{m[2i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} + 2^{2i} \cdot \color{#3a7dc9}{m[2i - 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} - 2^{2i + 1} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + (\color{#3a7dc9}{m[0]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} + 2^{n} \cdot \color{#3a7dc9}{m[n - 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}  + 2 \cdot \color{#3a7dc9}{m[1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} )\cr
\cr&     \quad\text{                       Some tidywork...}\cr \cr
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=1}^{\frac{n}{2}-1} (2^{2i} \cdot \color{#3a7dc9}{m[2i]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} + 2^{2i} \cdot \color{#3a7dc9}{m[2i - 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} - 2 \cdot 2^{2i} \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}) + (\color{#3a7dc9}{m[0]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} + 2^{n} \cdot \color{#3a7dc9}{m[n - 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}  + 2 \cdot \color{#3a7dc9}{m[1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} )\cr
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=1}^{\frac{n}{2}-1} ((2^{2i} \cdot \color{#DC6A76} \alpha \color{#4A4358} )\cdot(\color{#3a7dc9}{m[2i]}\color{#4A4358} + \color{#3a7dc9}{m[2i - 1]}\color{#4A4358} - 2 \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358} )) + (\color{#3a7dc9}{m[0]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} + 2^{n} \cdot \color{#3a7dc9}{m[n - 1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358}  + 2 \cdot \color{#3a7dc9}{m[1]}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} )\cr
\end{align}
$$

Whew. Did you get all of that? Why did we do all this? Well, note this part of the summation:

$$
\begin{aligned}
(\color{#3a7dc9}{m[2i]}\color{#4A4358} + \color{#3a7dc9}{m[2i - 1]}\color{#4A4358} - 2 \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358})
\end{aligned}
$$

This is always one of `(-2, -1, 0, 1, 2)`.


Multiplication by those five numbers is easy to calculate in hardware (well, negation is tricky - the algorithm implements negation as bitwise inversion, with an additional 1 added at a later stage. More information about this is given later).

Note also that if we define:

$$
\color{#3a7dc9}{m[-1]}\color{#4A4358} = 0
$$

and

$$
\begin{aligned}
\text{For}&\text{ unsigned multiplication:}\cr\cr
&x \geq n, \color{#3a7dc9}{m[x]}\color{#4A4358} = 0\cr\cr
\text{For}&\text{ signed multiplication:}\cr\cr
&x \geq n, \color{#3a7dc9}{m[x]}\color{#4A4358} = \color{#3a7dc9}m[n-1]\cr\cr
\end{aligned}
$$


Then the leftover three terms outside the summation can be absorbed into the summation, by expanding the summations range by one on both boundaries. And so we have:

$$
\begin{aligned}
\color{#3a7dc9}{m}\color{#4A4358} \cdot \color{#DC6A76} \alpha \color{#4A4358} &= \sum_{i=0}^{\frac{n}{2}} ((2^{2i} \cdot \color{#DC6A76} \alpha \color{#4A4358}) \cdot (\color{#3a7dc9}{m[2i]}\color{#4A4358} + \color{#3a7dc9}{m[2i - 1]}\color{#4A4358} - 2 \cdot \color{#3a7dc9}{m[2i + 1]}\color{#4A4358}))\cr
\end{aligned}
$$

Before all this mathematical chaos, we used to have `n` addends. Now we have just over half that many addends. We can model the generation of an addend sans the left shift using the following C code:

```c
// represents a 3-bit chunk that is used to determine an addend's value
typedef u8 BoothChunk;

struct BoothRecodingOutput {
    u64  recoded_output;
    bool carry;
};

// booth_chunk is a 3-bit number representing bits [2i - 1 .. 2i + 1]
// of the multiplier
struct BoothRecodingOutput booth_recode(u64 input, BoothChunk booth_chunk) {
    switch (booth_chunk) {
        case 0: return (struct BoothRecodingOutput) {            0, 0 };
        case 1: return (struct BoothRecodingOutput) {        input, 0 };
        case 2: return (struct BoothRecodingOutput) {        input, 0 };
        case 3: return (struct BoothRecodingOutput) {    2 * input, 0 };
        case 4: return (struct BoothRecodingOutput) { ~(2 * input), 1 };
        case 5: return (struct BoothRecodingOutput) {       ~input, 1 };
        case 6: return (struct BoothRecodingOutput) {       ~input, 1 };
        case 7: return (struct BoothRecodingOutput) {            0, 0 };
    }
}
```

## How to Add Stuff ✨ Efficiently ✨
Now that we have the addends, it's time to actually add them up to produce the result. However, using a
conventional full adder, the ARM7TDMI is only fast enough to add two numbers per cycle. Which means,
you gotta spend 16 cycles to add all 17 addends, which is uselessly slow. The reason full adders are so
slow is because of the carry propagation - bit `N` of the result can't be determined till bit `N - 1` is
determined. Can we eliminate this issue?

Introducing... *drum roll*... carry save adders (CSAs)! These are genius - instead of outputting a single `N-bit` result, CSAs output one `N-bit` result without carry propagation, and one `N-bit` list of carries computed from each bit. At first this seems kind of silly - are CSAs really adding two `N-bit` operands and
producing two `N-bit` results? What's the point? The point is that you can actually fit in an extra operand,
and turn three `N-bit` operands into two `N-bit` results. Like so:
```c
struct CSAOutput {
    u64 output;
    u64 carry;
};

struct CSAOutput perform_csa(u64 a, u64 b, u64 c) {
    // Bit i in result should be set if there is either 1 set bit in 
    // src1/src2/src3 at index i, or 3 set bits in src1/src2/src3 at
    // index i. Similarly, bit i in carries should be set if there's
    // 2 or 3 set bits in src1/src2/src3 at index i. See if you can
    // convince yourself why this is correct.

    u64 output = a ^ b ^ c;
    u64 carry  = (a & b) | (b & c) | (c & a);
    return (struct CSAOutput) { output, carry };
}
```
So you can chain a bunch of CSAs to get yourself down to two addends, and then you can shove the two
`N-bit` results into a regular adder, like so:
```c
u64 add_csa_results(u64 result, u64 carries) {
    // Exercise for the reader: Why do you suppose we multiply
    // carries by 2? Think about how a full adder is implemented,
    // and what the variable "carries" in the csa function actually
    // represents. The answer is given after this code block.

    return result + carries * 2;
}
```
The reason we multiply `carries` by two is because, if we think about how a full adder works, the carry out
from bit `i` is added to bits `i + 1` of the addends. So, bit `i` of carries has double the "weight" of bit `i` of
result. This is a **very** important detail that will come in handy later, so do make sure you understand
this.
## Parallelism
Until now, we've mostly treated "generate the addends" and "add the addends" as two separate, entirely
discrete steps of the algorithm. But, turns out, we can do both of these steps _at the same time_. We
know we can only add 4 addends per cycle, so what if we generate 4 addends per cycle, and compress
them using four CSAs to generate only two addends? So, we pipe 4 CSAs into each other, allowing us to process 6 `N`-bit inputs into two `N + 8` bit outputs. Each CSA widens the
output by `2`, because the carries that the CSA outputs has twice the weight of the sum, meaning the
carries needs to be represented using an `N + 1` bit number. Each cycle, we can generate 4 addends,
feed them into 4 of the 6 outputs of this CSA array, and then when we have our two results, feed those
results back to the very top of the CSA array for the next cycle. We can initialize those two inputs to the
CSA array with `0`s. Or, if we want to be clever, we can implement multiply accumulate by initializing one
of those two inputs with the accumulate value, and get multiply accumulate for free. This trick is what the
ARM7TDMI employs to do multiply accumulate. (This is a moot point, because the CPU is stupid and can only read two register values at a time per cycle. So, using an accumulate causes the CPU to take
an extra cycle _anyway_).


## Early Termination
The ARM7TDMI does something really clever here. In our current model of the algorithm, there are 4
cycles of CSA compression, where each cycle `i` processes bits `8 * i` to `8 * i + 7` of the <span style="color:#3a7dc9"> **multiplier**</span>.
(explain this in previous section). The observation is that if the remaining upper bits of the <span style="color:#3a7dc9"> **multiplier**</span> are all
zeros, then, we can skip that cycle, since the addends produced will be all zeros, which cannot possibly
affect the values of the partial result + partial carry. We can do the same trick if the remaining upper bits
are all ones (assuming we are performing a signed multiplication), as those also produce addends that
are all zeros.
## Putting it all together

Here's a rough diagram, provided by Steve Furber in his book, Arm System-On-Chip Architecture:

![An image of the high level overview of the multiplier's organization, provided by Steve Furber in his book, Arm System-On-Chip Architecture](/booth.png)

Partial Sum / Partial Carry contain the results obtained by the CSAs, and are rotated right by 8 on each cycle. Rm is recoded using booth's algorithm to produce the addends for the CSA array.

Ok, but remember when I said (make sure I said this) that there will be an elegant way to handle booth's negation of the addends? The way the algorithm gets around this is kind of genius. Remember how the carry output of a CSA has to be left shifted by 1? Well, this left-shift creates a zero in the LSB of the carry output of the CSA, so why don't we just put the carry in that bit? Like so:
<a name="perform_csa_array"></a>

```c
struct CSAOutput perform_csa_array(u64 partial_sum, u64 partial_carry, 
                                   BoothRecodingOutput addends) {
    struct CSAOutput csa_output = { partial_sum, partial_carry };
    struct CSAOutput final_csa_output = { 0, 0 };

    for (int i = 0; i < 4; i++) {
        csa_output.output &= 0x1FFFFFFFFULL;
        csa_output.carry  &= 0x1FFFFFFFFULL;

        struct CSAOutput result = perform_csa(csa_output.output, addends.m[i].recoded_output & 0x1FFFFFFFFULL, csa_output.carry);

        // Inject the carry caused by booth recoding
        result.carry <<= 1;
        result.carry |= addends.m[i].carry;

        // Take the bottom two bits and inject them into the final output.
        // The value of the bottom two bits will not be changed by future
        // addends, because those addends must be at least 4 times as big
        // as the current addend. By directly injecting these two bits, the
        // hardware saves some space on the chip.
        final_csa_output.output |= (result.output & 3) << (2 * i);
        final_csa_output.carry  |= (result.carry  & 3) << (2 * i);
        
        // The next CSA will only operate on the upper bits - as explained
        // in the previous comment.
        result.output >>= 2;
        result.carry  >>= 2;

        csa_output = result;
    }

    final_csa_output.output |= csa_output.output << 8;
    final_csa_output.carry  |= csa_output.carry  << 8;

    return final_csa_output;
}
``` 

(Yes, this insanity is indeed done by the actual CPU.)

## Fatal Contradiction
So I lied to you all. There's a small, but very meaningful difference between the algorithm I described and
the ARM7TDMI's algorithm. Let's consider the following multiplication:
$$
\color{#3a7dc9}0x000000FF\color{#4A4358}  \cdot  \color{#DC6A76}0x00000001 \color{#4A4358}
$$
How many cycles should this take? 1, right? Because the upper 24 bits of the <span style="color:#3a7dc9"> **multiplier**</span> are zeros, then the
second, third, and fourth cycles of addends will all be zeros... right?
Right?
Well, that's how long it takes the ARM7TDMI to do it. So what's the issue? Turns out, the second cycle of
the algorithm does have a single non-zero addend:

$$
\begin{aligned}
&\text{Chunk #1: }\color{#3a7dc9}\text{0b001}\cr
&\text{Chunk #2: }\color{#3a7dc9}\text{0b000}\cr
&\text{Chunk #3: }\color{#3a7dc9}\text{0b000}\cr
&\text{Chunk #4: }\color{#3a7dc9}\text{0b000}\cr
\end{aligned}
$$

Because the LSB of Chunk #1 uses the MSB of Chunk #3 of Cycle #1, our algorithm would be forced to
take 2 cycles of CSAs. And yet, on the ARM7TDMI, this multiplication would terminate early, after only 1 cycle of CSAs. And there doesn't seem to be a good way around this. And so I sat there thinking of
workarounds.

**Proposed solution #1:** What if the ARM7TDMI actually processes 5 chunks per cycle?

**Rebuttal:** If that were the case, then the algorithm would be able to process 9 bits on the first cycle,
which it cannot do.


**Proposed solution #2:** Ok but what if the ARM7TDMI has some way of processing chunk #1 of cycle
`n` on cycle `n - 1`, but only if cycle `n - 1` is the last cycle of the algorithm?

**Rebuttal:** Sure, maybe this is possible, but it feels like any solution that would allow the algorithm to do this would also
be capable of allowing the CPU to do 5 chunks per cycle.


**Proposed solution #3:** Fine, what if the CPU actually leverages the power of Cthulu and evil warlock
magic to pull this off?

**Rebuttal:** Yeah, that's assigning too much credit to this god forsaken bundle of wires that somehow
obtained the title of "CPU".


I was kind of out of ideas. I was pretty much ready to give up - my current algorithm was nowhere near
explaining the behavior of the CPU carry flag. And so I took a break, only looking at this problem every
once in a while.

## Descent into Madness
Congrats for getting this far, now comes the tricky stuff. I require anyone who wants to continue reading to
put on [this music](https://www.google.com/url?sa=t&source=web&rct=j&opi=89978449&url=https://www.youtube.com/watch?v=ntgaStqpmjQ&ved=2ahUKEwj03pfgvs2IAxWjJTQIHTp_EToQtwJ6BAgIEAI&usg=AOvVaw1qRD2jAXNcY-9YA6Uhb9Ig) in the background, as it most accurately models the trek into insanity we are about to endure.

So fast forward about a year, I'm out for a walk and I decide to give this problem a thought again. And so I considered something that, at the outset, sounds really, really stupid.

"What if we left shifted the  <span style="color:#3a7dc9"> **multiplier**</span> by 1 at the beginning of the algorithm?"

I mean, it's kind of dumb, right? The entire issue is that the <span style="color:#3a7dc9"> **multiplier**</span> is _too big_. Left shifting it would only exacerbate this issue. Congrats, we went from being able to process 7 bits on the first cycle to 6.

But pay attention to the **first addend** that would be produced. The corresponding **chunk** would either be `000` or `100`. Two options, both of which are really easy to compute. This is a **chunk** that would only exist on the first cycle of the algorithm. Coincidentally, if you refer to the diagram[have actual link or figure #] up above, you'll notice that, in the first cycle of the algorithm, we have an extra input in the CSA array that we initialized to zero. What if, instead, we initialize it to the addend produced by this mythical **chunk**?

It'd solve the issue. It'd get us the extra bit we needed, and make us match the ARM7TDMI's cycle counts completely.

But that's not all. Remember the carry flag from earlier? With this simple change, we go from matching hardware about 50% of the time (no better than randomly guessing) to matching hardware _**85%**_ of the time. This sudden increase was something no other theory was able to do, and made me really confident that I was on to something. However, this percentage only happens if we set the carry flag to bit `30` of the partial carry result, which seems super arbitrary. It turns out that bit of the partial carry result had a special meaning I did not realize at the time, and I would only find out that meaning much, much later.

## Mathematical Black Magic

It feels like we are finally making some sort of progress, however my algorithm still failed to calculate the carry flag properly around 15% of the time, and failed way more than that on long / signed multiplies. It was around this time that I found two patents [link later] that almost _entirely_ explained the algorithm. No idea how these hadn't been found up until this point, but they were quite illuminating.

After reading the patents, it turns out my implementation of the CSA array is slightly flawed (see [`perform_csa_array`](#perform_csa_array) above). In particular, that function uses CSAs with a width of _64_ bits. That's way too large and wastes space on the chip - the actual hardware gets away with only using _31_.

Another difference is that my algorithm has no way yet of supporting long accumulate values. Sure, I can initialize the partial output with the accumulate value, but the partial output is only 32 bits wide. 


Turns out, the patents describe a way to deal with both of these issues at once, using some mathematical trickery. This is the hardest part of the algorithm, so hang in there. (cite)

Roughly, on each CSA, we want to add three numbers together to produce two numbers. Let's give these five numbers some names. Define `S` to be a 33-bit value (even though the actual S is 32-bits, adding an extra bit allows us to handle both signed and unsigned multiplication) representing the previous CSA's sum, `C` to be a 33-bit value representing the previous CSA's carry, and `S'` and `C'` to be 33-bit values representing the resulting CSA sum / carry. Finally, define `X` to be a 34-bit value containing the current booths addend. Then we have:

$$
S', C' = S + C + X
$$

This, mathematically speaking, can be represented as a 65-bit addition. The reason why is that `X` can be left-shifted by as little as 0, and as much as 32. If we define `i` to be a number from `[0 - 3]` representing the CSA's position in the CSA array, we can divide the 65 bit CSA addition region into five chunks:
- Lower: A region of size `2i` that represents `final_csa_output`. This region is unaffected by future CSAs.
- TransL: The two bits of CSA `#i` that will become Lower bits in CSA `#(i + 1)`.
- Active: The 31-bit region where, including TransL, the actual CSA will be performed. 
- TransH: The two bits of CSA `#i` that will become Active bits in CSA `#(i + 1)`
- High: Contains values that have not yet been put into the CSA.
 34-bit value containing the current booths addend.

Define `A` to be a `65-bit` accumulate (even though the actual accumulate is 64-bits, adding an extra bit allows us to handle both signed and unsigned accumulates). Define `SL` and `CL` to be the analogue of `final_csa_output` in (see [`the code snippet above`](#perform_csa_array) above). Finally, define `XC` to be the carry flag produced by booth recoding. Then, we can model the CSA as follows:

| Region: | High  | TransH | Active | TransL | Lower |
| - | - | - | - | - | - |
| Size: |  30 - 2i | 2 |  31 | 2 | 2i | 
| Addend #1: | 0 | 0 | S[32:2] | S[1:0] | SL[2i:0] 
| Addend #2: | C[32], ..., C[32] | C[32] C[32] | C[32:2] | C[1:0] | CL[2i:0]
| Addend #3: | X[33], ..., X[33] | X[33] X[33] | X[32:2] | X[1:0] | 0
| Result Sum: |  0 | S'[32:31] | S'[30:0] | SL[2i+2:2i] | SL[2i:0] 
| Result Carry: |  C'[32], ..., C'[32]  | C'[32:31] | C'[30:0] | CL[2i+2], XC (aka CL[2i+1]) | CL[2i:0]

Seriously, take time to make sure you understand this table. It represents the CSA that we want to be able to perform.

Here's a simple way to implement long accumulates. 33 bits of the `A` will be placed in `S` as initialization. Meanwhile, we can shove the other `31` bits, two bits per CSA, into the high region of addend #1.

| Region: | High  | TransH | Active | TransL | Lower |
| - | - | - | - | - | - |
| Size: |  30 - 2i | 2 |  31 | 2 | 2i | 
| Addend #1: |0 | A[2i+35 : 2i+34] | S[32:2] | S[1:0] | SL[2i:0] 
| Addend #2: | C[32], ..., C[32] | C[32] C[32] | C[32:2] | C[1:0] | CL[2i:0]
| Addend #3: | X[33], ..., X[33] | X[33] X[33] | X[32:2] | X[1:0] | 0
| Result Sum: |  0 |S'[32:31] | S'[30:0] | SL[2i+2:2i] | SL[2i:0] 
| Result Carry: | C'[32], ..., C'[32] | C'[32:31] | C'[30:0] | CL[2i+2], XC (aka CL[2i+1]) | CL[2i:0]

We can ignore the Lower column, since the result there is always the same as the addends. We can also ignore the TransL and Active columns, as the operation in those two columns can be implemented using a simple 33-bit CSA (and we already have shown how to do so in [`perform_csa_array`](#perform_csa_array) above). This leaves:

| Region: | High  | TransH 
| - | - | - |
| Size: |  30 - 2i | 2 
| Addend #1: |0 | A[2i+35 : 2i+34] | S[32:2] | 
| Addend #2: | C[32], ..., C[32] | C[32] C[32] | C[32:2] | C[1:0] | CL[2i:0]
| Addend #3: | X[33], ..., X[33] | X[33] X[33] | X[32:2] | X[1:0] | 0
| Result Sum: |  0 |S'[32:31] | S'[30:0] | SL[2i+2:2i] | SL[2i:0] 
| Result Carry: | C'[32], ..., C'[32] | C'[32:31] | C'[30:0] | CL[2i+2], XC (aka CL[2i+1]) | CL[2i:0]

We can replace Addend #2 with one row of all ones, and another row with just `!C[N]`. Convince yourself why this is mathematically OK.

| Region: | High  | TransH 
| - | - | - |
| Size: |  30 - 2i | 2 
| Addend #1: |0 | A[2i+35 : 2i+34] | S[32:2] | 
| Addend #2: | 1, ..., 1 | 1 1 | C[32:2] | C[1:0] | CL[2i:0]
| Addend #2.5: | 0 | 0 !C[32] | C[32:2] | C[1:0] | CL[2i:0]
| Addend #3: | X[33], ..., X[33] | X[33] X[33] | X[32:2] | X[1:0] | 0
| Result Sum: |  0 |S'[32:31] | S'[30:0] | SL[2i+2:2i] | SL[2i:0] 
| Result Carry: | C'[32], ..., C'[32] | C'[32:31] | C'[30:0] | CL[2i+2], XC (aka CL[2i+1]) | CL[2i:0]

Do the same to Addend #3:

| Region: | High  | TransH 
| - | - | - |
| Size: |  30 - 2i | 2 
| Addend #1: |0 | A[2i+35 : 2i+34] | S[32:2] | 
| Addend #2: | 1, ..., 1 | 1 1 | C[32:2] | C[1:0] | CL[2i:0]
| Addend #2.5: | 0 | 0 !C[32] | C[32:2] | C[1:0] | CL[2i:0]
| Addend #3: | 1, ..., 1 | 1 1 | C[32:2] | C[1:0] | CL[2i:0]
| Addend #3.5: | 0 | 0 !X[33] | C[32:2] | C[1:0] | CL[2i:0]
| Result Sum: |  0 |S'[32:31] | S'[30:0] | SL[2i+2:2i] | SL[2i:0] 
| Result Carry: | C'[32], ..., C'[32] | C'[32:31] | C'[30:0] | CL[2i+2], XC (aka CL[2i+1]) | CL[2i:0]

Now, Addends #2 and #3 can be added together, being replaced by a new `Addend #4`.

| Region: | High  | TransH 
| - | - | - |
| Size: |  30 - 2i | 2 
| Addend #1: |0 | A[2i+35 : 2i+34] | S[32:2] | 
| Addend #2.5: | 0 | 0 !C[32] | C[32:2] | C[1:0] | CL[2i:0]
| Addend #3.5: | 0 | 0 !X[33] | C[32:2] | C[1:0] | CL[2i:0]
| Addend #4: | 1, ..., 1 | 1 0 | C[32:2] | C[1:0] | CL[2i:0]
| Result Sum: |  0 |S'[32:31] | S'[30:0] | SL[2i+2:2i] | SL[2i:0] 
| Result Carry: | C'[32], ..., C'[32] | C'[32:31] | C'[30:0] | CL[2i+2], XC (aka CL[2i+1]) | CL[2i:0]

Now we can define `S'` as:
`S'[32:31] = A[2i + 34] + !C[32] + !X[33]`

We can now remove `S'` and the bits used to calculate it. Let's see what's left.

| Region: | High  | TransH 
| - | - | - |
| Size: |  30 - 2i | 2 
| Addend #1: |0 | A[2i+35] 0 | S[32:2] | 
| Addend #4: | 1, ..., 1 | 1 0 | C[32:2] | C[1:0] | CL[2i:0]
| Result Carry: | C'[32], ..., C'[32] | C'[32:31] | C'[30:0] | CL[2i+2], XC (aka CL[2i+1]) | CL[2i:0]

Meaning   `C'[32] = !A[2i+35]`.



And with that, we managed to go from using 64 bits of CSA, to only 33. Our final algorithm for the CSAs is as follows:


```C
struct CSAOutput perform_csa_array(u64 partial_sum, u64 partial_carry, BoothRecodingOutput addends[4]) {
    struct CSAOutput csa_output = { partial_sum, partial_carry };
    struct CSAOutput final_csa_output = { 0, 0 };

    for (int i = 0; i < 4; i++) {
        csa_output.output &= 0x1FFFFFFFFULL;
        csa_output.carry  &= 0x1FFFFFFFFULL;

        struct CSAOutput result = perform_csa(csa_output.output, addends.m[i].recoded_output & 0x1FFFFFFFFULL, csa_output.carry);

        // Inject the carry caused by booth recoding
        result.carry <<= 1;
        result.carry |= addends.m[i].carry;

        // Take the bottom two bits and inject them into the final output.
        // The value of the bottom two bits will not be changed by future
        // addends, because those addends must be at least 4 times as big
        // as the current addend. By directly injecting these two bits, the
        // hardware saves some space on the chip.
        final_csa_output.output |= (result.output & 3) << (2 * i);
        final_csa_output.carry  |= (result.carry  & 3) << (2 * i);
        
        // The next CSA will only operate on the upper bits - as explained
        // in the previous comment.
        result.output >>= 2;
        result.carry  >>= 2;

        // Perform the magic described in the tables for the handling of TransH
        // and High. acc_shift_register contains the upper 31 bits of the acc
        // in its lower bits.
        u64 magic = bit(acc_shift_register, 0) + !bit(csa_output.carry, 32) + !bit(addends.m[i].recoded_output, 33);
        result.output |= magic << 31;
        result.carry |= (u64) !bit(acc_shift_register, 1) << 32;        
        acc_shift_register >>= 2;

        csa_output = result;
    }

    final_csa_output.output |= csa_output.output << 8;
    final_csa_output.carry  |= csa_output.carry  << 8;

    return final_csa_output;
}
```

# The Specifics of Early Termination

We already touched on early termination briefly, but turns out it gets a bit more complicated. The patents don't exactly explain how early termination works here in much detail, besides some cryptic references to shift types / shift values. More importantly, we can implement early termination quite simply, like so:

```c
bool should_terminate(u64 multiplier, enum MultiplicationFlavor flavor) {
    if (is_signed(flavor)) {
        return multiplier == 0x1FFFFFFFF || multiplier == 0;
    } else {
        return multiplier == 0;
    }
}
```

Note that <span style="color:#3a7dc9"> **multiplier**</span> is a signed 33-bit number. Now here's the main issue with early termination. After every cycle of booth's algorithm, a total of 41 bits of result are produced. To convince yourself of this, look at the final two lines of `perform_csa_array`. The bottom eight bits contain the result of each CSA, and the upper 33 bits above those 8 contain `csa_output`. After every cycle of booth's algorithm, the bottom eight bits are fed into a result register, since the _next_ cycle of booth's algorithm cannot change the value of those bottom eight bits. The upper 33 bits become the next input into the next cycle of booth's algorithm. Something like this:


```c
// I'm using this over a __uint128_t since the latter isn't available
// on a GBA, and I need this code to compile on a GBA so I can fuzz the 
// outputs.
struct u128 {
    u64 lo;
    u64 hi;
};

// Latches that contain the final results of the algorithm.
// Really, these only need to be 66 bits, but 128 is good enough.
// Why 66? Because:
// - We obtain 1 bit from initialization (EXPLAIN)
// - We obtain 8 * 4 bits from booths algorithm
// - We obtain 33 more bits also from booths algorithm.
__uint128_t partial_sum;
__uint128_t partial_carry;

do {
    csa_output = perform_one_cycle_of_booths_mutliplication(
        csa_output, multiplicand, multiplier);

    partial_sum.lo   |= csa_output.output & 0xFF;
    partial_carry.lo |= csa_output.carry  & 0xFF;

    csa_output.output >>= 8;
    csa_output.carry  >>= 8;

    partial_carry >>= 8;

    multiplier = asr(multiplier, 8, 33);
} while (!should_terminate(multiplier, flavor));

partial_sum.lo   |= csa_output.output;
partial_carry.lo |= csa_output.carry;
```

Since `partial_sum` and `partial_carry` are shift registers that get rotated with each iteration of booths algorithm, we need to rotate them again after the algorithm ends in order to correct them to their proper values. The `partial_carry`'s rotation is done via the ARM7TDMI's barrel shifter (explain what tha barrel shifteris), with the output of the barrel shifter going to the ALU. For long (64-bit) multiplies, two rotations occur, since the ALU can only add 32-bits at a time and so must be used twice. 

Spoiler alert, the value of the carry flag after a multiply instruction comes from the carryout of this barrel shifter.

So, what rotation values does the ARM7TDMI use? According to the patents, for an unsigned multiply, all (1 or 2) uses of the barrel shifter do:

| # Iterations | Type | Rotation | 
| - | - | - |
| 1 |ROR|22 | 
| 2 |ROR|14  |
| 3 |ROR|6  |
| 4 |ROR|30  |

Signed multiplies differ from unsigned multiplies in their **second** barrel shift. The second one for signed multiplies looks like this:

| # Iterations | Type | Rotation | 
| - | - | - |
| 1 |ASR|22 | 
| 2 |ASR|14  |
| 3 |ASR|6  |
| 4 |ROR|30  |

I'm not going to lie, I couldn't make sense of these rotation values. At all. Maybe they were wrong, since they patents already had a couple major errors at this point. No idea. Turns out it doesn't _really_ matter for calculating the carry flag of a multiply instruction. Observe the operation of the ARM7TDMI's `ROR` and `ASR`.

Code from fleroviux's NanoBoyAdvance:
```C++
void ROR(u32& operand, u8 amount, int& carry, bool immediate) {
  // Note that in booth's algorithm, the immediate argument will be true, and
  // amount will be non-zero

  // ROR #0 equals to RRX #1
  if (amount != 0 || !immediate) {
    if (amount == 0) return;

    amount %= 32;
    operand = (operand >> amount) | (operand << (32 - amount));
    carry = operand >> 31;
  } else {
    auto lsb = operand & 1;
    operand = (operand >> 1) | (carry << 31);
    carry = lsb;
  }
}

void ASR(u32& operand, u8 amount, int& carry, bool immediate) {
  // Note that in booth's algorithm, the immediate argument will be true, and
  // amount will be non-zero and less than 32.

  if (amount == 0) {
    // ASR #0 equals to ASR #32
    if (immediate) {
      amount = 32;
    } else {
      return;
    }
  }

  int msb = operand >> 31;

  if (amount >= 32) {
    carry = msb;
    operand = 0xFFFFFFFF * msb;
    return;
  }

  carry = (operand >> (amount - 1)) & 1;
  operand = (operand >> amount) | ((0xFFFFFFFF * msb) << (32 - amount));
}
```

Note that in both ROR and ASR the carry will always be set to the last bit of the `operand` to be shifted out. e.g., if I rotate a value by `n`, then the carry will always be bit `n - 1` of the `operand`, since that was the last bit to be rotated out. Same goes for ASR.

So, _it doesn't matter_ if I don't use the same rotation values as the patents. Since, no matter the rotation value, as long as the output from _my_ barrel shifter is the same as the output from the _ARM7TDMI's_ barrel shifter, and the input to _my_ barrel shift is the same as the input to the _ARM7TDMI's_ barrel shifter, then the last bit to be shifted out must be the same, and therefore the carry flag must _also_ have been the same.

So, here's my implementation. I tried to somewhat mimic the table from above, but I didn't do a very good job. But it works, so fuck it.


```C
// I'm using this over a __uint128_t since the latter isn't available
// on a GBA, and I need this code to compile on a GBA so I can fuzz the 
// outputs.
struct u128 {
    u64 lo;
    u64 hi;
};

// we have ror'd partial_sum and partial_carry by 8 * num_iterations + 1
// we now need to ror backwards, i tried my best to mimic the table, but
// i'm off by one for whatever reason.
int correction_ror;
if (num_iterations == 1) correction_ror = 23;
if (num_iterations == 2) correction_ror = 15;
if (num_iterations == 3) correction_ror = 7;
if (num_iterations == 4) correction_ror = 31;

partial_sum   = u128_ror(partial_sum, correction_ror);
partial_carry = u128_ror(partial_carry, correction_ror);

int alu_carry_in = bit(multiplier, 0);

if (is_long(flavor)) {
    if (num_iterations == 4) {
        struct AdderOutput adder_output_lo = 
            adder(partial_sum.hi, partial_carry.hi, alu_carry_in);
        struct AdderOutput adder_output_hi = 
            adder(partial_sum.hi >> 32, partial_carry.hi >> 32, 
                  adder_output_lo.carry);

        return (struct MultiplicationOutput) {
            ((u64) adder_output_hi.output << 32) | adder_output_lo.output,
            (partial_carry.hi >> 63) & 1
        };
    } else {
        struct AdderOutput adder_output_lo = 
            adder(partial_sum.hi >> 32, partial_carry.hi >> 32, alu_carry_in);

        int shift_amount = 1 + 8 * num_iterations;

        // why this is needed is unknown, but the multiplication doesn't work
        // without it
        shift_amount++;

        partial_carry.lo = sign_extend(partial_carry.lo, shift_amount, 64);
        partial_sum.lo |= acc_shift_register << (shift_amount);

        struct AdderOutput adder_output_hi = 
            adder(partial_sum.lo, partial_carry.lo, adder_output_lo.carry);
        return (struct MultiplicationOutput) { 
            ((u64) adder_output_hi.output << 32) | adder_output_lo.output,
            (partial_carry.hi >> 63) & 1
        };
    }
} else {
    if (num_iterations == 4) {
        struct AdderOutput adder_output = 
            adder(partial_sum.hi, partial_carry.hi, alu_carry_in);
        return (struct MultiplicationOutput) { 
            adder_output.output,
            (partial_carry.hi >> 31) & 1
        };
    } else {
        struct AdderOutput adder_output = 
            adder(partial_sum.hi >> 32, partial_carry.hi >> 32, alu_carry_in);
        return (struct MultiplicationOutput) { 
            adder_output.output,
            (partial_carry.hi >> 63) & 1
        };
    }
}

```

Anyway, that's basically it. If you're interested in the full code, take a look [here](https://github.com/zaydlang/multiplication-algorithm/tree/master).
