# Summary

> This is a very preliminary version of the audit.

## Audited contracts

Subjects of this audit are the files from the folder [`contracts/src/2.0.0/`](https://github.com/0xProject/0x-monorepo/tree/v2-prototype/packages/contracts/src/2.0.0) 
in commit `55dbb0ece06d17a9db7b93a0ffa274ff65298002`.

> Current version of audit only includes `protocol` subfolder.

## Documentation

System details are described in [specifications](https://github.com/0xProject/0x-protocol-specification/blob/master/v2/v2-specification.md).

## System description

In general, the system is well designed and provides broad functionality. 
It's good in terms of upgradability and can be easily paused in case if system security is compromised.
Major code refactoring was made and lots of new features were added since v1.


# General security tradeoffs


## Critical

> Need to doublecheck it in practice    

### Re-entrancy attack on `executeTransaction` allows stealing all the funds

Using `currentContextAddress` is very dangerous technique. If an attacker is able to reach Exchange contract in victims context, the attacker can steal all the funds from the victim. In order to do anything on behalf of the victim, the attacker only needs to make a re-entrancy attack (with malicious token or validator) while someone is calling `executeTransaction` on behalf of the victim.

Example:
1. The attacker creates order and signs it with `SignatureType.Wallet` signature type.
1. If the taker will try to take this order using `executeTransaction`. This will overwrite the `currentContextAddress` to the taker address.
1. Attacker(maker) can make re-entrancy on `isValid` function and execute a bunch of `fillOrder`(fake order in attackers benefit) in the context of the victim.
1. All the funds of the victim are stolen (except for the first order purchase).

**Solution** : context concept should be modified significantly. (TBD)


## Major


### Order reuse attack

Currently `Exchange` contract contains storage with all filled and canceled orders.
This creates a possibility to reuse the same orders in a new version of `Exchange`.
Or if someone will create a clone of 0x system, orders will be indistinguishable.

**Note** : some problems might appear with indistinguishable orders from mainnet and test networks.

**Solution 1** : order should be bounded to `Exchange` contract address.

**Solution 2** : keep order information in the separate storage contract that can be reused in a new version of `Exchange` contract.

## Medium

### Trust issues

It's recommended by the protocol to make unlimited allowance to the `AssetProxy` contracts.
`AssetProxy` allows it's owner to steal all the allowed tokens by adding a new authorized contract.
This situation is mitigated by the 2-weeks delay on every decision that `AssetProxyOwner` contract can make.
The problem is that 2-weeks delay logic has been put outside `AssetProxy` contracts which makes it harder to control contracts owner.
In order to make sure that tokens are safe, a user needs to constantly keep track of the `AssetProxy` owner and its upgrades, `secondsTimeLocked` (2-weeks delay can be changed) and all the authorized contracts.
This complexity makes it hard to keep track of everything and creates a risk of missing some backdoor in 2-weeks term. 
Also, it takes time for every user to withdraw their allowance and it's possible to spam the network so not everyone will be able to withdraw in time.


**Solution 1** : create some limitations (like 2-weeks delay) on the `AssetProxy` contract side. 

**Solution 2 (only for takers)** : use forwarder contract for every trade. This can be safer and requires only one transaction, but it's more expensive.


### Malicious token re-entrancy

A token is not guaranteed to be valid and non-malicious. There is no `TokenRegistry` in v2 and no precheck of tokens. 
A maker can put any address to the `makerToken` or `takerToken` field and almost no validation occurs on 0x protocol side. 

**Note 1** : *One dangerous attack is related to the `executeTransaction` and was described before*

**Note 2** : malicious token can try to benefit from the inappropriate use of arbitrage functions. For example, if someone will use `batchFillOrders` or `batchFillOrdersNoThrow` for arbitrage purposes. 

> TBD: Attack details will be shown later.

**Solution** : it's possible to create a token registry, but it will make the system more regulated. I would suggest to keep the system the way it is, but keep in mind this trade off.

### Malicious validator/wallet-validator re-entrancy

It's possible to make a re-entrancy while validating a signature. It's possible to do with either `SignatureType.Wallet` or `SignatureType.Validator`. It can be used to front-run this transaction or execute some other transactions using victims gas.

**Note 1** : *One dangerous attack is related to the `executeTransaction` and was described before*

**Solution** : TDB.


### Front-running

Since miners have full control over the transactions ordering in a block, few front-running techniques are possible. New functionality allows you to track all `matchOrders` call and try to front-run it with higher gas cost. It's a good use case because such calls are pure profit (in exchange for ZRX fees and transaction fees).

# Specific issues

## Medium

### `cancelOrdersUpTo` overflow issues

* Order with `salt == 2^256` can not be cancelled using this function.
* If someone will cancel orders up to `salt == 2^256 - 1` value, it can not be undone. No orders can be accepted from that address anymore.
* Exception and `uint` overflow on `cancelOrdersUpTo(2^256)` function call.

```uint256 newOrderEpoch = targetOrderEpoch + 1;``` - this statement can cause overflow.

**Note** : It's recommended by the protocol to use timestamp as the `salt`. It's a good advice and no problems will arise in that case.

**Solution** : keep `salt == 0` canceled by default. Change ```orderEpoch[order.makerAddress][order.senderAddress] > order.salt``` to ```orderEpoch[order.makerAddress][order.senderAddress] >= order.salt``` in 'if-canceled' checking.


### `registerAssetProxy` in `AssetProxyOwner` can register a proxy without checking its owner

`AssetProxyOwner` should only register valid proxies that are owned by this `AssetProxyOwner` contract.

**Solution** : add ownership check


## Minor

### Emit event on `preSign` in `MixinSignatureValidator`

Currently no event is emited. It's recomended to emit events on every sagnificant state change.

### Validation of `preSign` function call in `MixinSignatureValidator`

Functions `preSign` and `setSignatureValidatorApproval` are pretty similar by design and their impact, but they have different validation methods. `preSign` can't be called directly by signer without additional signing arguments.

**Solution** : make similar validation procedure in both functions for code consistency. No need to call `isValidSignature` in `preSign` if `signerAddress == msg.sender`. 

### `batchCancelOrders` should be `NoThrow`

Right now batch cancellation throws an exception if one of the orders is already filled or canceled.
Canceling all possible orders, even if one of them is already canceled/filled is more common behavior.

### Unbounded loop

There is unbounded loop in `MixinAuthorizable.sol` in `removeAuthorizedAddress` function.

**Solution 1** : change data structure to more appropriate. For example, keep values in array and indexes in mapping.

**Solution 2** : remove `removeAuthorizedAddress` function since it's not used anywhere. I prefer not to use this solution, because this function might be useful in future.


### Whitepaper and official website are outdated


# Missing functionality


### Atomically match more than 2 orders

In the current implementation, `matchOrders` can only match 2 orders, but arbitrage can be more complicated. 
For example if we have 3 orders (WETH -> ZRX), (ZRX -> ST), (ST -> WETH). 
All these orders can also be partially matched in order to do the arbitrage. 
It's possible to try to achieve this functionality by using `batchFillOrKillOrders`, but it has more limited functionality.

### Trading multiple erc721 is not possible

There is no way to atomically swap 2 ERC-721 tokens for 1 ERC-721 token.

### Create 'fillOrKill' order from maker side