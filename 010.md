Sneaky Coconut Turtle

high

# bidders suffer trade direction flip when currentNotional and targetNotional flip side in AuctionRebalanceModuleV1

## Summary
bidders suffer trade direction flip when currentNotional and targetNotional flip side in AuctionRebalanceModuleV1

## Vulnerability Detail
when a bidder places bid, he/she essentially place 1.) which setToken to interact, 2.) which component to buy/sell 3.) the acceptable input/output.

The Module then gets the latest `currentNotional` and `targetNotional`, which are compared in order to determine the direction of trade needed. if `currentNotional < targetNotional` then the auction needs to "buy"; and vice versa. 

```solidity
        (
            uint256 currentUnit,
            uint256 targetUnit,
            uint256 currentNotional,
            uint256 targetNotional
        ) = _getUnitsAndNotionalAmounts(_setToken, _component, _totalSupply);

        // Ensure that the current unit and target unit are not the same
        require(currentUnit != targetUnit, "Target already met");

        // Determine whether the component is being sold (sendToken) or bought
        isSellAuction = targetNotional < currentNotional;
```

However, the auction has a potential to "overshoot", namely the currentNotional becomes larger than the targetNotional even in a buy auction. 

While the actual logic is very intertwined with the virtual balance accounting in setToken, such case is handled clearly in the `_isQuoteAssetExcessOrAtTarget` where condition `isExcess` is possible. (One of the possibilities is actual executed price has advantages so more output tokens is acquired)

```solidity
    function _isQuoteAssetExcessOrAtTarget(ISetToken _setToken) internal view returns (bool) {
        RebalanceInfo storage rebalance = rebalanceInfo[_setToken];
        bool isExcess = _getDefaultPositionRealUnit(_setToken, rebalance.quoteAsset) > _getNormalizedTargetUnit(_setToken, rebalance.quoteAsset);
        bool isAtTarget = _getDefaultPositionRealUnit(_setToken, rebalance.quoteAsset).approximatelyEquals(_getNormalizedTargetUnit(_setToken, rebalance.quoteAsset), 1);
        return isExcess || isAtTarget;
    }
```
https://github.com/IndexCoop/index-protocol/blob/663e64efaa95df2247afa8926d4cfb42948f54fe/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L1210-L1215

when this happens, basically the bidder would indicate a trade that reverse in direction. Consider:

1. The auction offers some amount of X Token for Y Token, for simplicity let's say the price is 2 X for 1 Y.
2. Bidder A wants to participate in the auction to purchase 2X by sending in 1Y, he specifies a bid with the `_componentAmount==2` and `_quoteAssetLimit==1` in the bid parameters
3. The auction somehow overfills, the notional of Y is more than the targetNotional, isSellAuction becomes true.
4. If the overShoot quantity is more than the `_quoteAssetLimit`, then the check in `_createBidInfo` can be passed too.
5. Bidder A exchanged 1Y for 2X instead.

## Impact
Bidder suffers reverse trade.

## Code Snippet
https://github.com/IndexCoop/index-protocol/blob/663e64efaa95df2247afa8926d4cfb42948f54fe/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L910-L921

## Tool used

Manual Review

## Recommendation
allow bidder to explicitly specify the trade direction, this is desirable and more straight-forward for bidder to revert if the trade does not operate as expected.