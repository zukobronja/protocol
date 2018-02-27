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
- Amount that initiator locks into the channel
- Amount that participant locks into the channel
- A lock period determining for how long funds will be locked, when one of the parties initiates channel solo close
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

Channel creation transaction must be signed by both `initiator` and `participant`.

The transaction creates an channel object in the channels state tree.
Channel object is references by channel ID, which is the hash of `{initiator, nonce, participant}`.

**Questions:**
- Is `lock_period` countdown initiated only by channel solo close transaction?
Is there any way created channel will expire automatically?

### State channel updates

No channel update transaction will be accepted if a channel is already closed.

Both `channel_deposit` and `channel_withdraw` transactions must be signed by two peers.

#### `channel_deposit`

The transaction contains:
- ID of the channel
- The address of the peer that is locking funds in the channel
- The address of the peer whose channel receives the funds
- Amount to be deposited into the channel (amount will be taken from the address that submits the transaction)
- Transaction fee (`from_account` pays that)
- Nonce (refers to `from_account`)

```
{channel_id   :: binary(),
 from_account :: publick_key(),
 to_account   :: public_key(),
 amount       :: amount(),
 fee          :: amount(),
 nonce        :: non_neg_integer()
}
```

#### `channel_withdraw`

The transaction contains:
- ID of the channel
- The address of the peer from whose channel funds are withdrawn from
- The address of the account that receives the funds (must be one of the channel peers)
- Amount to be withdrawn from the channel (amount will be added to the balance to the `to_account` balance)
- Transaction fee
- Nonce

```
{channel_id   :: binary(),
 from_account :: public_key(),
 to_account   :: public_key(),
 amount       :: amount(),
 fee          :: amount(),
 nonce        :: non_neg_integer()
}
```

### State channel closing

No channel closing transaction will be accepted if a channel is already closed.

#### `channel_close_mutual`

The transaction contains:
- ID of the channel
- Amount
- The address of the party that submitted the transaction (needed for nonce)
- Transaction fee
- Nonce

Final balances are calculated based on the following formula:
```
initiator_final = initiator_start - amount
receiver_final  = receiver_start  + amount
```

```
{channel_id  :: binary(),
 amount      :: amount(),
 initiator   :: public_key(),
 participant :: public_key(),
 fee         :: amount(),
 nonce       :: non_neg_integer()
}
```

Channel close mutual transaction must be signed by two peers of the channel.

**Questions:**
- Is the `amount` formula correct?
Is initial peers balance in the formula impacted by `channel_deposit` and `channel_withdraw` transactions?

#### `channel_close_solo`

#### `channel_slash`

#### `channel_settle`
