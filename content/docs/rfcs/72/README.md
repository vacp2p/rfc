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
    [memeberHash | string]: {
      crypto: {
        cipher: "aes-128-ctr",
        cipherparams: { iv: “ “,  },
        ciphertext: "string",
        kdf: "pbkdf2",
        kdfparams: {
          dklen: interger,
          c: interger,
          prf: "hmac-sha256",
          salt: "hash",
        },
        mac: “Byte256 Hash",
      },
    }

```
# Specification
- The keystore is constructed with generated cryptographic constructions with password verification and secret decryption.
- - A keystore object contains a few modules including metadata, kdf, checksum, and cipher.
- Each contruct MUST include a keypair in credentials.
> key: [MembershipHash]: pair: [nWakuCredential]

nWakuCredential - follows EIP-2335

## Metadata:
The metadata of the keystore will consist of the declaration of `application`, `version`, and `appIdentifier` being used.

`application` : string </br>
`version` : string </br>
`appIdentifier`: string </br >

## Credentials:
The Waku RLN credentials consists of a `membershipHash` and `nWakuCredential`

### membershipHash 
- a 256 byte generated hash of `treeIndex`, `membershipContract`, and `identityCredential`

`treeIndex` : is a Merkle tree filled with identity commitments of peers. 
RLN membership tree, merkle tree data structure filled with identity commitments of users. 
As described in 32/RLN-V1

`membershipContract` : consist of a `contractId` and `contractAddress`

`identityCredential` : user’s commitments that are stored in a merkle tree.
Consists of:
`identity_secret`: `identity_nullifier` + `identity_trapdoor` 
- `identity_nullifier` : Random 32 byte value used for identity_secret generation.
- `identity_trapdoor` : Random 32 byte value for identity_secret generation.

`identity_secret_hash`: Created with `identity_secret` as parameter for hash function
- Used to decrypt the identity commitment of the user, and as a private input for zero knowlegde proof generation.
The secret hash should be kept private by the user.

`identity_commitment`: Created with `identity_secret_hash` as parameter for hash function. 
Used by users for registering protocol.

### Waku Credential: 
Waku credential is used for password verification
nWakuCredential follows EIP-2335 

- A Keystore credential object SHOULD include:
- password: used to encrypt keystore, for decryption key
- Secret: key to be encrypted
- PubKey: public key
- Path: HD path used to generate the secret

- checksum: hashing function 
- cipher: cipher function

### KDF:
The password based encryption SHOULD be KDF, key derivation function, produces a derived key from a password and other parameters.
Keystore SHOULD use PBKDF2 password based encryption, as described in RFC 2898

```js
	crypto: {
    		cipher: cipher.function,
    		cipherparams: cipher.parameters,
    		ciphertext: cipher.message,
    		kdf: kdf.function,
    		kdfparams: {
		- param = salt value and iteration count)
		- dklen= length in octets of derived key, MUST be positive integer
		- c= iteration count, MUST be positive integer
		- prf= Underlying pseudorandom function?
		- salt= produces a large set of keys based on the password.
		},
    		mac: checksum
	}
```
	
## Decryption: 
To decrypt a merkle proof with password and merkle proof PBKDF2.
Returns secert key.

# Security Considerations:
- Add a password to membership hash creation. Reason:


