Able Glossy Reindeer

high

# `_quoteQuantityLimit` , i.e max amount of quote asset to be spent in a sell auction specified by bidders may not really protect them at all times

## Summary
Specified `_quoteQuantityLimit` may not work in certain scenarios during a sell auction

## Vulnerability Detail
Max amount of quote asset to be spent during sell Auction (`_quoteQuantityLimit`) can be greater than quote asset available balance.

This is possible as there's no check in `AuctionRebalanceModuleV1._validateQuoteAssetQuantity()` to avoid such situations during a sell auction.

```solidity
function _validateQuoteAssetQuantity(bool isSellAuction, uint256 quoteAssetQuantity, uint256 _quoteQuantityLimit, uint256 preBidTokenSentBalance) private pure {
        if (isSellAuction) {//@audit-issue
            require(quoteAssetQuantity <= _quoteQuantityLimit, "Quote asset quantity exceeds limit"); //@audit-info  max to be spent..
        } else {//@audit-info what if max amount of quote asset to be spent during sell auction is greater than quote asset available balance??
            require(quoteAssetQuantity >= _quoteQuantityLimit, "Quote asset quantity below limit");//@audit-info min to be received
            require(quoteAssetQuantity <= preBidTokenSentBalance, "Insufficient quote asset balance");
        }
    }

```

Now the issue is if Quote asset available balance is < `_quoteQuantityLimit`, the max amount of quote asset to be spent in a sell auction specified by allowed bidders becomes invalid and this will cause much more quote assets to be spent.. much more than the bidders wanted.



    
Please i don’t think this should be considered as user input error, because the manager is in charge of adding quote asset, and bidders won't really be aware of the current balance of the quote asset


## Impact
`_quoteQuantityLimit` will be invalid and incapable of protecting bidders from spending more quote asset than they desire

A single bidder in a single bid may be able to finish the quote asset entire balance during a sell auction due to the `_quoteQuantityLimit` being invalid. This is possible because `_quoteQuantityLimit` is invalid whenever it is greater than quote asset balance

## Code Snippet
https://github.com/sherlock-audit/2023-06-Index/blob/main/index-protocol/contracts/protocol/modules/v1/AuctionRebalanceModuleV1.sol#L868-L869
## Tool used

Lofi Radio & Manual Review

## Recommendation
add a check to ensure that `_quoteQuantityLimit` is <= quote asset balance  during a sell auction in `AuctionRebalanceModuleV1._validateQuoteAssetQuantity()` function