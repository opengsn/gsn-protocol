---
eip: 
title: Gas Stations Network v2
author: Yoav Weiss <yoav@opengsn.org>, Dror Tirosh <dror@opengsn.org>, Alex Forshtat <alex@opengsn.org>
status: Draft
type: Standards Track
category: ERC
created: 2020-09-16
---

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

* `Client` - an etherless account that attempt to make a call, without paying for gas
* `RelayRecipient` - the contract that receives the relayed call from the client.
* `Relayer` - a server that relays the request from the client to the recipient. 
    The relayer pays for the gas, but ultimately gets reimbursed for it (with a fee)
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

Glossary of terms used in the processes below:

* `RelayHub` - the RelayHub singleton contract, used by everyone.
* `Recipient` - a contract implementing `RelayRecipient`, accepting relayed transactions from the RelayHub contract and paying for the incoming transactions.
* `Sender` - an external address with a valid key pair but no ETH to pay for gas.
* `Relayer` - a node holding ETH in an external address, listed in RelayHub and relaying transactions from Senders to RelayHub for a fee.

![Sequence Diagram](/assets/eip-1613/sequence.png)

The process of registering/refreshing a `Relayer`:

* Relayer starts listening as a web app (or on some other communication channel).
* If starting for the first time (no key yet), generate a key pair for Relayer's **manager** and **worker** addresss.
* The Relayer's owner should fund the manager address with some eth.
* Relayer's owner sends the required stake to `StakeManager` by calling `StakeManager.stakeForAddress(address relay, uint unstakeDelay)`.
* Relayer's owner should call `StakeManager.authorizeHubByOwner`
* `RelayHub` verifies the relayer has a stake and authorization in the StakeManager.
* Relay calls `RelayHub.registerRelayServer(baseFee, pctFee, url)` with the relay's `transaction fee` (as a base fee and a multiplier on transaction gas cost), and a URL for incoming transactions. 
* `RelayHub` ensures that Relay has a sufficient stake in the `StakeManager.
* `RelayHub` emits an event, `RelayAdded(Relay, owner, transactionFee, relayStake, unstakeDelay, url)`.
* `Relayer` goes to sleep and waits for signing requests.

The process of sending a relayed transaction:

* `Sender` selects a live `Relay` from RelayHub's list by looking at events from `RelayHub` in the past `lookup window`.
   Each relayer address is saved once.
* The sender may sort relayers based on its own criteria. Selection may be based on a mix of:
    * Relay published transaction fees.
    * Relay stake size and lock-up time.
    * Recent relay transactions (visible through `TransactionRelayed` events from `RelayHub`).
    * Optionally, reputation/blacklist/whitelist held by the sender app itself, or its backend, on per-app basis (not part of the gas stations network).
* Sender prepares the transaction with Sender's address, the recipient address, the actual transaction data, Relay's transaction fee, gas price, gas limit, its current nonce from `RelayHub.nonces`, RelayHub's address, and Relay's address, and then signs it.
* Sender verifies that `RelayHub.balances[recipient]` holds enough ETH to pay Relay's fee.
* Sender verifies that `Relay.balance` has enough eth to send the transaction
* Sender reads the Relay's current `nonce` value and decides on the `max_nonce` parameter.
* Sender sends the signed transaction amd metadata to Relay's web interface.
* `Relay` wraps the transaction with a transaction to `RelayHub`, with zero ETH value.
* `Relay` signs the wrapper transaction with its key in order to pay for gas.
* `Relay` verifies that:
    * The transaction's recipient contract will accept this transaction when submitted, by calling `RelayHub.canRelay()`, a view function, 
      which checks the recipient's `acceptRelayedCall`, also a view function, stating whether it's willing to accept the charges).
    * The transaction nonce matches `RelayHub.nonces[sender]`.
    * The relay address in the transaction matches Relay's address.
    * The transaction's recipient has enough ETH deposited in `RelayHub` to pay the transaction fee.
    * Relay has enough ETH to pay for the gas required by the transaction.
    * Value of `max_nonce` is higher than current Relay's `nonce`
* If any of Relay's checks fail, it returns an error to sender, and doesn't proceed.
* Relay submits the signed wrapped transaction to the blockchain.
* Relay immediately returns the signed wrapped transaction to the sender.  This step is discussed below, in attacks/mitigations.
* `Sender` receives the wrapped transaction and verifies that:
    * It's a valid relay call to `RelayHub`. from Relay's address.
    * The transaction's ethereum nonce matches Relay's current nonce.
    * The transaction's ethereum nonce is lower than or equal to `max_nonce`.
    * `Relay` is sufficiently funded to pay for it.
    * The wrapped transaction is valid and signed by `sender`.
    * Recipient contract has sufficient funds in `RelayHub.balances` to pay for Relay's fee as stated in the transaction.
* If any of sender's checks fails, it goes back to selecting a new Relay. Sender may also file a report on the unresponsive relay to its backend or save it locally, to down-sort this relay in future transactions.
* `Sender` may also submit the raw wrapped transaction to the blockchain without paying for gas, through any Ethereum node. 
  This submission is likely ignored because an identical transaction is already in the network's pending transactions, but no harm in putting it twice, to ensure that it happens. 
  This step is not strictly necessary, for reasons discussed below in attacks/mitigations, but may speed things up.
* `Sender` monitors the blockchain, waiting for the transaction to be mined. 
  The transaction was verified, with Relay's current nonce, so mining must be successful unless Relay submitted another (different) transaction with the same nonce. 
  If mining fails due to such attack, sender may call `RelayHub.penalizeRepeatedNonce` through another relay, to collect his reward and burn the remainder of the offending relay's stake, and then go back to selecting a new Relay for the transaction. 
  See discussion in the attacks/mitigations section below.
* `RelayHub` receives the transaction:
    * Records `gasleft()` as `initialGas` for later payment.
    * Verifies the transaction is sent from a registered relay.
    * Verifies that the signature of the internal transaction matches its stated origin (sender's key).
    * Verifies that the relay address written in the transaction matches msg.sender.
    * Verifies that the transaction's `nonce` matches the stated origin's nonce in `RelayHub.nonces`.
    * Calls recipient's `acceptRelayedCall` function, asking whether it's going to accept the transaction. If not, the `TransactionRelayed` will be emitted with status `CanRelayFailed`, and `chargeOrCanRelayStatus` will contain the return value of `acceptRelayedCall`. In this case, Relay doesn't get paid, as it was its responsibility to check `RelayHub.canRelay` before releasing the transaction.
    * Calls recipient's `preRelayedCall` function. If this call reverts the `TransactionRelayed` will be emitted with status `PreRelayedFailed`.
    * Sends the transaction to the recipient.  If this call reverts the `TransactionRelayed` will be emitted with status `RelayedCallFailed`.
      When passing gas to `call()`, enough gas is preserved by `RelayHub`, for post-call handling. Recipient may run out of gas, but `RelayHub` never does. 
      `RelayHub` also sends sender's address at the end of `msg.data`, so `RelayRecipient.getSender()` will be able to extract the real sender, and trust it because the transaction came from the known `RelayHub` address.
* Recipient contract handles the transaction.
* `RelayHub` calls recipient's `postRelayedCall`.
* `RelayHub` checks call's return value of call, and emits `TransactionRelayed(address relay, address from, address to, bytes4 selector, uint256 status, uint256 chargeOrCanRelayStatus)`.
* `RelayHub` increases `RelayHub.nonces[sender]`.
* `RelayHub` transfers ETH balance from recipient to `Relay.owner`, to pay the transaction fee, based on the measured transaction cost. 
  Note on relay payment: The relay gets paid for actual gas used, regardless of whether the recipient reverted. 
  The only case where the relay sustains a loss, is if `canRelay` returns non-zero, since the relay was responsible to verify this view function prior to submitting. 
  Any other revert is caught and paid for. See attacks/mitigations below.
* `Relay` keeps track of transactions it sent, and waits for `TransactionRelayed` events to see the charge. 
  If a transaction reverts and goes unpaid, which means the recipient's `acceptRelayedCall()` function was inconsistent, `Relay` refuses service to that recipient for a while (or blacklists it indefinitely, if it happens often). 
  See attacks/mitigations below.

The process of winding a `Relay` down:

* Relay's owner (the address that initially funded it) calls `RelayHub.removeRelayByOwner(Relay)`.
* `RelayHub` ensures that the sender is indeed Relay's owner, then removes `Relay`, and emits `RelayRemoved(Relay)`.
* `RelayHub` starts the countdown towards releasing the owner's stake.
* `Relay` receives its `RelayRemoved` event.
* `Relay` sends all its remaining ETH to its owner.
* `Relay` shuts down.
* Once the owner's unstake delay is over, owner calls `RelayHub.unstake()`, and withdraws the stake.

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
If a relay doesn't return a signed transaction within a couple of seconds, sender cancels the operation, drops the connection, and switches to another relay. 
It also marks Relay as unresponsive in its private storage to avoid using it in the near future.

Therefore, the maximal damage a relay can cause with such attack, is a one-time delay of a couple of seconds. After a while, senders will avoid it altogether.

##### Attack: Relay attempts to censor a transaction by signing it, returning it to the sender, but never putting it on the blockchain.
This attack will backfire and not censor the transaction. 
The sender can submit the transaction signed by Relay to the blockchain as a raw transaction through any node, so the transaction does happen, 
but Relay may be unaware and therefore be stuck with a bad nonce which will break its next transaction.

##### Attack: Relay attempts to censor a transaction by signing it, but publishing a different transaction with the same nonce.
Reusing the nonce is the only DoS performed by a Relay, that cannot be detected within a couple of seconds during the http request. 
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

##### Attack: Dapp attempts to burn relays funds by implementing an inconsistent acceptRelayedCall() and using multiple sender addresses to generate expensive transactions, thus performing a DoS attack on relays and reducing their profitability.
In this attack, a contract sets an inconsistent acceptRelayedCall (e.g. return zero for even blocks, nonzero for odd blocks), and uses it to exhaust relay resources through unpaid transactions. 
Relays can easily detect it after the fact. 
If a transaction goes unpaid, the relay knows that the recipient contract's acceptRelayedCall has acted inconsistently, because the relay has verified its view function before sending the transaction. 
It might be the result of a rare race condition where the contract's state has changed between the view call and the transaction, but if it happens too frequently, relays will blacklist this contract and refuse to serve transactions to it. 
Each offending contract can only cause a small damage (e.g. the cost of 2-3 transactions) to a relay, before getting blacklisted.

Relays may also look at recipients' history on the blockchain, looking for past unpaid transactions (reverted by RelayHub without pay), and denying service to contracts with a high failure rate. 
If a contract caused this minor loss to a few relays, all relays will stop serving it, so it can't cause further damage.

This attack doesn't scale because the cost of creating a malicious contract is in the same order of magnitude as the damage it can cause to the network. 
Causing enough damage to exhaust the resources of all relays, would be prohibitively expensive.

The attack can be made even more impractical by setting RelayHub to require a stake from dapps before they can be served, and enforcing an unstaking delay, 
so that attackers will have to raise a vast amount of ETH in order to simultaneously create enough malicious contracts and attack relays. 
This protection is probably an overkill, since the attack doesn't scale regardless.

##### Attack: User attempts to rob dapps by registering its own relay and sending expensive transactions to dapps.
If a malicious sender repeatedly abuses a recipient by sending meaningless/reverted transactions and causing the recipient to pay a relay for nothing, 
it is the recipient's responsibility to blacklist that sender and have its acceptRelayedCall function return nonzero for that sender. 
Collect calls are generally not meant for anonymous senders unknown to the recipient. 
Dapps that utilize the gas station networks should have a way to blacklist malicious users in their system and prevent Sybil attacks.

A simple method that mitigates such Sybil attack, is that the dapp lets users buy credit with a credit card, and credit their account in the dapp contract, 
so acceptRelayedCall() only returns zero for users that have enough credit, and deduct the amount paid to the relay from the user's balance, whenever a transaction is relayed for the user. 
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

A working implementation of the [**gas stations network**](https://github.com/tabookey-dev/tabookey-gasless) is being developed by **TabooKey**. It consists of `RelayHub`, `RelayRecipient`, `web3 hooks`, an implementation of a gas station inside `geth`, and sample dapps using the gas stations network.

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
