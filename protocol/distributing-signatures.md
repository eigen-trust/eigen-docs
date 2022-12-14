---
description: >-
  This page describes how we use DHT to distribute signatures which contain
  opinion our opinions toward other peers (our neighbours).
---

# Distributing Signatures

### Where do we store signatures?

* Our assumption is that, in a DHT network, when you insert a key, the protocol will find the `n` closest peers to that key, and make them providers for that key. In our case the keys will be the public keys of everyone in the network, and the values will be latest signatures from the peers we are managing.
* When a peer broadcast the signature into the DHT network using PUT\_VALUE command, the protocol will automatically assign the providers for the key.
* When a peer that is providing some key (in DHT network) disconnects, the DHT should update the neighbouring peers to make them providers of the keys that were previously provided by that peer.
* Peers should be able to find the providers of signatures for a given peer by using GET\_PROVIDERS command.
* Peers should be able to query their own buckets to see which keys they are providing. They should compute IVPs and provide them to the peers requesting it.

More on how DHT works: [https://github.com/libp2p/specs/blob/master/kad-dht/README.md](https://github.com/libp2p/specs/blob/master/kad-dht/README.md)

### Requirements for insertion

When values are to about to be added into the bucket:

* Signatures that cannot be verified should not be added into the bucket.
* If a message hash contains a public key of its own peer, it should not be added into the bucket.
* If a message hash scores don't add up to `scale` number (see [IVPs](../zk-circuits/iteration-validity-proofs-ivps.md)), it should not be added into the bucket.
* If a message hash has duplicate public keys listed as neighbours, it should not be added into the bucket.

### Signature Layout

We are using Eddsa as out signature scheme. The reason for this choice is simply the SNARK-friendliness of this scheme.

For keys we are using BabyJubJub EC point, so that would be 64 bytes without compression:

```rust
struct Key {
    x: [u8; 32],
    y: [u8; 32],
}
```

The structure of the value:

```rust
struct Value {
    sig_r_x: [u8; 32],
    sig_r_y: [u8; 32],
    sig_s: [u8; 32],
    neighbours: [[u8; 32]; 256],
    scores: [u8; 256],
}
```

To calculate the message hash, we do the following (pseudo Rust code):

```rust
// First, we want to construct pairs from neighbours and scores
let pairs = [(neighbours[0], scores[0]) ... (neighbours[n], scores[n])];
// Then we construct a hight-9 merkle root from these pairs
// which is the message hash
let message_hash = merkle_tree_from(pairs);
```

Each pair contains the neighbour public key and the score toward that neighbour. Hash of pair will be a leaf value of the Merkle tree. We will use m-ary where the `m` is chosen based on performance requirements of the ZK circuit.

Number of bytes would be: 32 + 32 + 32 + (32 \* 256) + 256 = 8544, which is more than 8 MB. We should try different compression techniques to reduce this number.
