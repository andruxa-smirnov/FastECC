FastECC implements FFT-based O(N*log(N)) [Reed-Solomon coder], running at [1.2 GB/s][Benchmarks]
on [i7-4770] for (n,k)=(2^20,2^19), i.e. calculating 524288 parity blocks from 524288 data blocks.

It's also [pretty fast for small orders](Benchmarks.md#small-ntt), outperforming previously fastest Reed-Solomon library, Intel [ISA-L] for 64+ parity blocks.

The encoding and decoding algorithms going to be implemented by FastECC were described in the paper
[An Efficient (n,k) Information Dispersal Algorithm based on Fermat Number Transforms](https://pdfs.semanticscholar.org/141d/c4ee4cca45b4ed1c07f890f758e427597db8.pdf)
by Sian-Jheng Lin and Wei-Ho Chung. FastECC v0.1 implements only encoding, so it isn't yet ready for real use.

<a name="what"/>

## What

Almost all existing Reed-Solomon [ECC] implementations employ matrix multiplication and thus have O(N^2) speed behavior,
i.e. they can produce N parity blocks in O(N^2) time, thus spending O(N) time per block.
F.e. the fastest implementation I know, [MultiPar], can compute 1000 parity blocks at the speed ~50MB/s,
but only at ~2 MB/s in its maximum configuration, 32000 parity blocks.
And computations in GF(2^32), implemented in the same way, will build one million parity blocks at 50 KB/s.

One of few exceptions is closed-source [RSC32 by persicum] with O(N*log(N)) speed, i.e. it spends O(log(N)) time per parity block.
Its speed with million parity blocks is 100 MB/s, i.e. it computes one million of 4 KB parity blocks
from one million of data blocks (processing 8 GB overall) in just 80 seconds.
Note that all speeds mentioned here are measured on [i7-4770], employing all features available in a particular program -
including multi-threading, SIMD and x64 support.

FastECC is open-source library implementing O(N*log(N)) encoding algorithm.
It computes million parity blocks at [1.2 GB/s].
Future versions will implement decoding that's also `O(N*log(N))`, although 1.5-3 times slower than encoding.
Current implementation is limited to 2^20 blocks, removing this limit is the main priority for future work
aside of decoder implementation.
And if you are interested in smaller configs, look at [small NTT benchmarks](Benchmarks.md#small-ntt) -
FastECC outperforms quadratic algorithms ([ISA-L], [CM256] and [MultiPar]) starting from 32-64 parity blocks.

[Leopard] is a new library, [faster](https://github.com/catid/leopard/blob/master/Benchmarks.md) than [FastECC][Benchmarks], especially for small orders.
It implements similar algorithm, described in newer paper by the same authors:
[Lin, Han and Chung "Novel Polynomial Basis and Its Application to Reed-Solomon Erasure Codes"](https://github.com/SianJhengLin/Fast-algorithms-of-Reed-Solomon-erasure-codes).

You can also find a few [research-grade libraries](https://www.livebusinesschat.com/smf/index.php?topic=5888.msg42216#msg42216) with O(N*log(N)) speed.

For comparison - [Wirehair], the best open-source LDPC codec I know, is O(N) and
[already as fast](https://github.com/catid/wirehair#benchmarks) as [FastECC][Benchmarks],
but can be made [several times faster using SSE1](https://github.com/catid/wirehair/issues/2).
It's limited to 64000 source blocks, but amount of parity blocks can be arbitrary.
It's an LDPC codec, so not [MDS](https://en.wikipedia.org/wiki/Singleton_bound#MDS_codes),
but chances that it needs even a single extra block to recover is [as low as 0.1%](https://github.com/catid/wirehair#discussion-overhead-reductions-with-gf216).
Moreover, it works with binary data, so no need for [recoding](GF.md#data-packing) and no need for extra space to store "overflow" bits.


<a name="how"/>

## How

All O(N*log(N)) Reed-Solomon implementations I'm aware of, use fast transforms like FFT or FWT.
FastECC employs fast Number-Theoretic Transform that is just an FFT over integer field or ring.
Let's see how it works. Note that below by `length-N polynomial` I mean any polynomial with order < N.

For any given set of N points, only one length-N polynomial may go through all these points.
Let's consider N input words as values of some length-N polynomial at N fixed points,
only one such polynomial may exist.

Typical Reed-Solomon encoding computes coefficients of this unique polynomial (so-called [polynomial interpolation]),
evaluates the polynomial at M another fixed points (the `polynomial evaluation`)
and outputs these M words as the resulting parity data.

At the decoding stage, we may receive any subset of N values out of those N source data words and M computed parity words.
But since they all belong to the original length-N polynomial, we may recover this polynomial from N known points
and then compute its values at other points, in particular those N points assigned to original data, thus restoring them.


<a name="fast"/>

## Fast

Usually, Reed-Solomon libraries implement encoding by multiplication with [Vandermonde matrix] (O(N^2) algo)
and decoding by multiplication with the matrix inverse.

But with special choice of fixed points we can perform polynomial interpolation and evaluation at these points
in O(N*log(N)) time, using NTT for evaluation and inverse NTT for interpolation. So, [the fast encoding] is as simple as:
- consider N input words as values of length-N polynomial at N special points
- compute the polynomial coefficients in O(N*log(N)) time using inverse NTT
- evaluate the polynomial at another M special points in O(M*log(M)) time using NTT

Decoding is more involved. We have N words representing values of length-N polynomial at **some** N points.
Since we can't choose these points, we can't just use iNTT to compute the polynomial coefficients.
So it's a generic [polynomial interpolation] problem that can be solved in [O(N*log(N)^2) time][fast polynomial interpolation].


<a name="faster"/>

## Faster

But this specific polynomial interpolation problem has faster solution.
Indeed, decoder knows values of length-N polynomial f(x) at N points a[i], but lost its values at M erasure points e[i].
Let's build "erasure locator" polynomial `l(x) = (x-e[1])*...*(x-e[M])` and compute polynomial product `p(x)=f(x)*l(x)`.
We have `order(p) = order(f)+order(l) < N+M` and l(e[i])=0, so by computing values l(a[i]) and then multiplying f(x) and l(x) in the value space
we can build polynomial p(x) with order<N+M described by its values at N+M points `p(a[i]) = f(a[i])*l(a[i]),  p(e[i]) = f(e[i])*l(e[i]) = 0`,
i.e. we have fully defined p(x).

Now we just need to perform [polynomial long division] f(x)=p(x)/l(x), that is [O(N*log(N)) operation][fast polynomial division].


<a name="fastest"/>

## Fastest

But there is even faster algorithm! Let's build formal derivative `p'(x) = f'(x)*l(x) + f(x)*l'(x)`
and evaluate it at each e[i]: `p'(e[i]) = f'(e[i])*l(e[i]) + f(e[i])*l'(e[i]) = f(e[i])*l'(e[i])` since l(e[i])=0.
Moreover, l'(e[i])!=0, so we can recover all the lost f(e[i]) values by simple scalar divisions: `f(e[i]) = p'(e[i]) / l'(e[i])`!!!

So, the entire algorithm is:
- transform polynomials p and l into coefficient space
- compute their formal derivatives p' and l'
- transform p' and l' into value space
- evaluate each `f(e[i]) = p'(e[i]) / l'(e[i])`

When M<=N, first operation on p is iNTT(2N),
third operation on p' is NTT(N) since we need to compute p'(x) values only at N points corresponding to original data,
and rest is either O(N) or operations on l(x) that is performed only once,
so overall decoding is 1.5-3 times slower than iNTT(N)+NTT(M) operations required for encoding.


<a name="why-not"/>

## Why not

FastECC is absolutely useless for RAID storage (such as hadoop). With RAID, when one sector is overwritten,
RAID software should read all other data sectors in the same shard in order to recompute parity sectors of the shard,
and then overwrite all these parity sectors. So, RAID software developers are looking for solutions that will allow them
to read/write less sectors on each operation (such as [pyramid codes](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/11/Pyramid-Codes.pdf)),
rather than opposite.

FastECC is probably useless for any hardware controllers (SSD, Ethernet, LTE, DVB...).
These controllers work with analog signals and tend to use **soft** decoders to extract as much data as possible
(soft decoders understand that data received as 7 have more chances to be decoded as 8 rather than to be decoded as 2,
and can deal even with something like 7.4 which is more probably 8 rather than 6). Soft decoders are absolutely out of my Math skills,
so if someone will build soft decoder, it will be not FastECC, but other great lib (and most probably not free).

Also, FastECC implements only erasure decoding. I have no idea how to implement error decoding nor plan to learn it.

There are some applications still. PAR3 is one of them - it's still interesting for some people, although not many.
Various communication applications and P2P data storage are also frequently mentioned in [discussions](#discussion).

---

I made a quick speed comparison and found that FastECC is faster than 16-bit RS codec in MultiPar starting from ~32 parity blocks.
8-bit RS codecs such as ISA-L should be even faster than 16-bit ones. And with 20% redundancy 32 parity blocks means 160 data blocks,
close to the maximum possible for 8-bit RS. So, it seems that FastECC territory starts right where 8-bit codecs territory ends -
if you need more than 256 data+parity blocks, FastECC should be faster than any 16-bit RS coders, otherwise 8-bit [ISA-L] or [CM256] is preferable.

Moreover, FastECC is free from patent restrictions that has any fast RS codec employing PSHUFB (i.e. SSSE3) including [Leopard].
And slow codecs are several times slower than MultiPar, so they have even less chances.

There is a great alternative to FastECC - [Wirehair] library, but afair it also may be covered with patents.
It's [already as fast](https://github.com/catid/wirehair#benchmarks) as [FastECC][Benchmarks],
but can be made [several times faster using SSE1](https://github.com/catid/wirehair/issues/2).
It's limited to 64000 source blocks, but amount of parity blocks can be arbitrary.
It's an LDPC codec, so not [MDS](https://en.wikipedia.org/wiki/Singleton_bound#MDS_codes),
but chances that it needs even single extra block to recover is [as low as 0.1%](https://github.com/catid/wirehair#discussion-overhead-reductions-with-gf216).

Unlike Wirehair and MultiPar, FastECC doesn't work directly with arbitrary binary data - it works in GF(p) or GF(p^2), default is GF(0xFFF00001).
This means that input data [should be converted](GF.md#efficient-data-packing) into 0..0xFFF00000 range,
that further means that parity sectors are slightly longer that input ones (f.e. 4096 byte data sectors and 4100 byte parity sectors).
This also limits its applications.

So, overall, FastECC should replace any use of 16-bit RS codecs, while LDPC and 8-bit RS codecs will keep their niches.


<a name="roadmap"/>

## Roadmap

- [x] Encoder (version 0.1)
- [ ] Decoder (version 0.2)
- [ ] Public API (see issue #1)
- [ ] SSE2/AVX2/NEON-intrinsics with runtime selection of scalar/sse2/avx2/neon code path
- [ ] NTT of sizes!=2^n: NTT5/NTT7/NTT13, PFA and MFA+PFA combined algo
- [ ] Optimizations for asymmetric cases (n!=2k)
- [ ] Optimized code for GF(2^31-1), GF(2^61-1) and GF(p^2)


<a name="more"/>

## Want to know more?

- [NTT: Number-theoretic transform](Overview.md): what one needs to know in order to implement O(N*log(N)) Reed-Solomon error-correcting codes
- [GF(p).cpp: fast computations in finite fields and rings](GF.md)
- [NTT.cpp: NTT implementation](NTT.md)
- [RS.cpp: Reed-Solomon coder](RS.md)
- [Benchmarks](Benchmarks.md)


<a name="discussion"/>

## Discussion

- [Encode.ru forum](https://encode.ru/threads/2750-FastECC-fastest-Reed-Solomon-codec-ever?p=52622)
- [MultiPar forum](https://www.livebusinesschat.com/smf/index.php?topic=6154.0)
- [Hacker News story](https://news.ycombinator.com/item?id=14290617)


[Reed-Solomon coder]: https://en.wikipedia.org/wiki/Reed%E2%80%93Solomon_error_correction
[Benchmarks]: Benchmarks.md#reed-solomon-encoding
[i7-4770]: https://ark.intel.com/products/75122/Intel-Core-i7-4770-Processor-8M-Cache-up-to-3_90-GHz
[ECC]: https://en.wikipedia.org/wiki/Error_detection_and_correction#Error-correcting_codes
[MultiPar]: https://www.livebusinesschat.com/smf/index.php?board=396.0
[RSC32 by persicum]: https://www.livebusinesschat.com/smf/index.php?board=399.0
[Vandermonde matrix]: https://en.wikipedia.org/wiki/Vandermonde_matrix
[the fast encoding]: https://github.com/Bulat-Ziganshin/FastECC/blob/bed3a3f4c228ee7ab61cee1b7c28b6d4d76df02d/RS.cpp#L37
[polynomial long division]: https://en.wikipedia.org/wiki/Polynomial_long_division
[fast polynomial division]: https://www.google.com/search?q=fast+polynomial+division "fast polynomial division"
[polynomial interpolation]: https://en.wikipedia.org/wiki/Polynomial_interpolation
[fast polynomial interpolation]: https://www.google.com/search?q=fast+polynomial+interpolation "fast polynomial interpolation"
[ISA-L]: https://github.com/01org/isa-l
[CM256]: https://github.com/catid/cm256
[Leopard]: https://github.com/catid/leopard
[Wirehair]: https://github.com/catid/wirehair
