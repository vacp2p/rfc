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
Encrypted credentials are stored in a JSON schema to securely exchange credentials between peers.

# Summary
A keystore is a construct to store a user’s keys. 
The keys will be encrypted and decrypted based on methods specified in the specification. 
This keystore specification uses [32/RLN-V1](https://rfc.vac.dev/spec/32/), Rate Limit Nullifiers, as a spam-prevention mechanism by generating zero-knowledge proofs and storing the proofs locally in the keystore.

# Background
The secure transfer of keys is important in peer-to-peer messaging applications.
A Waku RLN Keystore uses zero-knowledge proofs for anonymous rate-limiting for messaging frameworks. 
Generated credentials by a user are encrypted and stored in the keystore to be retrieved over a network.
With [32/RLN-V1](https://rfc.vac.dev/spec/32/), sending and receiving messages will ensure a message rate for a network is being followed while keeping the anonymity of the message owner. 

## Example Waku RLN Keystore:

This is an example of a keystore used by a [17/WAKU2-RLN-Relay](https://rfc.vac.dev/spec/17/).

```js

application: "waku-rln-relay",
appIdentifier: "string",
version: "string",
  credentials: {
    ["memberHash | string"]: {
      crypto: {
        cipher: "string",
        cipherparams: { object },
        ciphertext: "string",
        kdf: "pbkdf2 | string",
        kdfparams: {
          dklen: interger,
          c: interger,
          prf: "string",
          salt: "string",
        },
        mac: “SHA 256 Hash | string",
      },
    }

```

# Specification
The key words “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “NOT RECOMMENDED”, “MAY”, and “OPTIONAL” in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119) and [RFC 8174](https://datatracker.ietf.org/doc/html/rfc8174).
#

The keystore MUST be generated with a cryptographic construction for password verification and decryption.

Keystore modules MUST include metadata, key derivation function, checksum, cipher, and a membership hash.


## Metadata:
- Information about the keystore SHOULD be stored in the metadata. 
- The declaration of `application`, `version`, and `appIdentifier` COULD occur in the metadata.

`application` : current application

`version` : application version

`appIdentifier`: application identifier

## Credentials:
The Waku RLN credentials MUST consist of a `membershipHash` and `WakuCredential`
- Each contruct MUST include the keypair:
> key: [MembershipHash]: pair: [WakuCredential]

### membershipHash 
- MUST be a 256 byte hash.
- SHOULD be generated with `treeIndex`, `membershipContract`, and `identityCredential`.
- MUST not already exist in the keystore.

`treeIndex` : 
- it MUST be a RLN membership tree index in a merkle tree data structure filled with `identity_commitment` from user registrations.
As described in [32/RLN-V1](https://rfc.vac.dev/spec/32/)
- MUST be integer

`membershipContract` : 
- it MUST be a hash of a `contractId` and `contractAddress`
- `contractId` MUST be an integer.
- `contractAddess` MUST be a string.

`identityCredential` : 
- it MUST be a hash of `identity_commitment` stored in a Merkle tree.
- MUST be a string.

it MUST Consists of:
- `identity_secret`: `identity_nullifier` + `identity_trapdoor` 
  - `identity_nullifier` : Random 32 byte value used for `identity_secret` generation.
  - `identity_trapdoor` : Random 32 byte value for `identity_secret` generation.

`identity_secret_hash`: 
- it MUST be created with `identity_secret` as a parameter for the hash function.
- Used to decrypt the `identity_commitment` of the user, and as
a private input for zero-knowledge proof generation.
- The secret hash SHOULD be kept private by the user.

`identity_commitment`: 
- it MUST be created with `identity_secret_hash` for hash creation. 
- MUST be used by a user for contract registering.

### Waku Credential: 
`WakuCredential` MUST be used for password verification.

`WakuCredential` follows [EIP-2335](https://eips.ethereum.org/EIPS/eip-2335) 


### KDF:

The password-based encryption SHOULD be KDF, key derivation function, 
to produce a derived key from a password and other parameters.
Keystore COULD use PBKDF2 password based encryption, 
as described in [RFC 2898](https://www.ietf.org/rfc/rfc2898.txt).

A `WakuCredential` object MUST include:
- password: used to encrypt keystore and decryption key
- secret: key to be encrypted
- pubKey: public key
- path: HD, hardened derivation, path used to generate the secret

- checksum: hashing function 
- cipher: cipher function

```js

crypto: {
	cipher: cipher.function,
	cipherparams: cipher.parameters,
	ciphertext: cipher.message,
	kdf: kdf.function,
	kdfparams: {
		- param: salt value and iteration count
		- dklen: length in octets of derived key, MUST be positive integer
		- c: iteration count, MUST be positive integer
		- prf: Underlying pseudorandom function
		- salt: produces a large set of keys based on the password.
	},
	mac: checksum
}

```
	
### Decryption: 
- The keystore decrypts a keystore with a password and Merkle proof with PBKDF2.</br>
- Returns secret key.
To generate `decryptionKey`, MUST be constructed from password and KDF.
- The cipher.function encrypts the secret key using the decryption key.
- The `decryptionKey`, cipher.function and cipher.params MUST be used to encrypt the secret.
- If the `decryptionKey` is longer than the key size required by the cipher,
it MUST be truncated to the correct number of bits.

## Test Vectors
### Input:
Hashing function used: Poseidon Hash as described in [Poseidon Paper](https://eprint.iacr.org/2019/458.pdf)

`application`: "waku-rln-relay" <br />
`appIdentifier`: "01234567890abcdef" <br />
`version`: "0.2" <br />
`hash_function`: "poseidonHash" <br />

```js
identityCredential = {
	IDTrapdoor: [
		211, 23, 66, 42, 179, 130, 131, 111, 201, 205, 244, 34, 27, 238, 244,
        	216, 131, 240, 188, 45, 193, 172, 4, 168, 225, 225, 43, 197, 114, 176,
        	126, 9,
	],
	IDNullifier: [
        	238, 168, 239, 65, 73, 63, 105, 19, 132, 62, 213, 205, 191, 255, 209, 9,
        	178, 155, 239, 201, 131, 125, 233, 136, 246, 217, 9, 237, 55, 89, 81,
        	42,
	],
	IDSecretHash: [
        	150, 54, 194, 28, 18, 216, 138, 253, 95, 139, 120, 109, 98, 129, 146,
        	101, 41, 194, 36, 36, 96, 152, 152, 89, 151, 160, 118, 15, 222, 124,
        	187, 4,
	],
	IDCommitment: [
        	112, 216, 27, 89, 188, 135, 203, 19, 168, 211, 117, 13, 231, 135, 229,
		58, 94, 20, 246, 8, 33, 65, 238, 37, 112, 97, 65, 241, 255, 93, 171, 15,
	],
	IDCommitmentBigInt: buildBigIntFromUint8Array(
        	new Uint8Array([
          		112, 216, 27, 89, 188, 135, 203, 19, 168, 211, 117, 13, 231, 135, 229,
          		58, 94, 20, 246, 8, 33, 65, 238, 37, 112, 97, 65, 241, 255, 93, 171,
          		15,
		])
	)
}
membership = {
      chainId: "0xAA36A7",
      treeIndex: 8,
      address: "0x8e1F3742B987d8BA376c0CBbD7357fE1F003ED71",
}

```

### Output:

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
        ciphertext:     "9c72f47ce95de03ed34502d0288e7576b66b51b9e7d5ae882c27bd89f94e6a03c2c44c2ddf0c982e72003d67212105f1b64614f57cabb0ceadab7e07be165eee1121ad6b81951368a9f3be2dd99ea294515f6013d5f2bd4702a40e36cfde2ea298b23b31e5ce719d8040c3331f73d6bf44f88bca39bac0e917d8bf545500e4f40d321c235426a80f315ac70666acbd3bdf803fbc1e7e7103fed466525ed332b25d72b2dbedf6fa383b2305987c1fe276b029570519b3e79930edf08c1029868d05c2c08ab61d7c64f63c054b4f6a5a12d43cdc79751b6fe58d3ed26b69443eb7c9f7efce27912340129c91b6b813ac94efd5776a40b1dda896d61357de208c7c47a14af911cc231355c8093ee6626e89c07e1037f9e0b22c690e3e049014399ca0212c509cb04c71c7860d1b17a0c47711c490c27bad2825926148a1f15a507f36ba2cdaa04897fce2914e53caed0beaf1bebd2a83af76511cc15bff2165ff0860ad6eca1f30022d7739b2a6b6a72f2feeef0f5941183cda015b4631469e1f4cf27003cab9a90920301cb30d95e4554686922dc5a05c13dfb575cdf113c700d607896011970e6ee7d6edb61210ab28ac8f0c84c606c097e3e300f0a5f5341edfd15432bef6225a498726b62a98283829ad51023b2987f30686cfb4ea3951f3957654035ec291f9b0964a3a8665d81b16cec20fb40f944d5f9bf03ac1e444ad45bae3fa85e7465ce620c0966d8148d6e2856f676c4fbbe3ebe470453efb4bbda1866680037917e37765f680e3da96ef3991f3fe5cda80c523996c2234758bf5f7b6d052dc6942f5a92c8b8eec5d2d8940203bbb6b1cba7b7ebc1334334ca69cdb509a5ea58ec6b2ebaea52307589eaae9430eb15ad234c0c39c83accdf3b77e52a616e345209c5bc9b442f9f0fa96836d9342f983a7",
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

# Security Considerations
### 1.) Add a Password

An attacker can regenerate a keystore based on a user who registers to more than one keystore on the same `membershipContract`. 
Suggest Solution : Add a password to the construction of `membershipHash` to prevent this attack.

Suggested Construct:
- `membershipHash` SHOULD be contructed with `treeIndex`, `membershipContract`, `identityCredential`, `membershipPassword`
	- `membershipPassword` = a new password created and stored privately by the user.

# Copyright
Copyright and related rights waived via CC0.

# References
1. [58/RLN-V2](https://rfc.vac.dev/spec/58/)
2. [17/WAKU2-RLN-RELAY](https://rfc.vac.dev/spec/17/)
3. [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119)
4. [RFC 8174](https://datatracker.ietf.org/doc/html/rfc8174)
5. [EIP-2335](https://eips.ethereum.org/EIPS/eip-2335)
6. [RFC 2898](https://www.ietf.org/rfc/rfc2898.txt)



