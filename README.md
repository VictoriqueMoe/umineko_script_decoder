# Umineko Script Decoder

A tool to decode compiled `.file` script binaries from the [Umineko Project](https://github.com/umineko-project/umineko-scripting) back into readable `.txt` script files.

## How the encoding works

The Umineko Project translates the visual novel *Umineko no Naku Koro ni* into multiple languages. The translation workflow produces hundreds of individual `.txt` files, one per chapter, per episode, per language. At build time, all of these are merged into a single massive plaintext script (often 19+ MB), which is then encoded into a binary `.file` for the ONScripter-RU game engine to consume.

The encoding happens in `update-manager.php` via the `transformScript()` and `encodeScript()` functions. It's a three-layer process: two rounds of byte-level substitution cipher with a ZLIB compression sandwich in between.

### The binary header

Every `.file` starts with a 16-byte header:

```
Offset  Size  Field
0x00    4     Magic bytes: "ONS2" (ASCII)
0x04    4     Compressed data length (uint32, little-endian)
0x08    4     Original plaintext length (uint32, little-endian)
0x0C    4     Version number (uint32, little-endian; currently 110)
```

Everything after byte 16 is the encoded payload.

### Layer 1: XOR substitution cipher (pass 1)

The encoder processes the raw plaintext script in **128 KB chunks** (131,072 bytes). For each byte in a chunk, it applies:

```
output = keyTable[(input XOR 0x71)] XOR 0x45
```

`keyTable` is a hardcoded 256-byte permutation table where every value from `0x00` to `0xFF` appears exactly once, but in a scrambled order. This makes it a **substitution cipher**: each possible byte value maps to exactly one other byte value.

The XOR operations before and after the table lookup add two extra layers of scrambling. Without them, the substitution would be a simple 1:1 byte swap. With them, each byte is XORed with `0x71` *before* the table lookup, and the result is XORed with `0x45` *after*. This effectively creates a completely different substitution mapping than the raw table alone would produce.

### Layer 2: ZLIB compression

After pass 1 encrypts the entire script, the result is compressed using ZLIB (deflate with zlib wrapper, window bits = 15). This serves two purposes: it reduces the file size significantly (typically ~4x compression on script text), and it makes the encrypted data even harder to analyze since compression destroys the statistical patterns that cryptanalysis relies on.

### Layer 3: XOR substitution cipher (pass 2)

The compressed data goes through a second round of substitution, again in 128 KB chunks, but with different constants:

```
output = keyTable[(input XOR 0x23)] XOR 0x86
```

Same `keyTable`, but the XOR constants change from `0x71`/`0x45` to `0x23`/`0x86`. This means pass 1 and pass 2 produce entirely different substitution mappings despite sharing the same underlying table.

### The full encoding pipeline

```
Plaintext script
    │
    ▼
┌─────────────────────────────┐
│  XOR Pass 1 (128KB chunks)  │  keyTable[byte ^ 0x71] ^ 0x45
│  Substitution cipher        │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  ZLIB Compress              │  deflate, window bits = 15
│  ~4x size reduction         │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  XOR Pass 2 (128KB chunks)  │  keyTable[byte ^ 0x23] ^ 0x86
│  Substitution cipher        │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  Prepend ONS2 header        │  magic + sizes + version
└─────────────┬───────────────┘
              │
              ▼
         Final .file
```

## How the decoder reverses it

Decoding is the exact reverse of the encoding pipeline. The key insight is that the substitution cipher is **invertible** because `keyTable` is a permutation, meaning every input maps to a unique output, so we can build a reverse lookup table.

### The inverse table

The substitution cipher is invertible because `keyTable` is a permutation: if `keyTable[5] = 0xF3`, then the inverse mapping is `inverseKeyTable[0xF3] = 5`. The decoder ships with this inverse table pre-baked as a compile-time constant (the same table the ONScripter-RU engine uses internally as `CompressedConversionTable`), so there's no runtime computation needed to derive it.

### Reversing each XOR pass

The forward operation for pass 1 is:

```
encoded = keyTable[original ^ 0x71] ^ 0x45
```

To reverse it, we algebraically undo each step from the outside in:

1. XOR away the outer constant: `encoded ^ 0x45 = keyTable[original ^ 0x71]`
2. Reverse the table lookup: `inverseKeyTable[encoded ^ 0x45] = original ^ 0x71`
3. XOR away the inner constant: `inverseKeyTable[encoded ^ 0x45] ^ 0x71 = original`

So the reverse formula is:

```
original = inverseKeyTable[encoded ^ 0x45] ^ 0x71
```

Pass 2 works identically, just with different constants:

```
original = inverseKeyTable[encoded ^ 0x86] ^ 0x23
```

### Streaming architecture

Because the XOR substitution is stateless (each byte transforms independently, with no dependency on surrounding bytes), the decoder doesn't need to buffer the entire file in memory. Instead, it chains three `io.Reader` implementations into a streaming pipeline:

```
Input .file
    │
    ▼
┌─────────────────────────────┐
│  os.File                    │  read 16-byte header, then stream payload
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  xorReader (pass 2)         │  inverseKeyTable[byte ^ 0x86] ^ 0x23
│  transforms bytes in-place  │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  zlib.NewReader             │  inflate (streaming decompression)
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  xorReader (pass 1)         │  inverseKeyTable[byte ^ 0x45] ^ 0x71
│  transforms bytes in-place  │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│  bufio.Writer               │  buffered write to output file
└─────────────┬───────────────┘
              │
              ▼
       Plaintext script
```

`io.Copy` drives the entire pipeline, pulling 32 KB at a time through the reader chain. At no point is the full file or the full decompressed output held in memory. This brings memory usage down to ~5 MB (compared to ~121 MB for a naive load-everything-then-process approach), while also being slightly faster since there are no intermediate allocations or copies.

### Security note

This is obfuscation, not strong encryption. The substitution cipher has no key derivation, no initialization vector, no chaining between bytes. Every occurrence of the same input byte always produces the same output byte within a given pass. The ZLIB layer in the middle makes casual inspection harder, but anyone with access to the source code (which is open source) can reverse it trivially, as this tool demonstrates.

### Performance

Benchmarked against the C decoder (`nscdec`) bundled with the ONScripter-RU engine, and an earlier buffered Go implementation, decoding a 4.5 MB `es.file` (19.6 MB decompressed):

|                               | Time      | Memory    |
|-------------------------------|-----------|-----------|
| **Go streaming (this tool)**  | **0.06s** | **~5 MB** |
| Go buffered (earlier version) | 0.07s     | ~121 MB   |
| C nscdec                      | 0.25s     | ~25 MB    |

The C decoder loses on speed because it writes output one byte at a time via `fputc()` with no buffering (19.6 million individual calls). The earlier Go version loses on memory because it allocates full copies at every stage. The streaming version avoids both problems.

## Usage

```
./decode <input.file> <output.txt>
```

### Example

```
./decode en.file en.txt
```

```
2026/02/23 00:20:36 Compressed: 4577085 bytes, Original: 19624546 bytes, Version: 110
2026/02/23 00:20:36 Decoded 19624546 bytes
2026/02/23 00:20:36 Written to en.txt
```

## Installation

Pre-built binaries for macOS (arm64), Linux (amd64), and Windows (amd64) are available on the [releases page](https://github.com/VictoriqueMoe/umineko_script_decoder/releases).

### Building from source

Requires Go 1.21+.

```
go build -o decode .
```

## Supported files

Any `.file` produced by the Umineko Project build system, including:

- `en.file` (English)
- `es.file` (Spanish)
- `ru.file` (Russian)
- `pt.file` (Portuguese)
- `cn.file` (Chinese Simplified)
- `cht.file` (Chinese Traditional)
- `it.file` (Italian)
- `tr.file` (Turkish)
- `idn.file` (Indonesian)
- `vi.file` (Vietnamese)
