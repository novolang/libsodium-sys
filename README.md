# libsodium-sys

libsodium is a modern, easy-to-use software library for encryption,
decryption, signatures, password hashing and more. It is a portable,
cross-compilable, installable and packageable fork of NaCl, with a
compatible but extended API. For each job its high-level API offers one
construction, already chosen. The library is documented in the
[libsodium documentation](https://doc.libsodium.org/). This package
declares sixty-two of its entry points to novo-lang, one declaration
each.

Every function here is a declaration of a function in libsodium. The
package contains no logic of its own, and it does nothing without the C
library installed. The sixty-two entry points cover initialisation, the
random source, the constant-time helpers, guarded memory, hashing,
secret-key and public-key authenticated encryption, signatures and
password hashing. The section "What is not included" says what a
program cannot do with them alone.

## What it is

A **key** is a fixed-length block of secret bytes. Every construction
below states its key length. The library reads exactly that many bytes
from the key's address, so a shorter buffer is a programming error.

A **nonce** is a number used once. It is public, it goes beside the
ciphertext, and it may never repeat under one key. Repeating it does
not reveal the key. It can reveal the two messages encrypted under it,
and with a Poly1305 tag it lets an attacker forge messages.

An **authentication tag**, also called a MAC, is the short value that
proves a ciphertext was not altered. Every encryption call here
produces one, and every decryption call checks it. A decryption that
fails the check writes nothing a caller may use.

**Secret-key authenticated encryption** is for two parties that already
share a key. **Public-key authenticated encryption** is for two parties
that each hold a key pair and know each other's public half. A
**sealed box** is the anonymous form. The sender needs only the
recipient's public key, and the recipient cannot tell who wrote it.

A **signature** proves that the holder of a secret key produced a
message, to anyone holding the matching public key. It is not
encryption, and a signed message is readable by everyone.

**Password hashing** turns a password into a value that can be stored.
It is deliberately slow and deliberately memory-hungry, so that a
stolen store is expensive to attack. libsodium's function is Argon2id,
and the work is set by two numbers: the operations limit and the memory
limit.

## Install

```
novo pkg add libsodium-sys
```

Adding the package does not install the C library. On Debian and Ubuntu
the library and its header come from the system package
`libsodium-dev`:

```
sudo apt install libsodium-dev
```

On macOS the Homebrew formula is `libsodium`. On other systems the
library builds from the libsodium source with `./configure && make`.

## Example

A message encrypted under a shared key and read back:

```novo ignore
use libsodium

fn main() [io, ffi]
    // Call this once, before anything else in the library.
    if libsodium.sodium_init() == -1
        println("libsodium did not initialise")
        return

    let key = ptr.alloc(libsodium.crypto_secretbox_keybytes())
    libsodium.crypto_secretbox_keygen(key)

    // A nonce is public and must not repeat under this key.
    let noncelen = libsodium.crypto_secretbox_noncebytes()
    let nonce = ptr.alloc(noncelen)
    libsodium.randombytes_buf(nonce, noncelen)

    let message = "the quick brown fox jumps over the lazy dog"
    let m = ptr.alloc(64)
    ptr.write_bytes_buf(m, bytes.from_str(message))

    // The box is the message plus the tag.
    let maclen = libsodium.crypto_secretbox_macbytes()
    let c = ptr.alloc(43 + maclen)
    let _ = libsodium.crypto_secretbox_easy(c, m, 43, nonce, key)

    let back = ptr.alloc(64)
    if libsodium.crypto_secretbox_open_easy(back, c, 43 + maclen, nonce, key) == -1
        println("the box did not verify")
        return
    println(bytes.to_str(ptr.read_bytes_n(back, 43)))

    // A key leaves memory by being overwritten, not by being freed.
    libsodium.sodium_memzero(key, libsodium.crypto_secretbox_keybytes())
    ptr.free(back)
    ptr.free(c)
    ptr.free(m)
    ptr.free(nonce)
    ptr.free(key)
```

The example is not compiled, because it links against libsodium and the
link fails where that library is not installed. The same calls are in
`tests/libsodium_tests.nv`.

## What the package contains

| Module | Contents |
| --- | --- |
| `libsodium` | Every entry point, in ten groups: initialisation and version, the random source, the constant-time helpers, guarded memory, generic hashing, secret-key authenticated encryption, authenticated encryption with associated data, public-key authenticated encryption, signatures, and password hashing. |

The ten groups and their sizes:

| Group | Entry points | What it does |
| --- | --- | --- |
| Initialisation and version | 5 | Starts the library and reports which build is installed. |
| The random source | 5 | Draws unpredictable bytes and numbers, and a reproducible stream from a seed. |
| Constant-time helpers | 7 | Compares, clears, counts and converts secret bytes without leaking them through timing. |
| Guarded memory | 4 | Allocates a block between guard pages that is not swapped out, and locks and unlocks ordinary pages. |
| Generic hashing | 8 | BLAKE2b, in one call and a piece at a time, keyed or unkeyed. |
| Secret-key encryption | 6 | XSalsa20-Poly1305, encrypting and authenticating under a key both parties hold. |
| Authenticated encryption with associated data | 6 | XChaCha20-Poly1305, with a header that is authenticated and not encrypted. |
| Public-key encryption | 10 | X25519 key pairs, boxes between two parties, and anonymous sealed boxes. |
| Signatures | 6 | Ed25519 key pairs, detached signatures and their verification. |
| Password hashing | 5 | Argon2id, into a storable string and back. |

## How to choose an entry point

`crypto_secretbox_easy` is for two parties that already share a key.
It is the shortest way to encrypt, and it has no place for a header.

`crypto_aead_xchacha20poly1305_ietf_encrypt` is the same job with
associated data, such as a header, a sequence number or a record
identifier, that travels in the clear and is still covered by the tag. Its nonce is
24 bytes, which is long enough to choose at random for every message
rather than counting.

`crypto_box_easy` is for two parties that each hold a key pair. The
recipient learns which sender wrote the message, because verifying it
needs the sender's public key.

`crypto_box_seal` is for a sender with no key pair at all. Only the
recipient's public key is needed, and the sender cannot read the
message again afterwards.

`crypto_sign_detached` is for a message everyone may read and nobody
may alter. Use it where the point is provenance rather than secrecy.

`crypto_generichash` is a hash, not an encryption. With a key it is a
message authentication code. Without one it is a fingerprint of data
that is not secret.

`crypto_pwhash_str` is for a password, and only for a password.
Hashing a password with `crypto_generichash` is fast, and a fast hash
lets an attacker who stole the store try many guesses a second.

## The rules a user needs

1. **`sodium_init` comes first.** Call it before any other entry
   point. It is safe to call more than once and from several threads.
   It answers 0 the first time, 1 when the library was already
   initialised, and -1 on failure, after which the library is not safe
   to use.
2. **A pointer is an `Int`, and zero is null.** Every key, nonce,
   message and tag is an address the caller reserved with `ptr.alloc`,
   wrote with `ptr.write_bytes_buf` and read back with
   `ptr.read_bytes_n`.
3. **The lengths are calls, not numbers.** The C header spells them as
   `#define`s, which a binding cannot resolve, so every length is asked
   for, such as `crypto_secretbox_keybytes()`, `crypto_box_noncebytes()`
   and `crypto_sign_bytes()`. The current values are in the table
   below, and asking is still the correct way to reserve a buffer.
4. **A failure is -1 and it is the whole report.** There is no error
   code and no error string. A -1 from a decryption or a verification
   means the data was altered, or the key, nonce or associated data is
   wrong. Which of those it is cannot be learned.
5. **Nothing in the destination buffer may be used after a -1.** The
   documentation promises nothing about its contents. Treat it as
   unwritten.
6. **A nonce may never repeat under one key.** Draw a 24-byte nonce
   with `randombytes_buf` for every message, or count with
   `sodium_increment` and never restart the count.
7. **Compare secrets with `sodium_memcmp`.** Comparing byte by byte
   stops at the first difference, and how long that took is a measurement
   an attacker can make. `sodium_memcmp` reports equality only, and
   `sodium_compare` reports order.
8. **An answer of -1 compares equal to -1.** The entry points that
   answer a C `int` are declared `i32`, the width of that `int`, so
   `sodium_compare` and every call that fails with -1 need no
   conversion. `randombytes_random` and `randombytes_uniform` answer a
   C `uint32_t` and are declared `u32`, so their answers are never
   negative. A length is a C `size_t` in some calls and an `unsigned
   long long` in others, and both are 64 bits wide, so every length is
   an `Int`.
9. **A key leaves memory through `sodium_memzero`.** Freeing a buffer
   does not clear it, and a plain loop that clears it may be removed by
   the compiler.
10. **A block from `sodium_malloc` is released by `sodium_free`.**
    Never by `ptr.free`, and never the other way round. The guarded
    block is placed against an unreadable page and carries a canary, and
    `sodium_free` checks the canary before returning the pages.
11. **`sodium_mlock` may be refused.** A process has a limit on how much
    memory it may lock, and a -1 here is ordinary rather than a bug.
12. **The buffer lengths a caller must reserve.**

    | Buffer | Length in bytes | From |
    | --- | --- | --- |
    | secret-key key | 32 | `crypto_secretbox_keybytes` |
    | secret-key nonce | 24 | `crypto_secretbox_noncebytes` |
    | secret-key tag | 16 | `crypto_secretbox_macbytes` |
    | XChaCha20-Poly1305 key | 32 | `crypto_aead_xchacha20poly1305_ietf_keybytes` |
    | XChaCha20-Poly1305 nonce | 24 | `crypto_aead_xchacha20poly1305_ietf_npubbytes` |
    | XChaCha20-Poly1305 overhead | 16 | `crypto_aead_xchacha20poly1305_ietf_abytes` |
    | X25519 public key | 32 | `crypto_box_publickeybytes` |
    | X25519 secret key | 32 | `crypto_box_secretkeybytes` |
    | sealed box overhead | 48 | `crypto_box_sealbytes` |
    | Ed25519 public key | 32 | `crypto_sign_publickeybytes` |
    | Ed25519 secret key | 64 | `crypto_sign_secretkeybytes` |
    | Ed25519 signature | 64 | `crypto_sign_bytes` |
    | BLAKE2b digest, recommended | 32 | `crypto_generichash_bytes` |
    | BLAKE2b key, recommended | 32 | `crypto_generichash_keybytes` |
    | stored password hash | 128 | `crypto_pwhash_strbytes` |
    | deterministic seed | 32 | `randombytes_seedbytes` |

13. **The digest length is part of a BLAKE2b hash.** A digest is 16 to
    64 bytes and a key 16 to 64 bytes. A 16-byte digest is not the first
    16 bytes of a 32-byte digest, and the length given to
    `crypto_generichash_init` must be the one given to
    `crypto_generichash_final`.
14. **The streaming hash state is a buffer of an asked-for size.**
    Reserve `crypto_generichash_statebytes()` bytes and pass the
    address. The fields inside it are not documented, and nothing
    outside the library reads them. The C header declares the state
    with 64-byte alignment, and a buffer from `ptr.alloc` is not
    promised that alignment.
15. **`randombytes_buf_deterministic` is not a random source.** The
    same seed always gives the same bytes. It is for a reproducible
    test, and never for a key or a nonce.
16. **An Ed25519 secret key carries its public key.** The 64-byte
    secret key is the 32-byte seed followed by the 32-byte public key,
    so a signer keeps one buffer.
17. **Associated data is not stored anywhere.** The decryption must be
    handed exactly the bytes the encryption was handed, from wherever
    the program kept them, or the tag fails.

## Timing behaviour

The encryption, signature and hashing constructions in this package are
constant-time with respect to secret data. The number of operations and
the sequence of memory accesses do not depend on a key or a plaintext.
That is a property of libsodium, and this package adds nothing to it.

`sodium_memcmp`, `sodium_compare`, `sodium_is_zero`, `sodium_increment`
and `sodium_bin2hex` are the constant-time forms of operations whose
obvious implementation is not constant-time. Use them for anything
secret.

`crypto_pwhash_str` is deliberately slow, and how long it takes is set
by the operations and memory limits. Argon2id reads memory at addresses
that do not depend on the password in the first half of its first pass,
and at addresses that do for the rest. RFC 9106 section 3.4.1.3.

`sodium_hex2bin` is constant-time for the conversion itself. Whether
the text parsed at all is reported, and that report depends on the
input, which is a public fact about text that came from outside.

## What is not included

- **`randombytes_set_implementation`.** It replaces the random source
  with a caller-supplied one, and it takes a structure of C function
  pointers. The novo-lang foreign function interface passes integers,
  floats and strings.
- **`sodium_set_misuse_handler`.** It takes a C function pointer.
- **The secret stream**, `crypto_secretstream_xchacha20poly1305_*`.
  It encrypts a sequence of messages under one key with rekeying and a
  final tag. Its state has a `statebytes` accessor, so it can be
  declared. It is left out of this release.
- **Key derivation**, `crypto_kdf_*`, and key exchange, `crypto_kx_*`.
  They are left out of this release.
- **The raw `crypto_pwhash`.** It derives a key of a chosen length from
  a password and a caller-supplied salt. `crypto_pwhash_str`, which
  stores a password, is here.
- **Base64**, `sodium_bin2base64` and `sodium_base642bin`, and the
  padding helpers `sodium_pad` and `sodium_unpad`. They are left out of
  this release, and hexadecimal is here.
- **`sodium_mprotect_noaccess`, `sodium_mprotect_readonly` and
  `sodium_mprotect_readwrite`.** They change the protection of a
  `sodium_malloc` block. They are left out of this release.
- **The detached forms** of `crypto_secretbox` and `crypto_box`, which
  keep the tag in a separate buffer. The combined forms are here.
- **The lower-level constructions**, `crypto_stream`,
  `crypto_onetimeauth`, `crypto_scalarmult`, `crypto_hash_sha256` and
  `crypto_hash_sha512`. The high-level constructions above are built on
  them.
- **The `_easy` signature form**, `crypto_sign` and `crypto_sign_open`,
  which copy the message and the signature into one buffer. The
  detached form is here.

libsodium passes every buffer by address and returns no structure by
value, so every entry point it has could be declared here. The first
two omissions above take C function pointers, which the novo-lang
foreign function interface cannot pass. The rest are left out of this
release.

## Related packages

[crypto-nv](https://novo-lang.org/packages/crypto-nv) holds SHA-256,
SHA-512, SHA-1, MD5 and HMAC in novo-lang, with no C library. It has
none of the constructions this package declares.

The same primitives written in novo-lang are
[blake2-nv](https://novo-lang.org/packages/blake2-nv),
[chacha20-nv](https://novo-lang.org/packages/chacha20-nv),
[x25519-nv](https://novo-lang.org/packages/x25519-nv),
[ed25519-nv](https://novo-lang.org/packages/ed25519-nv) and
[argon2-nv](https://novo-lang.org/packages/argon2-nv). All five are
published as interface releases. Every function in them is declared and
none has a body yet, so a program that must encrypt, sign or hash a
password today uses this package. When their functions have bodies,
they are the choice for a program that must build for a microcontroller
or for WebAssembly, or does without a C toolchain. This package is the
choice for a program that must use libsodium itself, or interoperate
with a program that does.

## Tests

`tests/libsodium_tests.nv` holds twelve tests over the sixty-two entry
points. They call the C library, so `novo test` needs libsodium
installed and linkable:

```
novo test tests/libsodium_tests.nv
```

`novo pkg build` type-checks the declarations and needs nothing
installed.

The suite runs against libsodium 1.0.18 on Ubuntu, and all twelve
tests pass there.

Every test works in memory, so the suite reads and writes no file and
needs no privileges. The version test accepts both a full and a minimal
build. The helper test asserts that clearing, comparing, ordering and
incrementing agree with each other. The hexadecimal test round-trips
the message and asserts that text which is not hexadecimal is refused.
The guarded-memory test accepts both answers from `sodium_mlock`,
because a process limit may refuse it. The hash test asserts that the
one-shot and the streaming digest match and that a 16-byte digest is not
a prefix of the 32-byte one. Each encryption test round-trips the
message and then asserts a failure. The secret box is given an altered
byte, the authenticated encryption a changed associated-data length,
and the box the wrong sender's public key. The sealed-box test asserts
that the same message sealed twice gives two different ciphertexts. The
signature test asserts that a signature over a shorter message does not
verify. The password test asserts that the same password stored twice
gives two different strings.

## Licence

Apache-2.0. See [LICENSE](LICENSE).

libsodium itself is distributed under the ISC licence, and installing
it is the reader's own step.
