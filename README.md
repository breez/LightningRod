# Lightning Rod Protocol
## Introduction
In order to make a payment transaction between two Lightning Network nodes, the two nodes need to be online.

This restriction presents a challenge when the two nodes are not always online, for instance in the case of mobile devices.

Lightning Rod protocol is a way to circuveent this limitation by using an intermediate node which is always online and can securely forward the payment without needing that the payer and the payee are simultaneously online.

## Protocol

### Definitions

* `A`: the payer
* `pub_a`: the public id of `A`
* `device_id_a`: the device id of `A` used to receive notifications
* `amt`: the amount that `A` wants to pay to B. More specifically the amount that `A` wants `B` to receive
* `fee`: The fee `A` will pay to the Lightning Rod service in order to forward the payment.
####
* `B`: the paye3
* `pub_b`: the public id of `B`
* `device_id_b`: the device id of `B` used to receive notifications
####
* `S`: The Lightning Rod service
####
* `H`: a cryptographic hash function (sha256)
* `Sign(message)->signature` the node signature function using its private key.
* `Verify(message, signature)->(Ok|Error, pubkey)` verify signature function which returns the public key of the signer if the signature is valid.

### Notification Service
This is a service which can notify mobile device users by sending them messages. It's optional.

The Notification service `NS` has two endpoints:
* `register_notification(transaction_id, event_name, device_id)`: this is a way for a device to register itself to receive notifications for a specific event in a transaction.
 * `notify_device(transaction_id, event_name)`: tell `NS` to send a message to all registered devices for corresponding transaction and event/

### Payment Workflow

`A` wants to pay `amt` to `B`:
`A` is online. `B` can be offline
1. `A` generates a secret payment id `pid`
2. `A` sends an asynchronous message to `B` using a secure messaging app with the content: "I want to pay you `amt` using the secret payment id `pid`
3. `A` calls `NS::register_notification(H(pid), "InitiatePayment", device_id_a)`

`A` can go offline

`B` comes online

1. `B` generates a preimage `p` and it's hash `h = H(p)`
2. `B` calculates `b_proof = H(h || pid || pub_b)` and sign it `b_sign = Sign(b_proof)`
3. `B` sends `H(pid)`, `h`, `b_proof` and `b_sign` to `S`
4. `B` calls `NS::register_notification(H(pid), "ReceivePayment", device_id_b)`

`B` can go offline

`S` calls `NS:notify_device(H(pid), "InitiatePayment")` in order to notify `A`

`A` comes online

1. `A` sends `H(pid)` to `S`. `S` returns a "hold" invoice with hash: `h` and amount: `amt + fee`. `S` returns also `b_proof` and `b_sign`.
2. `A` verifies that `b_sign` is indeed a signature of `b_proof`, and obtain `pub_b` from `Verify`.
3. `A` verifies that `b_proof = H(h || pid || pub_b)`
4. `A` initiates the payment to `S`
5. `A` calls `NS::register_notification(H(pid), "SettlePayment", device_id_a)`

`A` can go offline

After receiving the payment from `A`, `S` calls `NS:notify_device(H(pid), "ReceivePayment")`

`B` comes online

1. `B` sends to `S` an invoice with hash: `h` and amount: `amt`
2. `S` pays the invoice to `B` and receives `p` as proof of payment

`B` can go offline

`S` calls NS:notify_device(H(pid), "SettlePayment")

`A` comes online

`S` settle the transaction from `A` using `p` in order to definitly receive `amt` from `A`
