---
eip: 
title: Gas Stations Network v2
author: Yoav Weiss <yoav@opengsn.org>, Dror Tirosh <dror@opengsn.org>, Alex Forshtat <alex@opengsn.org>
status: Draft
type: Standards Track
category: ERC
created: 2020-09-16
---

# The GSN (Gas Stations Network) Protocol
## Simple Summary

A Decentralized Meta-Transaction framework to let non-ether users to make
ethereum call into the blockchain - and let a 3rd party contract pay for the gas.
The caller account doesn't have to have any eth, 
 
Incentivize nodes to run "gas stations" or "relayers" to facilitate this. 
Require no network changes, and minimal contract changes.

## Abstract
Communicating with dapps currently requires paying ETH for gas, which limits dapp adoption to ether users. 
Therefore, contract owners may wish to pay for the gas to increase user acquisition, or let their users pay for gas with fiat money. 
Alternatively, a 3rd party may wish to subsidize the gas costs of certain contracts. 
Solutions such as described in [EIP-1077](https://eips.ethereum.org/EIPS/eip-1077) could allow transactions from addresses that hold no ETH.

The gas stations network is an [EIP-1077](https://eips.ethereum.org/EIPS/eip-1077) compliant effort to solve the problem by creating an incentive for nodes to run gas stations, where gasless transactions can be "fueled up". 
It abstracts the implementation details from both the dapp maintainer and the user, making it easy to convert existing dapps to accept "collect-calls".

The network consists of a single public contract trusted by all participating dapp contracts, and a decentralized network of relay nodes (gas stations) incentivized to listen on non-ether interfaces such as web or whisper, 
pay for transactions and get compensated by that contract. The trusted contract can be verified by anyone, and the system is otherwise trustless. 
Gas stations cannot censor transactions as long as there's at least one honest gas station. Attempts to undermine the system can be proven on-chain and offenders can be penalized.

## Motivation

* Increase user adoption of smart contracts by:
    * Removing the user hassle of acquiring ETH. Transactions are still paid by ETH but costs can be borne by a 3rd-party
      contract (Paymaster).
      Paymasters may implement logic to verify the user's eligibility - or even charge him through other means.
    * Removing the need to interact directly with the blockchain, while maintaining decentralization and censorship-resistance. 
      Contracts can "listen" on multiple public channels, and users can interact with the contracts through common protocols that are generally permitted even in restrictive environments.
* Ethereum nodes get a revenue source without requiring mining equipment. The entire network benefits from having more nodes.
* No protocol changes required. The gas station network is self-organized via smart contracts, and dapps interact with the network by implementing an interface.


## Design Guidelines

The system was built with decentralization and "mutual distrust" in its nature. 
We try to minimize the trust each component has on any other to limit the attack surface.
In particular
* No component should be able to perform an operation on another's behalf (e.g. a client must sign each request, and
    a RelayRecipient contract can be assured a request was actually signed by the client)
* No component should be able to steal money from other component: e.g., relayer gets paid only if it is guaranteed it
    delivered a service
* No component should be able to "grief" (cause monetarial damage) to other component.

## Definitions

* `Client` or `Sender` - an external account that attempt to make a call, without paying for gas
* `RelayRecipient` - the contract that receives the relayed call from the client.
* `Relayer` - a server that relays the request from the client to the recipient. 
    The relayer pays for the gas, but ultimately gets reimbursed for it (with a fee) from a Paymaster.
* `Paymaster` - a contract that pays for the actual transaction to the Relayer.
* `RelayHub` - a singleton that mediates the calls.
* `Forwarder` - a component to validate user's request, by checking the signature and nonce of a request.


### Internal Components

* `StakeManager` - a component of the RelayHub that holds the relayers stakes
* `Penalizer` - a component of the RelayHub that validates relayer's request - and can slash its stake if it is proven
    the relayer attempted a fraud.

The relayer is built from two (or more) distinct accounts:
* `RelayManager` - the account the "defines" the relayer (and hold a stake)    
* `RelayWorker` - actual relayer address that sends a request. a relayer has one or more workers    
## Specification

Roles of the `Forwarder`:
* validate the signature of the caller.
* validate the nonce in the request is unique
* call the target method on that target RelayRecipient.
* update the nonce, to prevent replay of the same request.

NOTE: the Forwarder is standalone generic contract, that can be used with different relaying frameworks - e.g multiple versions
of GSN or other networks. It is further defined in EIP-xxxxx

Roles of the `RelayHub`:

* Maintain a list of active relays. Senders select a `Relayer` from this list for each transaction. The selection process is discussed below.
* Mediate all communication between relayers and contracts.
* Provide contracts with trusted versions of the real msg.sender and msg.data.
* Validate a relay has a stake (with a minimum value and timeout) in the StakeManager.
* Hold ETH balance of Paymasters and use them to compensate relays.
* Using the Penalizer, penalize provably-offensive relays by giving their stakes to an address providing the proof, thus keeping relays honest.
* Provide a free way for relays to know whether they'll be compensated for a future transaction.

Roles of the `StakeManager`:

* hold stake for an address, with an unstake delay.
* only allow to remove the stake after the unstake delay.
* Provide the RelayHub with the stake status (value and time) for a given address.

Roles of a `Relayer` node:

* For each worker address, maintain a hot wallet with a small amount of ETH, to pay for gas.
* Provide a public interface for user apps to send gasless transactions via channels such as https or whisper.
* Publish it's public interfaces and its fees (as a base fee and multiplier of the actual transaction gas cost) in `RelayHub`.
* Optionally monitor reverted transactions of other relays through RelayHub, catching offending relays and claiming their stakes. This can be done by anyone, not just a relay.

Implementing a `RelayRecipient` contract:

* Know the address of `Forwarder` and trust it to provide information about the transaction.
* Use `_msgSender()` and `_msgData()` instead of `msg.sender` and `msg.data`, everywhere. `RelayRecipient` provides these functions and gets the information from `Forwarder`.

Implementing a `Paymaster`

* Maintain a small balance of ETH gas prepayment deposit in `RelayHub`. Can be paid directly by the `RelayRecipient` contract, or by the dapp's owner on behalf of the `RelayRecipient` address. 
  The Paymaster owner is responsible for ensuring sufficient balance for the next transactions, and can stop depositing if something goes wrong, thus limiting the potential for abuse of system bugs. In DAO usecases it will be up to the DAO logic to maintain a sufficient deposit.
* Implement a `preRelayedCall()` function to revert if it is NOT willing to accept a transaction and pay for it. 
  `preRelayedCall` is called by `RelayHub` before making the actual relayed call.
  Some examples of `preRelayedCall()` implementations:
    * Whitelist of trusted members.
    * Balance sheet of registered users, maintained by the paymaster owner. Users pay the paymaster with a credit card or other non-ETH means, and are credited in the `Paymaster` balance sheet. 
      Users can never cost the paymaster more than they were credited for.
    * A paymaster can provide off-chain a signed message called `approval` to a transaction sender and validate it.
    * Whitelist of known transactions used for onboarding new users. This allows certain anonymous calls and is subject to Sybil attacks.
      see below for various mitigation schemes.
  
* Implement `postRelayedCall()`. This method is called after a transaction is relayed. By default, it does nothing.
  
  These two methods can be used to charge the user in paymaster-specific manner.
  (e.g. transfer some tokens from the user in the `preRelayedCall`, and refund the excess in the `postRelayedCall`) 


![Sequence Diagram](https://www.websequencediagrams.com/cgi-bin/cdraw?lz=dGl0bGUgR1NOCnBhcnRpY2lwYW50IGNsaWVudAAGDVJlbGF5XG5Pd25lciBhcyBvd25lcgAQElNlcnYAGwZyZWxheQAxEkh1YiBhcyBodWIAZQ1TdGFrZU1hbmFnAFkGc20Kb3B0IHNldHVwAEQHCgoAbwUtPnNtOiAxLiBzdGFrZUZvckFkZHJlc3MoAGwFKQAZDDIuIGF1dGhvcml6ZUh1YkJ5AIE5BShodWIAHglodWI6IDMuIHJlZ2lzdGVyAIE6CwBJCGh1YgBuBjQuIGlzAIIDBQCBIgcAgTAFZABqDwBNBTUuIGFkZACCLwVXb3JrZXIAgRgHLHcACQUpCmVuZAoAgV0FbWFraW5nIGEgY2FsbABnBS0-AIJ5BjogNi4gZXZlbnQ6AD4LQWRkZWQKCgCDHAYtPgCCYQU6AIJnBkNhbGwKAIJyBQCBUQc3LgAOCwCBPgk4AIEpHgCBagVwACIGIHByZQCDdAVlZAA6CmZvcndhcmRlcgBGBWV4ZWN1dGUKbm90ZSBvdmVyIAASDnZlcmlmeVNpZ1xuAAUGTm9uY2UKADwJLT5yZWNpcACBawY4LiB0YXJnZXQtAIIJBWFjdGl2YXRlIAAaCQCBEQ1wb3N0AIESDGRlABoXAIJIClRyYW5zYWN0aW9uAIFRBwCDBAY&s=rose)

The process of registering/refreshing a `Relayer`:

* Relayer starts listening as a web app (or on some other communication channel).
* If starting for the first time (no key yet), generate a key pair for Relayer's **manager** and **worker** addresses.
* The Relayer's owner should fund the manager address with some eth.
* Relayer's owner sends the required stake to `StakeManager` by calling `StakeManager.stakeForAddress(address relay, uint unstakeDelay)`.
* Relayer's owner calls `StakeManager.authorizeHubByOwner(relayManager,relayHub)`
* Relay calls `RelayHub.registerRelayServer(baseFee, pctFee, url)` with the relayer's configured `transaction fee` (as a base fee and a multiplier on transaction gas cost), and a URL for incoming transactions. 
* `RelayHub` ensures that Relayer has an authorization and sufficient stake in the `StakeManager`.
* `RelayHub` emits an event, `RelayAdded(Relay, owner, transactionFee, relayStake, unstakeDelay, url)`.
* `Relayer` goes to sleep and waits for signing requests.

The process of winding a `Relayer` down:

* Relayer's owner (the address that initially funded it) calls `StakeManager.unlockStake(relayManager)`.
* The StakeManager validates its the actual owner, and it does have a stake.
* From that point, no transaction can go through this relayer.
* Once the owner's unstake delay is over, owner calls `RelayHub.unstake()`, and withdraws the stake.
* The relayer should stop the `Relay` shuts down.

The process of moving `Relayer` to a new RelayHub:

* The same RelayManager (but not relay workers!) can be used by multiple Hubs (e.g, when switching to a new hub version, and relay owner
  would like to give service through both the old hub and new hub, without requiring to add new stake.
* the Relayer is brought up with the new RelayHub (with the registering process above, but without re-staking)
* When the old RelayHub is no longer in use, the owner should call `StakeManager.unauthorizeHubByOwner(relayManager,relayHub)`
* The StakeManager will remove the authorization, and the relayManager will no longer be able to process requests by this RelayHub.
* When receiving the "HubUnauthorized" events, the Relayer will start a countdown of the unstake-delay.
* Only after this countdown is over, it will transfer the eth balance of the relay workers to the manager (before the time, the "transfer eth" operation balance can cause the relayer to be penalizeable)
* The client lookup mechanism filters out relayers for which a "HubUnauthorized" event was sent

The process of sending a relayed transaction:

* `Sender` selects a live `Relay` from RelayHub's list by looking at events from `RelayHub` in the past `lookup window`.
   Each relayer address is saved once (that is, a relayer is considred "active" regardless of how many requests it had 
   processed within the lookup window)
* The sender may maintain a list of "preferred relayers". These are always moved to the beginning of the list and tried 
  first, before other relayers (i.e probably dapp-owned relayers)
* The sender may sort relayers based on its own criteria. Selection may be based on a mix of:
    * Relay published transaction fees.
    * Relay stake size and lock-up time.
    * Recent relay transactions (visible through `TransactionRelayed` events from `RelayHub`).
    * Optionally, reputation/blacklist/whitelist held by the sender app itself, or its backend, on per-app basis (not part of the gas stations network).
* The Default sorting algorithm is:
    * preferred relays are always first, in their order from the configuration file.
    * sort relayers by calculated fee (for this transaction) 
    * downscore (move to the end of the list) relays that failed this client (see below)
* To select a relayer to use from the list, the client:
    * Pick the next 3 relayers (without mixing "preferred" with normal relayers)
    * send a "ping" request to all of them.
    * Verify the RelayManager and RelayHub in that ping response
    * save the RelayWorker address (used below)
    * Take the first relayer to answer the ping.
* Sender prepares the transaction with Sender's address, the recipient address, the actual transaction data, Relay's transaction fee, gas price, gas limit, its current nonce from `Forwarder.nonces`, RelayHub's address, and Relay-worker's address, and then signs it.
* The Sender makes a view-call to `RelayHub.relayCall()`, to make sure the call is valid, and paymaster agrees to pay.
* The Sender may also make the explicit verifications:
    * `RelayHub.balances[forwarder]` holds enough ETH to pay Relay's fee.
    * `RelayWorkder.balance` has enough eth to send the transaction
* Sender reads the Relay worker's current `nonce` value and decides on the `max_nonce` parameter.
* Sender sends the signed transaction and metadata to Relayer's web interface.
* `Relayer` receives the request, and verify the client's request:
    * the required gasPrice is acceptable by the relayer (too-low gas price might stall this worker for some time)
    * the next nonce to use is within "nonce_gap" from the nonce of the last mined transaction of this worker.
* `Relayer` receives the request, and make a view call to `RelayHub.relayCall()`, to verify it will get compensated:
* The `RelayHub.relayCall` validates (and reverts otherwise) that:
    * the relay manager is registered and staked.
    * the sender of the request is indeed a relayWorker of that manager.
    * the paymaster can pay for the max possible price.
    * the relayhub provided enough gas to complete the request.
    * the paymaster's preRelayedCall accepted the call (and didn't revert)
* If any of Relayer's checks fail, it returns an error to sender, and doesn't proceed.
* Relayer submits the signed relayCall transaction to the blockchain.
* Relayer immediately returns the signed relayCall transaction to the sender.  This step is discussed below, in attacks/mitigations.
* `Sender` receives the wrapped transaction and verifies that:
    * It's a valid call to `RelayHub.relayCall`, with its own transaction, from RelayWorker's address.
    * The transaction's ethereum nonce is within "nonce_gap" from RelayWorker's current nonce.
    * The `RelayWorker` is sufficiently funded to pay for it.
* If any of sender's checks fails, it goes back to selecting the next relayer.
  * As discussed above, this failed relayer is now downscore'd by this sender, so that it will avoid to select it again in future transaction.
* `Sender` may also submit the raw wrapped transaction to the blockchain without paying for gas, through any Ethereum node. 
  This submission is likely ignored because an identical transaction is already in the network's pending transactions, but no harm in putting it twice, to ensure that it happens. 
  This step is not strictly necessary, for reasons discussed below in attacks/mitigations, but may speed things up.
* `Sender` monitors the blockchain, waiting for the transaction to be mined. 
  The transaction was verified, with Relayer's current nonce, so mining must be successful unless Relayer submitted another (different) transaction with the same nonce. 
* If mining fails due to such attack, sender may call `RelayHub.penalizeRepeatedNonce` through another relay, to collect his reward and burn the remainder of the offending relay's stake, and then go back to selecting a new Relayer for the transaction. 
  See discussion in the attacks/mitigations section below.
* `RelayHub` receives the transaction:
    * Verifies the transaction is sent from a worker of a registered relay, and that's the same worker signed in the transaction.
    * Verifies the tx.gasPrice is at least as the request gasPrice
    * Verifies the given acceptanceBudget is above paymaster's acceptanceBudget (since #AcceptanceBudget section
    * Verifies the transaction is given enough gas to cover all gas limits (preRelayedCall, actual call and postRelayedCall)
    * Verifies the paymaster can pay for the max possible price of the transaction
    * Makes a call to `paymaster.preRelayedCall`, to make sure it accepts the request
      * the paymaster makes sure it trusts the forwarder of the request.
      * the paymaster then performs its other validations.
    * In case the paymaster reverts, a "TransactionRejectedByPaymaster" is emitted.
    * The RelayHub sends the transaction to the recipient through the forwarder.
      The forwarder verifies the nonce and signature, and then call the target recipient.
      When passing gas to `Forwrader.execute()`, enough gas is preserved by `RelayHub`, for post-call handling. Recipient may run out of gas, but `RelayHub` never does. 
    * The `Forwarder` verifies the signature of the sender, and that the nonce is correct, and increment the nonce.
    * The `Forwarder` makes a call to the recipient, after appending the sender's address at the end of the msg.data. This way, the recipient is assumed that when it receives a call through the Forwarder, it knows for sure the `msg.sender` is passed as the last 20 bytes of the **msg.data**.
    * Recipient contract handles the transaction.
    * `RelayHub` calls paymaster's `postRelayedCall`.
    * `RelayHub` checks call's return value of call, and emits `TransactionRelayed(address relay, address from, address to, bytes4 selector, uint256 status, uint256 chargeOrCanRelayStatus)`.
    * `RelayHub` transfers ETH balance from paymaster to `RelayManager`, to pay the transaction fee, based on the measured transaction cost. 

* `Relayer` keeps track of sent transactions. If the transaction doesn't get mined, it should boost it by putting a higher gas price.
  Any other modification to the transaction is not allowed, and is considered a violation of the protocol, for which the relayer will get penalized.
  Note that without boosting it, the relayWorker is blocked, and can't send other transactions.

### Handling of "unstable" transaction

The GSN protocol is based on an assumption of the client (and then of the relayer) that running a transaction in view mode will get the same result as running it on-chain.
The relayer trusts that if the view call succeeds, its OK to put it on-chain and it will refund the relayer for the transaction cost.
However, it is possible to write contracts (paymaster or recipient) that will cause a transaction to be "unstable", and thus succeed off-chain and later fail on-chain. In such a case, the relayer will suffer a loss of funds.

Examples of such "unstable" cases (but definitely not the only cases):
1. require(block.number % 1 == 0) - this naive code will "randomly" fail.
2. parallel sending the same transaction to multiple relays at the same time: obviously, one will succeed, and the other will fail (since the sender's nonce in the forwarder was modified by the first call) 

Note that in either case, the attack is not "free": in the first case, there is a change that it gets through anyway. In the latter case, one transaction gets paid, so the attack "cost" is higher than the actual damage to each relayer (though in total, it can be higher)

The GSN protocol contains several mechanisms to mitigate such cases:
1. **AcceptanceBudget**: For any given transaction, the Relayer risks at most "acceptanceBudget" of gas, and it is guaranteed to be refunded for paymaster rejects with higher gas usage. The Paymaster advertises (through Paymaster.getLimits()) its acceptanceBudget. The default of 150k is enough for most but extreme cases (such as zero-knowledge-based mixers)
Note that by "paymaster reject" we count both preRelayedCall's gas usage AND forwarder's check for signature and nonce. Since forwarder's checks take \~30kgas, the default preRelayedCallGasLimit was set to 100k, so that both are below the 150kgas acceptanceBudget.

Note that if the Relayer's owner knows the Paymaster code and trust it to be "stable" (which also means the trusted Forwrader is "stable"), then it can allow for higher acceptanceBudget


2. **AlertingMode**: this is a mechanism to help the relayer mitigate the 2nd unstable case, above: When the relayer detects an event "TransactionRejectedByPaymasterEvent", it knows it is probably because of parallel sending of such transaction. In alerted mode, the relayer waits a random time before sending the transaction. This way, there is a chance that the other relayer that was given this transaction will send it first, and this relayer will be able to see it (on-chain or in the mempool), and avoid re-sending the same TX.


## Types of Paymasters

A Paymaster is the on-chain entity that verifies the request and ultimately pays for the transaction.
One important aspect of the paymaster is that it has to block greifing attempts.

We see several types of Paymasters:
1. Test-Only EverythingAccepted - the most naive paymaster, which simply accept requests. It can only be used on testnets, as it will
  be immediately grief by wily hackers.

2. Whitelisting - In case the senders are known, the paymaster can whitelist the sender addresses it accepts. This is good if you already have an on-chain repository of known addresses (e.g, a DAO may pay for voting by the DAO participants)

3. Off-chain verification - in many cases, an off-chain mechanism for user verification can be used, such as OUath, Email, SMS, Captcha, etc. In this case, the off-chain service will perform a verification, and sign the request. The on-chain Paymaster will receive that signature as `approvalData`. The only thing the on-chain paymaster need is to know what signer addresses it supports.

4. Off-chain payment - a special case of the off-chain verification is "pay for gas": use a payment service (such as stripe or paypal), and sign the request, to be checked by the paymaster.

5. Pay-with-token - a paymaster that manages the balance of a user, and deduct the transation cost. While the user doesn't need to have ETH, he does need to have the right token, which means need to set an approval for the paymaster to manage it.


## Rationale
The rationale for the gas stations network design is a combination of two sets of requirements: Easy adoption, and robustness.

For easy adoption, the design goals are:

* No network changes.
* Minimal changes to contracts, apps and frameworks.

The robustness requirement translates to decentralization and attack resistance. The gas stations network is decentralized, and we have to assume that any entity may attack other entities in the system.

Specifically we've considered the following types of attacks:

* Denial-of-service attacks against individual senders, i.e. transactions censorship.
* Denial-of-service and financial attacks against individual relays.
* Denial-of-service and financial attacks against individual contracts.
* Denial-of-service attacks against the entire network, either by attacking existing entities, or by introducing any number of malicious entities.

#### Attacks and mitigations

##### Attack: Relay attempts to censor a transaction by not signing it, or otherwise ignoring a user request.
Relay is expected to return the signed transaction to the sender, immediately. 
Sender doesn't need to wait for the transaction to be mined, and knows immediately whether it's request has been served. 
If a relay doesn't return a signed transaction within few seconds, sender cancels the operation, drops the connection, and switches to another relay. 
It also marks Relay as unresponsive in its private storage to avoid using it in the near future.

Therefore, the maximal damage a relay can cause with such attack, is a one-time delay of few seconds. After a while, senders will avoid it altogether.

##### Attack: Relay attempts to censor a transaction by signing it, returning it to the sender, but never putting it on the blockchain.
This attack will backfire and not censor the transaction. 
The sender can submit the transaction signed by Relay to the blockchain as a raw transaction through any node, so the transaction does happen, 
but Relay may be unaware and therefore be stuck with a bad nonce which will break its next transaction.

##### Attack: Relay attempts to censor a transaction by signing it, but publishing a different transaction with the same nonce.
Reusing the nonce is the only DoS performed by a Relay, that cannot be detected within few seconds during the http request. 
It will only be detected when the malicious transaction with the same nonce gets mined and triggers the `RelayHub.TransactionRelayed` event. 
However, the attack will backfire and cost Relay its entire stake.

Sender has a signed transaction from Relay with nonce N, and also gets a mined transaction from the blockchain with nonce N, also signed by Relay. 
This proves that Relay performed a DoS attack against the sender. 
The sender calls `RelayHub.penalizeRepeatedNonce(bytes transaction1, bytes transaction2)`, which verifies the attack, confiscates Relay's stake, 
and sends half of it to the sender who delivered the `penalizeRepeatedNonce` call. The other half of the stake is burned by sending it to `address(0)`. Burning is done to prevent cheating relays from effectively penalizing themselves and getting away without any loss.
The sender then proceeds to select a new relay and send the original transaction.

The result of such attack is a delay of a few blocks in sending the transaction (until the attack is detected) but the relay gets removed and loses its entire stake. 
Scaling such attack would be prohibitively expensive, and actually quite profitable for senders and honest relays.

##### Attack: Relay attempts to censor a transaction by signing it, but using a nonce higher than it's current nonce.
In this attack, the Relay did create and return a perfectly valid transaction, but it will not be mined until this Relay fills the gap in the nonce with 'missing' transactions.
This may delay the relaying of some transactions indefinitely. In order to mitigate that, the sender includes a `max_nonce` parameter with it's signing request.
It is suggested to be higher by 2-3 from current nonce, to allow the relay process several transactions.

When the sender receives a transaction signed by a Relay he validates that the nonce used is valid, and if it is not, the client will ignore the given relay and use other relays to relay given transaction. Therefore, there will be no actual delay introduced by such attack.

##### Attack: Paymaster attempts to burn relays funds by implementing an inconsistent preRelayedCall() and using multiple sender addresses to generate expensive transactions, thus performing a DoS attack on relays and reducing their profitability.
In this attack, a contract sets an inconsistent preRelayedCall (e.g. revert on even blocks), and uses it to exhaust relay resources through unpaid transactions. 
Relays can easily detect it after the fact. 
If a transaction goes unpaid, the relay knows that the Paymaster contract's preRelayedCall has acted inconsistently, because the relay has verified its view function before sending the transaction. 
It might be the result of a rare race condition where the contract's state has changed between the view call and the transaction, but if it happens too frequently, relays will blacklist this paymaster and refuse to serve transactions to it. 
Each offending paymaster can only cause a small damage (e.g. the cost of 2-3 transactions) to a relay, before getting blacklisted.

Relays may also look at paymaster's history on the blockchain, looking for past unpaid transactions (reverted by RelayHub without pay), and denying service to contracts with a high failure rate. 
If a contract caused this minor loss to a few relays, all relays will stop serving it, so it can't cause further damage.

This attack doesn't scale because the cost of creating a malicious contract is in the same order of magnitude as the damage it can cause to the network. 
Causing enough damage to exhaust the resources of all relays, would be prohibitively expensive.

The attack can be made even more impractical by setting RelayHub to require a stake from paymasters before they can be served, and enforcing an unstaking delay, 
so that attackers will have to raise a vast amount of ETH in order to simultaneously create enough malicious contracts and attack relays. 

This protection is probably an overkill, since the attack doesn't scale regardless.

##### Attack: User attempts to rob dapps by registering its own relay and sending expensive transactions to dapps.
If a malicious sender repeatedly abuses a paymaster by sending meaningless/reverted transactions and causing the paymaster to pay a relay for nothing, 
it is the paymaster's responsibility to blacklist that sender and have its preRelayedCall function revert for that sender. 
Collect calls are generally not meant for anonymous senders unknown to the recipient. 
Dapps that utilize the gas station networks should have a way to blacklist malicious users in their system and prevent Sybil attacks.

A simple method that mitigates such Sybil attack, is that the dapp lets users buy credit with a credit card, and credit their account in the dapp contract, 
so preRelayedCall() only succeeds for users that have enough credit, and deduct the amount paid to the relay from the user's balance, whenever a transaction is relayed for the user. 
With this method, the attacker can only burn its own resources, not the dapp's.

A variation of this method, for free dapps (that don't charge the user, and prefer to pay for their users transactions) is to require a captcha during user creation in their web interface, 
or to login with a Google/Facebook account, which limits the rate of the attack to the attacker's ability to open many Google/Facebook accounts. 
Only a user that passed that process is given credit in RelayRecipient. The rate of such Sybil attack would be too low to cause any real damage.

##### Attack: Attacker attempts to reduce network availability by registering many unreliable relays.
Registering a relay requires placing a stake in RelayHub, and the stake can only be withdrawn after the relay is unregistered and a long cooldown period has passed, e.g. a month.

Each unreliable relay can only cause a couple of seconds delay to senders, once, and then it gets blacklisted by them, as described in the first attack above. 
After it caused this minor delay and got blacklisted, the attacker must wait a month before reusing the funds to launch another unreliable relay. 
Simultaneously bringing up a number of unreliable relays, large enough to cause a noticeable network delay, would be prohibitively expensive due to the required stake, 
and even then, all those relays will get blacklisted within a short time.

##### Attack: Attacker attempts to replay a relayed transaction.
Transactions include a nonce. RelayHub maintains a nonce (counter) for each sender. Transactions with bad nonces get reverted by RelayHub. Each transaction can only be relayed once.

##### Attack: User does not execute the raw transaction received from the Relayer, therefore blocking the execution of all further transactions signed by this relayer
The user doesn't really have to execute the raw transaction. It's enough that the user can. The relationship between relay and sender is mutual distrust. The process described above incentivizes the relay to execute the transaction, so the user doesn't need to wait for actual mining to know that the transaction has been executed.

Once relay returns the signed transaction, which should happen immediately, the relay is incentivized to also execute it on chain, so that it can advance its nonce and serve the next transaction. The user can (but doesn't have to) also execute the transaction. To understand why the attack isn't viable, consider the four possible scenarios after the signed transaction was returned to the sender:

1. Relay executes the transaction, and the user doesn't. In this scenario the transaction is executed, so no problem. This is the case described in this attack.
2. Relay doesn't execute the transaction, but the user does. Similarly to 1, the transaction is executed, so no problem.
3. Both of them execute the transaction. The transactions are identical in the pending transactions pool, so the transaction gets executed once. No problem.
4. None of them execute the transaction. In this case the transaction doesn't get executed, but the relay is stuck. It can't serve the next transaction with the next nonce, because its nonce hasn't been advanced on-chain. It also can't serve the next transaction with the current nonce, as this can be proven by the user, having two different transactions signed by the same relay, with the same nonce. The user could use this to take the relay's nonce. So the relay is stuck unless it executes the transaction.

As this matrix shows, the relay is __always__ incentivized to execute the transaction, once it returned it to the user, in order to end up in #1 or #3, and avoid the risk of #4. It's just a way to commit the relay to do its work, without requiring the user to wait for on-chain confirmation.

## Backwards Compatibility

The gas stations network is implemented as smart contracts and external entities, and does not require any network changes.

Dapps adding gas station network support remain backwards compatible with their existing apps/users. The added methods apply on top of the existing ones, so no changes are required for existing apps.

## Implementation

A working implementation of the [**gas stations network**](https://github.com/opengsn/gsn) is being developed by **OpenGsn**. 

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
