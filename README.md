# Lightning Rod - async Lightning payments protocol

## Introduction

In order to make a payment transaction between two Lightning Network nodes, the two nodes need to be online.

This restriction presents a challenge when the two nodes are not always online, for instance in the case of mobile devices.

Lightning Rod protocol is a way to circumvent this limitation by using [LSPs](https://medium.com/breez-technology/introducing-lightning-service-providers-fe9fb1665d5f) which are always online and can intercept incoming/outgoing HTLCs and break the payment flow to an asynchronous flow (payer and the payee are don't have to be simultaneously online). LSPs have also the advantage of being able to send notifications to their users.

See [Introducing Lightning Rod](https://medium.com/breez-technology/introducing-lightning-rod-2e0a40d3e44a) for a more detailed introduction. However, unlike the technique described in this post, this version of the Rod is simplified by using an HTLC inteceptor instead of executing two payments. Also, in this version, routing nodes' liquidity isn't being locked while the async payment is processed.

## Protocol

### Definitions

* `A`: the payer which is not always online.
* `B`: the payee which is not always online.
* `LSP_a` (resp `LSP_b`): the LSP that `A` (resp `B`) is connected to.

### Notifications

* `A` (resp `B`) can register itself using a platform dependant `deviceid_a` (resp `devideid_b`) to `LSP_a` (resp `LSP_b`).
* `LSP_a` (resp `LSP_b`) can use `deviceid_a` (resp `devideid_b`) to send a notification to `A` (resp `B`) in order to ask `A` (resp `B`) to go online.
* We assume here that `LSP_a` (resp `LSP_b`) sends a notification to `A` (resp `B`) in the following case:
  1. when an [Onion Message](https://github.com/rustyrussell/lightning-rfc/blob/guilt/offers/04-onion-routing.md#onion-messages) needs to be forwarded to `A` (resp `B`).
  2. when an preimage needs to be sent to `A` in order to settle a payment.

### Payment Workflow

`A` wants to pay `amt` to `B` using an invoice created by `B` and with a payment_hash `h` - this can be the result of an [offer](https://github.com/rustyrussell/lightning-rfc/commits/guilt/offers) message. Naturally,`A` is online, but `B` may be offline.

1. `A` generates a secret payment id `pid`.
2. `A` sends a onion message to `LSP_a`, asking it to intercept and block the HTLC for the payment with the payment_hash `h`. This can be a list of payment_hashes when the payment uses `MPP`.
3. `A` sends the payment with the payment_hash `h` to `B`. `LSP_a` intercepts and blocks the HTLC(s).
4. `A` sends an onion message to `B`, containing `pid`. `A` can go offline.
5. `LSP_b` [notifies](#notifications) `B`. `B` goes online.
6. `B` sends a message to `A` with a `pid` value for the field `payment_unblock` in the `encrypted_data_tlv` tlvstream for `LSP_a`
7. `LSP_a` intercepts this message and unblocks the HTLC(s) which reaches `B`. `B` settles the payment and sends back the preimage corresponding to `h`.
8. When the preimage reaches `LSP_a`, it [notifies](#notifications) `A`, so when `A` is back online it finally settles the payment.

![drawing](https://user-images.githubusercontent.com/31890660/166234202-e0d990e4-a547-4313-b1bc-dd08f2b312d0.png)

