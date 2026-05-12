ChaCha20](https://cr.yp.to/chacha.html) is a stream cipher based on [Salsa20](https://cr.yp.to/snuffle.html). There's also a nice [wikipedia article](https://en.wikipedia.org/wiki/Salsa20#ChaCha_variant) on it. You should read Salsa for background on ChaCha


# Salsa
Salsa20 takes a 256-bit key, and expands it into $2^{64}$ randomly accessible streams that each contain $2^{64}$ randomly accessible 64-byte blocks (512 bit blocks). More precisely:
- It takes a 256 bit key; defines $2^{64}$ streams
- It takes a 64-bit nonce; essentially this picks one of the streams.
- It takes a 64-bit "position"; essentially this picks one of the blocks in the stream.
- It then also has a 128-bit constant value that it uses (since we need to start somewhere; in fact this constant spells "expand 3 2-byte k")
Internally, it drops these into a 4x4 matrix where each  "cell" is a 32-bit word derived from the above values (*little-endian arrangment)*, and these make up the initial state. Diagnal holds the constant key, remaining cells are filled as follows: first 4 words of key - nonce - position - last 4 words of key. The algorithm then performs the following operation, which is to be performed on a column (or row, read ahead) of the matrix:
```
// Let a, b, c, d each be a 32-bit word, i.e. the cells of a column.
// (The general idea is that a = the diagonal of the column each time).
// Let W <<< x mean: rotate (left) the word W by x positions.
// "Rotate" means move each bit "x" places to the left with wrap-around
QR(a, b, c, d):
b ^= (a + d) <<<  7;
c ^= (b + a) <<<  9;
d ^= (c + b) <<< 13;
a ^= (d + c) <<< 18;
```

This operation is then applied for $r = 20$  rounds on the matrix.
```
// Let 0 through 15 signify an index to a cell of the aforemetioned matrix.

// Odd round
QR( 0,  4,  8, 12) // column 1
QR( 5,  9, 13,  1) // column 2
QR(10, 14,  2,  6) // column 3
QR(15,  3,  7, 11) // column 4
// Even round
QR( 0,  1,  2,  3) // row 1
QR( 5,  6,  7,  4) // row 2
QR(10, 11,  8,  9) // row 3
QR(15, 12, 13, 14) // row 4
```
Note that the original algorithm actually does a transpose after every round, but switching from rows to columns depending on parity is functionally identical and faster.
Then finally you add the resultng matrix (mod $2^32$) together with the initial matrix to get the final value; this is what prevents the output from being invertible.

**NOTE:** In the QR operation, you effectively start one cell below the diagonal, add the two cells above it together and then xor it with the current cell; then you do the same for the cell below it, and so on, until you wrap back to the diagonal cell.  Note thus that if you rotate each column of the initial matrix such that all constants (diagonals) are in the top row, *you can then vectorize each row of the matrix and perform the add-rotate-xor for each column in parallel by using SIMD.* Do note that at the end you must undo this permutation again before making the output block, since the input block still has the unpermuted form.*

# ChaCha
It starts off much the same as Salsa, including the same input sizes and the same constant. However, the initial matrix is arranged as follows, simply in the given order: constant - key - position - nonce.

However, it defines the following operation instead. It does the same number of operations, yet now each word is updated twice:
```
QR(a, b, c, d):
a += b; d ^= a; d <<<= 16;
c += d; b ^= c; b <<<= 12;
a += b; d ^= a; d <<<=  8;
c += d; b ^= c; b <<<=  7;
```
*Note that each operation depends on the previous, so no speedups there.*

In this case, the a permutation is "built-in" into the algorithm: fundamentally the only thing that has really changed is the internals of `QR`, however by building in the shift of the diagonals (constants) into the specfication it is  mainly to the advantage for SIMD platforms, as we can do the same trick **for SIMD as with Salsa** but now we don't need to do additional permutations at the start and end. The change in ordering of input fragments (key, nonce, position) is not believed to negatively affect securtity of Chacha compared to the positioning used by Salsa (or rather, there is no evidence that ordering the inputs that way in Salsa actually improves security).

## SIMD notes
*Note that eventhough the original paper and many example implementations work by doing four full quarter-rounds, one per column/diagonal, one after the other, many real-world implementations actually do each step of a quarter-round 4 times, once per column/diagonal, before moving on to the next step. This is generally better for pipelining and ILP: each step in a quarter-round depends on the previous one but quarter-rounds themselves are independent within an iteration, so by interleaving quarter-rounds you don't stall the pipeline as much between intra-QR steps.*

Cannot parellize operations within a single column, as the order of operations matter; for the same reason cannot parallelize between rounds either. Except for the final addition at the end, which you could do compeletely parallel. 
So ideally you wanna do all quarter rounds in parallel.

Alternatively, since this is a stream cipher, just do multiple chacha blocks at different counter/position values in parallel. When you have 256-bit wide registers, you can process 256 / 32 = 8 words at once, so in total you work on eight blocks of ChaCha. Each ChaCha block is 64 bytes, so you have 8 * 64 = 512 bytes of data per run of the algo in total. **This is NOT about parallelizing all quarter rounds in one go, but rather doing a quarter_round on multiple chacha states/blocks at once**

### YuriMyakotion ChaCha20 avx2 notes
Notes on a C-implementation of ChaCha20 via AVX2 found [here](https://github.com/YuriMyakotin/ChaCha20-SIMD/blob/2fa2d40d50597f913941dbe0189c2d11a8f45633/chacha20_avx2.c)
- 61-63: broadcast the first three (128-bit) rows of the state matrix across 256-bit registers, into variables `state0, ... state2` respectively.
- The reason state-3 happens inside the loop is to account for the calling function `ChaCha20EncryptBytes` being able to generate a stream to encrypt larger pieces of data than 512 bytes in one go (as explained above, a 256-bit simd implementation can generate at most 512 bytes of ChaCha in one go): state 3 holds the  'counter' and this must be incremented for every new chacha block being generated.

**AVX2: Yuri Vs Botan**
Botan:
Each register holds eight words at the same index from eight different state matrices. Each individual operation in a quarter-round is thus performed on eight different matrices at once, one word per matrix; quarter round operations themselves are interleaved between columns, per round.

Yuri:
Each register holds two rows at the same index from two different state matrices. Each quarter-round operation is thus performed on two different matrices at once, one row (four columns/words) per matrix, per round; "interleaving" is between four such sets of two matrices, per round.

### XoshiroSIMD impl notes
the cache does in fact get used. Whenever the `()` operatior is used to "generate" the next random number, what actually happens is this:
- Whenever the `m_index` member value is 0 (it is an 8-bit unsigned int), it runs `populate_cache`
- Otherwise it returns `m_index`'th index of the cache and then increments `m_index` by one.
In the `populate_cache` function,  essentially an unrolled for loop is called that writes to the entire cache at once by repeatedly calling `next` (which affects the internal state) and writing its result to the cache.. `next` operates on the `m_state` (stored as `xsimd` batches) and updates it, then returns the result.
Note that the larger caching here seems to be to take advantage of hardware-stuff like memory-locality (for cache hits) and pipelining, as it is in fact caching considerably more stuff than what the XSIMD parallelism generates.

The size of the state is `RNG_WIDTH x SIMD_WIDHT`, that is, `4 x <number of uint64_ts that fit in a SIMD reg` where each element is a `uint64_t`f