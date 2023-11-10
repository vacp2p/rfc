---
slug: 
title: WAKU-KEYSTORE
name: Waku Keystore
status: raw
category: Standards Track
editor: Jimmy Debe <jimmy@status.im>
contributors: 
---

# Abstract
Encrypted credentials in JSON format stored in a keystore to securely exchange credentials between peers.

# Summary
A keystore is a construct that store a peer’s keys. 
The keys will be encrypted and decrypted based on methods species in the keystore. 
This specification will introduce how a keystore is constructed.

# Background
Rate Limit Nullifiers, is a method that uses zero knowledge proofs for anonymous rate-limited for messaging/ signaling frameworks.
The secure transfer of keys are important in peer to peer messaging applications. 
The Waku RLN keystore can be used by peers to store credentials securely and exchange credentials anonymously.

# Example of a Waku Keystore
This is an example of a keystore that is used by a Waku RLN Relay.

```js
application: "waku-rln-relay",
  appIdentifier: "01234567890abcdef",
  version: "0.2",
  credentials: {
    "9DB2B4718A97485B9F70F68D1CC19F4E10F0B4CE943418838E94956CB8E57548": {
      crypto: {
        cipher: "aes-128-ctr",
        cipherparams: {
          iv: "fd6b39eb71d44c59f6bf5ff3d8945c80",
        },
        ciphertext:
        "9c72f47ce95de03ed34502d0288e7576b66b51b9e7d5ae882c27bd89f94e6a03c2c44c2ddf0c982e72003d67212105f1b64614f57cabb0ceadab7e07be165eee1121ad6b81951368a9f3be2dd99ea294515f6013d5f2bd4702a40e36cfde2ea298b23b31e5ce719d8040c3331f73d6bf44f88bca39bac0e917d8bf545500e4f40d321c235426a80f315ac70666acbd3bdf803fbc1e7e7103fed466525ed332b25d72b2dbedf6fa383b2305987c1fe276b029570519b3e79930edf08c1029868d05c2c08ab61d7c64f63c054b4f6a5a12d43cdc79751b6fe58d3ed26b69443eb7c9f7efce27912340129c91b6b813ac94efd5776a40b1dda896d61357de208c7c47a14af911cc231355c8093ee6626e89c07e1037f9e0b22c690e3e049014399ca0212c509cb04c71c7860d1b17a0c47711c490c27bad2825926148a1f15a507f36ba2cdaa04897fce2914e53caed0beaf1bebd2a83af76511cc15bff2165ff0860ad6eca1f30022d7739b2a6b6a72f2feeef0f5941183cda015b4631469e1f4cf27003cab9a90920301cb30d95e4554686922dc5a05c13dfb575cdf113c700d607896011970e6ee7d6edb61210ab28ac8f0c84c606c097e3e300f0a5f5341edfd15432bef6225a498726b62a98283829ad51023b2987f30686cfb4ea3951f3957654035ec291f9b0964a3a8665d81b16cec20fb40f944d5f9bf03ac1e444ad45bae3fa85e7465ce620c0966d8148d6e2856f676c4fbbe3ebe470453efb4bbda1866680037917e37765f680e3da96ef3991f3fe5cda80c523996c2234758bf5f7b6d052dc6942f5a92c8b8eec5d2d8940203bbb6b1cba7b7ebc1334334ca69cdb509a5ea58ec6b2ebaea52307589eaae9430eb15ad234c0c39c83accdf3b77e52a616e345209c5bc9b442f9f0fa96836d9342f983a7",
        kdf: "pbkdf2",
        kdfparams: {
          dklen: 32,
          c: 1000000,
          prf: "hmac-sha256",
          salt: "60f0aa92fbf63a8356dfdbed2ab18058",
        },
        mac: "51a227ac6db7f2797c63925880b3db664e034231a4c68daa919ab42d8df38bc6",
      },
    }

```
# Specification:
- The keystore MUST be in JSON format:
- 

 ## Header Options:
- application: string;
- version: string;
- appIdentifier: string;

## Credentials: 
- [key: MembershipHash]: nWakuCredential

MembershipHash : Compute MembershipHash
 - The hash SHOULD be created with three values `membershipContract`, `treeIndex`, `identityCredential`
 
 Encode
- Tree Index = Merkle tree filled with identity commitments of peers.
  COULD be obtained with the leaf_index of identity_commitment.
  RLN membership tree filled with identity commitments of peers, a data structure that ensures peer registrations.
### Membership Hash Values
- membershipContract = (contract - ChainId, contractAddress)
Peer identity = required by RLN/V1, generate zero-knowlege proof
- idCommitment: options.identity.IDCommitment
- idNullifier: options.identity.IDNullifier
- idSecretHash: options.identity.IDSecretHash
- idTrapdoor: options.identity.IDTrapdoor

### Peer’s identity is composed of:
identity_secret: identity_nullifier + identity_trapdoor 
- This hash is created with the poseidonHash Function.
It is used to obtain the identity commitment of a peer.
Also used as input for zk proof generation.
- identity_nullifier =  Random 32 byte value used as component for identity_secret generation.
- identity_trapdoor = Random 32 byte value used as component for identity_secret generation.

identity_secret_hash: poseidonHash(identity_secret)
- poseidonHash Function = hash function for Zero Knowledge proofs 
identity_commitment: poseidonHash([identiy_secret_hash])

### Waku IdentityCredential (nWakuCredential):

nWakuCredential - follows EIP-2335 
- checksum =  Keccak256Hash 

EIP Keystore
- password
- Secret
- stubPubKey
- stubPath
 
KDF as Pbkdf2kdf
```js
	crypto: {
    		cipher: eipCrypto.cipher.function;
    		cipherparams: eipCrypto.cipher.params;
    		ciphertext: eipCrypto.cipher.message;
    		kdf: eipkdf.function;
    		kdfparams: eipkdf.params;
    		mac: checksum; located fromEipToCredential
	}

```
	
- cipherFunction = is poseidonHash implementation, Poseidon paper. 
- nWaku uses Keccak256 for hash function

## KDF: 
- KDF, key derivation function, produces a derived key from a base key and other parameters.
For this specification usesPBKDF2.
RFC 2898 - Password-Based Cryptography
- baseKey = password, 
- param = salt value and iteration count) / is used by RLN to for peer encryption
dklen= length in octets of derived key, MUST be positive integer
c= iteration count, MUST be positive integer
prf= Underlying pseudorandom function?
salt= produces a large set of keys based on the password. Builds the derived key?

## Decryption: 

# Security Considerations:
- Add a password to membership hash creation. Reason:


