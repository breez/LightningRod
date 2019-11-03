# Lightning Rod Protocol
## Introduction
In order to make a payment transaction between two Lightning Network nodes, the two nodes need to be online.

This restriction presents a challenge when the two nodes are not always online, for instance in the case of mobile devices.

Lightning Rod protocol is a way to circumvent this limitation by using an intermediate node which is always online and can securely forward the payment without needing that the payer and the payee are simultaneously online.

## Protocol

### Definitions

* `A`: the payer
* `pub_a`: the public id of `A`
* `device_id_a`: the device id of `A` used to receive notifications
####
* `B`: the payee
* `pub_b`: the public id of `B`
* `device_id_b`: the device id of `B` used to receive notifications
####
* `S_A_1`, .., `S_A_i`, .., `S_A_n` (`n >= 1`): an ordered list of Lightning Rod service providers chosen by `A`. `A` pays to `S_A_1`, then each `S_A_i` pays to `S_A_(i+1)` until `n` is reached.
* `S_B_1`, .., `S_B_j`, .. `S_B_m` (`m >= 1`): an ordered list of Lightning Rod service providers chosen by `B` in addition the the list chosen by `A`.
* `S_1`, .., `S_p` is the final list of Lightning Rod service providers: `S_A_1`, .., `S_A_n`, `S_B_1`, .., `S_B_m` (`p = n + m`) renamed in order to simplify the notation.
####
* `amt`: the amount that `A` wants to pay to `B`. More specifically the amount that `S_A_n` will send to `S_B_1`.
* `fee_i`: The fee `S_i` will receive to participate to the workflow.
* `amt_i`: the amount `S_i` will receive. We have `S_B_1 = amt` and `S_(i+1) = S_i - fee_i`.
####
* `H`: a cryptographic hash function (sha256)
* `Sign(message)->signature` the node signature function using its private key.
* `Verify(message, signature)->(Ok|Error, pubkey)` verify signature function which returns the public key of the signer if the signature is valid.

### Notification Service
This is a service which can notify mobile device users by sending them messages. It's optional.

The Notification service `NS` has two endpoints:
* `register_notification(transaction_id, event_name, device_id)`: this is a way for a device to register itself to receive notifications for a specific event in a transaction.
 * `notify_device(transaction_id, event_name)`: tell `NS` to send a message to all registered devices for corresponding transaction and event.

### Payment Workflow

`A` wants to pay `amt` to `B`:
`A` is online. `B` can be offline
1. `A` generates a secret payment id `pid`
2. `A` sends an asynchronous message to `B` using a secure messaging app with the content: "I want to pay you `amt` using the secret payment id `pid` using `S_A_1`, .., `S_A_n` services"
3. `A` calls `NS::register_notification(H(pid), "InitiatePayment", device_id_a)`

`A` can go offline

`B` comes online

1. `B` generates a preimage `p` and it's hash `h = H(p)` and a random secret `r`
2. `B` calculates `b_proof = H(h || pid || pub_b)` and sign it `b_sign = Sign(b_proof)`
3. `B` asks `S_B_1` what is the fee to forward a payment when _receiving_ `amt`, then each `S_B_i` when `i` increases.
4. `B` asks `S_A_n` what is the fee it needs to receive in order to _pay_ `amt`, then each `S_A_i` when `i` decreases.
5. `B` asks `S_i (2<=i<=p)` an invoice for the amount `amt` and send it to `S_(i-1)`.
6. `B` sends an invoice using `h` and the amount `amt_p - fee_p` to `S_p`. It also sends `H(pid||r)` to `S_p`
7. `B` sends `H(pid)`, `h`, `b_proof` and `b_sign` to `S_1`
8. `B` calls `NS::register_notification(H(pid||r), "ReceivePayment", device_id_b)`

`B` can go offline

`S_1` calls `NS:notify_device(H(pid), "InitiatePayment")` in order to notify `A`

`A` comes online

1. `A` sends `H(pid)` to `S_1`. `S_1` returns a "hold" invoice with hash: `h` and amount: `amt_1`. `S_1` returns also `b_proof` and `b_sign`.
2. `A` verifies that `b_sign` is indeed a signature of `b_proof`, and obtain `pub_b` from `Verify`.
3. `A` verifies that `b_proof = H(h || pid || pub_b)`
4. `A` initiates the payment to `S_1`
5. `A` calls `NS::register_notification(H(pid), "SettlePayment", device_id_a)`

`A` can go offline

1. After receiving the payment from `A`, `S_1` pays to `S_2` the invoice it received from `B`, then `S_i (2<=i<=p)` will do the same.
2. After receiving the payment from `S_(p-1)`, `S_p` calls `NS:notify_device(H(pid||r), "ReceivePayment")`

`B` comes online

1. `B` sends to `S` an invoice with hash: `h` and amount: `amt`
2. `S_p` pays the invoice to `B` and receives `p` as proof of payment

`B` can go offline

`S_i (1<=i<=p)` can settle the payment it received using `p`.

As soon as `S_1` receives `p`, it calls `NS:notify_device(H(pid), "SettlePayment")`

`A` comes online

`S_1` settle the transaction from `A` using `p` in order to definitely receive `amt_1` from `A`
