---
slug: 72
title: 72/WAKU-RLN-KEYSTORE
name: Waku RLN Keystore
status: raw
category: Standards Track
editor: Jimmy Debe <jimmy@status.im>
contributors: 
---

# Abstract
Encrypted credentials stored in a JSON schema to securely exchange credentials between peers.

# Summary
A keystore is a construct to store a user’s keys. 
The keys will be encrypted and decrypted based on methods specified in the specification. 
This keystore uses RLN, Rate Limit Nullifiers, as a spam-prevention mechanism by generating zero knowledge proofs and storing proofs locally in the keystore.

# Background
A Waku RLN Keystore uses RLN which is a method that uses zero knowledge proofs for anonymous rate-limiting for messaging frameworks.
The secure transfer of keys are important in peer to peer messaging applications. 
RLN help users receive and send messages from trusted parties.
It will ensure a message rate is being followed in the network while keeping the anonymity of the message owner. 


## Example Waku RLN Keystore:

This is an example of a keystore that is used by a Waku RLN Relay.

```js
application: "waku-rln-relay",
appIdentifier: "01234567890abcdef",
version: "0.2",
  credentials: {
    "9DB2B4718A97485B9F70F68D1CC19F4E10F0B4CE943418838E94956CB8E57548": {
      crypto: {
        cipher: "aes-128-ctr",
        cipherparams: { iv: “ “,  },
        ciphertext: "",
        kdf: "pbkdf2",
        kdfparams: {
          dklen: interger,
          c: interger,
          prf: "hmac-sha256",
          salt: "hash",
        },
        mac: “Byte 256 Hash",
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


