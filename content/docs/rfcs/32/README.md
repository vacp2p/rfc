---
slug: 32
title: 32/RLN-SPEC
name: RLN specification
status: raw
editor: Blagoj Dimovski <blagoj.dimovski@yandex.com>
contributors:
---

# Abstract

The following document covers the specification of the RLN construct as well as the auxiliary libraries necessary for interaction with it.

# Motivation

RLN (Rate limiting nullfier) is a construct based on zero-knowledge proofs that enables spam prevention mechanism for decentralized, anonymous environments. In anonymous environments, the identity of the entities is unknown, and thus it is very hard to prevent spam attacks. Many applications would benefit from anonimity, but potential for spam could worsen their UX by large extent. RLN allows for anonymous signaling, while enabling spam prevention on protocol level.

# Technical overview

The main RLN construct is implemented using a [ZK-SNARK](https://z.cash/technology/zksnarks/) circuit. However it is helpful to describe the other necessary outside components for interaction with the circuit, which together with the ZK-SNARK circuit enable the above mentioned features.

## ZK Circuits specification

Anonymous signaling with spam detection is enabled by proving that the user is part of a group which has high bariers for entry (form of stake) and enabling secret reveal if more signals are produced than the spam threshold (system parameter) per epoch.
The membership part is implemented using membership [merkle trees](https://en.wikipedia.org/wiki/Merkle_tree) and merkle proofs, while the secret reveal part is enabled by using 
the Shamir's Secret Sharing scheme. 
Esentially the protocol requires the users to generate zero-knowledge proof to be able to send signals and participate in the application. The zero knowledge proof proves that the user is member of a group, but also enforces the user to share part of their secret for each signal in an epoch.
The epoch is an external nullifier which is usually represented by timestamp or a time interval. It can also be tought as voting booth in voting applications.

The ZK Circuit is implemented using a [Groth-16 ZK-SNARK](https://eprint.iacr.org/2016/260.pdf), using the [circomlib](https://docs.circom.io/) library.

Two different circuit implementation exists. One where the spam threshold (`limit`) = 2 - which is reffered to default RLN or just RLN, and one where the spam threshold (`limit`) > 2 - which is refered to generic RLN or NRLN. The difference is that in the circuit implementation, the NRLN accepts array of secret components as input rather than a secret hash, and also the secret equation is not a line equation but something else (depending on the `limit`). The following is a specification for the default implementation. At the end of this section the differences for NRLN are described in more depth.

### System parameters

- `n_levels` - merkle tree depth
- `limit` - spam threshold

### Circuit parameters

**Public Inputs**
- `x` - signal hash
- `epoch` - epoch
- `rln_identifier` - random finite field value, unique per RLN app (see below for more description)

**Private Inputs**
* `identity_secret` (NRLN) or `identity_secret_hash` (RLN) - the identity secret of the user
* `path_elements` - merkle tree membership proof component
* `identity_path_index` - merkle tree membership proof component

**Outputs**
- `y` - the output of the secret equation
- `root` - the membership tree root
- `nullifier` - the internal nullifier

### Hash function

Canonical [Poseidon hash implementation](https://eprint.iacr.org/2019/458.pdf) is used, as implemented in the circomlib library, according to the Poseidon paper.

### Membership implementation

For a valid signal, user's `identity commitment` (more on identity commitments below) must be exists in identity membership tree. Membership is proven by providing a membership proof (witness). The fields from the membership proof required for the verification are: `path_elements` and `identity_path_index`.

[IncrementalQuinTree](https://github.com/appliedzkp/incrementalquintree) structure is used for the Membership tree. The circuits are reused from this repository. You can find out more details about the IncrementalQuinTree algorithm [here](https://ethresear.ch/t/gas-and-circuit-constraint-benchmarks-of-binary-and-quinary-incremental-merkle-trees-using-the-poseidon-hash-function/7446).

### Slashing and Shamir's Secret Sharing

Slashing is enabled by using polynomials and [Shamir's Secret sharing](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing). In order to produce a valid proof, the user needs to provide their `identity_secret` (NRLN) or `identity_secret_hash` as a private input to the circuit. Then a secret equation is created in the form of:

`y = a_0 + x*a_1`,

where `a_0` is the `identity_secret_hash` and `a_1 = hash(a_0, epoch)`.

Along with the generated proof, the users need to provide a (x, y) share which satisfies the line equation, in order for their proof to be verified. `x` is the hashed signal, while the `y` is the circuit output. 

With more that the `limit` number of shares anyone can derive `a_0`, the `identity_secret_hash`  key. Hash of a signal will be evaluation point x. So that a member who sends more that `limit` number of signals per epoch reveals the secret key.

Note that shares used in different epochs cannot be used to derive the secret key.

### Nullifiers
`epoch` is external nullfier.

The internal nullifier is calculated as `nullifier = hash(a_1, rln_identifier)`. Note that `a_1` already contains a `identity_secret` ingredient `a_0` and epoch ingredient, so each epoch a member can signal to only one nullifier. 

The `rln_identifier` is a random value from a finite field, unique per RLN app, and is used for additional cross-application security - to protect the user secrets being compromised if they use the same credentials accross different RLN apps. If `rln_identifier` is not present, the user uses the same credentials and sends a different message for two different RLN apps using the same epoch, then their secret key can be revealed. With adding the `rln_identifier` field we obscure the nullifier, so this kind of attack cannot happen. The only kind of attack that is possible is if we have an entity with a global view of all messages, and they try to brute force different combinations of x and y shares for different nullifiers.

### NRLN differences

Other than the single signal per epoch usecase, we've implemented circuits that allow for (pseudo) arbitrary signals per epoch. To enable this we use polynomials of arbitrary degree, depending on the  spam threshold requirement for the certain usecase. 
For this implementation the user secret is represented as an array of random finite field values, and an array of secret ingredients is an input to the circuit (instead of a single finite field value such as `identity_secret_hash`). 

Let's say this array is denoted by: `rln_secret[n]`

The polynomial is now represented as:

```
y_n = a_0 + x*a_1 + x^2*a_2 + x^3*a_3 + ... + x^n*a_n 
```

where `a_0 = poseidonHash([rln_secret[0], rln_secret[1],..., rln_secret[limit - 1]])`,

each `a_i`, when `i > 0 && i <= n` = `a_i = poseidonHash([rln_secret[i-1] * epoch])` 

The user will need to send more than `n` signals per epoch to be eligible for slashing.

The internal nullifier is calculated as:

```
nullifier = poseidonHash([a_1, a_2, ... a_n, rln_identifier])
```

## Identity credentials generation

In order to be able to generate valid proofs, the users need to be part of the identity membership merkle tree. They are part of the identity membership merkle tree if their `identity_commitment` is placed in a leaf in the tree.

The identity credentials of a user are composed of:

- `identity_secret`
- `identity_commitment`

### `identity_secret`

The secret component array (`secret_component_arr`) for the `identity_secret` is generated in the following way:

```
    initial_component = identity_trapdoor^identity_nullifier
    secret_component_arr = [initial_component]
    for(i = 1; i < parts; i++) {
      secret_component_arr.push(initial_component^(i+1))
    }
    
    identity_secret = secret_component_arr
```

Although other methods can be used for the secret generation, this algorithm is used specifically to avoid collisions with other ZK protocols, such as Semaphore. The same secret should not be used accross different protocols, because revealing the secret at one protocol could break privacy for the user in the other protocols.  


### `identity_commitment`

The `identity_commitment` is generated in the following way:

```
identity_secret_hash = poseidonHash(identity_secret)
identity_commitment = poseidonHash([identity_secret_hash])
```


# Flow

The users participate in the protocol by first registering to the application. The application could have different form of registration. The goal of this step is the user to become a member of the group. Usually the membership requires financial or social stake, as a barrier for entry is required for disincentivising the users from spamming.
After they are part of the memebrship group, the users can generate ZK-Proofs for their signals and can participate in the application. 
If a user generates more signals than allowed in an epoch, the user risks being slashed - by revealing his secret and revealing his identity commitment. If financial stake is put in place, the user also risks his stake being taken.

## Registration

Depending on the application requirements, the registration can be implemented in different ways, for example: 
- centralized registrations, by using a central server
- decentralized registrations, by using a smart contract

What is important is that the user's identity commitment is stored in a merkle tree, and the users can obtain a merkle proof proving that they are part of the group.

Also depending on the application requirements, usually a financial or social stake is introduced.

An example for financial stake is: For each registration a certain amount of ETH is required.
An example for social stake is using InterRep as a registry - user's need to prove that they have a highly reputable social media account.

The user's identity is composed of:

```
{
    identity_secret: secret_component_arr,
    identity_secret_hash: poseidonHash(identity_secret),
    identity_commitment: poseidonHash([identity_secret_hash])
}
```

For registration, the user needs to submit their `identity_commitment` (along with any additional registration requirements) to the registry party. Upon registration, they should receive `leaf_index` value which represents their position in the merkle tree. Receiving a `leaf_index` is not a hard requirement and is application specific. The other way around is the users calculating the `leaf_index` themselves upon successful registration.

## Signaling

After registration, the users can participate in the application by sending signals to the other participants in a decentralised manner or to a centralised server. Along with their signal, they need to generate a ZK-Proof by using the circuit with the specification described above. 

For generating a proof, the users need to obtain the required parameters or compute them themselves, depending on the application implementation and client libraries supported by the application.
For example the users can store the memebrship merkle tree on their end and generate a merkle proof whenever they want to generate a signal. 

### Signal hash

The signal hash is generated by hashing the raw signal (or content) using the `keccak256` hash function, converting the hashed value to number and padding it:

```
  signal: string = "signal"
  converted: string = toHex(toUtf8Bytes(signal))
  signal_hash: bigint = BigInt(keccak256(converted)) >> BigInt(8)
```


### Epoch

The epoch (or external nullifier) is also obtained by hasing a raw string value using `keccak256`:

```
    raw_epoch: string = "1234"
    converted: string = keccak256(raw_epoch)
    hex_string: string = converted.slice(8, 64)
    epoch: string = "0x" + hex_string.padStart(64, "0")
```


### Merkle proof

The merkle proof should be obtained locally or from a trusted third party. By using the incremental merkle tree algorithm, the merkle can be obtained by providing the `leaf_index` of the `identity_commitment`. The proof (`merkle_proof`) is composed of the following fields:

```
{
  root: bigint
  indices: number[]
  path_elements: bigint[][]
}
```


### Generating proof

For proof generation, the user need to submit the following fields to the circuit:

```
    {
      identity_secret: identity_secret || identity_secret_hash,
      path_elements: merkle_proof.path_elements,
      identity_path_index: merkle_proof.indices,
      x: signal_hash,
      epoch: epoch,
      rln_identifier: rlnIdentifier
    }
```

The generated proof (`zk_proof`) is composed of the following fields:

```
{
  pi_a: Array<string>,
  pi_b: [ [Array<string>], [Array<string>], [Array<string>] ],
  pi_c: Array<string>
  protocol: string,
  curve: string
}
```

### Calculating output

The proof output is calculated locally, in order for the required fields for proof verification to be sent along with the proof. The proof output is composed of the `y` share of the secret equation and the `nullifier`. The following fields are needed for proof output calculation:

```
{
  identity_secret: bigint[] || bigint, 
  epoch: bigint, 
  x: bigint, 
  limit: number, 
  rln_identifier: bigint
}
```

The output `[y, nullifier]` is calculated in the following way:

```
a_0 = poseidonHash([identity_secret[0], identity_secret[1],..., identity_secret[limit - 1]]), for i == 0
a_i = poseidonHash([identity_secret[i-1] * epoch]), for i > 0 && i <= n

y = a_0 + x*a_1 + x^2*a_2 + x^3*a_3 +  ... + x^i*a_i + ... + x^n*a_n 

nullifier = poseidonHash([a_1, a_2, ... a_n, rln_identifier])
```



### Sending proof

The user's output message (`output_message`), containing the signal should contain the following fields at minimum:

```
{
    content: signal, # non-hashed signal
    proof: zk_proof,
    nullifier: nullifier,
    x: x, # signal_hash
    y: y,
    rln_identifier: rln_identifier
}
```

Additionally depending on the application, the following fields might be required:

```
  {
      root: merkle_proof.root,
      epoch: epoch
  }
```


## Verification and slashing

The slashing implementation is dependent on the type of application. If the application is implemented in a centralised manner, and everything is stored on a single server, the slashing will be implemented only on the server. Otherwise if the application is distributed, the slashing will be implemented on each user's client.

Each user of the protocol (server or otherwise) will need to store metadata for each message received by each user, for the given epoch. The data can be deleted when the epoch passes. Storing metadata is required, so that if a user sends more than the `limit` messages per epoch, they can be slashed and removed from the protocol. The metadata stored contains the `x`, `y` shares and the `nullifier` for the user for each message. If enough such shares are present, the user's secret can be retreived.

One way of storing received metadata (`messaging_metadata`) is the following format: 

```
{
    [epoch]: {
        [nullifier]: {
            x_shares: [],
            y_shares: []
        }
    }
}
```

### Verification

The output message verification consists of the following steps:
- `epoch` correctness
- non-duplicate message check
- `zk_proof` verification
- spam verification

**1. `epoch` correctness**
Upon received `output_message`, first the `epoch` field is checked, to ensure that the message matches the current epoch. 
If the epoch is correct the verification continues, otherwise the message is discarded.

**2. non-duplicate message check**
The received message is checked to ensure it is not duplicate. The duplicate message check is performed by verifying that the `x` and `y` fields do not exist in the `messaging_metadata` object. If the `x` and `y` fields exist in the `x_shares` and `y_shares` array for the `epoch` and the `nullifier` the message can be considered as a duplicate. Duplicate messages are discarded.

**3. `zk_proof` verification**

The `zk_proof` should be verified by providing the `zk_proof` field to the circuit verifier along with the `public_signal fields`:

```
[
    y,
    merkle_proof.root,
    nullifier,
    signal_hash,
    epoch,
    rln_identifier
]
```

If the proof verification is correct, the verification continues, otherwise the message is discarded.

**4. spam verification**

After the proof is verified The `x`, and `y` fields are added to the `x_shares` and `y_shares` arrays of the `messaging_metadata` `epoch` and `nullifier` object. If the length of the arrays is equal to the spam threshold (`limit`), the user can be slashed.

### Slashing

After the verification, the user can be slashed if enough shares are present to reconstruct their `identity_secret_hash` from `x_shares` and `y_shares` fields, for their `nullifier` in an `epoch`.
The secret can be retreived by the properties of the Shamir's secret sharing scheme. In particular the secret (`a_0`) can be retrieved by computing `Lagrange polynomials`. 

After the secret is retreived, the user's `identity_commitment` can be generated from the secret and it can be used for removing the user for the membership merkle tree (zeroing out the leaf that contains the user's `identity_commitment`). 
Additionally, depending on the application the `identity_secret_hash` can be used for taking the user's provided stake.

# Auxiliary tooling

There are few additional tools implemented for easier integrations and usage of the RLN protocol.

[`zk-kit`](https://github.com/appliedzkp/zk-kit) is a typescript library which exposes APIs for identity credentials generation, as well as proof generation. It supports various protocols (`Semaphore`, `RLN`, `NRLN`), you can read more about it [here](https://github.com/appliedzkp/zk-kit).

[`zk-keeper`](https://github.com/akinovak/zk-keeper) is a browser plugin which allows for safe credential storing and proof generation. You can think of MetaMask for ZK-Proofs. It uses `zk-kit` under the hood. You can read more about it here.


# References

- [1] https://medium.com/privacy-scaling-explorations/rate-limiting-nullifier-a-spam-protection-mechanism-for-anonymous-environments-bbe4006a57d
- [2] https://github.com/appliedzkp/zk-kit
- [3] https://github.com/akinovak/zk-keeper
- [4] https://z.cash/technology/zksnarks/
- [5] https://en.wikipedia.org/wiki/Merkle_tree
- [6] https://eprint.iacr.org/2016/260.pdf
- [7] https://docs.circom.io/
- [8] https://eprint.iacr.org/2019/458.pdf
- [9] https://github.com/appliedzkp/incrementalquintree
- [10] https://ethresear.ch/t/gas-and-circuit-constraint-benchmarks-of-binary-and-quinary-incremental-merkle-trees-using-the-poseidon-hash-function/7446
- [11] https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing
