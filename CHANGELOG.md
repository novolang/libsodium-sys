# Changelog

All notable changes to libsodium-sys are recorded here. The format
is [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.1.2 — 2026-09-24

The entry points that answer a C `int` were declared `Int`, so a -1
arrived as 4294967295 and no failure compared equal to -1; those 28
answers are now declared `i32`, and the two `uint32_t` answers of the
random source and the bound `randombytes_uniform` takes are declared
`u32`.

## 0.1.1 — 2026-09-24

The documentation and comments in plain prose; no declaration changed.

## 0.1.0 — 2026-09-16

The first release: sixty-two entry points of the libsodium C API, one
`@ffi` declaration each, and no logic.

### Added

- `libsodium` — the whole surface, in ten groups.
  - Initialisation and version: `sodium_init`, `sodium_version_string`,
    the two binary-interface numbers and `sodium_library_minimal`.
  - The random source: `randombytes_random`, `randombytes_uniform`,
    `randombytes_buf`, the seeded form and its seed length.
  - The constant-time helpers: `sodium_memzero`, `sodium_memcmp`,
    `sodium_compare`, `sodium_is_zero`, `sodium_increment` and the
    hexadecimal conversions both ways.
  - Guarded memory: `sodium_malloc`, `sodium_free`, `sodium_mlock` and
    `sodium_munlock`.
  - Generic hashing: `crypto_generichash` one-shot, the three streaming
    calls, the state size, the digest and key lengths and the key
    generator.
  - Secret-key authenticated encryption: `crypto_secretbox_easy`, its
    opening, the key generator and the three lengths.
  - Authenticated encryption with associated data:
    `crypto_aead_xchacha20poly1305_ietf_encrypt`, its decryption, the
    key generator and the three lengths.
  - Public-key authenticated encryption: `crypto_box_keypair`,
    `crypto_box_easy` and its opening, `crypto_box_seal` and its
    opening, and the five lengths.
  - Signatures: `crypto_sign_keypair`, `crypto_sign_detached`,
    `crypto_sign_verify_detached` and the three lengths.
  - Password hashing: `crypto_pwhash_str`, `crypto_pwhash_str_verify`,
    the stored length and the two interactive settings.
- `tests/libsodium_tests.nv` — twelve tests over the signatures. Every
  test works in memory, so the suite reads and writes no file and needs
  no privileges.

### The effect rows

A call that draws from the operating system's random source, or that
asks the kernel for memory it will not swap, is `[io, ffi]`. Everything
that only computes over buffers the caller supplied is `[ffi]`. That
puts `sodium_init`, the three drawing calls in `randombytes`, every
`*_keygen`, both key-pair generators, `crypto_box_seal`,
`crypto_pwhash_str` and the four memory calls on the wider row, and
leaves the rest of the surface — including every decryption and every
verification — on the narrow one.

### Where the surface came from

The groups are the ones libsodium's own documentation puts first: one
construction for each job, already chosen. The lengths are `#define`s
in the C header, which a binding cannot resolve, so the matching
`*bytes()` functions are declared and a caller asks rather than writing
a number.

### Not a `0.0.x` interface release

An interface release is the shape whose every `pub fn` body is a
`todo()`. Every `pub fn` here is an `@ffi` declaration with no body, so
`novo pkg publish` reads the package as a release with bodies and
refuses a `0.0.x` version for it. The first release of a bindings
package is therefore `0.1.0`.

### Unverified

libsodium was not installed on the machine this package was written on.
`novo pkg build` type-checks the declarations without it and is green;
`novo test` stopped at `cannot find -lsodium`, so the suite has never
been linked and no assertion in it has ever been observed to hold. The
declarations were checked against the libsodium documentation. Treat
the whole package as unmeasured until someone runs it against a real
libsodium.

### Named as missing

**Nothing is missing for the by-value rule.** libsodium passes every
buffer by address and returns no structure by value, so the rule that
removed entry points from libzstd-sys, libgit2-sys and libtree-sitter-sys
removes none here. The two exclusions that are the interface's and not a
choice of scope are both C function pointers.

**`randombytes_set_implementation`.** It replaces the random source with
a caller-supplied one, and it takes a structure of C function pointers.
The novo-lang foreign function interface passes integers, floats and
strings.

**`sodium_set_misuse_handler`.** It takes a C function pointer.

**The secret stream.** `crypto_secretstream_xchacha20poly1305_*`
encrypts a sequence of messages under one key, with rekeying and a
final tag. Its state has a `statebytes` accessor, so it is expressible
exactly as the streaming hash is; it is left out of the first release.

**Key derivation and key exchange.** `crypto_kdf_*` and `crypto_kx_*`
are left out of the first release.

**The raw `crypto_pwhash`.** It derives a key of a chosen length from a
password and a caller-supplied salt. `crypto_pwhash_str`, which stores
a password, is here.

**Base64 and the padding helpers.** `sodium_bin2base64`,
`sodium_base642bin`, `sodium_pad` and `sodium_unpad` are left out of
the first release. The hexadecimal pair is here.

**The memory protection changes.** `sodium_mprotect_noaccess`,
`sodium_mprotect_readonly` and `sodium_mprotect_readwrite` change the
protection of a `sodium_malloc` block, and are left out of the first
release.

**The detached and combined variants.** The detached forms of
`crypto_secretbox` and `crypto_box`, which keep the tag in a separate
buffer, and the combined form of `crypto_sign`, which copies the message
and the signature into one buffer, are left out. The other member of
each pair is here.

**The NaCl compatibility surface.** `crypto_stream`,
`crypto_onetimeauth`, `crypto_scalarmult`, `crypto_hash_sha256` and
`crypto_hash_sha512` are older constructions kept for compatibility, and
a program starting now uses the ones above instead.
