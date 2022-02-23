---
slug: 32
title: 32/RLN
name: RLN specification
status: raw
editor: Blagoj Dimovski <blagoj.dimovski@yandex.com>
contributors: <TO BE ADDED>
---

# Abstract

The following document covers the specification of the RLN construct as well as the auxiliary libraries necessary for interaction with it.

# Motivation

Rate limiting nullified (RLN) is a construct based on zero-knowledge proofs that provides an anonymous rate-limited signaling/messaging framework suitable for decentralized (and centralized) environments. 
Anonymity refers to the unlinkability of messages to their owner.
RLN guarantees a messaging rate is enforced cryptographically while preserving the anonymity of the message owners.
A wide range of applications can benefit from RLN and provide desirable security features. 
For example, an e-voting system can integrate RLN to contain the voting rate while protecting the voters-vote unlinkability. 
Another use case is to protect an anonymous messaging system against DDoS and spam attacks by containing messaging rate of users. 
This latter use case is explained in [17/Waku-RLN-Relay RFC](/spec/17).


# Technical overview

The main RLN construct is implemented using a [ZK-SNARK](https://z.cash/technology/zksnarks/) circuit. 
However it is helpful to describe the other necessary outside components for interaction with the circuit, 
which together with the ZK-SNARK circuit enable the above mentioned features.


## Terminology

| Term                      | Description                                                                         |
|---------------------------|-------------------------------------------------------------------------------------|
| **ZK-SNARK**    | https://z.cash/technology/zksnarks/              |
| **Stake**                 | Financial or social stake required for registering in the RLN applications. The staking requirement is necessary for disincentivising the end users to spam. A common stake examples are: locking cryptocurrency (financial), linking reputable social identity. |
| **RLN Registration**    | The process of the user registering themselves to the RLN protocol. The registrations can occur onchain or offchain. Financial or social stake is required for registering to the protocol.   |
| **Slashing**              | The process of retreiving the secret hash from the user that spams, and from the secret hash generating the identity commitment. Having the secret hash of a user implies that anyone could reveal their "identity" and also retreive their financial stake if it is enabled by the app.      |
| **Identity nullifier**    | Random 32 byte value used as component for identity secret generation.              |
| **Identity trapdoor**     | Random 32 byte value used as component for identity secret generation.              |
| **Identity secret**       | An array of two unique random components (identity nullifier and identity trapdoor), which must be kept secret from the users. Secret hash and identity commitment are derived from this array.        |
| **Identity secret hash**  | The hash of the identity secret, obtained using the Poseidon hash function. It is used for deriving the identity commitment of the user, and as a private input for zk proof generation. The secret hash should be treated as "private key" for the RLN protocol. The anonymity and security of the user is dependent on the secret hash.                                       |
| **Identity commitment**   | Hash obtained from the `Identity secret hash` by using the poseidon hash function. It is used by the users for registering in the protocol.                                       |
| **Signal**                | The message that needs to be sent by the user in a privacy preserving way. The message can be of different type, depending on the application. It can be a chat message, url request, protobuf message. |
| **Signal hash**           | Hash of the signal, used as an input in the RLN circuit. |
| **Epoch**                 | An identifier that groups signals and determines the validity of the signal. Can be though ot as voting booth. Usually a timestamp or time interval in the RLN apps. |
| **RLN Identifier**        | Random finite field value unique per RLN app. It is used for additional cross-application security. The role of the RLN identifier is protection of the user secrets being compromised if signals are being generated with the same credentials at different apps. |
| **RLN membership tree**   | Merkle tree data structure, filled with identity commitments of the users. Serves as a data structure that ensures user registrations.   |
| **Merkle proof**          | Proof that a user is member of the RLN membership tree.    |


## RLN ZK-Circuit specific terms

| Term           | Description                                                       |
|---------------------------|-------------------------------------------------------------------------------------|
| **X Share**               | Hash of the signal, public input in the RLN circuit.    |
| **A0**               | The identity secret hash.    |
| **A1**               | Poseidon hash of [a0, external_nullifier] (see about External nullifier below).    |
| **Y Share**          | The result of the polynomial equation (y = a0 + a1*x). Public output of the circuit.    |
| **External nullifier**    | `keccak256` hash of the Epoch.    |
| **Internal nullifier**    | Poseidon hash of [a1, rln_identifier]. This field ensures that a user can send only one valid signal per epoch without risking being slashed. Public output of the circuit.   |



## ZK Circuits specification

Anonymous signaling with spam detection is enabled by proving that the user is part of a group which has high barriers to entry (form of stake) and 
enabling secret reveal if more signals are produced than the spam threshold (system parameter) per epoch.
The membership part is implemented using membership [merkle trees](https://en.wikipedia.org/wiki/Merkle_tree) and merkle proofs, 
while the secret reveal part is enabled by using the Shamir's Secret Sharing scheme. 
Essentially the protocol requires the users to generate zero-knowledge proof to be able to send signals and participate in the application. 
The zero knowledge proof proves that the user is member of a group, 
but also enforces the user to share part of their secret for each signal in an epoch.
The epoch is an external nullifier which is usually represented by timestamp or a time interval. 
It can also be thought of as a voting booth in voting applications.

The ZK Circuit is implemented using a [Groth-16 ZK-SNARK](https://eprint.iacr.org/2016/260.pdf), 
using the [circomlib](https://docs.circom.io/) library.


### System parameters

- `n_levels` - merkle tree depth


### Circuit parameters

**Public Inputs**
- `x` 
- `epoch`
- `rln_identifier`

**Private Inputs**
* `identity_secret_hash`
* `path_elements` - rln membership proof component
* `identity_path_index` - rln membership proof component

**Outputs**
- `y`
- `root` - the rln membership tree root
- `internal_nullifier`

### Hash function

Canonical [Poseidon hash implementation](https://eprint.iacr.org/2019/458.pdf) is used, 
as implemented in the [circomlib library](https://github.com/iden3/circomlib/blob/master/circuits/poseidon.circom), according to the Poseidon paper.
This Poseidon hash version (canonical implementation) is uses the following parameters:

- `t`: 3
- `RF`: 8
- `RP`: 57

### Membership implementation

For a valid signal, a user's `identity_commitment` (more on identity commitments below) must exist in identity membership tree.
Membership is proven by providing a membership proof (witness). 
The fields from the membership proof required for the verification are: 
`path_elements` and `identity_path_index`.

[IncrementalQuinTree](https://github.com/appliedzkp/incrementalquintree) algorithm is used for constructing the Membership merkle tree. 
The circuits are reused from this repository. 
You can find out more details about the IncrementalQuinTree algorithm [here](https://ethresear.ch/t/gas-and-circuit-constraint-benchmarks-of-binary-and-quinary-incremental-merkle-trees-using-the-poseidon-hash-function/7446).

### Slashing and Shamir's Secret Sharing

Slashing is enabled by using polynomials and [Shamir's Secret sharing](https://en.wikipedia.org/wiki/Shamir%27s_Secret_Sharing). 
In order to produce a valid proof, `identity_secret_hash` as a private input to the circuit. 
Then a secret equation is created in the form of:

```
y = a_0 + x*a_1,
```

where `a_0` is the `identity_secret_hash` and `a_1 = hash(a_0, epoch)`.
Along with the generated proof, 
the users need to provide a (x, y) share which satisfies the line equation, 
in order for their proof to be verified. 
`x` is the hashed signal, while the `y` is the circuit output. 
With more than one pair of unique shares, anyone can derive `a_0`, the `identity_secret_hash` . 
Hash of a signal will be evaluation point x. 
So that a member who sends more that one unique signal per epoch risks their identity secret being revealed.

Note that shares used in different epochs and different RLN apps cannot be used to derive the secret key.

### Nullifiers
`epoch` is external nullfier (`external_nullifier`).

The internal nullifier is calculated as `internal_nullifier = hash(a_1, rln_identifier)`. 
Note that `a_1` already contains a secret ingredient `a_0` and epoch ingredient, 
so each epoch a member can signal to only one nullifier. 

The `rln_identifier` is a random value from a finite field, 
unique per RLN app, 
and is used for additional cross-application security - to protect the user secrets being compromised if they use the same credentials accross different RLN apps. 
If `rln_identifier` is not present, 
the user uses the same credentials and sends a different message for two different RLN apps using the same epoch,
then their secret key can be revealed. 
With adding the `rln_identifier` field we obscure the nullifier, 
so this kind of attack cannot happen. 
The only kind of attack that is possible is if we have an entity with a global view of all messages, 
and they try to brute force different combinations of `x` and `y` shares for different nullifiers.


## Identity credentials generation

In order to be able to generate valid proofs, the users need to be part of the identity membership merkle tree. 
They are part of the identity membership merkle tree if their `identity_commitment` is placed in a leaf in the tree.

The identity credentials of a user are composed of:

- `identity_secret`
- `identity_secret_hash`
- `identity_commitment`

### `identity_secret`

The `identity_secret` is generated in the following way:

```
    identity_nullifier = random_32_byte_buffer
    identity_trapdoor = random_32_byte_buffer
    identity_secret = [identity_nullifier, identity_trapdoor]
```

The same secret should not be used accross different protocols, 
because revealing the secret at one protocol could break privacy for the user in the other protocols.  

### `identity_secret_hash`

The `identity_secret_hash` is generated by obtaining a Poseidon hash of the `identity_secret` array:

```
    identity_secret_hash = poseidonHash(identity_secret)
```

### `identity_commitment`

The `identity_commitment` is generated by obtaining a Poseidon hash of the `identity_secret_hash`: 

```
identity_commitment = poseidonHash([identity_secret_hash])
```


# Flow

The users participate in the protocol by first registering to an application-defined group referred by the _membership group_.
Registration to the group is mandatory for signaling in the application. 
After registration, group members can generate Zero-knowledge Proof of membership for their signals and can participate in the application. 
Usually, the membership requires a financial or social stake which 
is beneficial for the prevention of Sybil attacks and double-signaling.
Group members are allowed to send one signal per epoch.
If a user generates more signals than allowed in an epoch, 
the user risks being slashed - by revealing his membership secret credentials. secret and revealing his identity commitment. 
If the financial stake is put in place, the user also risks his stake being taken.

Generally the flow can be described by the following steps:

1. Registration
2. Signaling
3. Verification and slashing


## Registration

Depending on the application requirements, the registration can be implemented in different ways, for example: 
- centralized registrations, by using a central server
- decentralized registrations, by using a smart contract

What is important is that the user's identity commitment is stored in a merkle tree, 
and the users can obtain a merkle proof proving that they are part of the group.

Also depending on the application requirements, 
usually a financial or social stake is introduced.

An example for financial stake is: For each registration a certain amount of ETH is required.
An example for social stake is using InterRep as a registry - 
users need to prove that they have a highly reputable social media account.

### Implementation notes

#### User identity

The user's identity is composed of:

```
{
    identity_secret: [identity_nullifier, identity_trapdoor],
    identity_secret_hash: poseidonHash(identity_secret),
    identity_commitment: poseidonHash([identity_secret_hash])
}
```

For registration, the user needs to submit their `identity_commitment` (along with any additional registration requirements) to the registry. 
Upon registration, they should receive `leaf_index` value which represents their position in the merkle tree. 
Receiving a `leaf_index` is not a hard requirement and is application specific. 
The other way around is the users calculating the `leaf_index` themselves upon successful registration.

## Signaling

After registration, 
the users can participate in the application by sending signals to the other participants in a decentralised manner or to a centralised server. 
Along with their signal, 
they need to generate a ZK-Proof by using the circuit with the specification described above. 

For generating a proof, 
the users need to obtain the required parameters or compute them themselves, 
depending on the application implementation and client libraries supported by the application.
For example the users can store the membership merkle tree on their end and 
generate a merkle proof whenever they want to generate a signal. 

### Implementation notes

#### Signal hash

The signal hash can be generated by hashing the raw signal (or content) using the `keccak256` hash function.

#### Epoch

The epoch can be generated by hashing a raw string (i.e UNIX timestamp) value using `keccak256`.

####  Obtaining merkle proof

The merkle proof should be obtained locally or from a trusted third party. 
By using the [incremental merkle tree algorithm](https://github.com/appliedzkp/incrementalquintree/blob/master/ts/IncrementalQuinTree.ts), 
the merkle can be obtained by providing the `leaf_index` of the `identity_commitment`. 
The proof (`merkle_proof`) is composed of the following fields:

```
{
  root: bigint
  indices: number[]
  path_elements: bigint[][]
}
```


#### Generating proof

For proof generation, 
the user need to submit the following fields to the circuit:

```
    {
      identity_secret: identity_secret_hash,
      path_elements: merkle_proof.path_elements,
      identity_path_index: merkle_proof.indices,
      x: signal_hash,
      epoch: epoch,
      rln_identifier: rln_identifier
    }
```


#### Calculating output

The proof output is calculated locally, 
in order for the required fields for proof verification to be sent along with the proof. 
The proof output is composed of the `y` share of the secret equation and the `internal_nullifier`. 
The following fields are needed for proof output calculation:

```
{
  identity_secret_hash: bigint, 
  epoch: bigint, 
  rln_identifier: bigint,
  x: bigint, 
}
```

The output `[y, internal_nullifier]` is calculated in the following way:

```
a_0 = identity_secret
a_1 = poseidonHash([a0, epoch])

y = a_0 + x * a_1

internal_nullifier = poseidonHash([a_1, rln_identifier])
```


#### Sending the output message

The user's output message (`output_message`), 
containing the signal should contain the following fields at minimum:

```
{
    content: signal, # non-hashed signal
    proof: zk_proof,
    internal_nullifier: internal_nullifier,
    x: x, # signal_hash
    y: y,
    rln_identifier: rln_identifier
}
```

Additionally depending on the application, 
the following fields might be required:

```
  {
      root: merkle_proof.root,
      epoch: epoch
  }
```


## Verification and slashing

The slashing implementation is dependent on the type of application. 
If the application is implemented in a centralised manner, 
and everything is stored on a single server, 
the slashing will be implemented only on the server. 
Otherwise if the application is distributed, 
the slashing will be implemented on each user's client.

### Implementation notes

Each user of the protocol (server or otherwise) will need to store metadata for each message received by each user, 
for the given epoch. 
The data can be deleted when the epoch passes. 
Storing metadata is required, so that if a user sends more than one unique signal per epoch, 
they can be slashed and removed from the protocol. 
The metadata stored contains the `x`, `y` shares and the `internal_nullifier` for the user for each message. 
If enough such shares are present, the user's secret can be retreived.

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

#### Verification

The output message verification consists of the following steps:
- `epoch` correctness
- non-duplicate message check
- `zk_proof` verification
- spam verification

**1. `epoch` correctness**
Upon received `output_message`, first the `epoch` field is checked, 
to ensure that the message matches the current epoch. 
If the epoch is correct the verification continues, otherwise the message is discarded.

**2. non-duplicate message check**
The received message is checked to ensure it is not duplicate. 
The duplicate message check is performed by verifying that the `x` and `y` fields do not exist in the `messaging_metadata` object. 
If the `x` and `y` fields exist in the `x_shares` and `y_shares` array for the `epoch` and the `internal_nullifier` the message can be considered as a duplicate. 
Duplicate messages are discarded.

**3. `zk_proof` verification**

The `zk_proof` should be verified by providing the `zk_proof` field to the circuit verifier along with the `public_signal`:

```
[
    y,
    merkle_proof.root,
    internal_nullifier,
    signal_hash,
    epoch,
    rln_identifier
]
```

If the proof verification is correct, 
the verification continues, otherwise the message is discarded.

**4. spam verification**

After the proof is verified The `x`, and `y` fields are added to the `x_shares` and `y_shares` arrays of the `messaging_metadata` `epoch` and `internal_nullifier` object. 
If the length of the arrays is equal to the spam threshold (`limit`), the user can be slashed.

#### Slashing

After the verification, the user can be slashed if enough shares are present to reconstruct their `identity_secret_hash` from `x_shares` and `y_shares` fields, 
for their `internal_nullifier` in an `epoch`.
The secret can be retreived by the properties of the Shamir's secret sharing scheme. 
In particular the secret (`a_0`) can be retrieved by computing `Lagrange polynomials`. 

After the secret is retreived,
 the user's `identity_commitment` can be generated from the secret and it can be used for removing the user from the membership merkle tree (zeroing out the leaf that contains the user's `identity_commitment`). 
Additionally, depending on the application the `identity_secret_hash` can be used for taking the user's provided stake.


# Appendix A: Security considerations

RLN is an experimental and still un-audited technology. This means that the circuits have not been yet audited. 
Another consideration is the security of the underlying primitives. 
zk-SNARKS require a trusted setup for generating a prover and verifier keys.
The standard for this is to use trusted [Multi-Party Computation (MPC)](https://en.wikipedia.org/wiki/Secure_multi-party_computation) ceremony, 
which requires two phases. 
Trusted MPC ceremony has not yet been performed for the RLN circuits.

# Apendix B: Auxiliary tooling

There are few additional tools implemented for easier integrations and usage of the RLN protocol.

[`zk-kit`](https://github.com/appliedzkp/zk-kit) is a typescript library which exposes APIs for identity credentials generation, 
as well as proof generation. 
It supports various protocols (`Semaphore`, `RLN`),.

[`zk-keeper`](https://github.com/akinovak/zk-keeper) is a browser plugin which allows for safe credential storing and proof generation. 
You can think of MetaMask for ZK-Proofs. 
It uses `zk-kit` under the hood.

# Apendix C: Example usage

The following examples are code snippets using the `zk-kit` library. 
The examples are written in [typescript](https://www.typescriptlang.org/).


## Generating identity credentials

```typescript
      import { ZkIdentity } from "@zk-kit/identity"

      const identity = new ZkIdentity()
      const identityCommitment = identity.genIdentityCommitment()
      const secretHash = identity.getSecretHash()
```

## Generating proof

```typescript
    import { RLN, MerkleProof, FullProof, genSignalHash, generateMerkleProof, genExternalNullifier } from '@zk-kit/protocols'
    import { ZkIdentity } from '@zk-kit/identity'
    import { bigintToHex, hexToBigint } from 'bigint-conversion'
    
      const ZERO_VALUE = BigInt(0);
      const TREE_DEPTH = 15;
      const LEAVES_PER_NODE = 2;
      const LEAVES = [...]; // leaves should be an array of the leaf values of the membership merkle tree
      // the identity commitment generated below should be present in the LEAVES array

      // this is for illustrative purposes only. The identityCommitment should be present in the LEAVES array above.
      const identity = new ZkIdentity()
      const secretHash = identity.getSecretHash()
      const identityCommitment = identity.genIdentityCommitment()

      const signal = "hey"
      const signalHash = genSignalHash(signal)
      const epoch = genExternalNullifier("test-epoch")
      const rlnIdentifier = RLN.genIdentifier()

      const merkleProof = generateMerkleProof(TREE_DEPTH, ZERO_VALUE, LEAVES_PER_NODE, LEAVES, identityCommitment)
      const witness = RLN.genWitness(secretHash, merkleProof, epoch, signal, rlnIdentifier)

      const [y, nullifier] = RLN.calculateOutput(secretHash, BigInt(epoch), rlnIdentifier, signalHash)
      const publicSignals = [y, merkleProof.root, nullifier, signalHash, epoch, rlnIdentifier]

      const wasmFilePath = path.join(zkeyFiles, "rln", "rln.wasm") // path to the WASM compiled circuit
      const finalZkeyPath = path.join(zkeyFiles, "rln", "rln_final.zkey") // path to the prover key

      const fullProof = await RLN.genProof(witness, wasmFilePath, finalZkeyPath)
```

## Verifiying proof

```typescript
    import { RLN } from '@zk-kit/protocols'
    
    // Public signal and the proof are received from the proving party
    // const publicSignals = [y, merkleProof.root, nullifier, signalHash, epoch, rlnIdentifier]
    // const proof = (await RLN.genProof(witness, wasmFilePath, finalZkeyPath)).proof

    const vkeyPath = path.join(zkeyFiles, "rln", "verification_key.json") // Path to the verifier key
    const vKey = JSON.parse(fs.readFileSync(vkeyPath, "utf-8")) // The verifier key

    const response = await RLN.verifyProof(vKey, { proof: proof, publicSignals })
```

For more details please visit the [`zk-kit`](https://github.com/appliedzkp/zk-kit) library.

# Copyright 

Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/)

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
- [12] https://research.nccgroup.com/2020/06/24/security-considerations-of-zk-snark-parameter-multi-party-computation/