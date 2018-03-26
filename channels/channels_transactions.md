[back](../CHANNELS.md)
## State channels on-chain transactions

State channels on-chain transactions are of the following types:
- Establishment
    - [`channel_create`](#channel_create)
- Updates
    - [`channel_deposit`](#channel_deposit)
    - [`channel_withdraw`](#channel_withdraw)
- Closing
    - [`channel_close_mutual`](#channel_close_mutual)
    - [`channel_close_solo`](#channel_close_solo)
    - [`channel_slash`](#channel_slash)
    - [`channel_settle`](#channel_settle)

### State channel establishment

#### `channel_create`

After off-chain agreement, one of the parties/peers can submit channel creation transaction to the blockchain.

The transaction contains:
- The address of the initiator peer
- The address of the participant peer
- Amount that initiator locks into the channel (must be non-negative value)
- Amount that participant locks into the channel (must be non-negative value)
- A lock period determining for how long dispute will take,
  in case one of the peers starts it with [`channel_close_solo`](#channel_close_solo) transaction
  (then [`channel_slash`](#channel_slash) transactions are acceptable)
- Transaction fee
- Nonce

```
{initiator          :: public_key(),
 participant        :: public_key(),
 initiator_amount   :: amount(),
 participant_amount :: amount(),
 lock_period        :: non_neg_integer(),
 fee                :: amount(),
 nonce              :: non_neg_integer()
}
```

Once channel is opened using channel creation transaction it will not be closed automatically without [`channel_close_mutual`](#channel_close_mutual) or [`channel_close_solo`](#channel_close_solo).

Funds are "locked" in the channel, so they are subtracted from `initiator` and `participant` balances at the moment of channel creation.

Transaction fee is taken from `initiator`'s balance, and is not included in `initiator_amount`, so `initiator`'s account balance must be at least `initiator_amount + fee`.

Nonce refers to `initiator`'s account nonce.

Channel create transaction creates channel object in the channels state tree.
Channel object is references by channel ID, which is the hash of `{initiator, nonce, participant}`.
This may be useful to be able to "recall" lost channel ID (may be changed random ID)?

Channel creation transaction must be signed by both `initiator` and `participant` accounts.

### State channel updates

Channel update transactions are accepted only for active channels, i.e. before channel is closed using [`channel_close_mutual`](#channel_close_mutual) transaction or
before a dispute is started using [`channel_close_solo`](#channel_close_solo) transaction.

Both `channel_deposit` and `channel_withdraw` transactions must be signed by two peers.

#### `channel_deposit`

The transaction contains:
- ID of the channel
- The address of the peer that is locking funds in the channel
- The address of the peer whose channel receives the funds
- Amount to be deposited into the channel (amount will be taken from the address that submits the transaction)
- The address of the initiator peer
- The address of the participant peer
- Transaction fee
- Nonce

```
{channel_id   :: binary(),
 from_account :: public_key(),
 to_account   :: public_key(),
 amount       :: amount(),
 initiator    :: public_key(),
 participant  :: public_key(),
 fee          :: amount(),
 nonce        :: non_neg_integer()
}
```

This transaction is used to lock more funds in the channel.

Both `from_account` and `to_account` must be peers of a channel (may be the same).

Transaction fee is taken from `from_account`'s balance, so `from_account`'s balance must be at least `amount + fee`.

Nonce refers to `from_account`'s account nonce.

Theoretically `from_account` and `to_account` may be the same peer, i.e. peer may want to lock more funds in the channel, as one did not lock sufficient amount during channel opening.
In the initial implementation `initiator` and `participant` fields are needed to be able to implement `aetx:signers/1` callback, without looking into channels state tree.
This may be changed by removal of preliminary check for [`channel_deposit`](#channel_deposit) transactions' signatures, before they lands in the mempool.

For `aetx:accounts/1` callback (which is used for APIs to list all transactions impacting user's balance) only `from_account` pubkey is returned, as this is the one whose balance is immediately impacted.

***Question:*** Should `to_account` be returned in `aetx:accounts/1` as well, as it's channel balance is impacted?

#### `channel_withdraw`

The transaction contains:
- ID of the channel
- The address of the peer from whose channel funds are withdrawn from
- The address of the account that receives the funds (must be one of the channel peers)
- Amount to be withdrawn from the channel (amount will be added to the balance to the `to_account` balance)
- The address of the initiator peer
- The address of the participant peer
- Transaction fee
- Nonce

```
{channel_id   :: binary(),
 from_account :: public_key(),
 to_account   :: public_key(),
 amount       :: amount(),
 initiator    :: public_key(),
 participant  :: public_key(),
 fee          :: amount(),
 nonce        :: non_neg_integer()
}
```

Both `from_account` and `to_account` must be peers of a channel (may be the same).

Transaction fee is taken from `from_account`'s balance.

Nonce refers to `from_account`'s account nonce.

Theoretically `from_account` and `to_account` may be the same peer.
In the initial implementation `initiator` and `participant` fields are needed to be able to implement `aetx:signers/1` callback, without looking into channels state tree.
This may be changed by removal of preliminary check for [`channel_withdraw`](#channel_withdraw) transactions' signatures, before they lands in the mempool.

For `aetx:accounts/1` callback (which is used for APIs to list all transactions impacting user's balance) only `to_account` pubkey is returned, as this is the one whose balance is immediately impacted.

***Question:*** Should `from_account` be returned in `aetx:accounts/1` as well, as it's channel balance is impacted?

### State channel closing

#### `channel_close_mutual`

No channel closing transaction will be accepted if a channel is already closed or there is an ongoing dispute.

The transaction contains:
- ID of the channel
- Amount
- The address of the initiator peer
- The address of the participant peer
- Transaction fee
- Nonce

```
{channel_id  :: binary(),
 amount      :: amount(),
 initiator   :: public_key(),
 participant :: public_key(),
 fee         :: amount(),
 nonce       :: non_neg_integer()
}
```

Final balances are calculated based on the following formula:
```
initiator_final = initiator_current_balance - amount
receiver_final  = receiver_current_balance  + amount
```
, where `initiator_current_balance` and `receiver_current_balance` are initial peers balances amended by all submitted deposit/withdraw transactions.

This transaction then `initiator_final` and `receiver_final` channel balances and adds them to `initiator` and `participant` account balances.
In the end channel is closed, i.e. it is removed from channels state tree.

Channel's `initiator` is the one paying transaction fee.

Nonce refers to `initiator`'s account nonce.

Channel close mutual transaction must be signed by both `initiator` and `participant` accounts.

`initiator` and `participant` fields are needed to implement `aetx:signers/1` and `aetx:accounts/1` callbacks without looking into channels state tree.

#### `channel_close_solo`

This transaction is to be submitted when one of peers is not cooperative and the other wants to close the channel.

Channel close solo transaction is accepted only for active channels, i.e. before channel is closed using [`channel_close_mutual`](#channel_close_mutual) transaction or
before any dispute is started using [`channel_close_solo`](#channel_close_solo) transaction.

The transaction contains:
- ID of the channel
- The address of the peer that is submitting transaction
- Payload, which is serialized, mutually signed local state, that is exchanged between peers off-chain
- Transaction fee
- Nonce

```
{channel_id :: binary(),
 account    :: public_key(),
 payload    :: binary(),
 fee        :: amount(),
 nonce      :: non_neg_integer()
}
```

Submitting this opens a dispute, i.e. the other peer has `lock_period` time to submit [`channel_slash`](#channel_slash) transaction with newer state (i.e. higher sequence number).

If no [`channel_slash`](#channel_slash) transactions are submitted, [`channel_settle`](#channel_settle) is needed to redistribute funds locked in the channel (as opposed to [`channel_close_mutual`](#channel_close_mutual) transaction, where funds are distributed automatically).

`account` is the one paying transaction fee.

Nonce refers to `account`'s account nonce.

#### `channel_slash`

This transaction is to be submitted when peer wants to submit local state with higher `sequence_number` than the one submitted in [`channel_close_solo`](#channel_close_solo) or in previous channel slash.

The transaction contains:
- ID of the channel
- The address of the peer that is submitting transaction
- Payload, which is serialized, mutually signed local state, that is exchanged between peers off-chain
- Transaction fee
- Nonce

```
{channel_id :: binary(),
 account    :: public_key(),
 payload    :: binary(),
 fee        :: amount(),
 nonce      :: non_neg_integer()
}
```

`account` is the one paying transaction fee.

Nonce refers to `account`'s account nonce.

#### `channel_settle`

This transaction is used to distribute funds after a dispute is finished (i.e. `lock_period` expires).

The transaction contains:
- ID of the channel
- The address of the peer that is submitting transaction
- The address of the peer that was participating in a channel
- Transaction fee
- Nonce

```
{channel_id :: binary(),
 account    :: public_key(),
 party      :: public_key(),
 fee        :: amount(),
 nonce      :: non_neg_integer()
}
```

[`channel_settle`](#channel_settle) is needed to redistribute funds locked in the channel (as opposed to [`channel_close_mutual`](#channel_close_mutual) transaction, where funds are distributed automatically).

`account` is the one paying transaction fee.

Nonce refers to `account`'s account nonce.

`party` fields is needed to implement `aetx:accounts/1` callback without looking into channels state tree.
