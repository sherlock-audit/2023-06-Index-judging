# Issue H-1: price is calculated wrongly in BoundedStepwiseExponentialPriceAdapter 

Source: https://github.com/sherlock-audit/2023-06-Index-judging/issues/39 

## Found by 
0x007, 0x52, Brenzee, dany.armstrong90
## Summary
The BoundedStepwiseExponentialPriceAdapter contract is trying to implement price change as `scalingFactor * (e^x - 1)` but the code implements `scalingFactor * e^x - 1`. Since there are no brackets, multiplication would be executed before subtraction. And this has been confirmed with one of the team members.

## Vulnerability Detail
The [getPrice code](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/integration/auction-price/BoundedStepwiseExponentialPriceAdapter.sol#L40-L73) has been simplified as the following when boundary/edge cases are ignored

```solidity
(
    uint256 initialPrice,
    uint256 scalingFactor,
    uint256 timeCoefficient,
    uint256 bucketSize,
    bool isDecreasing,
    uint256 maxPrice,
    uint256 minPrice
) = getDecodedData(_priceAdapterConfigData);

uint256 timeBucket = _timeElapsed / bucketSize;

int256 expArgument = int256(timeCoefficient * timeBucket);

uint256 expExpression = uint256(FixedPointMathLib.expWad(expArgument));

uint256 priceChange = scalingFactor * expExpression - WAD;
```

When timeBucket is 0, we want priceChange to be 0, so that the returned price would be the initial price. Since `e^0 = 1`, we need to subtract 1 (in WAD) from the `expExpression`. 

However, with the incorrect implementation, the returned price would be different than real price by a value equal to `scalingFactor - 1`. The image below shows the difference between the right and wrong formula when initialPrice is 100 and scalingFactor is 11. The right formula starts at 100 while the wrong one starts at 110=100+11-1

![newplot (11)](https://github.com/sherlock-audit/2023-06-Index-bizzyvinci/assets/22333930/56cfc3e4-2bca-40d3-99bd-9e02df94bf33)



## Impact
Incorrect price is returned from BoundedStepwiseExponentialPriceAdapter and that will have devastating effects on rebalance.


## Code Snippet
https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/integration/auction-price/BoundedStepwiseExponentialPriceAdapter.sol#L73

## Tool used

Manual Review

## Recommendation
Change the following [line](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/integration/auction-price/BoundedStepwiseExponentialPriceAdapter.sol#L73)
```solidity
- uint256 priceChange = scalingFactor * expExpression - WAD;
+ uint256 priceChange = scalingFactor * (expExpression - WAD);
```




## Discussion

**pblivin0x**

Confirmed ✅

# Issue M-1: int256 type conversion may overflow, resulting in a price calculation error 

Source: https://github.com/sherlock-audit/2023-06-Index-judging/issues/1 

## Found by 
0xpinky, Topmark, Vagner, auditsea, ethan-crypto, kutugu, pep7siup, qandisa, qpzm, shogoki
## Summary

In `BoundedStepwiseExponentialPriceAdapter` and `BoundedStepwiseLogarithmicPriceAdapter`,  `getPrice` function converts `uint256` to `int256`, but did not check the overflow. 
This causes problems with conversions greater than `type(uint256).max / 2`, resulting in negative number.

## Vulnerability Detail

```solidity
        // Protect against exponential argument overflow
        if (timeBucket > type(uint256).max / timeCoefficient) {
            return _getBoundaryPrice(isDecreasing, maxPrice, minPrice);
        }
        int256 expArgument = int256(timeCoefficient * timeBucket);
        
        // Protect against exponential overflow and increasing relative error
        if (expArgument > MAX_EXP_ARG) {
            return _getBoundaryPrice(isDecreasing, maxPrice, minPrice);
        }
        uint256 expExpression = uint256(FixedPointMathLib.expWad(expArgument));

        // Protect against priceChange overflow
        if (scalingFactor > type(uint256).max / expExpression) {
            return _getBoundaryPrice(isDecreasing, maxPrice, minPrice);
        }
        uint256 priceChange = scalingFactor * expExpression - WAD;
```

As you can see from the code, it only checks if it exceeds uint256, not check int256 overflow. To give a specific example:
```solidity
    function expWad(int256 x) internal pure returns (int256 r) {
        unchecked {
            // When the result is < 0.5 we return zero. This happens when
            // x <= floor(log(0.5e18) * 1e18) ~ -42e18
            if (x <= -42139678854452767551) return r;
```
When `timeBucket * timeCoefficient = type(uint256).max / 2 + 1`, `expArgument = type(int256).min < MAX_EXP_ARG`, and then it will caculate `expExpression = 0`.  Since the divisor is 0, revert. 

## Impact

int256 type conversion may overflow, resulting in a price calculation error.

## Code Snippet

- https://github.com/sherlock-audit/2023-06-Index/blob/8d348ed344635a068d458aa04956f966b6d3d4f3/index-protocol/contracts/protocol/integration/auction-price/BoundedStepwiseExponentialPriceAdapter.sol#L61
- https://github.com/sherlock-audit/2023-06-Index/blob/8d348ed344635a068d458aa04956f966b6d3d4f3/index-protocol/contracts/protocol/integration/auction-price/BoundedStepwiseLogarithmicPriceAdapter.sol#L61C39-L61C39

## Tool used

Manual Review

## Recommendation

Check int256 conversion for overflow




## Discussion

**pblivin0x**

The fix here is to replace the usage of `type(uint256).max` in the overflow checks with `type(int256).max`?

```solidity
        // Protect against exponential argument overflow
        if (timeBucket > type(int256).max / timeCoefficient) {
            return _getBoundaryPrice(isDecreasing, maxPrice, minPrice);
        }
        int256 expArgument = int256(timeCoefficient * timeBucket);
        
        // Protect against exponential overflow and increasing relative error
        if (expArgument > MAX_EXP_ARG) {
            return _getBoundaryPrice(isDecreasing, maxPrice, minPrice);
        }
        uint256 expExpression = uint256(FixedPointMathLib.expWad(expArgument));

        // Protect against priceChange overflow
        if (scalingFactor > type(int256).max / expExpression) {
            return _getBoundaryPrice(isDecreasing, maxPrice, minPrice);
        }
        uint256 priceChange = scalingFactor * expExpression - WAD;
```

**pblivin0x**

Noting that this looks to be a duplicate of #18 

# Issue M-2: SetToken can't be unlocked early. 

Source: https://github.com/sherlock-audit/2023-06-Index-judging/issues/38 

## Found by 
0x52, Yuki, qandisa
## Summary
SetToken can't be unlocked early

## Vulnerability Detail
The function unlock() is used to unlock the setToken after rebalancing, as how it is right now there are two ways to unlock the setToken.

- can be unlocked once the rebalance duration has elapsed
- can be unlocked early if all targets are met, there is excess or at-target quote asset, and raiseTargetPercentage is zero
```solidity
    function unlock(ISetToken _setToken) external {
        bool isRebalanceDurationElapsed = _isRebalanceDurationElapsed(_setToken);
        bool canUnlockEarly = _canUnlockEarly(_setToken);

        // Ensure that either the rebalance duration has elapsed or the conditions for early unlock are met
        require(isRebalanceDurationElapsed || canUnlockEarly, "Cannot unlock early unless all targets are met and raiseTargetPercentage is zero");

        // If unlocking early, update the state
        if (canUnlockEarly) {
            delete rebalanceInfo[_setToken].rebalanceDuration;
            emit LockedRebalanceEndedEarly(_setToken);
        }

        // Unlock the SetToken
        _setToken.unlock();
    }
```
```solidity
    function _canUnlockEarly(ISetToken _setToken) internal view returns (bool) {
        RebalanceInfo storage rebalance = rebalanceInfo[_setToken];
        return _allTargetsMet(_setToken) && _isQuoteAssetExcessOrAtTarget(_setToken) && rebalance.raiseTargetPercentage == 0;
    }
```

The main problem occurs as the value of raiseTargetPercentage isn't reset after rebalancing. The other thing is that the function setRaiseTargetPercentage can't be used to fix this issue as it doesn't allow giving raiseTargetPercentage a zero value.

A setToken can use the AuctionModule to rebalance multiple times, duo to the fact that raiseTargetPercentage value isn't reset after every rebalancing. Once changed with the help of the function setRaiseTargetPercentage this value will only be non zero for every next rebalancing. A setToken can be unlocked early only if all other requirements are met and the raiseTargetPercentage equals zero.

This problem prevents for a setToken to be unlocked early on the next rebalances, once the value of the variable raiseTargetPercentage is set to non zero. 

On every rebalance a manager should be able to keep the value of raiseTargetPercentage to zero (so the setToken can be unlocked early), or increase it at any time with the function setRaiseTargetPercentage.

```solidity
    function setRaiseTargetPercentage(
        ISetToken _setToken,
        uint256 _raiseTargetPercentage
    )
        external
        onlyManagerAndValidSet(_setToken)
    {
        // Ensure the raise target percentage is greater than 0
        require(_raiseTargetPercentage > 0, "Target percentage must be greater than 0");

        // Update the raise target percentage in the RebalanceInfo struct
        rebalanceInfo[_setToken].raiseTargetPercentage = _raiseTargetPercentage;

        // Emit an event to log the updated raise target percentage
        emit RaiseTargetPercentageUpdated(_setToken, _raiseTargetPercentage);
    }
```

## Impact
Once the value of raiseTargetPercentage is set to non zero, every next rebalancing of the setToken won't be eligible for unlocking early. As the value of raiseTargetPercentage isn't reset after every rebalance and neither the manager can set it back to zero with the function setRaiseTargetPercentage().

## Code Snippet

https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L389

## Tool used

Manual Review

## Recommendation
Recommend to reset the value raiseTargetPercentage after every rebalancing.

```solidity
    function unlock(ISetToken _setToken) external {
        bool isRebalanceDurationElapsed = _isRebalanceDurationElapsed(_setToken);
        bool canUnlockEarly = _canUnlockEarly(_setToken);

        // Ensure that either the rebalance duration has elapsed or the conditions for early unlock are met
        require(isRebalanceDurationElapsed || canUnlockEarly, "Cannot unlock early unless all targets are met and raiseTargetPercentage is zero");

        // If unlocking early, update the state
        if (canUnlockEarly) {
            delete rebalanceInfo[_setToken].rebalanceDuration;
            emit LockedRebalanceEndedEarly(_setToken);
        }

+       rebalanceInfo[_setToken].raiseTargetPercentage = 0;

        // Unlock the SetToken
        _setToken.unlock();
    }
```



## Discussion

**pblivin0x**

The risk here is a stale `raiseTargetPercentage` can lead to a SetToken that cannot be unlocked early?

Debating if we should include this or not.

**FlattestWhite**

Should raiseTargetPercentage be part of the check for unlocking early? I'm thinking it doesn't since that's only used when needing to `raiseTargetAssets` because we met all our targets and we have leftover WETH.

**pblivin0x**

since we can't set the raiseAssetTarget to zero, this is a valid issue. will fix 

**ElmoArmy**

> since we can't set the raiseAssetTarget to zero, this is a valid issue. will fix

Yea I agree with the assessment and looks like the fix is more or less clear. 

# Issue M-3: No check for sequencer uptime can lead to dutch auctions executing at bad prices 

Source: https://github.com/sherlock-audit/2023-06-Index-judging/issues/40 

## Found by 
0x52
## Summary

When purchasing from dutch auctions on L2s there is no considering of sequencer uptime. When the sequencer is down, all transactions must originate from the L1. The issue with this is that these transactions use an aliased address. Since the set token contracts don't implement any way for these aliased addressed to interact with the protocol, no transactions can be processed during this time even with force L1 inclusion. If the sequencer goes offline during the the auction period then the auction will continue to decrease in price while the sequencer is offline. Once the sequencer comes back online, users will be able to buy tokens from these auctions at prices much lower than market price.

## Vulnerability Detail

See summary.

## Impact

Auction will sell/buy assets at prices much lower/higher than market price leading to large losses for the set token

## Code Snippet

[AuctionRebalanceModuleV1.sol#L772-L836](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L772-L836)

## Tool used

Manual Review

## Recommendation

Check sequencer uptime and invalidate the auction if the sequencer was ever down during the auction period



## Discussion

**pblivin0x**

What exactly is the remediation here? To check an external uptime feed https://docs.chain.link/data-feeds/l2-sequencer-feeds ?

Not sure if we will fix this issue. This may be on manager parameterizing the auction to select tight upper/lower bounds. 

**FlattestWhite**

Agree won't fix - will look at again if we launch on an L2.

**ElmoArmy**

The equivalent effect to this on L1 would be a reorg that would move time forward but not have had any bids on the canonical chain. The protection on this is the manager setting an appropriate floor for the auction as the "loss" outcome is no different than having no participants.

# Issue M-4: Full inventory asset purchases can be DOS'd via frontrunning 

Source: https://github.com/sherlock-audit/2023-06-Index-judging/issues/41 

## Found by 
0x52
## Summary

Users who attempt to swap the entire component value can be frontrun with a very small bid making their transaction revert

## Vulnerability Detail

[AuctionRebalanceModuleV1.sol#L795-L796](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L795-L796)

        // Ensure that the component quantity in the bid does not exceed the available auction quantity.
        require(_componentQuantity <= bidInfo.auctionQuantity, "Bid size exceeds auction quantity");

When creating a bid, it enforces the above requirement. This prevents users from buying more than they should but it is also a source of an easy DOS attack. Assume a user is trying to buy the entire balance of a component, a malicious user can frontrun them buying only a tiny amount. Since they requested the entire balance, the call with fail. This is a useful technique if an attacker wants to DOS other buyers to pass the time and get a better price from the dutch auction.

## Impact

Malicious user can DOS legitimate users attempting to purchase the entire amount of component

## Code Snippet

[AuctionRebalanceModuleV1.sol#L772-L836](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L772-L836)

## Tool used

Manual Review

## Recommendation

Allow users to specify type(uint256.max) to swap the entire available balance



## Discussion

**pblivin0x**

The recommendation here is that we allow `componentQuantity` to be specified in excess of the auction size? so replace the current check

```solidity
// Ensure that the component quantity in the bid does not exceed the available auction quantity.
require(_componentQuantity <= bidInfo.auctionQuantity, "Bid size exceeds auction quantity");
```

with a enforced cap

```solidity
if (_componentQuantity > bidInfo.auctionQuantity) {
    _componentQuantity = bidInfo.auctionQuantity;
}
```

I was originally hesitant because of some unintuitive UX, but if this removes the potential for a DOS attack, i think it is worth implementing.

**FlattestWhite**

Hmmm we should probably allow user to specify `maxQuantity` rather than the absolute quantity they want to buy

**ElmoArmy**

> The recommendation here is that we allow `componentQuantity` to be specified in excess of the auction size? so replace the current check
> 
> ```solidity
> // Ensure that the component quantity in the bid does not exceed the available auction quantity.
> require(_componentQuantity <= bidInfo.auctionQuantity, "Bid size exceeds auction quantity");
> ```
> 
> with a enforced cap
> 
> ```solidity
> if (_componentQuantity > bidInfo.auctionQuantity) {
>     _componentQuantity = bidInfo.auctionQuantity;
> }
> ```
> 
> I was originally hesitant because of some unintuitive UX, but if this removes the potential for a DOS attack, i think it is worth implementing.

I believe what the submitter was recommending was something more akin to keeping the :

```solidity
 if (_componentQuantity == type(uint256).max) {
     _componentQuantity = bidInfo.auctionQuantity;
} else {
require(_componentQuantity <= bidInfo.auctionQuantity, "Bid size exceeds auction quantity");
}
```

note: the difference in gas is trivial, but it isn't equivalent to your suggestion so I wanted to point it out. My examples assumes pragma > 0.7 to use the type().max syntax but the older idiomatic `uint256(-1) ` would.

# Issue M-5: Exponential and logarithmic price adapters will return incorrect pricing when moving from higher dp token to lower dp token 

Source: https://github.com/sherlock-audit/2023-06-Index-judging/issues/42 

## Found by 
0x52
## Summary

The exponential and logarithmic price adapters do not work correctly when used with token pricing of different decimal places. This is because the resolution of the underlying expWad and lnWad functions is not fit for tokens that aren't 18 dp.

## Vulnerability Detail

[AuctionRebalanceModuleV1.sol#L856-L858](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L856-L858)

    function _calculateQuoteAssetQuantity(bool isSellAuction, uint256 _componentQuantity, uint256 _componentPrice) private pure returns (uint256) {
        return isSellAuction ? _componentQuantity.preciseMulCeil(_componentPrice) : _componentQuantity.preciseMul(_componentPrice);
    }

The price returned by the adapter is used directly to call _calculateQuoteAssetQuantity which uses preciseMul/preciseMulCeil to convert from component amount to quote amount. Assume we wish to sell 1 WETH for 2,000 USDT. WETH is 18dp while USDT is 6dp giving us the following price:

    1e18 * price / 1e18 = 2000e6

Solving for price gives:

    price = 2000e6

This establishes that the price must be scaled to:

    price dp = 18 - component dp + quote dp

Plugging in our values we see that our scaling of 6 dp makes sense.

[BoundedStepwiseExponentialPriceAdapter.sol#L67-L80](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/integration/auction-price/BoundedStepwiseExponentialPriceAdapter.sol#L67-L80)

        uint256 expExpression = uint256(FixedPointMathLib.expWad(expArgument));

        // Protect against priceChange overflow
        if (scalingFactor > type(uint256).max / expExpression) {
            return _getBoundaryPrice(isDecreasing, maxPrice, minPrice);
        }
        uint256 priceChange = scalingFactor * expExpression - WAD;

        if (isDecreasing) {
            // Protect against price underflow
            if (priceChange > initialPrice) {
                return minPrice;
            }
            return FixedPointMathLib.max(initialPrice - priceChange , minPrice);

Given the pricing code and notably the simple scalingFactor it also means that priceChange must be in the same order of magnitude as the price which in this case is 6 dp. The issue is that on such small scales, both lnWad and expWad do not behave as expected and instead yield a linear behavior. This is problematic as the curve will produce unexpected behaviors under these circumstances selling the tokens at the wrong price. Since both functions are written in assembly it is very difficult to determine exactly what is going on or why this occurs but testing in remix gives the following values:

    expWad(1e6) - WAD = 1e6
    expWad(5e6) - WAD = 5e6
    expWad(10e6) - WAD = 10e6
    expWad(1000e6) - WAD = 1000e6

As seen above these value create a perfect linear scaling and don't exhibit any exponential qualities. Given the range of this linearity it means that these adapters can never work when selling from higher to lower dp tokens. 

## Impact

Exponential and logarithmic pricing is wrong when tokens have mismatched dp

## Code Snippet

[BoundedStepwiseExponentialPriceAdapter.sol#L28-L88](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/integration/auction-price/BoundedStepwiseExponentialPriceAdapter.sol#L28-L88)

[BoundedStepwiseLogarithmicPriceAdapter.sol#L28-L88](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/integration/auction-price/BoundedStepwiseLogarithmicPriceAdapter.sol#L28-L88)

## Tool used

Manual Review

## Recommendation

scalingFactor should be scaled to 18 dp then applied via preciseMul instead of simple multiplication. This allows lnWad and expWad to execute in 18 dp then be scaled down to the correct dp.



## Discussion

**pblivin0x**

Agree that scalingFactor should be 18 decimals and applied with preciseMul, will fix. 

**Oot2k**

Not a duplicate 

# Issue M-6: Target raises can be highly damaging for dutch auctions with multiple components 

Source: https://github.com/sherlock-audit/2023-06-Index-judging/issues/45 

## Found by 
0x52
## Summary

Multi-component dutch auctions are fundamentally incompatible with target raises and will lead to inefficient pricing causing loss to set token.

## Vulnerability Detail

The AuctionRebalanceModuleV1 allows targets to be increased when all component targets have been met and there is still excess quote token. When combined with multiple components, it his highly likely that these target raises will lead to inefficient pricing which will cause loss to the set token.

Consider the following a set token has the following composition that has target raises enabled:

40% USDC
30% WBTC
30% WETH

The manager wishes to rebalance the set to the following using USDC as the quote token:

20% USDC
40% WBTC
40% WETH

Assume the WETH portion of the execute within the first hour of the auction. The WBTC on the other hand doesn't execute until 12 hours in. Assume there is excess quote so the target is increased. The issue is that now because of the change in time, the WETH auction is now well above the market price. This buys the WETH for a large loss compared to the market price of WETH.

## Impact

Pricing after target raises will likely be heavily skewed from market prices for some components lead to set token losses

## Code Snippet

[AuctionRebalanceModuleV1.sol#L359-L380](https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L359-L380)

## Tool used

Manual Review

## Recommendation

Target raises should reset `rebalanceStartTime` allowing the dutch auction to restart and properly price the assets



## Discussion

**pblivin0x**

Agree, especially since the raising of targets is onlyAllowedBidder, we should reset the pricings. 

In the fix, I think we will make it such that the rebalance still ends at the same time. 

