Large Cotton Hippo

high

# int256 type conversion may overflow, resulting in a price calculation error

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
