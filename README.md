# Florentines Documentation

Florentines are a new authorization token format, intended to replace some uses of [JSON Web Tokens](https://jwt.io) (JWTs). This repository contains draft specifications for the token format and related standards, as well as general documentation for developers.

They are named after ["Florentine" biscuits](https://www.deliaonline.com/recipes/main-ingredient/chocolate/florentines), continuing the theme of [cookies](https://en.wikipedia.org/wiki/HTTP_cookie), [macaroons](https://research.google/pubs/pub41892/), [biscuits](https://www.biscuitsec.org), etc.

> "If there was such a thing as a prize for the very best biscuit in the world, one bite of a Florentine would tell you this was the winner."

![Image of Florentine biscuits](/images/Florentines.jpeg)

(Image credit: [becks & posh](https://becksposhnosh.blogspot.com/2005/11/spiced-sesame-orange-florentines-with.html) License: [CC-By-NC-ND](https://creativecommons.org/licenses/by-nc-nd/3.0/))

## Why another new auth token format?

The development of Florentines is driven by the observation that many authorization or identity tokens contain 
[personally identifiable information (PII)](https://ico.org.uk/for-organisations/guide-to-data-protection/guide-to-the-general-data-protection-regulation-gdpr/key-definitions/what-is-personal-data/) 
such as usernames, email addresses, location, or other potentially sensitive information. However, almost all existing auth token formats provide only digital
signatures and do not protect the confidentiality of claims within the token. Florentines eschew digital signatures in favour of
[authenticated encryption](https://neilmadden.blog/2018/11/14/public-key-authenticated-encryption-and-why-you-want-it-part-i/),
which provides similar authenticity guarantees to a signature but also ensures the contents are encrypted.

The gold standard of encryption is [forward secrecy](https://en.wikipedia.org/wiki/Forward_secrecy), which ensures that 
the confidentiality of messages is assured even if the long-term keys of either sender or recipient are subsequently compromised.
Previously, this property was only available through interactive protocols such as TLS or secure messengers like Signal, and it
is not provided by JWTs or other existing token formats. Florentines support a notion of "replyable encryption", which is a half-way
house between one-shot encryption formats like JWTs or PGP and forward-secret interactive protocols like TLS. An initial Florentine
message does not provide forward secrecy, but any reply to it does. This provides an easy way to upgrade the security of
redirect-based request-response protocols such as OAuth or OpenID Connect.

Some other differences between Florentines and JWTs are as follows:

* Support for encrypting the same token to multiple recipients in a very compact format. JWTs only support this for the little-used JSON Serialization format. Additional recipients add around 50 bytes of overhead, allowing 7-8 recipients to be encoded in the same space as a single RSA-3072 signature.
* Support for any number of payload blocks in a token, rather than putting everything into a single payload section. This allows you to rigidly separate untrusted user-supplied data from trusted system-generated data.
* Payload blocks can be individually encrypted, using misuse-resistant authenticated encryption.
* No algorithm identifiers or key material are communicated or negotiated in-band within the header, preventing any kind of `alg=none` attacks. Algorithm identifiers are [associated with keys instead](https://neilmadden.blog/2018/09/30/key-driven-cryptographic-agility/).
* Support for Macaroon-style caveats, which can be appended to a token after it has been created to attenuate the privileges granted by a token. Just like in the original Macaroon paper, Florentines use simple HMAC-SHA-256 authentication, rather than complicated and expensive public key signature schemes.

## Technical details

This section is intended for cryptography engineers and assumes some background knowledge of applied cryptography.

If you are familiar with Macaroons, then you can think of a Florentine as a Macaroon with a fresh random HMAC root key. This root key 
is encrypted for each recipient using an authenticated KEM and the encrypted blobs are prefixed to the Macaroon to create the Florentine. 
The identifier portion of the Macaroon part is divided into an arbitrary number of sections, each of which can optionally be encrypted 
(using a variant of SIV mode to provide misuse-resistance). The authenticated KEM allows recipients to be sure that the encrypted root
key was produced by a known sender, and it also provides "insider security": no recipient can produce a new message that appears to come
from the same sender, even though they know the root key.

To go into a bit more details, Florentines use a typical KEM/DEM paradigm for hybrid encryption (and authentication), using the following
general approach:

1. First, a fresh 256-bit DEM key is generated.
2. This DEM key is used to MAC and encrypt the payload sections of the token. The DEM must provide Deterministic Authenticated Encryption (DAE, from Shrimpton/Rogaway) and must be compactly-commiting.
3. An authenticated KEM is then used to encapsulate the fresh DEM key for each recipient. The KEM takes the authentication tag from step 2 as an additional input (i.e., it is a Tag-KEM) and incorporates it into the key-wrapping process to provide insider-security. This is what prevents a genuine recipient of one message from simply creating a new message reusing the same DEM key, and this is also why the DEM must be compactly commiting.

The initial algorithm suite uses the following cryptographic primitives to implement this scheme:
* X25519 key agreement with both ephemeral-static and static-static agreements between recipient and sender key pairs.
* HKDF is then used to derive a unique key-wrapping key. The KDF process includes all public keys (sender, recipient, ephemeral) to ensure non-malleability and key binding. It also includes an algorithm identifier that uniquely identifiers the KEM+DEM in use, and optionally allows an application-specific context string and user identifiers to be included.
* Key-wrapping and the DEM used for encryption are based on AES in Synthetic IV (SIV) mode, but using HMAC-SHA-256 rather than CMAC for authentication. The [safe cascade MAC construction](https://neilmadden.blog/2021/10/27/multiple-input-macs/) is used instead of s2v to allow multiple inputs to be authenticated, which is simpler to implement and fits better with how Macaroon caveats are supported.

### Replyable Encryption

Florentines support a `reply()` operation that constructs a new Florentine in response to one that has just been received. This works
by creating a new Florentine but replacing the original sender's public key with the ephemeral public key from the first Florentine,
and including the shared secret from the original in the new KDF inputs. This effectively ensures that the reply Florentine is encrypted
using a key that incorporates both static and ephemeral keys from both parties, providing similar security properties to the 
[Noise `KK` handshake pattern](http://noiseprotocol.org/noise.html#payload-security-properties).

To support this, when Alice sends a Florentine to Bob she needs to remember the ephemeral secret key until she receives a reply. The
Florentines API is designed to encapsulate this ephemeral secret in an opaque state object to support this. A special header is used
to indicate that replying is supported and to specify a deadline for how long the sender is willing to process replies before they
destroy the original ephemeral secret.

Although in principle you could use this facility to build a simple sort of secure messenger service, in practice this would be quite
cumbersome and you are better off using something like Signal or Noise in that case. Replies in Florentine are really intended only for
very basic request-reply handshakes such as OAuth or custom challenge-response authentication protocols where each message expects a
reply and you are not sending multiple messages concurrently etc. 
