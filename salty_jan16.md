# TItle 1: Since the `formPOL` function doesn't refund the left token, tokens can be locked in the `DAO` contract.

## Severity
High

## Impact
Tokens(SALT, DAI, USDS) can be locked in the `DAO` contract because the contract doesn't refund remaining amount of the tokens.

## Proof of Concept
If a user calls the `performUpkeep` function of the `Upkeep` contract, `step3` and `step4` is called and these functions also called the `_formPOL` function.
```solidity
    // 3. Convert a default 5% of the remaining WETH to USDS/DAI Protocol Owned Liquidity.
    function step3() public onlySameContract
        {
        uint256 wethBalance = weth.balanceOf( address(this) );
        if ( wethBalance == 0 )
            return;

        // A default 5% of the remaining WETH will be swapped for USDS/DAI POL.
        uint256 amountOfWETH = wethBalance * stableConfig.percentArbitrageProfitsForStablePOL() / 100;
        _formPOL(usds, dai, amountOfWETH);
        }


    // 4. Convert a default 20% of the remaining WETH to SALT/USDS Protocol Owned Liquidity.
    function step4() public onlySameContract
        {
        uint256 wethBalance = weth.balanceOf( address(this) );
        if ( wethBalance == 0 )
            return;

        // A default 20% of the remaining WETH will be swapped for SALT/USDS POL.
        uint256 amountOfWETH = wethBalance * daoConfig.arbitrageProfitsPercentPOL() / 100;
        _formPOL(salt, usds, amountOfWETH);
        }
```
And then, the `_formPOL` function calls `formPOL` function of the `DAO` contact.
```solidity
    function _formPOL( IERC20 tokenA, IERC20 tokenB, uint256 amountWETH) internal
        {
        uint256 wethAmountPerToken = amountWETH >> 1;

        // Swap WETH for the specified tokens
        uint256 amountA = pools.depositSwapWithdraw( weth, tokenA, wethAmountPerToken, 0, block.timestamp );
        uint256 amountB = pools.depositSwapWithdraw( weth, tokenB, wethAmountPerToken, 0, block.timestamp );

        // Transfer the tokens to the DAO
        tokenA.safeTransfer( address(dao), amountA );
        tokenB.safeTransfer( address(dao), amountB );

        // Have the DAO form POL
        dao.formPOL(tokenA, tokenB, amountA, amountB);
        }
```
```solidity
    function formPOL( IERC20 tokenA, IERC20 tokenB, uint256 amountA, uint256 amountB ) external
        {
        require( msg.sender == address(exchangeConfig.upkeep()), "DAO.formPOL is only callable from the Upkeep contract" );

        // Use zapping to form the liquidity so that all the specified tokens are used
        collateralAndLiquidity.depositLiquidityAndIncreaseShare( tokenA, tokenB, amountA, amountB, 0, block.timestamp, true );

        emit POLFormed(tokenA, tokenB, amountA, amountB);
        }
```
And `formPOL` function calls the `depositLiquidityAndIncreaseShare` function of the `collateralAndLiquidity` contract, and then `_depositLiquidityAndIncreaseShare` function is called.
```solidity
    function depositLiquidityAndIncreaseShare( IERC20 tokenA, IERC20 tokenB, uint256 maxAmountA, uint256 maxAmountB, uint256 minLiquidityReceived, uint256 deadline, bool useZapping ) external nonReentrant ensureNotExpired(deadline) returns (uint256 addedAmountA, uint256 addedAmountB, uint256 addedLiquidity)
        {
        require( PoolUtils._poolID( tokenA, tokenB ) != collateralPoolID, "Stablecoin collateral cannot be deposited via Liquidity.depositLiquidityAndIncreaseShare" );

        return _depositLiquidityAndIncreaseShare(tokenA, tokenB, maxAmountA, maxAmountB, minLiquidityReceived, useZapping);
        }
```
```solidity
    function _depositLiquidityAndIncreaseShare( IERC20 tokenA, IERC20 tokenB, uint256 maxAmountA, uint256 maxAmountB, uint256 minLiquidityReceived, bool useZapping ) internal returns (uint256 addedAmountA, uint256 addedAmountB, uint256 addedLiquidity)
        {
        require( exchangeConfig.walletHasAccess(msg.sender), "Sender does not have exchange access" );

        // Transfer the specified maximum amount of tokens from the user
        tokenA.safeTransferFrom(msg.sender, address(this), maxAmountA );
        tokenB.safeTransferFrom(msg.sender, address(this), maxAmountB );

        // Balance the token amounts by swapping one to the other before adding the liquidity?
        if ( useZapping )
            (maxAmountA, maxAmountB) = _dualZapInLiquidity(tokenA, tokenB, maxAmountA, maxAmountB );

        // Approve the liquidity to add
        tokenA.approve( address(pools), maxAmountA );
        tokenB.approve( address(pools), maxAmountB );

        // Deposit the specified liquidity into the Pools contract
        // The added liquidity will be owned by this contract. (external call to Pools contract)
        bytes32 poolID = PoolUtils._poolID( tokenA, tokenB );
        (addedAmountA, addedAmountB, addedLiquidity) = pools.addLiquidity( tokenA, tokenB, maxAmountA, maxAmountB, minLiquidityReceived, totalShares[poolID]);

        // Increase the user's liquidity share by the amount of addedLiquidity.
        // Cooldown is specified to prevent reward hunting (ie - quickly depositing and withdrawing large amounts of liquidity to snipe rewards as they arrive)
        // _increaseUserShare confirms the pool as whitelisted as well.
        _increaseUserShare( msg.sender, poolID, addedLiquidity, true );

        // If any of the user's tokens were not used, then send them back
@>      if ( addedAmountA < maxAmountA )
            tokenA.safeTransfer( msg.sender, maxAmountA - addedAmountA );

@>     if ( addedAmountB < maxAmountB )
            tokenB.safeTransfer( msg.sender, maxAmountB - addedAmountB );

        emit LiquidityDeposited(msg.sender, address(tokenA), address(tokenB), addedAmountA, addedAmountB, addedLiquidity);
        }
```
As you can see in the `_depositLiquidityAndIncreaseShare` function, when the added amount of `tokenA` or `tokenB` is greater than `maxAmount` of them, it refund the left amount to the `msg.sender`.
However, `msg.sender` is the address of the `DAO` contract. And the `DAO` contract doesn't refund the received tokens to the user again. Therefore, the tokens(SALT, USDS, DAI) would remain and be locked in the `DAO` contract.

## Tools Used
VS Code

## Recommended Mitigation Steps
Add the logic that refund the exceeded amount of tokens to the user.


# Title 2: Unexpected result can occur since the logic of `winningParameterVote` is incorrect.

## Severity
Medium

## Impact
Unexpected result can occur because the result of the `winningParameterVote` function would be `NO_CHANGE` when the `increaseTotal` and `decreaseTotal` are the same.

## Proof of Concept
In `_finalizeParameterBallot` function of `DAO` contract, the `winningParameterVote` function of `Proposals` contract is called.
```solidity
    function _finalizeParameterBallot( uint256 ballotID ) internal
        {
        Ballot memory ballot = proposals.ballotForID(ballotID);

        Vote winningVote = proposals.winningParameterVote(ballotID);

        if ( winningVote == Vote.INCREASE )
            _executeParameterChange( ParameterTypes(ballot.number1), true, poolsConfig, stakingConfig, rewardsConfig, stableConfig, daoConfig, priceAggregator );
        else if ( winningVote == Vote.DECREASE )
            _executeParameterChange( ParameterTypes(ballot.number1), false, poolsConfig, stakingConfig, rewardsConfig, stableConfig, daoConfig, priceAggregator );

        // Finalize the ballot even if NO_CHANGE won
        proposals.markBallotAsFinalized(ballotID);

        emit BallotFinalized(ballotID, winningVote);
        }
```
```solidity
function winningParameterVote( uint256 ballotID ) external view returns (Vote)
    {
    mapping(Vote=>uint256) storage votes = _votesCastForBallot[ballotID];

    uint256 increaseTotal = votes[Vote.INCREASE];
    uint256 decreaseTotal = votes[Vote.DECREASE];
    uint256 noChangeTotal = votes[Vote.NO_CHANGE];

    if ( increaseTotal > decreaseTotal )
    if ( increaseTotal > noChangeTotal )
        return Vote.INCREASE;

    if ( decreaseTotal > increaseTotal )
    if ( decreaseTotal > noChangeTotal )
        return Vote.DECREASE;

    return Vote.NO_CHANGE;
    }
```
However, in `winningParameterVote` function, when the `increaseTotal` and `decreaseTotal` are the same, it returns the `NO_CHANCE` even if the `noChangeTotal` is 0.
This can produce unexpected results for ballots with intense voting competition, and in the worst case, it can be exploited by malicious users.

## Tools Used
VS Code

## Recommended Mitigation Steps
Should decide how return the result when the `increaseTotal` and `decreaseTotal` are the same and update the `winningParameterVote` function with that logic.

