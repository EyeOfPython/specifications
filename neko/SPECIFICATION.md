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

# Layer 1 Header

+-+-+
| 32 bytes | Previous block hash |
| 32 bytes | Hash of Layer 2 Metadata |
+-+-+

# Layer 2 Header

+-+-+
| 8 bytes | Timestamp |
| 4 bytes | nBits |
| 32 bytes | Hash of Layer 3 Metadata |
+-+-+

# Layer 3 Header

+-+-+
| 32 bytes | Merkel root |
| 8 bytes | Block size |
| 32 bytes | Epoch block hash, 5040 block epochs |
| 32 bytes | Extended Metadata hash |
+-+-+

# Layer 4

+-+-+
| 8 byte | nonce | 
| var_int | number of fields |
| metadata_field[] | |
+-+-+

Metadata field format:

+-+-+
| 4 bytes | field ID |
| var_int | length |
| uchar[] | data |
+-+-+

The nonce is 