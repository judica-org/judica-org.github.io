+++
title="From the Sapio Zoo: HODL Chicken + Tux Preview"
date=2020-09-06
front_pic='/img/chickens.png'
summary="An on chain game of chance."
author="Pyskell and Jeremy Rubin"
+++

Hello! I'm really excited to have [@pyskell](https://twitter.com/pyskell) making the first guest post
on the Judica blog! The post below is a description of Pyskell's efforts to make a new game for
Bitcoin, a sort of "proof of strong hands" called HodlChicken. The blog will describe how the
contract is written and show how the Sapio + Tux ecosystem works together to produce compelling
Bitcoin Applications.

Let's hand it off to Pyskell to tell you more about what he built.


-----

A few weeks ago I got access to the Sapio language developer preview, and I had a little fun
designing a smart contract enabling people to play something like Chicken.

If you're not familiar, Chicken is a dangerous game where two drivers drive their cars directly at
each other. Three things can happen: either neither driver gets out of the way, and they crash; one
driver yields the right of way (the chicken), or both players yield.

We can make a version of this for Bitcoin using Sapio. We want the following outcomes:

1. Alice waits, Bob waits: coins remain locked
1. Alice yields, Bob waits: Bob gets a prize that Alice pays for
1. Alice waits, Bob yields: Alice gets a prize that Bob pays for
1. Alice yields, Bob yields: Not Allowed, we're playing the extreme version of Chicken here


To do this, I created the below contract: 

```forth
OP_IF <bob_chicken_tx_h> OP_CHECKTEMPLATEVERIFY OP_DROP <bob_key> OP_ELSE <alice_chicken_tx_h> OP_CHECKTEMPLATEVERIFY OP_DROP <alice_key> OP_ENDIF, OP_CHECKSIG
```

Can you read this? Maybe! But can you properly derive what the transaction hashes should be? Of course not. I can't either. That's why I wrote it in Sapio.

```python
# code is Copyright (c) 2020 Pyskell & Judica, Inc.
# Available under BSD-new license
@contract
class HodlChicken:
    alice_contract: Callable[[Amount], Contract]
    bob_contract: Callable[[Amount], Contract]
    alice_key: PubKey
    bob_key: PubKey
    alice_deposit: Amount
    bob_deposit: Amount
    winner_gets: Amount
    chicken_gets: Amount
```

This first section lays out parameters necessary to construct our contract.

The winner_gets and chicken_gets determine the amounts of the prize.

The alice_deposit and bob_deposit parameters aren't required to make the contract, and could be
forged, but are used to ensure some invariants in the contract later on.

The alice_key and bob_key are used to determine which party capitulates by signing a transaction.

Lastly, and most importantly, the functions alice_contract and bob_contract return a contract
instance depending on how much value is paid. This allows a recipient to have a generic receiving
function that creates different types of keys (e.g., composable smart contracts or plain addresses)
depending on how much value is sent. So for example Alice could receive her winnings out to a bech32 Lightning channel
while Bob, who likes paying unnecessary fees, can choose a P2SH address.


The next section adds the chicken logic for Alice:
```python
@HodlChicken.let
def alice_is_a_chicken(self) -> Clause:
    return SignedBy(self.alice_key)


@alice_is_a_chicken
@HodlChicken.then
def alice_redeem(self) -> TransactionTemplate:
    tx = TransactionTemplate()
    tx.add_output(self.winner_gets, self.bob_contract(self.winner_gets))
    tx.add_output(self.chicken_gets, self.alice_contract(self.chicken_gets))
    return tx

```
The let binding allows us to declare a reusable restriction that we can apply to
as a decorator to our `alice_redeem` function.

The 'then' binding ensures that the `TransactionTemplate` returned can be used if the
alice_is_a_chicken condition is met.

The `TransactionTemplate` we return pays the winner amount to Bob (because Alice capitulated), and the
chicken amount to Alice.


Next, we add the same logic for Bob.
```python

@HodlChicken.let
def bob_is_a_chicken(self):
    return SignedBy(self.bob_key)


@bob_is_a_chicken
@HodlChicken.then
def bob_redeem(self) -> TransactionTemplate:
    tx = TransactionTemplate()
    tx.add_output(self.winner_gets, self.alice_contract(self.winner_gets))
    tx.add_output(self.chicken_gets, self.bob_contract(self.chicken_gets))
    return tx
```

Lastly, we add some requirements. Requirements are essentially assertions about the correctness of
the parameters passed to the HodlChicken contract.

```python

@HodlChicken.require
def amounts_sum_correctly(self) -> bool:
    # Make sure all sats will be spent when the game completes
    return (
        self.alice_deposit + self.bob_deposit
    ) == self.winner_gets + self.chicken_gets


@HodlChicken.require
def equal_amounts(self) -> bool:
    # Both participants should commit the same amount
    return self.alice_deposit == self.bob_deposit

```


And there you have it, a fun little Bitcoin game in a short bit of Sapio.
Any programmer can read this code, understand what it's doing, and expand on it.
For example I bet you could easily add a `draw` function where both participants
can agree to exit the game without losing any of their deposit. You'd at least have
a much easier time doing so than if I gave you a plain Bitcoin Script.

I'll leave it up to Jeremy to explain how the Tux browser works to inspect smart contracts.

-----------

Thanks Pyskell, awesome idea!

Now, how do we set up and use the HodlChicken contract?

Well, we can use it in a lot of ways! For this part let's set it up with a simple PayToSegwitAddress
receiver and normal PubKeys.

```python
alice_script = script_to_p2wsh(CScript([b"Alice's Key Goes Here!"]))
bob_script = script_to_p2wsh(CScript([b"Bob's Key Goes Here!"]))
alice_key = random_k()
bob_key = random_k()

hodl_chicken = HodlChicken.create(
    alice_contract=lambda x: PayToSegwitAddress.create(
        amount=AmountRange.of(x), address=alice_script
    ),
    bob_contract=lambda x: PayToSegwitAddress.create(
        amount=AmountRange.of(x), address=bob_script
    ),
    alice_key=alice_key,
    bob_key=bob_key,
    alice_deposit=Sats(100_000_000),
    bob_deposit=Sats(100_000_000),
    winner_gets=Sats(150_000_000),
    chicken_gets=Sats(50_000_000),
)
```

Now we can derive an address for the `hodl_chicken` contract instance, and pay to it.
Suppose we've done that and created a COutPoint X. All we need to do is:

```
hodl_chicken.bind(X)
```

This will automatically generate a list of transactions and ABI data.


We can also add a single line of code to the server which allows HodlChicken be imported into the
[Tux](https://github.com/sapio-lang/tux) tool automatically by websocket. Though in the future
this will be configurable without editing Tux.

```python
CompilerWebSocket.add_contract("Chicken", HodlChicken)
```

Now, without writing any new code, we have the ability to create a new instance of the contract via webform:


{{< figure src="/img/chickens/tux3.png" width="100%" caption="Webform showing HodlChicken Contract Creation.">}}

Let's check out our newly created contract:

{{< figure src="/img/chickens/tux1.png" width="100%" caption="Freshly created HodlChicken Contract.">}}

We can use Tux to check out and inspect the transactions and outputs in our contract.
{{< figure src="/img/chickens/tux2.png" width="100%" caption="Inspecting HodlChicken">}}

When we're satisfied, we can use the Tux create button to connect to our node and fund a raw
transaction to create our contract, and link the children to the parent.
{{< figure src="/img/chickens/tux4.png" width="100%" caption="Paying To a HodlChicken Contract.">}}



The Load/Save windows can be used to dump a JSON object that can be passed around for future Tux
sessions or for compatibility with other tools.

```json
{
  "program": [
    {
      "hex": "0200000001174c0cee8f1c8a1dd743d663defb4c9670c7c21af17f1e2c5f93e7171ee0ac0a0000000000000000000280d1f00800000000220020108abc959debbf561218facf9ce66cecb11c526eafc4e4031e4fc68de4c8955780f0fa0 2000000002200205688d4a74bf4b739e69c34a4fde6733831d739714acc42aac756e9a371e72f6f00000000",
      "label": "150cc0b95f23d17c",
      "color": "green",
      "utxo_metadata": [
        {
          "color": "grey",
          "label": "Segwit Address"
        },
        {
          "color": "grey",
          "label": "Segwit Address"
        }
      ]
    },
    {
      "hex": "0200000001174c0cee8f1c8a1dd743d663defb4c9670c7c21af17f1e2c5f93e7171ee0ac0a0000000000000000000280d1f008000000002200205688d4a74bf4b739e69c34a4fde6733831d739714acc42aac756e9a371e72f6f80f0fa0 200000000220020108abc959debbf561218facf9ce66cecb11c526eafc4e4031e4fc68de4c8955700000000",
      "label": "cee1b11a58baf2bb",
      "color": "green",
      "utxo_metadata": [
        {
          "color": "grey",
          "label": "Segwit Address"
        },
        {
          "color": "grey",
          "label": "Segwit Address"
        }
      ]
    },
    {
      "hex": "01000000018e87e856904dac9bb5cf3f70a1d500cef506e544519a68b2fd0d46022e9de6a70000000000feffffff0200c2eb0b000000002200206320cc4055e9b2de0c5459e5eb25c2c010a2633fc32ec9aa97254e8931249cf40c241a1 e01000000160014e1d1aaf05e4531a70becab819f9ce5729c265ea900000000",
      "label": "0aace01e17e7935f",
      "color": "white",
      "utxo_metadata": [
        {
          "color": "white",
          "label": "Missing"
        },
        {
          "color": "white",
          "label": "72ae200f3fc43178"
        }
      ]
    }
  ]
}
```


There's still a lot of work to do to make everything seamless and perfect, but this shows the
direction that things are headed! If you're excited and want to work with Judica on Sapio, Tux,
and whatever else we dream up, [drop us a line!](/join/)
