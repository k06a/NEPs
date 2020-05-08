# Light Client

The state of the light client is defined by:

1. `BlockHeaderInnerLiteView` for the current head (which contains `height`, `epoch_id`, `next_epoch_id`, `prev_state_root`, `outcome_root`, `timestamp` and the hash of the block producers set for the next epoch `next_bp_hash`);
2. The set of block producers for the current and next epochs.

Light clients operate by periodically fetching instances of `LightClientBlockView` via particular RPC end-point desdribed [below](#rpc-end-point).

```rust
pub enum ApprovalInner {
    Endorsement(CryptoHash),
    Skip(BlockHeight)
}

pub struct ValidatorStakeView {
    pub account_id: AccountId,
    pub public_key: PublicKey,
    pub stake: Balance,
}

pub struct BlockHeaderInnerLiteView {
    pub height: BlockHeight,
    pub epoch_id: CryptoHash,
    pub next_epoch_id: CryptoHash,
    pub prev_state_root: CryptoHash,
    pub outcome_root: CryptoHash,
    pub timestamp: u64,
    pub next_bp_hash: CryptoHash,
}

pub struct LightClientBlockView {
    pub prev_block_hash: CryptoHash,
    pub next_block_inner_hash: CryptoHash,
    pub inner_lite: BlockHeaderInnerLiteView,
    pub inner_rest_hash: CryptoHash,
    pub next_bps: Option<Vec<ValidatorStakeView>>,
    pub approvals_after_next: Vec<Option<Signature>>,
}
```

Recall that the hash of the block is

```rust
sha256(concat(
    prev_hash,
    sha256(concat(
        sha256(borsh(inner_lite)),
        sha256(borsh(inner_rest))
    ))
))
```

The fields `prev_block_hash`, `next_block_inner_hash` and `inner_rest_hash` are used to reconstruct the hashes of the current and next block, and the approvals that will be signed, in the following way (where `block_view` is an instance of `LightClientBlockView`):

```rust
current_block_hash = sha256(concat(
    block_view.prev_block_hash,
    sha256(concat(
        sha256(borsh(block_view.inner_lite)),
        block_view.inner_rest_hash,
    ))
))

next_block_hash = sha256(concat(
    current_block_hash,
    block_view.next_block_inner_hash
))

approval_message = concat(
    borsh(ApprovalInner::Endorsement(next_block_hash)),
    big_endian(block_view.inner_lite.height + 2)
)
```

The light client updates its head with the information from `LightClientBlockView` iff:

1. The height of the block is higher than the height of the current head;
2. The epoch of the block is equal to the `epoch_id` or `next_epoch_id` known for the current head;
3. If the epoch of the block is equal to the `next_epoch_id` of the head, then `next_bps` is not `None`;
4. `approvals_after_next` contain valid signatures on `approval_message` from the block producers of the corresponding epoch (see next section);
5. The signatures present in `approvals_after_next` correspond to more than 2/3 of the total stake (see next section).
6. If `next_bps` is not none, `sha256(borsh(next_bps))` corresponds to the `next_bp_hash` in `inner_lite`.

## Signature verification

To simplify the protocol we require that the next block and the block after next are both in the same epoch as the block that `LightClientBlockView` corresponds to. It is guaranteed that each epoch has at least one final block for which the next two blocks that build on top of it are in the same epoch.

By construction by the time the `LightClientBlockView` is being validated, the block producers set for its epoch is known. Specifically, when the first light client block view of the previous epoch was processed, due to (3) above the `next_bps` was not `None`, and due to (6) above it was corresponding to the `next_bp_hash` in the block header.

The sum of all the stakes of `next_bps` in the previous epoch is `total_stake` referred to in (5) above.

The signatures in the `LightClientBlockView::approvals_after_next` are signatures on `approval_message`. The  `i`-th signature in `approvals_after_next`, if present, must validate against the `i`-th public key in `next_bps` from the previous epoch. `approvals_after_next` can contain fewer elements than `next_bps` in the previous epoch.

## RPC end-point

There's a single end-point that full nodes exposed that light clients can use to fetch new `LightClientBlockView`s:

```
http post http://127.0.0.1:3030/ jsonrpc=2.0 method=next_light_client_block params:="[<last known hash>]" id="dontcare"
```

The RPC returns the `LightClientBlock` for the block as far into the future from the last known hash as possible for the light client to still accept it. Specifically, it either returns the last final block of the next epoch, or the last final known block. If there's no newer final block than the one the light client knows about, the RPC returns an empty result.

A standalone light client would bootstrap by requesting next blocks until it receives an empty result, and then periodically request the next light client block.

A smart contract-based light client that enables a bridge to NEAR on a different blockchain naturally cannot request blocks itself. Instead external oracles query the next light client block from one of the full nodes, and submit it to the light client smart contract. The smart contract-based light client performs the same checks described above, so the oracle doesn't need to be trusted.