# Protocol mints less `rsETH` on deposit than intended

## Severity

High

# Impact

Price of `rsETH` is calculated as `totalLockedETH / rsETHSupply`. `rsETH` price is used to calculate `rsETH` amount to mint when user deposits. Formulas are following:

- `rsethAmountToMint = amount * assetPrice / rsEthPrice`
- `rsEthPrice = totalEthLocked / rsETHSupply`

Problem is that it transfers deposit amount before calculation of rsethAmountToMint. It increases totalEthLocked. As a result rsethAmountToMint is less than intended because rsEthPrice is higher.

For example:

1. Suppose `totalEthLocked = 10e18`, `assetPrice = 1e18`, `rsETHSupply = 10e18`
2. User deposits `30e18`. He expects to receive `30e18 rsETH`
3. However, actual received amount will be `30e18 * 1e18 / ((30e18 * 1e18 + 10e18 * 1e18) / 10e18) = 7.5e18`

## Proof of Concept

Here you can see that it firstly transfers asset to address(this), then calculates amount to mint:

```solidity
    function depositAsset(
        address asset,
        uint256 depositAmount
    )
        external
        whenNotPaused
        nonReentrant
        onlySupportedAsset(asset)
    {
        ...

        if (!IERC20(asset).transferFrom(msg.sender, address(this), depositAmount)) {
            revert TokenTransferFailed();
        }

        // interactions
        uint256 rsethAmountMinted = _mintRsETH(asset, depositAmount);

        emit AssetDeposit(asset, depositAmount, rsethAmountMinted);
    }
```

There is long chain of calls:

```solidity
_mintRsETH()
    getRsETHAmountToMint()
        LRTOracle().getRSETHPrice()
            getTotalAssetDeposits()
                getTotalAssetDeposits()
```

Finally `getTotalAssetDeposits()` uses current `balanceOf()`, which was increased before by transferrign deposit amount:

```solidity
    function getAssetDistributionData(address asset)
        public
        view
        override
        onlySupportedAsset(asset)
        returns (uint256 assetLyingInDepositPool, uint256 assetLyingInNDCs, uint256 assetStakedInEigenLayer)
    {
        // Question: is here the right place to have this? Could it be in LRTConfig?
@>      assetLyingInDepositPool = IERC20(asset).balanceOf(address(this));

        uint256 ndcsCount = nodeDelegatorQueue.length;
        for (uint256 i; i < ndcsCount;) {
            assetLyingInNDCs += IERC20(asset).balanceOf(nodeDelegatorQueue[i]);
            assetStakedInEigenLayer += INodeDelegator(nodeDelegatorQueue[i]).getAssetBalance(asset);
            unchecked {
                ++i;
            }
        }
    }
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Transfer tokens in the end:

```diff
    function depositAsset(
        address asset,
        uint256 depositAmount
    )
        external
        whenNotPaused
        nonReentrant
        onlySupportedAsset(asset)
    {
        ...

+       uint256 rsethAmountMinted = _mintRsETH(asset, depositAmount);

        if (!IERC20(asset).transferFrom(msg.sender, address(this), depositAmount)) {
            revert TokenTransferFailed();
        }

-       // interactions
-       uint256 rsethAmountMinted = _mintRsETH(asset, depositAmount);

        emit AssetDeposit(asset, depositAmount, rsethAmountMinted);
    }
```
