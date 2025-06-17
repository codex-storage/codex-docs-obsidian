---
related-to:
  - "[[Codex Encryption Design]]"
  - "[[Codex Encryption Implementation]]"
---
We want some kind of encryption of user data, definitely on by default (maybe even enforced in all cases).

Some observations:

- we want the encryption and decryption to happen on a machine under the user's control; otherwise somebody else would have access to the data
- encryption needs to happen before (network-level) erasure coding, otherwise repair is impossible (storage nodes would need the key to decrypt to be able to repair)
- we probably want to be able to encrypt and decrypt individual network blocks
- the only option is to use symmetric key encryption, because that's the only viable option for large data (and here "large" means already megabytes, not terabytes :)
- we don't need [authenticated encryption](https://en.wikipedia.org/wiki/Authenticated_encryption), because the data is already authenticated by the Merkle commitment and block Merkle proofs

I would argue that at the moment we should only do something very basic, that is, no key management, no access control, etc. The reason for this is that everything else is extremely complicated, and we have more pressing issues to solve. I'm not sure secure file-sharing is even possible in full generality... 

## Choice of cipher

We need to choose a [stream cipher](https://en.wikipedia.org/wiki/Stream_cipher). Which one to choose doesn't really matter as long as we don't fuck up the many subtle details. Some natural options:

- something based on the [AES block cipher](https://en.wikipedia.org/wiki/Advanced_Encryption_Standard)
- [Salsa20](https://en.wikipedia.org/wiki/Salsa20) or ChaCha
- just `XOR` with Shake or TurboShake (Shake is the SHA3 XOF)

AES is usually a default choice because it's a standard, it's well-known (you don't have to explain the choice), and hardware-accelerated. However, there are many details allowing you to shoot yourself in the foot.

Note: some modern stream ciphers allow independent decryption of pieces of the data. However, as we aim for block-by-block decryption, that already solves this issue.

## Initialization vector and padding

Many stream ciphers require an initialization vector and/or nonce, and potentially specific padding too.

As we want to encrypt fix-sized blocks (divisible by eg. 16 bytes), padding is not required.

The initialization vector (IV) is an additional piece of random information required to decrypt (often this doesn't have to be secret). The advantages of using a random IV are easy to see: Encrypting the same data with the same key twice results in different ciphertext, so you cannot do comparision of encrypted data. However, an IV may be also required just because of how a cipher is constructed.

Often the security of the cipher breaks if the IV / nonce is reused, so it's very important in general for it be random and never reused.

An approach to ensure that is synthetic initialization vector (SIV), which is computed from the data and the key (and maybe other additional information) by a deterministic process, then encrypted using the key and added to the ciphertext. This has the disadvantage of additional calculation over the data (usually some kind of hash function). In our case a further disadvantage would be a slightly larger ciphertext than plaintext (not compatible with fixed size blocks and per-block encryption).

Note that in some situations, by manipulating the IV one can manipulate the first bytes of the plaintext.

## AES modes of operation

AES, as many other stream ciphers, is based a block cipher. The API of the AES _block cipher_ is:

    encrypt : Key x PlainBlock  -> CipherBlock
    decrypt : Key x CipherBlock -> PlainBlock
    
where blocks are 128 bits, and key can be 128, 192 or 256 bit (depending on the AES variant).

There are ["modes of operations"](https://en.wikipedia.org/wiki/Block_cipher_mode_of_operation) which transform block ciphers into stream ciphers.

Some of these are:

- ECB (Electronic code book) - totally broken, never ever use this
- CBC (Cipher block chaining)
- CFB (Cipher feedback)
- OFB (Output feedback)
- CTR (Counter mode)
- GCM (Galois/Counter Mode) - an authenticated version of counter mode

These has some advantages and disadvantages; for example propagation of errors / bit flips, possibility of parallel encryption / decryption, etc.

A common choice these days is AES-GCM-SIV. There are subtle details though, for example you shouldn't encrypt too much data with the same key, and the nonce space is too small.

Note that we don't really need the advantages of AES-GCM-SIV because of how Codex is constructed.

## A note on randomness

Getting good quality random numbers for creation of keys is notoriously tricky, and easy to mess up, as it depends on the hardware, operating system, programming language, libraries, APIs, and even extrenal circumstances (for example cloud virtual machines at startup usually have less entropy then machines used by humans).

As weak randomness can have catastrophic consequences, special care should be given to this.

A common mitigation strategy is to include additional entropy based for example on the data to encrypt. The disadvantage of this is more computation.

## A proposal

Based on the above, I have a very simple proposal:

- use a freshly generated random master key (at least 256 bits) per dataset (generated and kept on the user's machine)
- derive a new encryption key and also an IV for each block from the master key and the block index
- use for example AES192-CBC

For example, we could have

    key = SHA256( MASTER_KEY || block_index ), truncated to 192 bits
    IV  = SHA256( MASTER_IV  || block_index ), truncated to 128 bits
    
where both `MASTER_KEY` and `MASTER_IV` are 256 bit random numbers, and `||` denotes concatenation.

If storing 512 bits (as opposed to a 256 bit minimum) of key material is a problem, we could derive both by the same key, for example as

    key' = SHA256( MASTER_KEY || 0x01 || block_index ), truncated to 192 bits
    IV'  = SHA256( MASTER_KEY || 0x02 || block_index ), truncated to 128 bits

## What if the user loses their key?

Of course, if the key is lost, the data becames unaccessible. However, this is not really different from losing the CID in a private unencrypted setting.

We can make the life of the user easier by combining the CID with the encryption key into a single string.

However, as the CID and the key are fundamentelly different objects, I strongly believe that the user should be able to separate and re-combine them, and that the nature of them should be explained to the user very carefully.

## Some further thoughts

Encryption in the cloud storage setting is a very very deep rabbit hole.

### Proxy re-encryption and updateable encryption

A general problem with long-term storage of encrypted data, is that once the key leaks, it's fully broken. 

An important component of any solution for that would be something like "proxy re-encryption", that is, the ability of a server to rotate the encryption key without actually being able to decrypt the data. 

Unfortuntaly, we are dealing with large data, so our only choice is symmetric key encryption, which severly limits the options:

- the user downloads their data, decrypts, re-encrypts and re-uploads. Clearly we would like to avoid this.
- add further layers of encryptions with new keys (like an onion): The obvious problem is that both the key, and the decryption time grows linearly with each such re-encryption
- key-homomorphic encryption: using a block cipher which is a homomorphic function from the key space to a permutation space
- there are more complicated UE schemes which I need to study

There is some literature about key-homomorphic pseudo-random functions, for example this paper from 2014:
- [Dan Boneh et al.: Key Homomorphic PRFs and Their Applications](https://eprint.iacr.org/2015/220)

I'm not sure about the security properties (apparently the strong mathematical structure has effects on the security). Speed can be also a problem: these constructions are presumably significantly slower than say AES.

A further complication of re-encrypting data is that we are also erasure coding the encrypted data, so the erasure coding needs to be redone too, which is non-local (re-encrypting the original data can ideally happen locally).

### Key management

Some form of distributed key management could provide added value for the users:

- convenience of not storing too many keys
- partial protection against losing keys
- (limited forms of) access control
- peer discovery / authentication (?) for file sharing
- etc

Of course key management can be very complicated and very tricky, and would also add a lot of extra complexity to the protocol (question: maybe it can be separate from the core protocol? however, if we already have a distributed network, you want to use that...)

Probably the most interesting paper I've seen in this area is:

- [Updatable Oblivious Key Management for Storage Systems](https://eprint.iacr.org/2019/1275) by Jarecki, Krawczyk and Resch 

I believe it's worth to think about this direction, but I would recommend not to do anything like this right now.

### File sharing

Many people simply want (secure) file sharing. This comes with a lot of new problems:

- small files
- many people want to use passwords instead of keys
- they most probably don't want to remember many different CIDs, and instead want the metadata to be also stored in the cloud
- the latter requires long-term identity
- they want access control (probably including revocation!)
- etc etc

Secure file sharing looks like another very hard problem.
