# Public-key authenticated encryption

## Example

```c
#define MESSAGE (const unsigned char *) "test"
#define MESSAGE_LEN 4
#define CIPHERTEXT_LEN (crypto_box_MACBYTES + MESSAGE_LEN)

unsigned char alice_publickey[crypto_box_PUBLICKEYBYTES];
unsigned char alice_secretkey[crypto_box_SECRETKEYBYTES];
crypto_box_keypair(alice_publickey, alice_secretkey);

unsigned char bob_publickey[crypto_box_PUBLICKEYBYTES];
unsigned char bob_secretkey[crypto_box_SECRETKEYBYTES];
crypto_box_keypair(bob_publickey, bob_secretkey);

unsigned char nonce[crypto_box_NONCEBYTES];
unsigned char ciphertext[CIPHERTEXT_LEN];
randombytes_buf(nonce, sizeof nonce);
if (crypto_box_easy(ciphertext, MESSAGE, MESSAGE_LEN, nonce,
                    bob_publickey, alice_secretkey) != 0) {
    /* error */
}

unsigned char decrypted[MESSAGE_LEN];
if (crypto_box_open_easy(decrypted, ciphertext, CIPHERTEXT_LEN, nonce,
                         alice_publickey, bob_secretkey) != 0) {
    /* message for Bob pretending to be from Alice has been forged! */
}
```

## Purpose

Using public-key authenticated encryption, Bob can encrypt a confidential
message specifically for Alice, using Alice's public key.

Using Bob's public key, Alice can compute a shared secret key. Using Alice's
public key and his secret key, Bob can compute the exact same shared secret key.
That shared secret key can be used to verify that the encrypted message was not
tampered with, before eventually decrypting it.

Alice only needs Bob's public key, the nonce and the ciphertext. Bob should
never ever share his secret key, even with Alice.

And in order to send messages to Alice, Bob only needs Alice's public key. Alice
should never ever share her secret key either, even with Bob.

Alice can reply to Bob using the same system, without having to generate a
distinct key pair.

The nonce doesn't have to be confidential, but it should be used with just one
invocation of `crypto_box_open_easy()` for a particular pair of public and
secret keys.

One easy way to generate a nonce is to use `randombytes_buf()`, considering the
size of the nonces the risk of any random collisions is negligible. For some
applications, if you wish to use nonces to detect missing messages or to ignore
replayed messages, it is also acceptable to use a simple incrementing counter as
a nonce. An better alternative is to use the `crypto_secretstream()` API.

When doing so you must ensure that the same value can never be re-used (for
example you may have multiple threads or even hosts generating messages using
the same key pairs).

As stated above, senders can decrypt their own messages, and compute a valid
authentication tag for any messages encrypted with a given shared secret key.
This is generally not an issue for online protocols. If this is not acceptable,
check out the Sealed Boxes section, as well as the Key Exchange section in this
documentation.

## Key pair generation

```c
int crypto_box_keypair(unsigned char *pk, unsigned char *sk);
```

The `crypto_box_keypair()` function randomly generates a secret key and a
corresponding public key. The public key is put into `pk`
(`crypto_box_PUBLICKEYBYTES` bytes) and the secret key into `sk`
(`crypto_box_SECRETKEYBYTES` bytes).

```c
int crypto_box_seed_keypair(unsigned char *pk, unsigned char *sk,
                            const unsigned char *seed);
```

Using `crypto_box_seed_keypair()`, the key pair can also be deterministically
derived from a single key `seed` (`crypto_box_SEEDBYTES` bytes).

```c
int crypto_scalarmult_base(unsigned char *q, const unsigned char *n);
```

In addition, `crypto_scalarmult_base()` can be used to compute the public key
given a secret key previously generated with `crypto_box_keypair()`:

```c
unsigned char pk[crypto_box_PUBLICKEYBYTES];
crypto_scalarmult_base(pk, sk);
```

## Combined mode

In combined mode, the authentication tag and the encrypted message are stored
together. This is usually what you want.

```c
int crypto_box_easy(unsigned char *c, const unsigned char *m,
                    unsigned long long mlen, const unsigned char *n,
                    const unsigned char *pk, const unsigned char *sk);
```

The `crypto_box_easy()` function encrypts a message `m` whose length is `mlen`
bytes, with a recipient's public key `pk`, a sender's secret key `sk` and a
nonce `n`.

`n` should be `crypto_box_NONCEBYTES` bytes.

`c` should be at least `crypto_box_MACBYTES + mlen` bytes long.

This function writes the authentication tag, whose length is
`crypto_box_MACBYTES` bytes, in `c`, immediately followed by the encrypted
message, whose length is the same as the plaintext: `mlen`.

`c` and `m` can overlap, making in-place encryption possible. However do not
forget that `crypto_box_MACBYTES` extra bytes are required to prepend the tag.

```c
int crypto_box_open_easy(unsigned char *m, const unsigned char *c,
                         unsigned long long clen, const unsigned char *n,
                         const unsigned char *pk, const unsigned char *sk);
```

The `crypto_box_open_easy()` function verifies and decrypts a ciphertext
produced by `crypto_box_easy()`.

`c` is a pointer to an authentication tag + encrypted message combination, as
produced by `crypto_box_easy()`. `clen` is the length of this authentication
tag + encrypted message combination. Put differently, `clen` is the number of
bytes written by `crypto_box_easy()`, which is `crypto_box_MACBYTES` + the
length of the message.

The nonce `n` has to match the nonce used to encrypt and authenticate the
message.

`pk` is the public key of the sender that encrypted the message. `sk` is the
secret key of the recipient that is willing to verify and decrypt it.

The function returns `-1` if the verification fails, and `0` on success. On
success, the decrypted message is stored into `m`.

`m` and `c` can overlap, making in-place decryption possible.

## Detached mode

Some applications may need to store the authentication tag and the encrypted
message at different locations.

For this specific use case, "detached" variants of the functions above are
available.

```c
int crypto_box_detached(unsigned char *c, unsigned char *mac,
                        const unsigned char *m,
                        unsigned long long mlen,
                        const unsigned char *n,
                        const unsigned char *pk,
                        const unsigned char *sk);
```

This function encrypts a message `m` of length `mlen` with a nonce `n` and a
secret key `sk` for a recipient whose public key is `pk`, and puts the encrypted
message into `c`.

Exactly `mlen` bytes will be put into `c`, since this function does not prepend
the authentication tag.

The tag, whose size is `crypto_box_MACBYTES` bytes, will be put into `mac`.

```c
int crypto_box_open_detached(unsigned char *m,
                             const unsigned char *c,
                             const unsigned char *mac,
                             unsigned long long clen,
                             const unsigned char *n,
                             const unsigned char *pk,
                             const unsigned char *sk);
```

The `crypto_box_open_detached()` function verifies and decrypts an encrypted
message `c` whose length is `clen` using the recipient's secret key `sk` and the
sender's public key `pk`.

`clen` doesn't include the tag, so this length is the same as the plaintext.

The plaintext is put into `m` after verifying that `mac` is a valid
authentication tag for this ciphertext, with the given nonce `n` and key `k`.

The function returns `-1` if the verification fails, or `0` on success.

## Precalculation interface

Applications that send several messages to the same receiver or receive several
messages from the same sender can gain speed by calculating the shared key only
once, and reusing it in subsequent operations.

```c
int crypto_box_beforenm(unsigned char *k, const unsigned char *pk,
                        const unsigned char *sk);
```

The `crypto_box_beforenm()` function computes a shared secret key given a public
key `pk` and a secret key `sk`, and puts it into `k` (`crypto_box_BEFORENMBYTES`
bytes).

```c
int crypto_box_easy_afternm(unsigned char *c, const unsigned char *m,
                            unsigned long long mlen, const unsigned char *n,
                            const unsigned char *k);

int crypto_box_open_easy_afternm(unsigned char *m, const unsigned char *c,
                                 unsigned long long clen, const unsigned char *n,
                                 const unsigned char *k);

int crypto_box_detached_afternm(unsigned char *c, unsigned char *mac,
                                const unsigned char *m, unsigned long long mlen,
                                const unsigned char *n, const unsigned char *k);

int crypto_box_open_detached_afternm(unsigned char *m, const unsigned char *c,
                                     const unsigned char *mac,
                                     unsigned long long clen, const unsigned char *n,
                                     const unsigned char *k);
```

The `_afternm` variants of the previously described functions accept a
precalculated shared secret key `k` instead of a key pair.

Like any secret key, a precalculated shared key should be wiped from memory (for
example using `sodium_memzero()`) as soon as it is not needed any more.

## Constants

* `crypto_box_PUBLICKEYBYTES`
* `crypto_box_SECRETKEYBYTES`
* `crypto_box_MACBYTES`
* `crypto_box_NONCEBYTES`
* `crypto_box_SEEDBYTES`
* `crypto_box_BEFORENMBYTES`

## Algorithm details

* Key exchange: X25519
* Encryption: XSalsa20 stream cipher
* Authentication: Poly1305 MAC

## Notes

The original NaCl `crypto_box` API is also supported, albeit not recommended.

`crypto_box()` takes a pointer to 32 bytes before the message, and stores the
ciphertext 16 bytes after the destination pointer, the first 16 bytes being
overwritten with zeros. `crypto_box_open()` takes a pointer to 16 bytes before
the ciphertext and stores the message 32 bytes after the destination pointer,
overwriting the first 32 bytes with zeros.

The `_easy` and `_detached` APIs are faster and improve usability by not
requiring padding, copying or tricky pointer arithmetic.
