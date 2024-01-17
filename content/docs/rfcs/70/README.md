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

Some data, such as the key pairs `(ik, IK)` for Alice and Bob, MAY NOT be regenerated after a period of time. 
Therefore the prekey bundle MAY be stored in long-term storage solutions, such as a dedicated smart contract which outputs such a key pair when receiving an Ethereum wallet address.

## Ephemeral data

Storing ephemeral data on Ethereum can be done using a combination of on-chain and off-chain solutions. 
This approach provides an efficient solution to the problem of storing updatable data in Ethereum.
1. Ethereum can store a reference or a hash that points to the off-chain data.
2. Off-chain solutions can include systems like IPFS, traditional cloud storage solutions, or decentralized storage networks such as a [Swarm](https://www.ethswarm.org). 
In any case, the user stores the associated IPFS hash, URL or reference in Ethereum.

The fact of a user not updating the ephemeral information can be understood as Bob not willing to participate in any communication.

## Interaction with Ethereum

Storing static data is done using a dedicated smart contract `PublicKeyStorage` which associates the Ethereum wallet address of a user with his public key.
This mapping is done by `PublicKeyStorage` using a `publicKeys` function, or a `setPublicKey` function.
This mapping is done if the user passed an authorization process.
A user who wants to retrieve a public key associated with a specific wallet address calls a function `getPublicKey.
The user provides the wallet address as the only input parameter for `getPublicKey`.
The function outputs the associated public key from the smart contract.

# Extension to group chat: the MLS Framework

The Messaging Layer Security (MLS) protocol aims at providing a group of users with end-to-end encryption in an authenticated and asynchronous way. 
It is a key establishment protocol that provides efficient asynchronous group key establishment with forward secrecy (FS) and post-compromise security (PCS) for groups in size ranging from two to thousands.
The main security characteristics of the protocol are: Message confidentiality and authentication, sender authentication, membership agreement, post-remove and post-update security, and forward secrecy and post-compromise security.
The MLS protocol achieves: low-complexity, group integrity, synchronization and extensibility.

## Cryptographic suites
Each MLS session uses a single cipher suite that specifies the primitives to be used in group key computations. The cipher suite MUST use:
- `X488` as Diffie-Hellman function.
- `SHA256` as KDF.
- `AES256-GCM` as AEAD algorithm.
- `SHA512` as hash function.
- `XEd448` for digital signatures.

Formats for public keys, signatures and public-key encryption MUST be as defined in Section 5.1 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Hash-based identifiers
Some MLS messages refer to other MLS objects by hash.
These identifiers MUST be computed according to Section 5.2 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Credentials
Each member of a group presents a credential that provides one or more identities for the member and associates them with the member's signing key. 
The identities and signing key are verified by the Authentication Service in use for a group.
Credentials MUST follow the specifications of section 5.3 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).  

## Message framing
Handshake and application messages use a common framing structure providing encryption to ensure confidentiality within the group, and signing to authenticate the sender.

The structures are:
- `PublicMessage`: represents a message that is only signed, and not encrypted. 
- `PrivateMessage`: represents a signed and encrypted message, with protections for both the content of the message and related metadata.

Applications MUST use `PrivateMessage` to encrypt application messages. 
Applications SHOULD use `PrivateMessage` to encode handshake messages.

The definition of `PublicMessage` MUST follow the specification in section 6.2 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).
The definition, and the encoding/decoding of `PrivateMessage` MUST follow the specification in section 6.3 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Nodes contents
The nodes of a ratchet tree contain several types of data. 
Leaf nodes describe individual members.
Parent nodes describe subgroups.
Contents of each kind of node, and its structure MUST follow the indications described in sections 7.1 and 7.2 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Leaf node validation
`KeyPackage` objects describe the client's capabilities and provides keys that can be used to add the client to a group.

The validity of a leaf node needs to be verified at the following stages:
- When a leaf node is downloaded in a `KeyPackage`, before it is used to add the client to the group.
- When a leaf node is received by a group member in an Add, Update, or Commit message.
- When a client validates a ratchet tree.

A client MUST verify the validity of a leaf node following the instructions of section 7.3 in [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Ratchet tree evolution
Whenever a member initiates an epoch change, they MAY need to refresh the key pairs of their leaf and of the nodes on their direct path. This is done to keep forward secrecy and post-compromise security.
The member initiating the epoch change MUST follow this procedure procedure.
A member updates the nodes along its direct path as follows:
- Blank all the nodes on the direct path from the leaf to the root.
- Generate a fresh HPKE key pair for the leaf.
- Generate a sequence of path secrets, one for each node on the leaf's filtered direct path.
It MUST follow the procedure described in section 7.4 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).
- Compute the sequence of HPKE key pairs `(node_priv,node_pub)`, one for each node on the leaf's direct path.
It MUST follow the procedure described in section 7.4 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Views of the tree synchronization
After generating fresh key material and applying it to update their local tree state, the generator broadcasts this update to other members of the group.
This operation MUST be done according to section 7.5 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Leaf synchronization
Changes to group memberships MUST be represented by adding and removing leaves of the tree.
This corresponds to increasing or decreasing the depth of the tree, resulting in the number of leaves being doubled or halved.
These operations MUST be done as described in section 7.7 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Tree and parent hashing
Group members can agree on the cryptographic state of the group by generating a hash value that represents the contents of the group ratchet tree and the member’s credentials. 
The hash of the tree is the hash of its root node, defined recursively from the leaves.
Tree hashes summarize the state of a tree at point in time.
The hash of a leaf is the hash of the `LeafNodeHashInput` object. 
At the same time, the hash of a parent node including the root, is the hash of a `ParentNodeHashInput` object.
Parent hashes capture information about how keys in the tree were populated.

Tree and parent hashing MUST follow the directions in Sections 7.8 and 7.9 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Key schedule
Group keys are derived using the `Extract` and `Expand` functions from the KDF for the group's cipher suite, as well as the functions defined below:

```
ExpandWithLabel(Secret, Label, Context, Length) = KDF.Expand(Secret, KDFLabel, Length)
DeriveSecret(Secret, Label) = ExpandWithLabel(Secret, Label, "", KDF.Nh)
```
`KDFLabel` MUST be specified as:
```
struct {
    uint16 length;
    opaque label<V>;
    opaque context<V>;
} KDFLabel;
```
The fields of `KDFLabel` MUST be:
```
length = Length;
label = "MLS 1.0 " + Label;
context = Context;
```

Each member of the group MUST maintaint a `GroupContext` object summarizing the state of the group. 
The sturcture of such object MUST be:

```
struct {
ProtocolVersion version = mls10;
CipherSuite cipher_suite;
opaque group_id<V>;
uint64 epoch;
opaque tree_hash<V>;
opaque confirmed_trasncript_hash<V>;
Extension extension<V>;
} GroupContext;
```

The use of key scheduling MUST follow the indications in sections 8.1 - 8.7 in [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Key packages
KeyPackage objects are used to ease the addition of clients to a group asynchronously.
A KeyPackage object specifies:

- Protocol version and cipher suite supported by the client.
- Public keys that can be used to encrypt Welcome messages. Welcome messages provide new members with the information to initialize their state for the epoch in which they were added or in which they want to add themselves to the group
- The content of the leaf node that should be added to the tree to represent this client.

KeyPackages are intended to be used only once and SHOULD NOT be reused. 
Clients MAY generate and publish multiple KeyPackages to support multiple cipher suites.
The structure of the object MUST be:

```
struct {
ProtocolVersion version;
CipherSuite cipher_suite;
HPKEPublicKey init_key;
LeafNode leaf_node;
Extension extensions<V>;
/* SignWithLabel(., "KeyPackageTBS", KeyPackageTBS) */
opaque signature<V>;
}
```
```
struct {
ProtocolVersion version;
CipheSuite cipher_suite;
HPKEPublicKey init_key;
LeafNode leaf_node;
Extension extensions<V>;
}
```
`KeyPackage` object MUST be verified when:
- A `KeyPackage` is downloaded by a group member, before it is used to add the client to the group.
- When a `KeyPackage` is received by a group member in an `Add` message.

Verification MUST be done as follows:
- Verify that the cipher suite and protocol version of the `KeyPackage` match those in the `GroupContext`.
- Verify that the `leaf_node` of the `KeyPackage` is valid for a `KeyPackage`.
- Verify that the signature on the `KeyPackage` is valid.
- Verify that the value of `leaf_node.encryption_key` is different from the value of the `init_key field`.

HPKE public keys are opaque values in a format defined by Section 4 of [RFC9180](https://datatracker.ietf.org/doc/rfc9180/).
Signature public keys are represented as opaque values in a format defined by the cipher suite's signature scheme.

## Group creation
A group is always created with a single member. 
Other members are then added to the group using the usual Add/Commit mechanism.
The creator of a group MUST set:
- the group ID. 
- cipher suite.
- initial extensions for the group. 

If the creator intends to add other members at the time of creation, then it SHOULD fetch `KeyPackages` for those members, and select a cipher suite and extensions according to their capabilities. 

The creator MUST use the capabilities information in these `KeyPackages` to verify that the chosen version and cipher suite is the best option supported by all members.

Group IDs SHOULD be constructed so they are unique with high probability. 

To initialize a group, the creator of the group MUST initialize a one-member group with the following initial values:
- Ratchet tree: A tree with a single node, a leaf node containing an HPKE public key and credential for the creator.
- Group ID: A value set by the creator.
- Epoch: `0`.
- Tree hash: The root hash of the above ratchet tree.
- Confirmed transcript hash: The zero-length octet string.
- Epoch secret: A fresh random value of size `KDF.Nh`.
- Extensions: Any values of the creator's choosing.

The creator MUST also calculate the interim transcript hash:
- Derive the `confirmation_key` for the epoch according to Section 8 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).
- Compute a `confirmation_tag` over the empty `confirmed_transcript_hash` using the `confirmation_key` as described in Section 6.1 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).
- Compute the updated `interim_transcript_hash` from the `confirmed_transcript_hash` and the `confirmation_tag` as described in Section 8.2 [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

All members of a group MUST support the cipher suite and protocol version in use. Additional requirements MAY be imposed by including a `required_capabilities` extension in the `GroupContext`.

```
struct {
ExtensionType extension_types<V>;
ProposalType proposal_types<V>;
CredentialType credential_types<V>;
}
```

## Group evolution
Group membership can change, and existing members can change their keys in order to achieve post-compromise security. 
In MLS, each such change is accomplished by a two-step process:
- A proposal to make the change is broadcast to the group in a Proposal message.
- A member of the group or a new member broadcasts a Commit message that causes one or more proposed changes to enter into effect.

The group evolves from one cryptographic state to another each time a Commit message is sent and processed. 
These states are called epochs and are uniquely identified among states of the group by eight-octet epoch values. 

Proposals are included in a FramedContent by way of a Proposal structure that indicates their type:

```
struct {
ProposalType proposal_type;
select (Proposal.proposal_type) {
case add:			Add:
case update:			Update;
case remove:			Remove;
case psk:			PreSharedKey;
case reinit:			ReInit;
case external_init:		ExternalInit;
case group_context_extensions:	GroupContextExtensions;
}
```
On receiving a FramedContent containing a Proposal, a client MUST verify the signature inside `FramedContentAuthData` and that the epoch field of the enclosing FramedContent is equal to the epoch field of the current GroupContext object. 
If the verification is successful, then the Proposal SHOULD be cached in such a way that it can be retrieved by hash in a later Commit message.

Proposals are organized as follows:
- Add: requests that a client with a specified KeyPackage be added to the group.
- Update: similar to Add, it replaces the sender's LeafNode in the tree instead of adding a new leaf to the tree.
- Remove: requests that the member with the leaf index removed be removed from the group.
- ReInit: requests to reinitialize the group with different parameters.
- ExternalInit: used by new members that want to join a group by using an external commit.
- GroupContentExtensions: it is used to update the list of extensions in the GroupContext for the group.

Proposals structure and semantics MUST follow sections 12.1.1 - 12.1.7 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

Any list of commited proposals MUST be validated either by a the group member who created the commit, or any group member processing such commit.
The validation MUST be done according to one of the procedures described in Section 12.2 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

When creating or processing a Commit, a client applies a list of proposals to the ratchet tree and GroupContext. 
The client MUST apply the proposals in the list in the order described in Section 12.3 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Commit messages
Commit messages initiate new group epochs. 
It informs group members to update their representation of the state of the group by applying the proposals and advancing the key schedule.

Each proposal covered by the Commit is included by a `ProposalOrRef` value.
`ProposalOrRef` identify the proposal to be applied by value or by reference. 
Commits that refer to new Proposals from the committer can be included by value. 
Commits for previously sent proposals from anyone can be sent by reference. 
Proposals sent by reference are specified by including the hash of the `AuthenticatedContent`.

Group members that have observed one or more valid proposals within an epoch MUST send a Commit message before sending application data.
A sender and a receiver of a Commit MUST verify that the committed list of proposals is valid.
The sender of a Commit SHOULD include all valid proposals received during the current epoch.

Functioning of commits MUST follow the instructions of Section 12.4 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

## Application messages
Handshake messages provide an authenticated group key exchange to clients. 
To protect application messages sent among the members of a group, the `encryption_secret` provided by the key schedule is used to derive a sequence of nonces and keys for message encryption.

Each client MUST maintain their local copy of the key schedule for each epoch during which they are a group member. 
They derive new keys, nonces, and secrets as needed. This data MUST be deleted as soon as they have been used.

Group members MUST use the AEAD algorithm associated with the negotiated MLS ciphersuite to encrypt and decrypt Application messages according to the Message Framing section.
The group identifier and epoch allow a device to know which group secrets should be used and from which Epoch secret to start computing other secrets and keys. 
Application messages SHOULD be padded to provide resistance against traffic analysis techniques. 
This avoids additional information to be provided to an attacker in order to guess the length of the encrypted message.
Padding SHOULD be used on messages with zero-valued bytes before AEAD encryption.

Functioning of application messages MUST follow the instructions of Section 15 of [RFC9420](https://datatracker.ietf.org/doc/rfc9420/).

# Authentication process: SIWE

[EIP 4361](https://eips.ethereum.org/EIPS/eip-4361) is the authentication method required. 
Sign-In with Ethereum describes how Ethereum accounts authenticate with off-chain services by signing a standard message format 
parameterized by scope, session details, and security mechanisms

## Message format (ABNF)

A SIWE Message MUST conform with the following Augmented Backus–Naur Form ([RFC 5234](https://datatracker.ietf.org/doc/html/rfc5234)) expression.

```
sign-in-with-ethereum =
    [ scheme "://" ] domain %s" wants you to sign in with your Ethereum account:" LF
    address LF
    LF
    [ statement LF ]
    LF
    %s"URI: " uri LF
    %s"Version: " version LF
    %s"Chain ID: " chain-id LF
    %s"Nonce: " nonce LF
    %s"Issued At: " issued-at
    [ LF %s"Expiration Time: " expiration-time ]
    [ LF %s"Not Before: " not-before ]
    [ LF %s"Request ID: " request-id ]
    [ LF %s"Resources:"
    resources ]

scheme = ALPHA *( ALPHA / DIGIT / "+" / "-" / "." )
    ; See RFC 3986 for the fully contextualized
    ; definition of "scheme".

domain = authority
    ; From RFC 3986:
    ;     authority     = [ userinfo "@" ] host [ ":" port ]
    ; See RFC 3986 for the fully contextualized
    ; definition of "authority".

address = "0x" 40*40HEXDIG
    ; Must also conform to captilization
    ; checksum encoding specified in EIP-55
    ; where applicable (EOAs).

statement = *( reserved / unreserved / " " )
    ; See RFC 3986 for the definition
    ; of "reserved" and "unreserved".
    ; The purpose is to exclude LF (line break).

uri = URI
    ; See RFC 3986 for the definition of "URI".

version = "1"

chain-id = 1*DIGIT
    ; See EIP-155 for valid CHAIN_IDs.

nonce = 8*( ALPHA / DIGIT )
    ; See RFC 5234 for the definition
    ; of "ALPHA" and "DIGIT".

issued-at = date-time
expiration-time = date-time
not-before = date-time
    ; See RFC 3339 (ISO 8601) for the
    ; definition of "date-time".

request-id = *pchar
    ; See RFC 3986 for the definition of "pchar".

resources = *( LF resource )

resource = "- " URI
```

This specification defines the following SIWE Message fields that can be parsed from a SIWE Message by following the rules in ABNF Message Format:

- `scheme` OPTIONAL. The URI scheme of the origin of the request. 
Its value MUST be a [RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986) URI scheme.

- `domain` REQUIRED. The domain that is requesting the signing. 
Its value MUST be a [RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986) authority. The authority includes an OPTIONAL port. 
If the port is not specified, the default port for the provided scheme is assumed. 

If scheme is not specified, HTTPS is assumed by default.
- `address` REQUIRED. The Ethereum address performing the signing. 
Its value SHOULD be conformant to mixed-case checksum address encoding specified in ERC-55 where applicable.

- `statement` OPTIONAL. A human-readable ASCII assertion that the user will sign which MUST NOT include '\n' (the byte 0x0a).
- `uri` REQUIRED. An [RFC 3986](https://datatracker.ietf.org/doc/html/rfc3986) URI referring to the resource that is the subject of the signing.

- `version` REQUIRED. The current version of the SIWE Message, which MUST be 1 for this specification.

- `chain-id` REQUIRED. The EIP-155 Chain ID to which the session is bound, and the network where Contract Accounts MUST be resolved.

- `nonce` REQUIRED. A random string (minimum 8 alphanumeric characters) chosen by the relying party and used to prevent replay attacks.

- `issued-at` REQUIRED. The time when the message was generated, typically the current time. 
Its value MUST be an ISO 8601 datetime string.

- `expiration-time` OPTIONAL. The time when the signed authentication message is no longer valid. 
Its value MUST be an ISO 8601 datetime string.

- `not-before` OPTIONAL. The time when the signed authentication message will become valid. 
Its value MUST be an ISO 8601 datetime string.

- `request-id` OPTIONAL. An system-specific identifier that MAY be used to uniquely refer to the sign-in request.

- `resources` OPTIONAL. A list of information or references to information the user wishes to have resolved as part of authentication by the relying party. 
Every resource MUST be a RFC 3986 URI separated by "\n- " where \n is the byte 0x0a.

## Signing and Verifying Messages with Ethereum Accounts

- For Externally Owned Accounts, the verification method specified in [ERC-191](https://eips.ethereum.org/EIPS/eip-191) MUST be used.

- For Contract Accounts,

	- The verification method specified in [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) SHOULD be used. 
Otherwise, the implementer MUST clearly define the verification method to attain security and interoperability for both wallets and relying parties.

	- When performing [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) signature verification, the contract performing the verification MUST be resolved from the specified `chain-id`.

	- Implementers SHOULD take into consideration that [ERC-1271](https://eips.ethereum.org/EIPS/eip-1271) implementations are not required to be pure functions. 
They can return different results for the same inputs depending on blockchain state. 
This can affect the security model and session validation rules.

## Resolving Ethereum Name Service (ENS) Data

- The relying party or wallet MAY additionally perform resolution of ENS data, as this can improve the user experience by displaying human-friendly information that is related to the `address`. 
Resolvable ENS data include:
	- The primary ENS name.
	- The ENS avatar.
	- Any other resolvable resources specified in the ENS documentation.

- If resolution of ENS data is performed, implementers SHOULD take precautions to preserve user privacy and consent. 
Their `address` could be forwarded to third party services as part of the resolution process.

## Relying Party Implementer Steps
### Specifying the Request Origin
The `domain` and, if present, the `scheme`, in the SIWE Message MUST correspond to the origin from where the signing request was made. 

### Verifying a signed Message
The SIWE Message MUST be checked for conformance to the ABNF Message Format and its signature MUST be checked as defined in Signing and Verifying Messages with Ethereum Accounts.

### Creating Sessions
Sessions MUST be bound to the address and not to further resolved resources that can change.

### Interpreting and resolving Resources
Implementers SHOULD ensure that that URIs in the listed resources are human-friendly when expressed in plaintext form.

## Wallet Implementer Steps

### Verifying the Message Format
The full SIWE message MUST be checked for conformance to the ABNF defined in ABNF Message Format.

Wallet implementers SHOULD warn users if the substring `"wants you to sign in with your Ethereum account"` appears anywhere in an [ERC-191](https://eips.ethereum.org/EIPS/eip-191) message signing request unless the message fully conforms to the format defined ABNF Message Format.

### Verifying the Request Origin
Wallet implementers MUST prevent phishing attacks by verifying the origin of the request against the `scheme` and `domain` fields in the SIWE Message. 

The origin SHOULD be read from a trusted data source such as the browser window or over WalletConnect [ERC-1328](https://eips.ethereum.org/EIPS/eip-1328) sessions for comparison against the signing message contents.

Wallet implementers MAY warn instead of rejecting the verification if the origin is pointing to localhost.

The following is a RECOMMENDED algorithm for Wallets to conform with the requirements on request origin verification defined by this specification.

The algorithm takes the following input variables:

- fields from the SIWE message.
- `origin` of the signing request: the origin of the page which requested the signin via the provider.
- `allowedSchemes`: a list of schemes allowed by the Wallet.
- `defaultScheme`: a scheme to assume when none was provided. Wallet implementers in the browser SHOULD use https.
- developer mode indication: a setting deciding if certain risks should be a warning instead of rejection. Can be manually configured or derived from `origin` being localhost.

The algorithm is described as follows:

- If `scheme` was not provided, then assign `defaultScheme` as scheme.
- If `scheme` is not contained in `allowedSchemes`, then the `scheme` is not expected and the Wallet MUST reject the request. 
Wallet implementers in the browser SHOULD limit the list of allowedSchemes to just 'https' unless a developer mode is activated.
- If `scheme` does not match the scheme of origin, the Wallet SHOULD reject the request. 
Wallet implementers MAY show a warning instead of rejecting the request if a developer mode is activated. 
In that case the Wallet continues processing the request.
- If the `host` part of the `domain` and `origin` do not match, the Wallet MUST reject the request unless the Wallet is in developer mode. 
In developer mode the Wallet MAY show a warning instead and continues procesing the request.
- If `domain` and `origin` have mismatching subdomains, the Wallet SHOULD reject the request unless the Wallet is in developer mode. 
In developer mode the Wallet MAY show a warning instead and continues procesing the request.
- Let `port` be the port component of `domain`, and if no port is contained in domain, assign port the default port specified for the scheme.
- If `port` is not empty, then the Wallet SHOULD show a warning if the `port` does not match the port of `origin`.
- If `port` is empty, then the Wallet MAY show a warning if `origin` contains a specific port.
- Return request origin verification completed.

### Creating Sign-In with Ethereum Interfaces
Wallet implementers MUST display to the user the following fields from the SIWE Message request by default and prior to signing, if they are present: `scheme`, `domain`, `address`, `statement`, and `resources`. 
Other present fields MUST also be made available to the user prior to signing either by default or through an extended interface.

Wallet implementers displaying a plaintext SIWE Message to the user SHOULD require the user to scroll to the bottom of the text area prior to signing.

Wallet implementers MAY construct a custom SIWE user interface by parsing the ABNF terms into data elements for use in the interface. 
The display rules above still apply to custom interfaces.

### Supporting internationalization (i18n)
After successfully parsing the message into ABNF terms, translation MAY happen at the UX level per human language.

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
