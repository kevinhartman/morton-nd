# Morton ND
A C++14 header-only Morton encoding library for N dimensions, based on the Lookup Table (LUT) method. LUTs are defined with constant expressions and thus can be generated (and even used) at compile-time.

The encoder supports chunking, meaning input fields can be automatically split into parts prior to lookup. The results of these partial lookups are then combined. This allows a smaller LUT to be used at the expense of the extra 'shift' and 'or' operations required to combine the results.

## Usage
The MortonNDEncoder class encapsulates a LUT and provides a corresponding encode function. To instantiate it, you must specify the number of fields (N), the number of chunks in each field, and the size of each chunk in bits as template parameters.

Field size in bits is calculated as the number of chunks multiplied by the size of each chunk in bits (chunks * chunk size).

### Example: 2D encoding (N = 2)
For a 2 field encoder, where the result must fit in a 32-bit field, the max field size is 16 bits (32 bits / 2 fields).

To create a MortonNDEncoder to accommodate 16 bits, parameterize it such that chunks * bits per chunk = 16:

```c++
// 1) 1 chunk, 16 bits. LUT size is 2^16 entries * 4 bytes per entry = 262.144 KB
constexpr auto MortonND_2D_32 = mortonnd::MortonNDEncoder<2, 1, 16>();

// 2) 2 chunks, 8 bits. LUT size is 2^8 entries * 2 bytes per entry = 512 bytes
constexpr auto MortonND_2D_32 = mortonnd::MortonNDEncoder<2, 2, 8>();

// 3) 4 chunks, 4 bits. LUT size is 2^4 entires * 1 byte per entry = 16 bytes
constexpr auto MortonND_2D_32 = mortonnd::MortonNDEncoder<2, 4, 4>();

// 4) 8 chunks, 2 bits
// ... this is supported (as is 16 chunks, 1 bit), but probably not very useful
```

Note the size of each LUT. A 16-bit chunk size requires a few hundred KB, but will provide the fastest encoding. A 4-bit chunk size only requires a 16 byte LUT, but each field will require roughly 3 times the number of lookup, shifting, or-ing and masking operations. See the performance section below for a comparison.

The encode function is variadic, but will assert that exactly 2 fields are specified for 2D (and N for ND):
```c++
auto encoding = MortonND_2D_32.Encode(9, 5);
```

<blockquote>
<b>Note:</b></p>
If targeting a 2D 64-bit result, the max field size is 32 bits (64 bits / 2 fields). However, in this case, the max chunk size is still 16 (not 32). This is due to compiler limitations which prevent a LUT of size 2^32 from being generated (which would be greater than 17 GB in size anyway – likely impractical). See the notes section below on various compiler limitations.
</blockquote>

### Example: 3D encoding (N = 3)
For a 3 field encoder, where the result must fit in a 64 bit field, the max size of each field is 21 bits (⌊64 bits / 3 fields⌋).

To create a suitable MortonNDEncoder, parameterize it such that chunks * bits per chunk = 21:

```c++
// 1) 1 chunk, 21 bits. LUT size is 2^21 entries * 8 bytes per entry = 16.777216 MB
constexpr auto MortonND_3D_64 = mortonnd::MortonNDEncoder<3, 1, 21>();

// 2) 3 chunks, 7 bits. LUT size is 2^7 entries * 4 bytes per entry = 512 bytes
constexpr auto MortonND_3D_64 = mortonnd::MortonNDEncoder<3, 3, 7>();

auto encoding = MortonND_3D_64.Encode(9, 5, 1);
```

As is the case in 2D, the larger table (while fairly hefty at around 17 MB) will offer the best performance (compared below).

A 3 field encoder, where the result must fit in a 32 bit field (max field size = ⌊32 bits / 3 fields⌋ = 10 bits) can be achieved as follows:

```c++
// 1 chunk, 10 bits.
constexpr auto MortonND_3D_32 = mortonnd::MortonNDEncoder<3, 1, 10>();

// 2 chunks, 5 bits.
constexpr auto MortonND_3D_32 = mortonnd::MortonNDEncoder<3, 2, 5>();
```

### Example: 5D encoding (N = 5)
For a 5 field encoder, where the result must fit in a 64 bit field, the max size of each field is 12 bits (⌊64 bits / 5 fields⌋).

Suitable MortonNDEncoders:

```c++
// 1) 1 chunk, 12 bits. LUT size is 2^12 entries * 8 bytes per entry = 32.768 KB
constexpr auto MortonND_5D_64 = mortonnd::MortonNDEncoder<5, 1, 12>();

// 2) 2 chunks, 6 bits. LUT size is 2^6 entries * 4 bytes per entry = 256 bytes
constexpr auto MortonND_3D_64 = mortonnd::MortonNDEncoder<5, 2, 6>();

// 3) 3 chunks, 4 bits. LUT size is 2^4 entries * 4 bytes per entry = 64 bytes
constexpr auto MortonND_3D_64 = mortonnd::MortonNDEncoder<5, 3, 4>();

auto encoding = MortonND_3D_64.Encode(17, 13, 9, 5, 1);
```

## 3D Performance
Performance metrics were gathered using [Forceflow's libmorton library](https://github.com/Forceflow/libmorton), which contains a suite of different Morton encoding algorithms for 2D and 3D. The snippets below show performance comparisons to the algorithms found there, as well as comparisons between different MortonND LUT size configurations.

libmorton also includes an approach using the BMI2 instruction set, the performance of which is not captured here (due to incompatible test hardware).

Depending on your use case and hardware, a particular configuration of Morton ND or perhaps one of the many libmorton algorithms may offer the best performance.

The following metrics were collected on an i7-6920HQ (6th generation Quad-core Intel 2.9GHz mobile CPU), compiled with GCC 8.1 on macOS 10.13 using "-O3 -DNDEBUG".

### 32-bit
```
++ Encoding 512^3 morton codes (134217728 in total)

    Linear      Random
    ======      ======
    925.439 ms  874.468 ms  : 32-bit MortonND: 2 chunks, 5 bits
    743.166 ms  711.925 ms  : 32-bit MortonND: 1 chunk, 10 bits  (fastest)
    
    libmorton algorithms
    ====================
    2663.162 ms 2599.181 ms : 32-bit For
    2330.065 ms 2014.106 ms : 32-bit For ET
    1171.010 ms 1135.522 ms : 32-bit Magicbits
    857.056 ms  814.186 ms  : 32-bit LUT
    831.843 ms  816.522 ms  : 32-bit LUT Shifted
```

### 64-bit
```
++ Encoding 512^3 morton codes (134217728 in total)

    Linear      Random
    ======      ======
    1349.667 ms 1329.539 ms : 64-bit MortonND: 3 chunks, 7 bits  (fastest, random)
    894.632 ms  1657.213 ms : 64-bit MortonND: 1 chunk, 21 bits  (fastest, linear)
    
    libmorton algorithms
    ====================
    1354.036 ms 1357.236 ms : 64-bit LUT Shifted
    1471.270 ms 1467.422 ms : 64-bit LUT
    1759.593 ms 1756.538 ms : 64-bit Magicbits
    4726.462 ms 8649.010 ms : 64-bit For ET
    8279.347 ms 8258.069 ms : 64-bit For

```

## Compiling
* GCC is much faster than Clang when generating LUTs at compile-time. Expect long compilation times with a large chunk size (LUT size) when using Clang. GCC can also handle up to a chunk size of 27 bits (28+ fails gracefully), whereas Clang will crash with a chunk size > 16 bits. For this reason, use GCC for large LUTs if possible.
* Compile with release/optimization flags.
* VC++ is untested.