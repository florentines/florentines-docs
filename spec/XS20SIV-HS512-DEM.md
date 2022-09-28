# An XSalsa20-SIV-based DEM using HMAC-SHA-512

## Abstract

This specification describes the `XS20SIV-HS512` Data Encapsulation Mechanism (DEM) for the Florentine auth token 
format. This DEM provides deterministic authenticated encryption (DAE) or misuse-resistant authenticated encryption 
(MRAE), and is compactly committing. The DEM is based on the XSalsa20 stream cipher in a Synthetic IV (SIV) mode of
operation, using HMAC with SHA-512 (truncated to 256 bits) for authentication. These primitives are chosen due to their
wide available in the NaCl cryptographic library and its derivatives.

## Security Goals

The DEM is designed to achieve the following security goals:

* Deterministic Authenticated Encryption (DAE) as defined by [Rogaway and Shrimpton][1]. This implies standard security
  against chosen ciphertext attacks (IND-CCA) under the assumption that each DEM key is used only once. If a nonce or 
  random IV is supplied as associated data then the DEM achieves misuse-resistant authenticated encryption (MRAE), which
  implies IND-CCA security under standard assumptions, degrading to DAE security in the case of nonce reuse.
* Provide at least 128-bit security against classical attacks against confidentiality or authenticity in a multiuser
  setting.
* The authentication tag is [compactly-committing][2] (and key committing), ensuring that it is computationally 
  infeasible to find two (key, message) pairs that produce an identical authentication tag.



[1]: https://web.cs.ucdavis.edu/~rogaway/papers/keywrap.pdf
[2]: https://eprint.iacr.org/2017/664 