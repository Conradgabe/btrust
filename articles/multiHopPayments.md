# A Technical Deep Dive into Lightning's Multi-Hop Payments from the Terminal

Lightning's approach to scaling Bitcoin was to move payments off-chain and settle back on-chain only periodically. This turns out to be fundamentally a balance-routing problem. While payment channels handle bilateral payments between directly connected peers, multi-hop payments turn isolated channels into a connected, routable network graph. By combining Hash Time-Locked Contracts (HTLCs) with Sphinx onion routing, Lightning enables trustless atomic transfers across multi-node paths while shielding the full payment path from any single intermediate relayer.

This deep dive tries to dissect the technical mechanics of multi-hop payments, from payment setup and route selection to onion construction and the cryptographic enforcement that makes the whole path settle atomically using the terminal.

Before you continue, you should have a basic understanding of the following:

- How a Lightning payment channel works.
- HTLC mechanism: hashlocks, timelocks and how a preimage unlocks a payment.

To follow along with the code sections, you'll need to have installed:

- Polar and Docker installed for regtest trace.

## Payment Setup: The invoice and the Hash Commitment

A BOLT11 invoice is a single Bech32-encoded string. It starts with "ln" followed by a BIP-0173 currency prefix ("lnbc" for mainnet, "lntb" for testnet and "lnbcrt" for regtest), then followed by an optional amount in that currency and an optional multiplier letter (m for milli, u for micro, n for nano, p for pico).

During a payment transaction, the final recipient (let's use Dave in this instance) generates a random 32-byte value called the preimage (R), then he computes the hash of the preimage, H = hash(R), and embeds H, along with the payment amount, into a BOLT11 invoice. Dave now sends this invoice to the sender (let's use Alice).

Here is the anatomy of a BOLT11 invoice:

```
lnbc1pvjluezsp5zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zyg3zygspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpl2pkx2ctnv5sxxmmwwd5kgetjypeh2ursdae8g6twvus8g6rfwvs8qun0dfjkxaq9qrsgq357wnc5r2ueh7ck6q93dj32dlqnls087fxdwk8qakdyafkq3yap9us6v52vjjsrvywa6rt52cm9r9zqt8r2t7mlcwspyetp5h2tztugp9lfyql
```


<img width="2727" height="1319" alt="bolt11-invoice-breakdown" src="https://github.com/user-attachments/assets/b4341688-dc2e-4fcd-8273-c4f3f3925ab8" />


### Working with the terminal

Let's dive right in. Set up 4 nodes on your polar network, and deposit 2M sats into each node, then run "lncli walletbalance" in each node's terminal to confirm the funds have been deposited there. The names of the nodes in this piece are Alice (sender), Carol, Erin and Dave (recipient).

Alice's (sender) Terminal:

```bash
lnd@alice:/$ lncli walletbalance
```

```json
{
    "total_balance":  "2000000",
    "confirmed_balance":  "2000000",
    "unconfirmed_balance":  "0",
    "locked_balance":  "0",
    "reserved_balance_anchor_chan":  "10000",
    "account_balance":  {
        "default":  {
            "confirmed_balance":  "994175",
            "unconfirmed_balance":  "0"
        }
    }
}
```

```
lnd@alice:/$
```

Next, Dave is the one receiving the payment, so, you create an invoice of 100k sats on Dave's node via the terminal.

```bash
lnd@dave:/$ alias lncli="lncli - network regtest"
lnd@dave:/$
lnd@dave:/$ lncli addinvoice - amt 100000 -memo "multihop demo payment"
```

```json
{
 "r_hash": "a0d05d169472b7dcb5729846926161115ff387149f57d7198b8de03c1ee75f83",
 "payment_request": "lnbcrt1m1p42508ppp55rg96955w2maedtjnprfyctpz90l8pc5natawxvt3hsrc8h8t7psdpzd46kcarfdphhqgryv4kk7grsv9uk6etwwscqzzsxqyz5vqsp50nnldr4ykrqx5k2klfclewq453u6r66sgmt7gnl4d3w9pemktkgs9qxpqysgqc4x6dygkpu682458rwaw5hdrazyw3nujl26a45a23kr3zlaxsyhxyjvkyequpf2uxd7xuglmhmlav9dammhautd6st7jdgcaph5w6tcqmcny2h",
 "add_index": "1",
 "payment_addr": "7ce7f68ea4b0c06a5956fa71fcb815a479a1eb5046d7e44ff56c5c50e7765d91"
}
```

```
lnd@dave:/$
```

r_hash: This is H. The SHA256 hash of the preimage R that Dave generated. Every HTLC along the route will lock against it.

payment_request: The Bech32-encoded full BOLT11 invoice string, starting with "lnbcrt"; "rt" confirms this was generated on regtest. Dave hands this over to Alice.

add_index: A simple sequential counter LND assigns to invoices as they're created on this node; this one is invoice #1.

payment_addr: This is the payment secret, a separate random 32-byte value (distinct from r_hash) that gets embedded in the invoice specifically to prevent a class of attack where a malicious intermediate node tries to guess/probe payment details by sending partial payment attempts. It's why modern invoices carry both a hash and a secret rather than just a hash alone.

Next, to see the details of the invoice that was sent by Dave, copy the payment request string from the response, and then on Alice's terminal, decode it like so:

```bash
lnd@alice:/$ alias lncli="lncli --network regtest"
lnd@alice:/$
lnd@alice:/$ lncli decodepayreq lnbcrt1m1p42508ppp55rg96955w2maedtjnprfyctpz90l8pc5natawxvt3hsrc8h8t7psdpzd46kcarfdphhqgryv4kk7grsv9uk6etwwscqzzsxqyz5vqsp50nnldr4ykrqx5k2klfclewq453u6r66sgmt
7gnl4d3w9pemktkgs9qxpqysgqc4x6dygkpu682458rwaw5hdrazyw3nujl26a45a23kr3zlaxsyhxyjvkyequpf2uxd7xu
glmhmlav9dammhautd6st7jdgcaph5w6tcqmcny2h
```

```json
{
    "destination":  "03122afa9745d2e765bbd5229efba53a42d1bc90c23d8860c6aeed1a0e18566c96",
    "payment_hash":  "a0d05d169472b7dcb5729846926161115ff387149f57d7198b8de03c1ee75f83",
    "num_satoshis":  "100000",
    "timestamp":  "1789541601",
    "expiry":  "86400",
    "description":  "multihop demo payment",
    "description_hash":  "",
    "fallback_addr":  "",
    "cltv_expiry":  "80",
    "route_hints":  [],
    "payment_addr":  "7ce7f68ea4b0c06a5956fa71fcb815a479a1eb5046d7e44ff56c5c50e7765d91",
    "num_msat":  "100000000",
    "features":  {
        "8":  {
            "name":  "tlv-onion",
            "is_required":  true,
            "is_known":  true
        },
        "14":  {
            "name":  "payment-addr",
            "is_required":  true,
            "is_known":  true
        },
        "17":  {
            "name":  "multi-path-payments",
            "is_required":  false,
            "is_known":  true
        },
        "25":  {
            "name":  "route-blinding",
            "is_required":  false,
            "is_known":  true
        }
    },
    "blinded_paths":  []
}
```

```
lnd@alice:/$
```

destination: this is Dave's node public key; this is where the funds ultimately need to reach, and this is not the same as who Alice sends the packet to first; that would be Carol, the first hop.

payment_hash: this is H, the same value from the r_hash.

num_satoshis/num_msat: the amount Dave is actually owed. Msats exist because Lightning supports sub-satoshi precision for fee calculation across hops.

cltv_expiry: this is the final-hop timelock delta; it's the minimum number of blocks Dave requires before his HTLC can expire.

features→route-blinding: this is a newer BOLT4 privacy feature that hides the final recipient's identity even further.

features→multi-path-payments: this confirms Dave's node can accept a single payment split across multiple routes (MPP)

Next, connect to the other nodes in the network and open channels to them. To do that, we need the URI, which includes the public key, host, and port of the node we want to connect to. Connecting to the other peers is only needed if the nodes were not previously connected. We need to get the URI of the nodes in the network

Carol's terminal:

```bash
lnd@carol:/$ lncli getinfo
```

```json
{
    "version":  "0.20.0-beta commit=v0.20.0-beta",
    "commit_hash":  "b9ea7070c20ad2ca8514a47d9b4d560a501f0487",
    "identity_pubkey":  "02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf2958ac1440e9d42",
    ... (truncated)
    "chains":  [
        {
            "chain":  "bitcoin",
            "network":  "regtest"
        }
    ],
    "uris":  [
        "02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf2958ac1440e9d42@172.21.0.3:9735"
    ],
    "features":  {
        ...
     }
```

Erin's Terminal:

```bash
lnd@erin:/$ lncli getinfo
```

```json
{
    "version":  "0.20.0-beta commit=v0.20.0-beta",
    "commit_hash":  "b9ea7070c20ad2ca8514a47d9b4d560a501f0487",
    "identity_pubkey":  "0344220697489632d8f511467b9dcdd7757a7244e3f8f018d33045f19f32f1038c",
    ...
    "chains":  [
        {
            "chain":  "bitcoin",
            "network":  "regtest"
        }
    ],
    "uris":  [
        "0344220697489632d8f511467b9dcdd7757a7244e3f8f018d33045f19f32f1038c@172.21.0.6:9735"
    ],
    "features":  {
        ...
     }
```

Dave's Terminal:

```bash
lnd@dave:/$ lncli getinfo
```

```json
{
    "version":  "0.20.0-beta commit=v0.20.0-beta",
    "commit_hash":  "b9ea7070c20ad2ca8514a47d9b4d560a501f0487",
    "identity_pubkey":  "03122afa9745d2e765bbd5229efba53a42d1bc90c23d8860c6aeed1a0e18566c96",
    ...
    "chains":  [
        {
            "chain":  "bitcoin",
            "network":  "regtest"
        }
    ],
    "uris":  [
        "03122afa9745d2e765bbd5229efba53a42d1bc90c23d8860c6aeed1a0e18566c96@172.21.0.2:9735"
    ],
    "features":  {
        ...
     }
```

Next, we want to create a channel from ALICE (sender) → CAROL → ERIN → DAVE (recipient).

Alice's Terminal:

```bash
lnd@alice:/$ lncli connect 02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf2958ac1440e9d42@172.21.0.3:9735
[lncli] rpc error: code = Unknown desc = already connected to peer: 02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf2958ac1440e9d42@172.21.0.3:46892
lnd@alice:/$
lnd@alice:/$ lncli openchannel --node_key=02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf29
58ac1440e9d42 --local_amt=2000000
lnd@alice:/$
[lncli] rpc error: code = Unknown desc = not enough witness outputs to create funding transaction, need 0.02003043 BTC, only have 0.02000000 BTC available
lnd@alice:/$ lncli openchannel --node_key=02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf2958ac1440e9d42 --local_amt=2000000
```

```json
{
    "funding_txid": "d0e9cb9c8fdd906cce3c0d1560df6aa426f28d9b9f7c7029dcd8d1d1bda26fe7"
}
```

```bash
lnd@alice:/$
lnd@alice:/$ lncli listchannels
```

```json
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf2958ac1440e9d42",
            "channel_point": "d0e9cb9c8fdd906cce3c0d1560df6aa426f28d9b9f7c7029dcd8d1d1bda26fe7:1",
            "chan_id": "e76fa2bdd1d1d8dc29707c9f9b8df226a46adf60150d3cce6c90dd8f9ccbe9d1",
            "scid": "222101348876289",
            "scid_str": "202x1x1",
            "capacity": "2000000",
            "local_balance": "1996530",
            "remote_balance": "0",
            "commit_fee": "3140",
            "commit_weight": "772",
            "fee_per_kw": "2500",
            "unsettled_balance": "0",
            "total_satoshis_sent": "0",
            "total_satoshis_received": "0",
            "num_updates": "0",
            "pending_htlcs": [],
            "csv_delay": 240,
            "private": false,
            "initiator": true,
            "chan_status_flags": "ChanStatusDefault",
            "local_chan_reserve_sat": "20000",
            "remote_chan_reserve_sat": "20000",
            "static_remote_key": false,
            "commitment_type": "ANCHORS",
            "lifetime": "1152",
            "uptime": "1152",
            "close_address": "",
            "push_amount_sat": "0",
            "thaw_height": 0,
            "local_constraints": {
                "csv_delay": 240,
                "chan_reserve_sat": "20000",
                "dust_limit_sat": "354",
                "max_pending_amt_msat": "1980000000",
                "min_htlc_msat": "1",
                "max_accepted_htlcs": 483
            },
            "remote_constraints": {
                "csv_delay": 240,
                "chan_reserve_sat": "20000",
                "dust_limit_sat": "354",
                "max_pending_amt_msat": "1980000000",
                "min_htlc_msat": "1",
                "max_accepted_htlcs": 483
            },
            "alias_scids": [],
            "zero_conf": false,
            "zero_conf_confirmed_scid": "0",
            "peer_alias": "carol",
            "peer_scid_alias": "0",
            "memo": "",
            "custom_channel_data": ""
        }
    ]
}
```

The RPC errors were deliberate; the first error tells you that you have already connected to the node previously, while the second error came as a shock to me as well; it's telling you that you don't have enough to cover the on-chain funding transaction's mining fee. All you have to do is deposit more sats into your wallet to cover the mining fee. We got a response with the funding_txid; it shows that we have successfully opened a channel with Carol.

The flag "node_key" equals the node's public key

Carol's Terminal:

```bash
lnd@carol:/$ lncli openchannel --node_key=0344220697489632d8f511467b9dcdd7757a7244e3f8f018d33045f19f32f1038c --local_amt=2000000
[lncli] rpc error: code = Unknown desc = peer 0344220697489632d8f511467b9dcdd7757a7244e3f8f018d33045f19f32f1038c is not online
lnd@carol:/$
lnd@carol:/$ lncli connect 0344220697489632d8f511467b9dcdd7757a7244e3f8f018d33045f19f32f1038c@172.21.0.6:9735
```

```json
{
    "status":  "connection to 0344220697489632d8f511467b9dcdd7757a7244e3f8f018d33045f19f32f1038c@172.21.0.6:9735 initiated"
}
```

```bash
lnd@carol:/$ lncli openchannel --node_key=0344220697489632d8f511467b9dcdd7757a7244e3f8f018d33045f19f32f1038c --local_amt=2000000
```

```json
{
    "funding_txid": "cfaa54eb1110e8586d2cd5e1a818da2820ff465d9a75d3e7b80de5ee3c56b6fe"
}
```

```bash
lnd@carol:/$lnd@carol:/$ lncli listchannels
```

```json
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "039ac8df86cbe28991911a8fc824d78b2e80e78dcaf183aebb658a59dee4311251",
            "channel_point": "d0e9cb9c8fdd906cce3c0d1560df6aa426f28d9b9f7c7029dcd8d1d1bda26fe7:1",
            "chan_id": "e76fa2bdd1d1d8dc29707c9f9b8df226a46adf60150d3cce6c90dd8f9ccbe9d1",
            "scid": "222101348876289",
            "scid_str": "202x1x1",
            "capacity": "2000000",
            "local_balance": "0",
            "remote_balance": "1996530",
           ...
            "local_constraints": {
                ...
            },
            "remote_constraints": {
                ...
            },
            ...
        }
    ]
}
```

The RPC error here, "peer 03... is not online" means that you are not yet connected to the node you want to open a channel with. So you have to connect to it, then open a channel.

Erin's Terminal:

```bash
lnd@erin:/$ lncli openchannel --node_key=03122afa9745d2e765bbd5229efba53a42d1bc90c23d8860c6aeed1a0e18566c96 --local_amt=2000000
```

```json
{
    "funding_txid": "5616fd9c842f61c521932e05c18a49e03c057fb4cdcd587e99f484d05f2e1231"
}
```

```bash
lnd@erin:/$ lncli listchannels
```

```json
{
    "channels":  []
}
```

```bash
lnd@erin:/$ lncli pendingchannels
```

```json
{
    "total_limbo_balance":  "0",
    "pending_open_channels":  [
        {
            "channel":  {
                "remote_node_pub":  "02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf2958ac1440e9d42",
                "channel_point":  "cfaa54eb1110e8586d2cd5e1a818da2820ff465d9a75d3e7b80de5ee3c56b6fe:1",
                "capacity":  "2000000",
                "local_balance":  "0",
                "remote_balance":  "1996530",
                "local_chan_reserve_sat":  "20000",
                "remote_chan_reserve_sat":  "20000",
                "initiator":  "INITIATOR_REMOTE",
                "commitment_type":  "ANCHORS",
                "num_forwarding_packages":  "0",
                "chan_status_flags":  "",
                "private":  false,
                "memo":  "",
                "custom_channel_data":  ""
            },
            "commit_fee":  "2810",
            "commit_weight":  "772",
            "fee_per_kw":  "2500",
            "funding_expiry_blocks":  2014,
            "confirmations_until_active":  1,
            "confirmation_height":  215
        },
        {
            "channel":  {
                "remote_node_pub":  "03122afa9745d2e765bbd5229efba53a42d1bc90c23d8860c6aeed1a0e18566c96",
                "channel_point":  "5616fd9c842f61c521932e05c18a49e03c057fb4cdcd587e99f484d05f2e1231:1",
                "capacity":  "2000000",
                "local_balance":  "1996530",
                "remote_balance":  "0",
                "local_chan_reserve_sat":  "20000",
                "remote_chan_reserve_sat":  "20000",
                "initiator":  "INITIATOR_LOCAL",
                "commitment_type":  "ANCHORS",
                "num_forwarding_packages":  "0",
                "chan_status_flags":  "",
                "private":  false,
                "memo":  "",
                "custom_channel_data":  ""
            },
            "commit_fee":  "2810",
            "commit_weight":  "772",
            "fee_per_kw":  "2500",
            "funding_expiry_blocks":  2014,
            "confirmations_until_active":  1,
            "confirmation_height":  215
        }
    ],
    "pending_closing_channels":  [],
    "pending_force_closing_channels":  [],
    "waiting_close_channels":  []
}
```

```
lnd@erin:/$
```

You can confirm whether the channel is open or it 
is pending via the terminal in Polars UI. To confirm via the terminal, run the "lncli pendingchannels" command to see the pending channels. The dotted lines in the image below mean that the channel to that node is still opening, while the full line means that a channel has been established (Alice to Carol). You would have to mine a couple more blocks for confirmation.


<img width="2727" height="1319" alt="polar-channel-opening-status" src="https://github.com/user-attachments/assets/f615ed64-2b9b-4d22-807a-62feefd8e8b8" />


## Route Selection

We can actually see the route Alice's payment could/would take without actually sending the payment.

Alice's Terminal:

```bash
lnd@alice:/$ lncli queryroutes --dest 03122afa9745d2e765bbd5229efba53a42d1bc90c23d8860c6aeed1a0e18566c96 --amt 100000
```

```json
{
    "routes":  [
        {
            "total_time_lock":  462,
            "total_fees":  "2",
            "total_amt":  "100002",
            "hops":  [
                {
                    "chan_id":  "222101348876289",
                    "chan_capacity":  "2000000",
                    "amt_to_forward":  "100001",
                    "fee":  "1",
                    "expiry":  382,
                    "amt_to_forward_msat":  "100001100",
                    "fee_msat":  "1100",
                    "pub_key":  "02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf2958ac1440e9d42",
                    "tlv_payload":  true,
                    "mpp_record":  null,
                    "amp_record":  null,
                    "custom_records":  {},
                    "metadata":  "",
                    "blinding_point":  "",
                    "encrypted_data":  "",
                    "total_amt_msat":  "0"
                },
                {
                    "chan_id":  "236395000102913",
                    "chan_capacity":  "2000000",
                    "amt_to_forward":  "100000",
                    "fee":  "1",
                    "expiry":  302,
                    "amt_to_forward_msat":  "100000000",
                    "fee_msat":  "1100",
                    "pub_key":  "0344220697489632d8f511467b9dcdd7757a7244e3f8f018d33045f19f32f1038c",
                    "tlv_payload":  true,
                    "mpp_record":  null,
                    "amp_record":  null,
                    "custom_records":  {},
                    "metadata":  "",
                    "blinding_point":  "",
                    "encrypted_data":  "",
                    "total_amt_msat":  "0"
                },
                {
                    "chan_id":  "236395000037377",
                    "chan_capacity":  "2000000",
                    "amt_to_forward":  "100000",
                    "fee":  "0",
                    "expiry":  302,
                    "amt_to_forward_msat":  "100000000",
                    "fee_msat":  "0",
                    "pub_key":  "03122afa9745d2e765bbd5229efba53a42d1bc90c23d8860c6aeed1a0e18566c96",
                    "tlv_payload":  true,
                    "mpp_record":  null,
                    "amp_record":  null,
                    "custom_records":  {},
                    "metadata":  "",
                    "blinding_point":  "",
                    "encrypted_data":  "",
                    "total_amt_msat":  "0"
                }
            ],
            "total_fees_msat":  "2200",
            "total_amt_msat":  "100002200",
            "first_hop_amount_msat":  "0",
            "custom_channel_data":  ""
        }
    ],
    "success_prob":  1
}
```

```
lnd@alice:/$
```

From the response, we can see the hop flow from Alice to Carol to Erin to Dave (based on their public key). One important field we can see from the response is amt_to_forward and the fee attached to the amount to forward. Carol receives a fee of 1 sat, Erin receives a fee of 1 sat as well, while Dave doesn't receive any fees because that is the destination node. The amt_to_forward reduces at each hop by the fee amount.

total_fees: the entire 3-hop route costs Alice just 2 sats total(2200 msat, per total_fees_msat). The fees are small because regtest channels default to tiny fee rates, but the mechanism is identical to mainnet.

success_prob: This is LND's confidence estimate that this route will actually succeed. It's 1.0 here because this is Alice's own small test network with fully known channel states; on mainnet, this number is rarely exactly 1 since nodes can't see the others' real-time liquidity and rely on probabilistic estimates

total_time_lock and per-hop expiry values: They show the timelock getting pushed further into the future, giving each intermediary a safety margin to claim their HTLC before their own upstream obligation expires.

## Onion Construction

Now we know the route that Alice's payment would take and the exact amount and fee at each hop. During an onion construction, according to the Sphinx onion routing (BOLT 4), Alice encrypts the payment instructions in nested layers, one per hop, so each intermediary decrypts only its own layer. Each hop's decryption key comes from an ECDH (Elliptic Curve Diffie-Hellman) exchange with an ephemeral key Alice generates fresh for this payment; the packet is padded to a fixed size so a 3-hop and a 20-hop route look identical on the wire, and an HMAC chain lets each hop verify the packet wasn't tampered with.

The example tries to replicate the peeling property when each hop decrypts exactly one layer, learns only its own instructions, and passes on a blob it cannot read. The encryption is done from the inside out (from the last hop to the first hop).

```rust
// Dave's layer is the last hop
let dave_layer = FinalHopLayer {
    role: "final_hop".into(),
    amount_to_forward_msat: 100_000_000,
    payment_hash: "a0d05d169472b7dcb5729846926161115ff387149f57d7198b8de03c1ee75f83".into(),
};
// Dave's layer is encrypted and would be passed into Erin's layer encryption
// &dave_key is known only to Dave
let encrypted_for_dave = encrypt(&dave_key, serde_json::to_string(&dave_layer).unwrap().as_bytes());

// Erin's layer is the next hop
let erin_layer = ForwardingHopLayer {
    role: "forwarding_hop".into(),
    next_hop_chan_id: "236395000037377".into(),
    amount_to_forward_msat: 100_000_000, // forwards this amount and keeps 1100 msat(1 sats)
    encrypted_payload_for_next_hop: encrypted_for_dave,
};
// Erin's layer is encrypted and passed into Carol's layer encryption
// &erin_key is known only by Erin
let encrypted_for_erin = encrypt(&erin_key, serde_json::to_string(&erin_layer).unwrap().as_bytes());

// Carol's layer is the first hop
let carol_layer = ForwardingHopLayer {
    role: "forwarding_hop".into(),
    next_hop_chan_id: "236395000102913".into(),
    amount_to_forward_msat: 100_001_110, // forwards this amount and keeps 1100 msat(1 sats)
    encrypted_payload_for_next_hop: encrypted_for_erin,
};
// Carol's layer is encrypted and would be passed into Alice's encryption
// &carol_key is known only by Carol
let encrypted_for_carol = encrypt(&carol_key, serde_json::to_string(&ecarol_layer).unwrap().as_bytes());
```

```
=== What Alice sends to Carol ===
NKdIoHY8zhHMKWp/GZEogBJWw242qVUqL+ChAB1n7fwRrZ0LeVdgMc...(truncated)

=== What Carol learns after decrypting her layer ===
{
  "role": "forwarding_hop",
  "next_hop_chan_id": "236395000102913",
  "amount_to_forward_msat": 100001100,
  "encrypted_payload_for_next_hop": "Bsiuxb+DXlZC0PSXiKIb..."
}

>> Carol trying to use her own key on the inner blob would read decryption
>> failed: wrong key or tampered blob

=== What Erin learns after decrypting her layer ===
{
  "role": "forwarding_hop",
  "next_hop_chan_id": "236395000037377",
  "amount_to_forward_msat": 100000000,
  "encrypted_payload_for_next_hop": "031L4xOTBbld8elWlFA..."
}

=== What Dave learns after decrypting the final layer ===
{
  "role": "final_hop",
  "amount_to_forward_msat": 100000000,
  "payment_hash": "a0d05d169472b7dcb5729846926161115ff387149f57d7198b8de03c1ee75f83"
}

>> Dave sees he's the final hop and checks the amount and payment hash against
>> his own invoice, but nothing in this packet identifies Alice, Carol,
>> or Erin as intermediaries.
```

Source @ lightning-multi-hop-demo

This is a simplified implementation illustration, not BOLT 4 Sphinx. Real onion packets derive per-hop keys via ECDH, are padded to a fixed 1300 bytes, and carry an HMAC chain. See BOLT #4: Onion Routing Protocol as reference.

## Cryptographic Enforcement

This is where all we have discussed so far ties together: the invoice hash from the Payment Setup, the route from Route Selection, and the encrypted layers from Onion Construction all exist to serve one mechanism, which is the HTLC that locks funds at each hop.

Now let's check the channel state before we pay the invoice, so we can compare afterwards

Alice's Terminal:

```bash
lnd@alice:/$ lncli listchannels
```

```json
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf2958ac1440e9d42",
            "channel_point": "d0e9cb9c8fdd906cce3c0d1560df6aa426f28d9b9f7c7029dcd8d1d1bda26fe7:1",
            "chan_id": "e76fa2bdd1d1d8dc29707c9f9b8df226a46adf60150d3cce6c90dd8f9ccbe9d1",
            "scid": "222101348876289",
            ...
            "local_balance": "1996530",
            "remote_balance": "0",
            ...
            "peer_alias": "carol",
            "peer_scid_alias": "0",
            ...
        }
    ]
}
```

Carol's Terminal:

```bash
lnd@carol:/$ lncli listchannels
```

```json
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "0344220697489632d8f511467b9dcdd7757a7244e3f8f018d33045f19f32f1038c",
            "channel_point": "cfaa54eb1110e8586d2cd5e1a818da2820ff465d9a75d3e7b80de5ee3c56b6fe:1",
            "chan_id": "feb6563ceee50db8e7d3759a5d46ff2028da18a8e1d52c6d58e81011eb54aace",
            "scid": "236395000102913",
            "scid_str": "215x2x1",
            "capacity": "2000000",
            "local_balance": "1996530",
            "remote_balance": "0",
            ...
            "peer_alias": "erin",
            "peer_scid_alias": "0",
            ...
        },
        {
            "active": true,
            "remote_pubkey": "039ac8df86cbe28991911a8fc824d78b2e80e78dcaf183aebb658a59dee4311251",
            "channel_point": "d0e9cb9c8fdd906cce3c0d1560df6aa426f28d9b9f7c7029dcd8d1d1bda26fe7:1",
            "chan_id": "e76fa2bdd1d1d8dc29707c9f9b8df226a46adf60150d3cce6c90dd8f9ccbe9d1",
            "scid": "222101348876289",
            "scid_str": "202x1x1",
            "capacity": "2000000",
            "local_balance": "0",
            "remote_balance": "1996530",
            ...
            "peer_alias": "alice",
            "peer_scid_alias": "0",
            ...
        }
    ]
}
```

// The 2 channels shown here are this node's connection to Alice and Erin

Erin's Terminal:

```bash
lnd@erin:/$ lncli listchannels
```

```json
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf2958ac1440e9d42",
            "channel_point": "cfaa54eb1110e8586d2cd5e1a818da2820ff465d9a75d3e7b80de5ee3c56b6fe:1",
            "chan_id": "feb6563ceee50db8e7d3759a5d46ff2028da18a8e1d52c6d58e81011eb54aace",
            "scid": "236395000102913",
            "scid_str": "215x2x1",
            "capacity": "2000000",
            "local_balance": "0",
            "remote_balance": "1996530",
            ...
            "peer_alias": "carol",
            "peer_scid_alias": "0",
            ...
        },
        {
            "active": true,
            "remote_pubkey": "03122afa9745d2e765bbd5229efba53a42d1bc90c23d8860c6aeed1a0e18566c96",
            "channel_point": "5616fd9c842f61c521932e05c18a49e03c057fb4cdcd587e99f484d05f2e1231:1",
            "chan_id": "31122e5fd084f4997e58cdcdb47f053ce0498ac1052e9321c5612f849cfd1657",
            "scid": "236395000037377",
            "scid_str": "215x1x1",
            "capacity": "2000000",
            "local_balance": "1996530",
            "remote_balance": "0",
            ...
            "peer_alias": "dave",
            "peer_scid_alias": "0",
            ...
        }
    ]
}
```

// The 2 channels shown here are this node's connection to Carol and Dave

Dave's Terminal:

```bash
lnd@dave:/$ lncli listchannels
```

```json
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": "0344220697489632d8f511467b9dcdd7757a7244e3f8f018d33045f19f32f1038c",
            "channel_point": "5616fd9c842f61c521932e05c18a49e03c057fb4cdcd587e99f484d05f2e1231:1",
            "chan_id": "31122e5fd084f4997e58cdcdb47f053ce0498ac1052e9321c5612f849cfd1657",
            "scid": "236395000037377",
            "scid_str": "215x1x1",
            "capacity": "2000000",
            "local_balance": "0",
            "remote_balance": "1996530",
            ...
            "peer_alias": "erin",
            "peer_scid_alias": "0",
            ...
        }
    ]
}
```

// The 2 channels shown here are this node's connection to Carol and Dave

Now let's pay Dave's invoice from Alice's Terminal

Alice's Terminal:

```bash
lnd@alice:/$ lncli payinvoice --json lnbcrt1m1p42508ppp55rg96955w2maedtjnprfyctpz90l8pc5natawxv
t3hsrc8h8t7psdpzd46kcarfdphhqgryv4kk7grsv9uk6etwwscqzzsxqyz5vqsp50nnldr4ykrqx5k2klfclewq453u6r6
6sgmt7gnl4d3w9pemktkgs9qxpqysgqc4x6dygkpu682458rwaw5hdrazyw3nujl26a45a23kr3zlaxsyhxyjvkyequpf2u
xd7xuglmhmlav9dammhautd6st7jdgcaph5w6tcqmcny2h
Payment hash: a0d05d169472b7dcb5729846926161115ff387149f57d7198b8de03c1ee75f83
Description: multihop demo payment
Amount (in satoshis): 100000
Fee limit (in satoshis): 5000
Destination: 03122afa9745d2e765bbd5229efba53a42d1bc90c23d8860c6aeed1a0e18566c96
Confirm payment (yes/no): yes
```

```json
{
    "payment_hash":  "a0d05d169472b7dcb5729846926161115ff387149f57d7198b8de03c1ee75f83",
    "value":  "100000",
    "creation_date":  "1789593579",
    "fee":  "2",
    "payment_preimage":  "f41234e52caa9d43a1419ed38138ac75e578d9da5c27ab22a84326ef0d6c5d87",
    "value_sat":  "100000",
    "value_msat":  "100000000",
    "payment_request":  "lnbcrt1m1p42508ppp55rg96955w2maedtjnprfyctpz90l8pc5natawxvt3hsrc8h8t7psdpzd46kcarfdphhqgryv4kk7grsv9uk6etwwscqzzsxqyz5vqsp50nnldr4ykrqx5k2klfclewq453u6r66sgmt7gnl4d3w9pemktkgs9qxpqysgqc4x6dygkpu682458rwaw5hdrazyw3nujl26a45a23kr3zlaxsyhxyjvkyequpf2uxd7xuglmhmlav9dammhautd6st7jdgcaph5w6tcqmcny2h",
    "status":  "SUCCEEDED",
    "fee_sat":  "2",
    "fee_msat":  "2200",
    "creation_time_ns":  "1789593579046493425",
    "htlcs":  [
        {
            "attempt_id":  "1",
            "status":  "SUCCEEDED",
            "route":  {
                "total_time_lock":  466,
                "total_fees":  "2",
                "total_amt":  "100002",
                "hops":  [
                    {
                        "chan_id":  "222101348876289",
                        "chan_capacity":  "2000000",
                        "amt_to_forward":  "100001",
                        "fee":  "1",
                        "expiry":  386,
                        "amt_to_forward_msat":  "100001100",
                        "fee_msat":  "1100",
                        "pub_key":  "02d8ae0f5d0089ff5da86843eb4dd0919aa385eb36a5bedffdf2958ac1440e9d42",
                        "tlv_payload":  true,
                        "mpp_record":  null,
                        "amp_record":  null,
                        "custom_records":  {},
                        "metadata":  "",
                        "blinding_point":  "",
                        "encrypted_data":  "",
                        "total_amt_msat":  "0"
                    },
                    {
                        "chan_id":  "236395000102913",
                        "chan_capacity":  "2000000",
                        "amt_to_forward":  "100000",
                        "fee":  "1",
                        "expiry":  306,
                        "amt_to_forward_msat":  "100000000",
                        "fee_msat":  "1100",
                        "pub_key":  "0344220697489632d8f511467b9dcdd7757a7244e3f8f018d33045f19f32f1038c",
                        "tlv_payload":  true,
                        "mpp_record":  null,
                        "amp_record":  null,
                        "custom_records":  {},
                        "metadata":  "",
                        "blinding_point":  "",
                        "encrypted_data":  "",
                        "total_amt_msat":  "0"
                    },
                    {
                        "chan_id":  "236395000037377",
                        "chan_capacity":  "2000000",
                        "amt_to_forward":  "100000",
                        "fee":  "0",
                        "expiry":  306,
                        "amt_to_forward_msat":  "100000000",
                        "fee_msat":  "0",
                        "pub_key":  "03122afa9745d2e765bbd5229efba53a42d1bc90c23d8860c6aeed1a0e18566c96",
                        "tlv_payload":  true,
                        "mpp_record":  {
                            "payment_addr":  "7ce7f68ea4b0c06a5956fa71fcb815a479a1eb5046d7e44ff56c5c50e7765d91",
                            "total_amt_msat":  "100000000"
                        },
                        "amp_record":  null,
                        "custom_records":  {},
                        "metadata":  "",
                        "blinding_point":  "",
                        "encrypted_data":  "",
                        "total_amt_msat":  "0"
                    }
                ],
                "total_fees_msat":  "2200",
                "total_amt_msat":  "100002200",
                "first_hop_amount_msat":  "100002200",
                "custom_channel_data":  ""
            },
            "attempt_time_ns":  "1789593579120937459",
            "resolve_time_ns":  "1789593579980452341",
            "failure":  null,
            "preimage":  "f41234e52caa9d43a1419ed38138ac75e578d9da5c27ab22a84326ef0d6c5d87"
        }
    ],
    "payment_index":  "1",
    "failure_reason":  "FAILURE_REASON_NONE",
    "first_hop_custom_records":  {}
}
```

```
lnd@alice:/$
```

// Hashing the payment_preimage would equal to the payment_hash, this is the
// verification that the transaction succeeded.

We can check the channels to see the newer balances after that transaction

Alice's Terminal:

```bash
lnd@alice:/$ lncli listchannels
```

```json
{
    "channels": [
        {
            "active": true,
            ...
            "local_balance": "1896527",
            "remote_balance": "100002",
            ...
}
```

// You can see and compare that the balance amount has changed

Carol's Terminal:

```bash
lnd@carol:/$ lncli listchannels
```

```json
{
    "channels": [
        {
            "active": true,
            ...
            "local_balance": "1896528",
            "remote_balance": "100001",
            ...
        },
        {
            "active": true,
            ...
            "local_balance": "100002",
            "remote_balance": "1896527",
            ...
    ]
}
```

// You can see and compare the balance amount has changed

Erin's Terminal:

```bash
lnd@erin:/$ lncli listchannels
```

```json
{
    "channels": [
        {
            "active": true,
            ...
            "local_balance": "100001",
            "remote_balance": "1896528",
            ...
        },
        {
            "active": true,
            ...
            "local_balance": "1896530",
            "remote_balance": "100000",
            ...
    ]
}
```

// You can see and compare the balance amount have changed

Dave's Terminal:

```bash
lnd@dave:/$ lncli listchannels
```

```json
{
    "channels": [
        {
            "active": true,
            ...
            "capacity": "2000000",
            "local_balance": "100000",
            "remote_balance": "1896530",
            ...
}
```

// You can see and compare that the balance amount has changed

To further confirm that "each hop only sees its own slice", we check the forwarding history in the terminal of each node to see and know what they saw as well

Carol's Terminal:

```bash
lnd@carol:/$ lncli fwdinghistory --start_time 0
```

```json
{
    "forwarding_events":  [
        {
            "timestamp":  "1789593579",
            "chan_id_in":  "222101348876289",
            "chan_id_out":  "236395000102913",
            "amt_in":  "100002",
            "amt_out":  "100001",
            "fee":  "1",
            "fee_msat":  "1100",
            "amt_in_msat":  "100002200",
            "amt_out_msat":  "100001100",
            "timestamp_ns":  "1789593579960099781",
            "peer_alias_in":  "alice",
            "peer_alias_out":  "erin",
            "incoming_htlc_id":  "0",
            "outgoing_htlc_id":  "0"
        }
    ],
    "last_offset_index":  1
}
```

Erin's Terminal:

```bash
lnd@erin:/$ lncli fwdinghistory --start_time 0
```

```json
{
    "forwarding_events":  [
        {
            "timestamp":  "1789593579",
            "chan_id_in":  "236395000102913",
            "chan_id_out":  "236395000037377",
            "amt_in":  "100001",
            "amt_out":  "100000",
            "fee":  "1",
            "fee_msat":  "1100",
            "amt_in_msat":  "100001100",
            "amt_out_msat":  "100000000",
            "timestamp_ns":  "1789593579956111178",
            "peer_alias_in":  "carol",
            "peer_alias_out":  "dave",
            "incoming_htlc_id":  "0",
            "outgoing_htlc_id":  "0"
        }
    ],
    "last_offset_index":  1
}
```

From both terminal outputs, you can see that each node was only aware of 2 things: one that it received from a node and that it is routing to another node. It is never aware of the entire route, or other nodes outside of its immediate neighbours.

## Wrap-Up

Whew!!! What a ride!!! This piece was written to try to dissect multi-hop payments end-to-end using the terminal, so the outputs are live and similar to the main environment. To summarise, Dave generated a preimage and hash commitment. Alice's node picked a real 3-hop route through Carol and Erin, with fees calculated before sending any sats. A simplified version of the onion construction, built in Rust using AES-GCM encryption, showed why Carol can't tell whether Erin is the final recipient or just another forwarder. And when the payment actually settled, SHA256 of the revealed preimage produced the same hash that appeared in Dave's invoice at the very start.

## Conclusion

Along the way, a few things turned out to matter more than expected. Carol always knows she's forwarding from Alice to Erin, but it's never aware of any other nodes on the route. And the entire 3-hop payment, invoice creation, route discovery, onion construction, HTLC propagation, and settlement, resolved in well under a second. This shows how fast the Lightning Network actually is.

This piece leaves out how Alice's node chose that particular route out of every possible path through the network, and what happens when a payment doesn't succeed. I believe that both are substantial enough to deserve their own piece.

Next in the series:

- How Lightning picks a route (Fee weighting, success probability estimation, Mission Control and routing MPP)
- What happens when a Lightning Payment Fails (BOLT4's error onions)
