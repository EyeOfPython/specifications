# Neko Initial Spec

# General Changes

Neko will be based on Bitcoin ABC, with the following changes initially.

* Removal of all hardfork activation code
* Reset difficulty to 1
* New Genesis Block
* New Netmagic
* Blocktime to 2 minutes
* The issuance of Coinbase rewards will be changed to be a fixed 1 gigasat (10^9 sats) in perpetuity. 262800 gigasats will be issued per year.
  * Half will go to miners
  * Half will go to servicing the blockchain and users
    * 25% of is will go to node development
    * 25% of this will go to wallet development
    * 25% of this will go to education
    * 25% of this will go to user wellbeing
* Half of fees will be burned, the other half will go to miners as a fee for including transactions.

# Block Header Changes

The hash algorithm will be multiple iterations of SHA256 that are layered on top of each other via hash commitments. This allows low resource devices to prune various unimportant data.

Usage: Entropy, Blockchaining

The header data structure looks as follows:

| Size | Name | Meaning |
|------|------|---------|
| 32 bytes | hashPrevBlock | Previous block hash |
| 4 bytes | nBits | Compact encoding of the target for this block |
| 6 bytes | nTime | Timestamp of the block (UNIX time), good for 8 million years |
| 2 bytes | nReserved | Reserved for future use, always 0x0000 |
| 8 bytes | nNonce | Nonce for miners to tweak the hash |
| 1 bytes | nMetaVersion | Version for the data that follows; it should not yet be considered set in stone |
| 7 bytes | nSize | Block size in bytes |
| 4 bytes | nHeight | Block height, where the genesis block is 0 |
| 32 bytes | hashEpochBlock | Hash of the last block which had a height divisible by 5040 |
| 32 bytes | hashMerkleRoot | Merkle root of the transactions of the block |
| 32 bytes | hashExtendedMetadata | Hash of the extended metadata |

# Layer 1 Header: Chain Layer

This layer allows to very cheaply verify that the blocks form a chain and that PoW has been performed.

Can also be used as a very cheap entropy source if previous block hash is known.

| Size | Name | Meaning |
|------|------|---------|
| 32 bytes | hashPrevBlock | Previous block hash |
| 32 bytes | hashPowLayer | SHA256 of Layer 2 Metadata |

# Layer 2 Header: PoW Layer

This layer allows to verify the PoW and DAA (ASERT only requires time and height).

SPV wallets can store only this layer for blocks that don't contain txs.

A 6 byte timestamp is good for 8 million years; until then we will have reversed SHA256 anyway and have to do a PoW change.

| Size | Name | Meaning |
|------|------|---------|
| 4 bytes | nBits | Compact encoding of the target for this block |
| 6 bytes | nTime | Timestamp of the block (UNIX time) |
| 2 bytes | nReserved | Reserved for future use, always 0x0000 |
| 8 bytes | nNonce | Nonce for miners to tweak the hash |
| 32 bytes | hashTxLayer | SHA256 of Layer 3 Metadata |

# Layer 3 Header: Tx layer

This contains the txs merkle root, so we can verify the transactions of the block.

SPV wallets store this layer for blocks that do contain txs they're interested in.

Epochs are 7 days and allow skipping many blocks while still knowing they're connected. This is useful for very low spec devices and smart contracts.

| Size | Name | Meaning |
|------|------|---------|
| 1 bytes | nMetaVersion | Version for the data that follows; it should not yet be considered set in stone |
| 7 bytes | nSize | Block size in bytes |
| 4 bytes | nHeight | Block height, where the genesis block is 0 |
| 32 bytes | hashEpochBlock | Epoch block hash, 5040 block epochs (7 days) |
| 32 bytes | hashMerkleRoot | Txs Merkle root |
| 32 bytes | hashExtendedMetadata | Extended Metadata hash |

# Layer 4: Extended Metadata

This contains additional data which is only relevant for consensus.

For better upgradability, it has a key-value layout. The maximum size of the metadata, for now, is 100 bytes.

Optional fields are:
* `1`: extra nonce (1-32 bytes), to tweak the hash some more.
* `2`: memo (1-32 bytes), to leave some arbitrary message

| Type | Meaning |
|------|---------|
| var_int | number of fields |
| metadata_field[] | fields |

Metadata field format:

| Type | Meaning |
|------|---------|
| 4 bytes | field ID |
| var_int | length |
| uchar[] | data |
