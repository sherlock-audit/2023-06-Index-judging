Spare Carob Dinosaur

medium

# Full inventory asset purchases can be DOS'd via frontrunning

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