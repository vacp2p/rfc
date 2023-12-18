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
2.  Execution of X3DH: This step will output a common secret key SK together with an additional data vector AD. Both will be used in the double ratchet algorithm initialization.
3.  Execution of the double ratchet algorithm for forward secure, authenticated communications, using the common secret key SK, obtained from X3DH, as a root key.

## Cryptographic functions required
-   XEd448 and SHA512 for digital signatures involved in the X3DH key generation.
-   SHA256 for the key generation related to AES256-GCM.
-   AES256-GCM for the encryption/decryption of messages.

## Considerations on the X3DH initialization
This scheme MUST work on a specific elliptic curves which differ from those used by Ethereum. The curve Curve448 MUST be chosen: 
since it offers a higher security level: 224-bit security instead of the 128-bit security provided by X25519. This document specifies X3DH in terms of the Noise Framework.
The X3DH algorithm corresponds to the IX pattern in Noise.

Bob and Alice MUST define personal key pairs `(ik_B, IK_B)` and `(ik_A, IK_A)` respectively where:
-   The key ik must be kept secret,
-   and the key IK is public.

Bob will not be able to use his Ethereum public key during this stage due to incompatibilities with the involved elliptic curves, therefore he MUST generate new keys. 
Using the Noise framework notation, these key pairs MUST BE generated using `(ik_B, IK_B) = GENERATE_KEYPAIR(curve = curve448)`.

Bob MUST also generate a public key SPK using `(spk_B, SPK_B) = GENERATE_KEYPAIR(curve = curve448)`.

SPK is a public key generated and stored at medium-term. 
It is called a signed prekey because Bob MUST store a public key certificate of SPK using IK. 
Both signed prekey and the certificate MUST undergo periodic replacement, 
a process that entails the generation of a fresh signed prekey. 
After replacing the key, 
Bob keeps the old private key of SPK for some interval, dependant on the implementation.
This allows Bob to decrypt delayed messages. 
It is important that Bob MUST NOT reuse SPKs. 
This action is pivotal for ensuring forward secrecy, as these keys are integral for recalculating the shared secret employed in decrypting historical messages.

Bob MUST sign SPK for authentication. Following the specification of X3DH, one will use the digital signature scheme XEd448 and define `SigSPK = XEd448(ik, Encode(SPK))`

A final step requires the definition of a  _prekey bundle_  given by the tuple `prekey_bundle = (IK, SPK, SigSPK, OPK_i)`

Where the different one-time keys OPK are generated as `(opk_B, OPK_B) = GENERATE_KEYPAIR(curve = curve448)`.

Before sending an initial message to Bob, Alice MUST generate an AD vector as described in the documentation: `AD = Encode(IK_A) || Encode(IK_B)`.

Alice MUST generate ephemeral key pairs `(ek, EK) = GENERATE_KEYPAIR(curve = curve448)`.

The function Encode() transforms an X448 public key into a byte sequence. 
It consists of a single-byte constant to represent the type of curve, followed by little-endian encoding of the u-coordinate. 
This is specified in the [RFC 7748](http://www.ietf.org/rfc/rfc7748.txt) on elliptic curves for security.

The algorithm used for digital signatures MUST be XEd448 described below. 
One MUST consider `q = 2^446 - 13818066809895115352007386748515426880336692474882178609894547503885` and Z a random 64-bytes sequence:
```
XEd448_sign((ik, IK), message):
    Z = randbytes(64)  
    r = SHA512(2^456 - 2 || ik || message || Z )
    R = (r * convert_mont(5)) % q
    h = SHA512(R || IK || M)
    s = (r + h * ik) % q
    return (R || s)
```
```
XEd448_verify(u, message, (R || s)):
    if (R.y >= 2^448) or (s >= 2^446): return FALSE
    h = (SHA512(R || 156326 || message)) % q
    R_check = s * convert_mont(5) - h * 156326
    if R == R_check: return TRUE
    return FALSE 
```
The auxiliary function `convert_mont(u)` follows:
```
convert_mont(u):
    u_masked = u % mod 2^448
    inv = ((1 - u_masked)^(2^448 - 2^224 - 3)) % (2^448 - 2^224 - 1)
    P.y = ((1 + u_masked) * inv)) % (2^448 - 2^224 - 1)
    P.s = 0
    return P
```

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
```
dh1 = DH(IK_A, SPK_B, curve = curve448)
dh2 = DH(EK_A, IK_B, curve = curve448)
dh3 = DH(EK_A, SPK_B)
SK = KDF(dh1 || dh2 || dh3)
```
Alice MUST send to Bob a message containing: 

-  `IK_A, EK_A`.
-  An identifier to the Bob's prekeys used.
-  A message encrypted with AES256 using AD and SK.

The Key Derivation Function (KDF) ratchet and the associated encryption protocols used by the double ratchet are also included by the Noise framework: 
SHA256 for the KDF and AES256 for AEAD encryption.

Upon reception of the initial message, Bob MUST perform the same computations above with the DH() function.
Bob derives SK and constructs AD.
Bob decrypts the initial message encrypted with AES256.
If decryption fails, Bob MUST abort the protocol.

# The Double Ratchet

## Initialization
In this stage Bob and Alice have generated key pairs and agreed a shared secret SK using X3DH.

Alice calls `RatchetInitAlice()` defined below:
```
RatchetInitAlice(SK, IK_B):
    state.DHs = GENERATE_KEYPAIR(curve = curve448)
    state.DHr = IK_B
    state.RK, state.CKs = HKDF(SK, DH(state.DHs, state.DHr)) 
    state.CKr = None
    state.Ns, state.Nr, state.PN = 0
    state.MKSKIPPED = {}
```
The HKDF function MUST be the proposal by [Krawczyk and Eronen](http://www.ietf.org/rfc/rfc5869.txt).
In this proposal `chaining_key` and `input_key_material` MUST be replaced with `SK` and the output of `DH` (respectively).

Similarly, Bob calls the function `RatchetInitBob()` defined below:
```
RatchetInitBob(SK, (ik_B,IK_B)):
    state.DHs = (ik_B, IK_B)
    state.Dhr = None
    state.RK = SK
    state.CKs, state.CKr = None
    state.Ns, state.Nr, state.PN = 0
    state.MKSKIPPED = {}
```

## Encryption
This function performs the symmetric key ratchet.

```
RatchetEncrypt(state, plaintext, AD):
    state.CKs, mk = HMAC-SHA256(state.CKs)
    header = HEADER(state.DHs, state.PN, state.Ns)
    state.Ns = state.Ns + 1
	return header, AES256-GCM_Enc(mk, plaintext, AD || header)
```
The `HEADER` function creates a new message header containing the public key from the key pair output of the `DH`function.  
It outputs the previous chain length `pn`, and the message number `n`. 
The returned header object contains ratchet public key `dh` and integers `pn` and `n`.

## Decryption
This function will decrypt incoming messages. One introduces auxiliary functions required.

```
DHRatchet(state, header):
    state.PN = state.Ns
    state.Ns = state.Nr = 0
    state.DHr = header.dh
    state.RK, state.CKr = HKDF(state.RK, DH(state.DHs, state.DHr))
    state.DHs = GENERATE_KEYPAIR(curve = curve448)
    state.RK, state.CKs = HKDF(state.RK, DH(state.DHs, state.DHr))
```
```
SkipMessageKeys(state, until):
    if state.NR + MAX_SKIP < until:
        raise Error
    if state.CKr != none:
        while state.Nr < until:
            state.CKr, mk = HMAC-SHA256(state.CKr)
            state.MKSKIPPED[state.DHr, state.Nr] = mk
            state.Nr = state.Nr + 1
```
```
TrySkippedMessageKey(state, header, ciphertext, AD):
    if (header.dh, header.n) in state.MKSKIPPED:
        mk = state.MKSKIPPED[header.dh, header.n]
        delete state.MKSKIPPED[header.dh, header.n]
        return AES256-GCM_Dec(mk, ciphertext, AD || header)
    else: return None
```

The main function of this section follows:
```
RatchetDecrypt(state, header, ciphertext, AD):
    plaintext = TrySkippedMessageKeys(state, header, ciphertext, AD)
    if plaintext != None:
        return plaintext
    if header.dh != state.DHr:
        SkipMessageKeys(state, header.pn)
        DHRatchet(state, header)
    SkipMessageKeys(state, header.n)
    state.CKr, mk = HMAC-SHA256(state.CKr)
    state.Nr = state.Nr + 1
    return AES256-GCM_Dec(mk, ciphertext, AD || header)
```

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

## The MLS Framework

The Messaging Layer Security (MLS) protocol aims at providing a group of users with end-to-end encryption in an authenticated and asynchronous way. 
It is a key establishment protocol that provides efficient asynchronous group key establishment with forward secrecy (FS) and post-compromise security (PCS) for groups in size ranging from two to thousands.

### Main security characteristics

The main security characteristics of the protocol follow:

- **Message confidentiality**: If a user C sends a message $M$ in epoch $E$ of group $G$, 
and C believes that the membership of G in E is C_0, …, C_n, 
then M is kept secret from an adversary as long as none of these members is compromised.
- **Message authentication**: If a user C accepts a message M in epoch E of group G, 
and if C believes that the membership of G in E is C_0, …, C_n, 
and if none of these members are compromised at the time of reception, then M must have been sent by one of these group members for the group G in epoch E.
- **Sender authentication**: If a user C accepts a message M seemingly sent by a client C' in epoch E of group G, 
and if C' is uncompromised at the time of reception, 
then M must have been sent by C' in epoch E of group G.
- **Membership agreement**: If a user C accepts a message M from a client C' in epoch E of group G, 
then C and C' must agree on the membership of G at E.
- **Post-remove security**: If a user C was member of group G in epoch E, 
and was no longer a member in epoch E+1, 
then even if C was compromised in epochs before E, it does not affect the confidentiality of messages sent in epochs following E+1.
- **Post-update security**: If a user C was member of group G in epoch E, 
and has updated its cryptographic keys in epoch E+1, 
then even if the previous state of C in epochs before E was compromised, 
it does not affect the confidentiality of messages sent in following E+1.
- **Forward secrecy**: If a user C sends (or receives) a message M in epoch E of group G, 
then any compromise of C after this point does not affect the confidentiality of M.
- **Post-compromise security**: If the key of a user C has been compromised in epoch E, 
then all members og the group G have security guarantees about communicating with C. 
To recover from a compromise of a single member of the group, all other members have to broadcast an update of their key material. 
This leads to an overall cost of computation and bandwidth of O(n^2) for a group size of n and requires all group members to come online at least once. 
MLS has an update operation with complexity of O(log(n)) that requires only the compromised member to be online for the group to recover from the compromise.

### Strengths of the protocol

- **Low complexity**: The use of binary trees allows MLS achieveing low complexity levels, 
meaning that the number of required operations and the payload size do not increase linearly with the group size but logarithmically after a short period.
- **Group integrity**: This property guarantees full agreement among group members regarding the group's current state and its constituents. 
Consequently, a member can decrypt messages exclusively from others in the group when both sender and recipient align on the group's status, particularly its membership. 
This feature thwarts any attempt by a third party to add a new member to the group without the explicit acknowledgment of all existing members, enhancing the group's security by preventing unauthorized inclusions.
- **Synchronization**: MLS tackles data synchronization among group members by leveraging its group integrity property. 
Beyond merely concurring on a member list, the protocol enables agreement on diverse data sets among group members. 
Central to this process is the Delivery Service component, which enforces sequential delivery of MLS messages, dictating the progressive transition from one group state to another. 
This mechanism establishes an MLS group as a reliable distribution channel for synchronizing various data among different entities. 
The cryptographic agreement on prior extension messages ensures the secure transfer of data, enhancing the protocol's integrity and synchronization capabilities.
> The Delivery Service can be completely abstract if users are able to distribute credentials and messages without relying on a central Delivery Service. 
- **Extensibility**: MLS offers extensibility by permitting modifications or additions of data to a group's state. 
It incorporates a negotiation mechanism that mandates unanimous support from all group members for any introduced extensions. 
Moreover, members collectively decide which extensions are obligatory within the group. 
As a group key negotiation protocol, MLS facilitates the export of additional cryptographic key material by all members, enabling the augmentation of cryptographic keys within the group's context.

### Components

The MLS protocol is composed of 3 subprotocols, namely: TreeSync, TreeKEM and TreeDEM, which are described below:

- **TreeSync** (Authenticated Group Management): This sub-protocol guarantees consensus on membership and upholds the integrity of the group's state. 
The sub-protocol maintains a synchronized and authenticated representation of the group's state for all members. 
It encompasses group management functionalities, ensuring coherence across membership arrays and keys within the MLS tree data structure. 
TreeSync employs several tree hashing methodologies similar to Merkle Trees alongside signatures.
- **TreeKEM** (Efficient Group Key Establishment): This sub-protocol utilizes a tree structure to create subgroup keys for internal nodes, including a shared group key at the root for all members within the present group epoch. 
When group membership changes, TreeKEM generates a new group key and efficiently distributes it to all members. 
The efficiency of TreeKEM remains logarithmic to the group's size when all members actively participate in the group. However, if only a subset of members actively engage, operational costs can escalate linearly with the group size, compromising the protocol's efficiency.
    
	TreeKEM is based in a previous protocol called Asynchronous Ratcheting Tree (ART). 
	ART was replaced by the more efficient alternative TreeKEM, which is based on Hybrid Public Key Encryption. 
	Subsequent drafts refined and improved TreeKEM, but the fundamental key establishment mechanism remains the same.

	TreeKEM can manage the situation of a user associated to several devices. 
	There are two approaches:
	- Providing each device with a set of keys and including it in the tree as separate leaf.
	This corresponds to session `NM` [here](https://rfc.vac.dev/spec/37/).
	- Allowing the synchronization of keys so all devices appear as a single leaf.
	This corresponds to session `N11M` [here](https://rfc.vac.dev/spec/37/).
    
- **TreeDEM** (Forward Secure Group Messaging): This sub-protocol builds upon TreeKEM's established group keys to secure application messages, handshake messages and *welcome* messages, for new members, transmitted within each epoch. 
Employing the tree structure, TreeDEM ensures forward security for these messages by determining key derivation and deletion based on the tree's configuration. 
This approach safeguards the integrity of application messages while providing forward security measures by effectively managing key usage and expiration within the protocol's framework.

### Inactive users: Quarantined treeKEM and Tainted treeKEM

Ensuring post-compromise security and forward secrecy requires active involvement from all users within a group, including both compromised and uncompromised individuals.
However, there's a potential risk posed by inactive users, termed as *ghosts*, who remain offline for extended periods and fail to update their keys. These inactive users create a vulnerability that affects the entire group.

Ghosts are considered an inherent aspect of the protocol's asynchronicity. 
The compromise of just one ghost can jeopardize the forward secrecy of the entire system. 
Moreover, the presence of a single corrupted ghost is sufficient to undermine the post-compromise security of the entire group.

In order to overcome the associated issues with ghosts one finds two similar soluctions, namely: [Quarantined treeKEM](https://eprint.iacr.org/2023/1903) (QTK) and [Tainted treeKEM](https://eprint.iacr.org/2019/1489) (TTK).

QTK is a continuous group key agreement which builds upon treeKEM and aligns with the MLS standard. 
It introduces a "quarantine" feature designed to address the risks associated with ghosts while retaining their presence within the group.

In the quarantine process of QTK, a randomly selected user takes charge of blanking the ghost's direct path and updating its encryption keys on its behalf. 
This method bears a striking resemblance to TTK's strategy for adding or removing users without blanking their direct paths. 
In TTK, nodes within new or former users' direct paths are updated by a temporary group member.

The main distinction between QTK and TTK lies in how they handle the initiator's access to the ghost's (secret) decryption key. 
QTK diverges by employing a secret sharing scheme to share the secret seed used for generating the ghost's encryption key pair. 
These shares are then distributed among all group members, and the ghost's secret seed and private key are deleted from the initiator's internal state. 
Consequently, the confidentiality of the ghost's secret key no longer hinges on the security of a single user (the quarantine initiator) but relies on the collective security of several active group members.

When a ghost finally reconnects and updates its keying material, its quarantine automatically stops and the users that kept shares related to that ghost send them to it. 
The former ghost is therefore able to reconstruct the secret seeds corresponding to its quarantine keys.

# Privacy and Security Considerations

- The double ratchet "recommends" using AES in CBC mode. Since encryption must be with an AEAD encryption scheme, we will use AES in GCM mode instead (supported by Noise).
- For the information retrieval, the algorithm MUST include a access control mechanisms to restrict who can call the set and get functions.
- One SHOULD include event logs to track changes in public keys.
- The curve Curve448 MUST be chosen as the elliptic curve, since it offers a higher security level: 224-bit security instead of the 128-bit security provided by X25519.
- Concerning the hardness of the ADKG, the proposal lies on the Discrete Logarithm assumption.
- In order to improve forward-secrecy in TreeKEM users MUST use a UPKE scheme.

# Copyright

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

# References

- https://rfc.vac.dev/spec/20/
- https://signal.org/docs/specifications/x3dh/
- https://signal.org/docs/specifications/doubleratchet/
- https://eprint.iacr.org/2022/1389
- https://github.com/sourav1547/htadkg
- https://github.com/farcasterxyz/protocol/discussions/99
- http://www.ietf.org/rfc/rfc5869.txt
- https://eprint.iacr.org/2022/1389
- https://inria.hal.science/hal-02425247/file/treekem%20(1).pdf
- https://eprint.iacr.org/2019/1189.pdf
- https://eprint.iacr.org/2023/1903
- https://eprint.iacr.org/2019/1489
