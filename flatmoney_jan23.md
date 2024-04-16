# Title 1: `_profitLoss` function of the `PerpMath` calculate the `PnL` incorrectly

## Severity

Medium

## Summary

The calculation of `PnL` in `_profitLoss` function of `PerpMath` is wrong.

## Vulnerability Detail

The `_profitLoss` function calculates and returns `pnl` based on passed params of `position` and `price`.

```solidity
    function _profitLoss(FlatcoinStructs.Position memory position, uint256 price) internal pure returns (int256 pnl) {
        int256 priceShift = int256(price) - int256(position.lastPrice);
        int256 profitLossTimesTen = (int256(position.additionalSize) * (priceShift) * 10) / int256(price);

        if (profitLossTimesTen % 10 != 0) {
            return profitLossTimesTen / 10 - 1;
        } else {
            return profitLossTimesTen / 10;
        }
    }
```
First, following params may be passed to this function.

- `position.additionalSize` is 22
- `price` is 30
- `priceShift` is 3

In this case, the `profitLossTimesTen` is 22 * 3 * 10 / 30 = 22

Next, think of the following params.

`position.additionalSize` is 20
`price` is 30
`priceShift` is 3

In this case, the `profitLossTimesTen` is 20 * 3 * 10 / 30 = 20

In first condition the PnL is 1 and last conditin, PnL is 2. This is not fair.

## Impact

Above wrong calculation may leads to loss of user's fund.

## Tool used

Manual Review

## Recommendation

The `_profitLoss` function should be updated as follow.
```diff
    function _profitLoss(FlatcoinStructs.Position memory position, uint256 price) internal pure returns (int256 pnl) {
        int256 priceShift = int256(price) - int256(position.lastPrice);
        int256 profitLossTimesTen = (int256(position.additionalSize) * (priceShift) * 10) / int256(price);

-       if (profitLossTimesTen % 10 != 0) {
-           return profitLossTimesTen / 10 - 1;
-       } else {
-           return profitLossTimesTen / 10;
-       }
+       return profitLossTimesTen / 10;
    }
```


# Title 2: `checkSkewMax` function of the `FlatcoinValut` contract calculate the `longSkewFraction` incorrectly.

## Severity

High

## Summary

In order to accurately calculate the `longSkewFraction`, it is necessary to subtract the `_additionalSkew` amount from the `stableCollateralTotal` instead of adding it to the `sizeOpenedTotal`. And it may result in the bypassing of the skew check, leading to an excessive skew towards the long side within the system since the `FlatcoinVault.checkSkewMax` is utilized to determine whether the skew is disabled.

## Vulnerability Detail

To calculate the `longSkewFraction` correctly, the `additonalSkew` should be subtracted from the `stableCollateralTotal` instead of being added to the `sizeOpenedTotal`
```solidity
   function checkSkewMax(uint256 _additionalSkew) public view {
        // check that skew is not essentially disabled
        if (skewFractionMax < type(uint256).max) {
            uint256 sizeOpenedTotal = _globalPositions.sizeOpenedTotal;

            if (stableCollateralTotal == 0) revert FlatcoinErrors.ZeroValue("stableCollateralTotal");

@>      uint256 longSkewFraction = ((sizeOpenedTotal + _additionalSkew) * 1e18) / stableCollateralTotal;

            if (longSkewFraction > skewFractionMax) revert FlatcoinErrors.MaxSkewReached(longSkewFraction);
        }
    }
```

## Impact

Due to wrong calculation in `checkSkewMax` function, the system would be too skewed towards the long.

## Tool used

Manual Review

## Recommendation

`checkSkewMax` function should be updated as follow.
```diff
   function checkSkewMax(uint256 _additionalSkew) public view {
        // check that skew is not essentially disabled
        if (skewFractionMax < type(uint256).max) {
            uint256 sizeOpenedTotal = _globalPositions.sizeOpenedTotal;

            if (stableCollateralTotal == 0) revert FlatcoinErrors.ZeroValue("stableCollateralTotal");

-          uint256 longSkewFraction = ((sizeOpenedTotal + _additionalSkew) * 1e18) / stableCollateralTotal;
+          uint256 longSkewFraction = (sizeOpenedTotal * 1e18) / (stableCollateralTotal - _additionalSkew);

            if (longSkewFraction > skewFractionMax) revert FlatcoinErrors.MaxSkewReached(longSkewFraction);
        }
    }
```


# Title 3: In `settleFundingFees` function of `FlatcoinVault` contract, `_globalPositions.marginDepositedTotal` is updated incorrectly.

## Summary
Since the signature of the `_fundingFees` is wrong, the `_globalPositions.marginDepositedTotal` is updated incorrectly.

## Vulnerability Detail

In the protocol, they execute `settleFundingFees` function of the `FlatcoinValut` contract to settle the funding fees. However, the `_globalPositions.marginDepositedTotal` would be updated incorrectly due to _fundingFees has the wrong signature.
```solidity
    function settleFundingFees() public returns (int256 _fundingFees) {
        (int256 fundingChangeSinceRecomputed, int256 unrecordedFunding) = _getUnrecordedFunding();


        // Record the funding rate change and update the cumulative funding rate.
        cumulativeFundingRate = PerpMath._nextFundingEntry(unrecordedFunding, cumulativeFundingRate);


        // Update the latest funding rate and the latest funding recomputation timestamp.
        lastRecomputedFundingRate += fundingChangeSinceRecomputed;
        lastRecomputedFundingTimestamp = (block.timestamp).toUint64();


        // Calculate the funding fees accrued to the longs.
        // This will be used to adjust the global margin and collateral amounts.
        _fundingFees = PerpMath._accruedFundingTotalByLongs(_globalPositions, unrecordedFunding);


        // In the worst case scenario that the last position which remained open is underwater,
        // we set the margin deposited total to 0. We don't want to have a negative margin deposited total.
@>      _globalPositions.marginDepositedTotal = (int256(_globalPositions.marginDepositedTotal) > _fundingFees)
            ? uint256(int256(_globalPositions.marginDepositedTotal) + _fundingFees)
            : 0;


        _updateStableCollateralTotal(-_fundingFees);
```
To be correct, the signature of the `_fundingFees` should be -.

## Impact

Due to wrong signature of the `_fundingFees`, `_globalPositions.marginDepositedTotal` is calculated incorrectly.

## Tool used
Manual Review

## Recommendation
The signature of the `_fundingFees` should be `-` in calculation.
```diff
    function settleFundingFees() public returns (int256 _fundingFees) {
        (int256 fundingChangeSinceRecomputed, int256 unrecordedFunding) = _getUnrecordedFunding();

        // Record the funding rate change and update the cumulative funding rate.
        cumulativeFundingRate = PerpMath._nextFundingEntry(unrecordedFunding, cumulativeFundingRate);

        // Update the latest funding rate and the latest funding recomputation timestamp.
        lastRecomputedFundingRate += fundingChangeSinceRecomputed;
        lastRecomputedFundingTimestamp = (block.timestamp).toUint64();

        // Calculate the funding fees accrued to the longs.
        // This will be used to adjust the global margin and collateral amounts.
        _fundingFees = PerpMath._accruedFundingTotalByLongs(_globalPositions, unrecordedFunding);

        // In the worst case scenario that the last position which remained open is underwater,
        // we set the margin deposited total to 0. We don't want to have a negative margin deposited total.
-       _globalPositions.marginDepositedTotal = (int256(_globalPositions.marginDepositedTotal) > _fundingFees)
+       _globalPositions.marginDepositedTotal = (int256(_globalPositions.marginDepositedTotal) > -_fundingFees)
            ? uint256(int256(_globalPositions.marginDepositedTotal) + _fundingFees)
            : 0;

        _updateStableCollateralTotal(-_fundingFees);
    }
```
