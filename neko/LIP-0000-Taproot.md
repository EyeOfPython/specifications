# LIP 0000: Taproot

Backporting a modified version of Taproot ([BIP341](https://github.com/bitcoin/bips/blob/master/bip-0341.mediawiki)) would give us a bunch of advantages:

* MAST, which allows representing scripts with a lot of IFs in a much more efficient and private way.
    * This also on *some* level increases the opcode limit by a factor of 2^128, which is the number of possible leaves. With proper CashScript support, this would make writing smart contracts a lot easier, as developers don't have to check whether their script broke the limit as much.
* Spending P2SH-esque outputs without revealing the script, adding privacy for complex scripts.
* Advanced txs insight through a new sighash algorithm, allowing introspecting the amounts and `scriptPubKey`s of other inputs.
    * Also allows much more efficient introspection as parts of the serialization are simply left out if they're not part of the sig hash type.
* Wallet developers could be encouraged to always use Taproot but with a plain public key, making payments a little bit more efficient and all unrevealed smart contract payments much more private.
* We don't change the sighash algorithm, so it's easy for multicoin hardware wallets that support taproot to also support Logos.
* If we add `OP_ECADD` and `OP_ECMUL`, smart contract developers can still write self-modifying scripts.
* We will add the ability to store 'state' in a taproot output, which will make recursive smart contracts a lot more efficient and simple.

## Proposal

We will implement Taproot in a way different from BTC. Doing so both makes the implementation much simpler, compatible with our hardfork philosophy and more flexible.

We can split the implementation into the following parts, in order:
1. **Sighash Tap**: Add a new sighash algorithm (available in all Scripts), indicated by adding 0x20 to the sighash flags.
2. **OP_TAPROOT**: Rename `OP_VER` to `OP_TAPROOT`, but keep it disabled.
3. **Add Taproot logic**: These can be either `OP_TAPROOT <pubkey>` or `OP_TAPROOT <pubkey> <32-byte state>`, and implement the taproot logic similar to BIP341 (see below).
4. **Pay-To-Taproot addresses**: These are yet undefined. They should encode both the pubkey and the state somehow. One idea is to just encode the scriptPubKey.

### Sighash Tap
The bits 0x01, 0x02, 0x40 and 0x80 are already taken, which leaves 0x04, 0x08, 0x10 and 0x20. `0x20` will indicate the sighash algorithm introduced by BIP341. For simplicity, we enable it in all scripts, both P2SH and P2Taproot, and signers can choose either. As we introduce a new sighash bit, all sigs will be invalid on all other Bitcoin chains (Bitcoin, Bitcoin Cash, Bitcoin SV).

For non-taproot scripts, the sighash will encode the SHA256 of the redeemScript if it's P2SH in place of `m_tapleaf_hash`, and is left out otherwise, like in a taproot sighash.

### OP_TAPROOT
Since we disabled all non-standard scripts, we can add any semantics we want without breaking existing standard scripts.

Bitcoin Core has to use SegWit, which adds special meaning to this scriptPubKey: `1 <32-byte program>`.

For us, the simplest way is by just adding a new opcode to indicate that the scriptPubKey is a taproot program.

Repurposing `OP_VER` makes the most sense, since it's in the "control" section of the script. However, the opcode never gets executed, instead, it is just used for pattern matching on the scriptPubKey. This is dissimilar from P2SH, which actually gets executed. However, even the very first semantics of taproot are currently not expressible in script (requiring a EC multiplication and addition), therefore we can't do something similar as for P2SH.

Since the opcode is just for pattern matching, it doesn't have to obey opcode semantics and should just be easy for matching.

We chose the following two possible patterns:
1. `OP_TAPROOT <33-byte program>`
2. `OP_TAPROOT <33-byte program> <32-byte state>`

These are easy to match on, and a quick test for taproot can be done by just looking at the first byte. Also, it's easy to work with in Script, since you only need to do one `OP_SPLIT` and `OP_CAT` to swap out the state, which is optimal.

In the future, we *might* relax the contraints on the state to allow other sizes, but since this will be directly part of the UTXO set, we want it to be very simple to deal with.

### Add Taproot logic
#### Schnorr signatures
Unlike Bitcoin (BTC), Bitcoin Cash (BCH) and derivatives (ABC) already have Schnorr signatures activated. There, if a signature (without sighashflags) is of length 64, it's considered a Schnorr signature.

On Bitcoin Core, Schnorr signatures are only enabled in Taproot scripts, and there *only* Schnorr signatures are allowed, but no ECDSA signatures.

The simplest and most flexible implementation, which is also compatible with both Bitcoin and Bitcoin Cash, is to keep the semantics of Bitcoin Cash both in P2SH and Tapscript. I.e., if the sig length is 64, it's interpreted as Schnorr sig, otherwise as ECDSA sig.

#### X-only pubkeys
Bitcoin introduces X-only pubkeys for Schnorr signatures, which are always implicitly `0x02` prefixed pubkeys.

This adds a lot of complexity on the top of the existing Schnorr sig logic, and we should not implement any of that.

This also means that our Taproot programs in the `scriptPubKey` will be 33 bytes, not 32 bytes as in Bitcoin Core, but we're treating those differently anyway.

#### Annex
BIP 341 adds an optional (and currently meaningless) 'annex' field, which allows future features to be soft-forked in. We don't need to do that; instead, we just disallow items that match the annex. This way, we can hardfork additional logic that Core comes up with without breaking existing scripts.

#### State
To allow advanced smart contract functionality, we optionally allow storing state in the `scriptPubKey`. The state always has 32 bytes in size.

Before evaluating the tap script, the state gets pushed onto the stack generated by the scriptSig, and can then be used normally.

#### Taproot
We otherwise implement the same taproot logic as proposed in BIP341, only that pubkeys are 33 bytes, even the one in the control bytes.

Merkle branch order is encoded by placing items lexicographically such that they hash to the root hash.

#### Tapscript
Otherwise, tapscripts behave exactly how P2SH script currently behave, i.e the semantics are exactly the same. This allows Script developers to migrate scripts between the two without having to worry about different meanings.

`OP_CHECKSIGADD` will not be added.

## Implementation

### Sighash Tap

Things that need to change (each could be a single commit + tests):
1. Add `TaggedHash` function: hash_tag(x) = SHA256(SHA256(tag) || SHA256(tag) || x).
2. Add `SHA256Uint256` (doing a single SHA256 round).
3. Add `CHashWriter::GetSHA256`.
4. Refactor `PrecomputedTransactionData` so it has access to the spent outputs.
5. Use single hash SHA256 for current sighash calculation:
    1. Change `GetPrevoutHash` to `GetPrevoutsSHA256`.
    2. Change `GetSequenceHash` to `GetSequencesSHA256`.
    3. Change `GetOutputsHash` to `GetOutputsHash`.
    4. Add `GetSpentAmountsSHA256`.
    5. Add `GetSpentScriptsSHA256`.
    6. Refactor `SignatureHash` to use `SHA256Uint256(GetXXXsSHA256)` instead of `GetXXX`. 
6. Refactor `PrecomputedTransactionData` so it computes + stores `prevouts_single_hash`, `sequences_single_hash`, `outputs_single_hash`, `spent_amounts_single_hash`, `spent_scripts_single_hash`.
7. Split contents of `SignatureHash` into `SignatureHashLegacy` (with the old `FindAndDelete` logic and all) and `SignatureHashBIP143` (the new sighash used by SegWit and Bitcoin Cash), make `SignatureHash` call it depending on `0x40` sighashflag.
8. Add `tapleaf_hash` and `codeseparator_pos` (`std::optional`) to `ScriptExecutionMetrics`.
9. Add `SignatureHashBIP341`:
    1. Thread `ScriptExecutionMetrics` into `SignatureHash`.
    2. Implement `SignatureHashBIP341`, if `ScriptExecutionMetrics::tapleaf_hash` is not `std::optional::none`, add `tapleaf_hash` and `codeseparator_pos` to the sighash as in BIP341.
10. Make `SignatureHash` call `SignatureHashBIP341` if `0x20` sighashflags are set.
11. Make `0x20` sighashflags legal.

### OP_TAPROOT
1. Rename `OP_VER` to `OP_TAPROOT`.
2. Add `ILLEGAL_OP_TAPROOT` script error.
3. Instead of throwing `DISABLED_OPCODE` when encountering `OP_TAPROOT` (whether executed or not executed), throw `ILLEGAL_OP_TAPROOT`.

### Add Taproot logic
1. Add `CPubKey::CheckPayToContract`, which verifies `Q == P + kG`.
2. Add `VerifyTaprootCommitment`, verifying the MAST path and that `P + hash(P||m)G` holds, and returns `tapleaf_hash`.
3. Add `CScript::IsPayToTaproot`, returning if the script is of the form `0 <pubkey> OP_TAPROOT` or `<32-byte state> <pubkey> OP_TAPROOT`.
4. Modify `VerifyScript`:
    1. After running `scriptSig`, branch if `scriptPubKey.IsPayToTaproot()`.
    2. Return `TAPROOT_DISABLED_ANNEX` if there's at least 2 items on the stack and bottom one starts with `0x50`.
    3. Extract 32-byte state from `scriptPubKey`, if any
    4. Extract 33-byte program (a public key) from `scriptPubKey`
    5. Branch on stack size:
        1. If just one element on the stack, it's a signature. Verify it with the 33-byte program as public key, using the `SignatureHashBIP341` sighash as message.
        2. Otherwise, the elements on the stack are `inputs... script_bytes control`:
            1. Verify the length of `control` is 34 + 32m.
            2. Call `VerifyTaprootCommitment`
            3. Verify `(control[0] & TAPROOT_LEAF_MASK) == TAPROOT_LEAF_TAPSCRIPT`, otherwise error.
            4. If the 32-byte state is not empty, push it onto the top of the stack.
            5. Run `script_bytes` script with the `inputs`.
    6. If the above succeeds, return success early; don't execute the `scriptPubKey`.
