Interesting Merlot Cricket

high

# price is calculated wrongly in BoundedStepwiseExponentialPriceAdapter

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

![253827539-56cfc3e4-2bca-40d3-99bd-9e02df94bf33](https://github.com/sherlock-audit/2023-06-Index-judging/assets/1225563/5256c121-a35d-4ce0-8740-15b7eef42109)



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
