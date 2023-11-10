---
slug: 70
title: 70/ETH-SECPM
name: Secure channel setup using Ethereum accounts
status: raw
category: Standards Track
tags:
editor: Ramses Fernandez <ramses@status.im>
contributors:
---

# Abstract
This document specifies an Ethereum-based private messaging service. 
This proposal is built upon this [model](https://rfc.vac.dev/spec/20/) and 
amends the limitations of the latter concerning forward privacy and authentication. 
The document is still work in progress. 
Next steps will include a description of how to implement the different functions and algorithms in terms of the Noise framework.

# Background

Alice wants to send an encrypted message to Bob. 
Here Bob is the only individual able to decrypt the message.
Alice knows Bob’s Ethereum address.

# Theory and Description of the Protocol

The proposed protocol must adhere to the following design requirements:
-   Alice knows Bob’s Ethereum address.
-   Bob is willing to participate in the protocol, and publishes his public key.
-   Bob’s ownership of his public key is verifiable,
-   Alice wants to send message M to Bob.
-   An eavesdropper cannot read M’s content even if she is storing it or relaying it.

The specification is based on the noise protocol framework.
It corresponds to the double ratchet scheme combined with the X3DH algorithm, which will be used to initialize the former. 
We chose to express the protocol in noise to be be able to use the noise streamlined implementation and proving features.
The X3DH algorithm provides both authentication and forward secrecy, as stated in the [X3DH specification](https://signal.org/docs/specifications/x3dh/).

## High level description
This protocol will consist of several stages:

1.  Key setting for X3DH: this step will produce prekey bundles for Bob which will be fed into X3DH. It will also allow Alice to generate the keys required to run the X3DH algorithm correctly.
2.  Execution of X3DH: This step will output a common secret key SK together with an additional data vector AD. Both will be used in the Double Ratchet algorithm initialization.
3.  Execution of the Double Ratchet algorithm for forward secure, authenticated communications, using the common secret key SK, obtained from X3DH, as a root key.

## Cryptographic functions required
-   XEd448 for digital signatures involved in the X3DH key generation.
-   SHA512 for hashing and the generation of HMACs.
-   AES256-CBC for the encryption/decryption of messages.

## Considerations on the X3DH initialization
This scheme MUST work on a specific elliptic curves which differ from those used by Ethereum. The curve Curve448 MUST be chosen: 
since it offers a higher security level: 224-bit security instead of the 128-bit security provided by X25519.

Bob and Alice MUST define personal key pairs $(ik_B, IK_B)$ and $(ik_A, IK_A)$ respectively where:
-   The key ik must be kept secret,
-   and the key IK is public.

Bob will not be able to use his Ethereum public key during this stage due to incompatibilities with the involved elliptic curves, therefore he MUST generate new keys. 
Using the Noise framework notation, these key pairs MUST BE generated using $(ik_B, IK_B) = \text{GENERATE-KEYPAIR}(curve = Curve448)$.

Bob MUST also generate a public key SPK using $(spk_B, SPK_B) = \text{GENERATE-KEYPAIR}(curve = Curve448)$.

SPK is a public key generated and stored at medium-term. 
It is called a signed prekey because Bob MUST store a public key certificate of SPK using IK. 
Both signed prekey and the certificate MUST undergo periodic replacement, 
a process that entails the generation of a fresh signed prekey. 
After replacing the key, 
Bob keeps the old private key of SPK for some interval, dependant on the implementation.
This allows Bob to decrypt delayed messages. 
It is important that Bob MUST NOT reuse SPKs. 
This action is pivotal for ensuring forward secrecy, as these keys are integral for recalculating the shared secret employed in decrypting historical messages.

Bob MUST sign SPK for authentication. Following the specification of X3DH, one will use the digital signature scheme XEd448 and define:

$$ SigSPK = XEd448(ik, Encode(SPK)) $$

A final step requires the definition of a  _prekey bundle_  given by the tuple

$$ prekey-bundle = (IK, SPK, SigSPK, \{OPK_i\}_i) $$

Where the different one-time keys OPK are generated as $(opk_B, OPK_B) = \text{GENERATE-KEYPAIR}(curve = Curve448)$.

Before sending an initial message to Bob, Alice MUST generate an AD vector as described in the documentation:

$$ AD = Encode(IK_A)||Encode(IK_B) $$

Alice MUST generate ephemeral key pairs $(ek, EK) = \text{GENERATE-KEYPAIR}(curve = Curve448)$.

The function Encode() transforms an X448 public key into a byte sequence. 
It consists of a single-byte constant to represent the type of curve, followed by little-endian encoding of the u-coordinate. 
This is specified in the [RFC 7748](http://www.ietf.org/rfc/rfc7748.txt) on elliptic curves for security.

## Using X3DH in Double Ratchet

According to [Signal specifications](https://signal.org/docs/specifications/doubleratchet/) 
this specification uses the double ratchet in combination with X3DH using the following data as initialization for the former:

-   The SK output from X3DH becomes the SK input of the double ratchet. See section 3.3 of [Signal Specification](https://signal.org/docs/specifications/doubleratchet/) for a detailed description.
-   The AD output from X3DH becomes the AD input of the double ratchet. See sections 3.4 and 3.5 of  [Signal Specification](https://signal.org/docs/specifications/doubleratchet/)  for a detailed description.
-   Bob’s signed prekey SigSPKB from X3DH is used as Bob’s initial ratchet public key of the double ratchet.

Once this initialization has been set, Alice and Bob can start exchanging messages with forward secrecy and authentication.

X3DH has three phases:

1.  Bob publishes his identity key and prekeys to a server, a network, or dedicated smart contract.
2.  Alice fetches a "prekey bundle" from the server, and uses it to send an initial message to Bob.
3.  Bob receives and processes Alice's initial message.

Alice MUST perform the following computations:

-  $dh1 = DH(IK_A, SPK_B, curve = Curve448)$
-  $dh2 = DH(EK_A, IK_B, curve = Curve448)$
-  $dh3 = DH(EK_A, SPK_B)$
-  $SK = KDF(dh1 || dh2 || dh3)$

Alice MUST send to Bob a message containing: 

-  $IK_A, EK_A$.
-  An identifier to the Bob's prekeys used.
-  A message encrypted with AES256 using AD and SK.

The Key Derivation Function (KDF) ratchet and the associated encryption protocols used by the double ratchet are also included by the Noise framework: 
SHA256 for the KDF and AES256 for AEAD encryption.

Upon reception of the initial message, Bob MUST perform the same computations above with the DH() function.
Bob derives SK and constructs AD.
Bob decrypts the initial message encrypted with AES256.
If decryption fails, Bob MUST abort the protocol.

# Retrieving information

## Static data

Some data, such as the key pairs (ik, IK) for Alice and Bob, MAY NOT be regenerated after a period of time. 
Therefore the prekey bundle MAY be stored in long-term storage solutions, such as a dedicated smart contract which outputs such a key pair when receiving an Ethereum wallet address.

## Ephemeral data

Storing ephemeral data on Ethereum can be done using a combination of on-chain and off-chain solutions. 
This approach provides an efficient solution to the problem of storing updatable data in Ethereum.
1. Ethereum can store a reference or a hash that points to the off-chain data.
2. Off-chain solutions can include systems like IPFS, traditional cloud storage solutions, or decentralized storage networks such as a [Swarm](https://www.ethswarm.org). 
In any case, the user stores the associated IPFS hash, URL or reference in Ethereum.

The fact of a user not updating the ephemeral information can be understood as Bob not willing to participate in any communication.

## Interaction with Ethereum

Storing static data is done using a dedicated smart contract *PublicKeyStorage* which associates the Ethereum wallet address of a user with his public key.
This mapping is done by PublicKeyStorage using a *publicKeys* function, or a *setPublicKey* function.
This mapping is done if the user passed an authorization process.
A user who wants to retrieve a public key associated with a specific wallet address calls a function *getPublicKey*.
The user provides the wallet address as the only input parameter for *getPublicKey*.
The function outputs the associated public key from the smart contract.

# Extension to group chat

## 1-to-1 version

In order to extend the protocol to a group chat, this document specifies using an Asynchronous Distributed Key Generation (ADKG) to replace the X3DH step in the previous combination X3DH + Double Ratchet.

Distributed Key Generation (DKG) is a method for initiating threshold cryptosystems in a decentralized manner, all without the need for a trusted third party. 
DKG serves as a fundamental component for numerous decentralized protocols, including systems like randomness beacons, threshold signatures, Byzantine consensus, and multiparty computation.

Most DKG protocols assume synchronous networks. 
Asynchronous DKG (ADKG) has been studied only recently and the state-of-the-art high-threshold ADKG protocols is very inefficient compared to its low-threshold counterpart. 

Here low-threshold means that the reconstruction threshold is set to be one higher than the number of corrupt nodes, 
whereas high-threshold protocols admit reconstruction thresholds much higher than the number of malicious nodes.

Existing ADKG constructions tend to become inefficient when the reconstruction threshold surpasses one-third of the total nodes. 
In this proposal we suggest using the scheme by [Kokoris-Kogias et al.](https://eprint.iacr.org/2022/1389) which is designed for $n = 3t + 1$ nodes. 

This protocol can withstand the presence of up to $t$ malicious nodes and can adapt to any reconstruction threshold in $l \in [t, n-t-1]$. 
The key point of the proposal is an asynchronous method for securely distributing a random polynomial of degree $l\geq t$. 
The proposal includes [Python and Rust implementations](https://github.com/sourav1547/htadkg).

The suggested ADKG makes assumes the existence of a PKI. 
In case of requiring removing such assumption, one can replace the VSS scheme with the [Alhaddad et al.](https://eprint.iacr.org/2021/118) at the price of increasing the complexity.

The output of the ADKG is an integer (modulo a prime), 
meaning that one should apply a KDF to that output 
in order to obtain a result which could be used as an input for the double ratchet.

One observes that using an ADKG allows a set of users, 
which want to define a group chat, 
defining a common secret key which will be used as a root key for the double ratchet. 
Using an ADKG defines a room key, 
which essentially defines the group itself.

This approach share similarities with the point of view of [Farcaster](https://github.com/farcasterxyz/protocol/discussions/99).

Once the double ratchet is initialized, 
the communication in this group is 1-to-1, 
meaning that group member C cannot see the messages between group members A and B. 
The fact of defining a room key makes impossible for outsiders to communicate with group members if the latter are not willing to.

## n-to-n version

Using the above approach leads to a situation where a group of users can set a group for 1-to-1 messages,
meaning that any group member external to a communication between any other two members will not be able to read the contents of the messages.

An approach to generalize this situation to the setting of a group of users exchanging messages without any kind of restriction is using asynchronous ratcheting trees, as suggested in the proposal from [Cohn-Gordon et al.](https://eprint.iacr.org/2017/666) where a group of people can derive a shared secret key even in the event of if no two users are ever online at the same time. 
The proposal suggested provides both forward secrecy and post-compromise security. 
The shared key can be then used in any symmetric encryption scheme, such as AES256.

# Privacy and Security Considerations

- For the information retrieval, the algorithm MUST include a access control mechanisms to restrict who can call the set and get functions.
- One SHOULD include event logs to track changes in public keys.
- The curve Curve448 MUST be chosen as the elliptic curve, since it offers a higher security level: 224-bit security instead of the 128-bit security provided by X25519.
- Concerning the hardness of the ADKG, the proposal lies on the Discrete Logarithm assumption.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

- https://rfc.vac.dev/spec/20/
- https://signal.org/docs/specifications/x3dh/
- https://signal.org/docs/specifications/doubleratchet/
- https://eprint.iacr.org/2022/1389
- https://github.com/sourav1547/htadkg
- https://github.com/farcasterxyz/protocol/discussions/99
