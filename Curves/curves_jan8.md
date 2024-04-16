# Title 1: In `_buyCurvesToken` function, the left amount of ETH sent by the buyer is not refunded to him.

https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L267-L270

## Severity
High

## Impact
If the amount of Ether to be sent is greater than the amount actually needed in `_buyCurvesToken` function, the left amount will not be refunded to the buyer. Therefore, the buyer will lose the left amount.

## Proof of Concept
In `_buyCurvesToken` function, they calculates the price and fees that to be needed for buying curves token of certain amount and compare if received ETH is greater than sum of the price and fees.
```solidity
        uint256 price = getPrice(supply, amount);
        (, , , , uint256 totalFee) = getFees(price);

        if (msg.value < price + totalFee) revert InsufficientPayment();
```
But they don't refund the left amount of ETH to the buyer. Total supply of the curves token may change before the transaction is executed due to various actions like selling curves token by the other account and it affects the price and fees. Therefore, received ETH amount may become greater than needed amount indeedly and buyer will lost left ETH because left amount is not refunded.

## Tools Used
VS Code

## Recommended Mitigation Steps
Refund left amount of ETH to the account when received ETH is greater than sum of `price` and `totalFee`.

# Title 2: When an account sell its curves token, the protocol fee is not transfered to `protocolFeeDestination`.

https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L225-L233

## Severity
High

## Impact
When an account sell its curves token, the protocol is calculated but not transfered to anywhere.

## Proof of Concept
When an account sell its curves token, the internal function `_transferFees` is executed to distribute fees to right destinations.
```solidity
    function _transferFees(
        address curvesTokenSubject,
        bool isBuy,
        uint256 price,
        uint256 amount,
        uint256 supply
    ) internal {
```
This function has `isBuy` param and it is passed as `false` in the case of selling. Otherwise, it is passed as `true` in the case of buying
```solidity
        address firstDestination = isBuy ? feesEconomics.protocolFeeDestination : msg.sender;
        uint256 buyValue = referralDefined ? protocolFee : protocolFee + referralFee;
        uint256 sellValue = price - protocolFee - subjectFee - referralFee - holderFee;
        (bool success1, ) = firstDestination.call{value: isBuy ? buyValue : sellValue}("");
        if (!success1) revert CannotSendFunds();
```
If `isBuy` is `true`, `firstDestination` is set as `priceFeeDestination` and the protocol fee is transfered to it. But if `isBuy` is `false`, `firstDestination` is set as seller and calculated protocol fee isn't transfered to anywhere.
Therefore, the protocol will lose their fee amount about `sellCurvesToken` action.

## Tools Used
VS Code

## Recommended Mitigation Steps
Add logic that send protocol fee to right destination when `isBuy` is `false` in `_transferFees` function.
```diff
    function _transferFees(
        address curvesTokenSubject,
        bool isBuy,
        uint256 price,
        uint256 amount,
        uint256 supply
    ) internal {
        (uint256 protocolFee, uint256 subjectFee, uint256 referralFee, uint256 holderFee, ) = getFees(price);
        {
            bool referralDefined = referralFeeDestination[curvesTokenSubject] != address(0);
            {
                address firstDestination = isBuy ? feesEconomics.protocolFeeDestination : msg.sender;
                uint256 buyValue = referralDefined ? protocolFee : protocolFee + referralFee;
                uint256 sellValue = price - protocolFee - subjectFee - referralFee - holderFee;
                (bool success1, ) = firstDestination.call{value: isBuy ? buyValue : sellValue}("");
                if (!success1) revert CannotSendFunds();
+               if (!isBuy) {
+                   (bool success4, ) = protocolFeeDestination.call{value: buyValue}("");
+                   if (!success4) revert CannotSendFunds();
+               }
            }
            {
                (bool success2, ) = curvesTokenSubject.call{value: subjectFee}("");
                if (!success2) revert CannotSendFunds();
            }
            {
                (bool success3, ) = referralDefined
                    ? referralFeeDestination[curvesTokenSubject].call{value: referralFee}("")
                    : (true, bytes(""));
                if (!success3) revert CannotSendFunds();
            }


            if (feesEconomics.holdersFeePercent > 0 && address(feeRedistributor) != address(0)) {
                feeRedistributor.onBalanceChange(curvesTokenSubject, msg.sender);
                feeRedistributor.addFees{value: holderFee}(curvesTokenSubject);
            }
        }
        emit Trade(
            msg.sender,
            curvesTokenSubject,
            isBuy,
            amount,
            price,
            protocolFee,
            subjectFee,
            isBuy ? supply + amount : supply - amount
        );
    }
```

# Title 3: As the length of an account's `ownedCurvesTokenSubjects` continues to increase, a DoS may occur when that account purchases new curves tokens.

https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L328-L336

## Severity
High

## Impact
As the number of curves tokens owned by an account increases, buying new curve token of the account may fall into DoS.

## Proof of Concept
Whenever an account buy a new curve token, the token is pushed to `ownedCurvesTokenSubjects` of the account.
```solidity
    function _addOwnedCurvesTokenSubject(address owner_, address curvesTokenSubject) internal {
        address[] storage subjects = ownedCurvesTokenSubjects[owner_];
        for (uint256 i = 0; i < subjects.length; i++) {
            if (subjects[i] == curvesTokenSubject) {
                return;
            }
        }
        subjects.push(curvesTokenSubject);
    }
```
But there is no any logic that decrease the length `ownedCurvesTokenSubjects[_owned]` in entire protocol so that it would be increasing whenever the account buy new curve token. It may exceed gas limit of transaction due to `for` loop with too long array length and fall into denial of service.

## Tools Used
VS Code

## Recommended Mitigation Steps
In `_addOwnedCurvesTokenSubject` function, change the logic that check if passed `curvesTokenSubject` is already owned token by introducing new state `ownedCurvesTokenSubjectStatus`.
```diff
+   mapping(address => mapping(address => bool)) public ownedCurvesTokenSubjectStatus;
    
    ...

    function _addOwnedCurvesTokenSubject(address owner_, address curvesTokenSubject) internal {
        address[] storage subjects = ownedCurvesTokenSubjects[owner_];
-       for (uint256 i = 0; i < subjects.length; i++) {
-           if (subjects[i] == curvesTokenSubject) {
-               return;
-           }
-       }

+       if (ownedCurvesTokenSubjectStatus[owner_][curvesTokenSubject]) {
+           return;
+       } else {
+           ownedCurvesTokenSubjectStatus[owner_][curvesTokenSubject] = true;
+       }
        subjects.push(curvesTokenSubject);
    }
```


# Title 4: `setCurves` function of `FeeSplitter` contract should be protected by a trusted role.

https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/FeeSplitter.sol#L35-L37

## Severity
High

## Impact
Since `setCurves` function of `FeeSplitter` contract isn't be protected by a trusted role, anyone can change `curves` contract address.

## Proof of Concept
`setCurves` function of `FeeSplitter` contract is public function but it doesn't have any modifier that protects it with trusted role.
```solidity
    function setCurves(Curves curves_) public {
        curves = curves_;
    }
```
Therefore, anyone can change `curves` address of the contract and it may occur error while performing transaction.
In worst case, an attacker can manipulate balance and supply of particular curve token by deploying own mocked `Curves` contract and setting its address as `curves` of `FeeSplitter`.

## Tools Used
VS Code

## Recommended Mitigation Steps
Add modifier that protect with a trusted role to `setCurves` function.


# Title 5: In `Curves.setWhiteList` function `merkleRoot` can be zero

https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L394-L402

## Severity
Medium

## Impact
In `setWhitelist` function of `Curves` contract, `merkleRoot` can be zero and anyone can be whitelisted since there is no validation.

## Proof of Concept
In `setWhitelist` function of `Curves` contract, there isn't any validation for `merkleRoot`.
```solidity
    function setWhitelist(bytes32 merkleRoot) external {
        uint256 supply = curvesTokenSupply[msg.sender];
        if (supply > 1) revert CurveAlreadyExists();


        if (presalesMeta[msg.sender].merkleRoot != merkleRoot) {
            presalesMeta[msg.sender].merkleRoot = merkleRoot;
            emit WhitelistUpdated(msg.sender, merkleRoot);
        }
    }
```
Therefore, `merkleRoot` can be zero and it allows everyone can be whitelisted.

## Tools Used
VS Code

## Recommended Mitigation Steps
Add validation for `merkleRoot` as follow.
```diff
    function setWhitelist(bytes32 merkleRoot) external {
        uint256 supply = curvesTokenSupply[msg.sender];
        if (supply > 1) revert CurveAlreadyExists();
+       require(merkleRoot != bytes32(0), "MerkleRoot can't be zero")

        if (presalesMeta[msg.sender].merkleRoot != merkleRoot) {
            presalesMeta[msg.sender].merkleRoot = merkleRoot;
            emit WhitelistUpdated(msg.sender, merkleRoot);
        }
    }
```


# Title 6: Holders can lose their holder fee due to updating of userFeeOffset of `onBalanceChange` function

https://github.com/code-423n4/2024-01-curves/blob/516aedb7b9a8d341d0d2666c23780d2bd8a9a600/contracts/Curves.sol#L246-L249

## Severity
Medium

# Impact
When holders buy/sell the curves tokens, they can't claim the past `cumulativeFeePerToken` since `userFeeOffset` is updated in `onBalanceChange` function.

# Proof of Concept
When holders buy/sell the curves tokens, `_transferFees` function is called to distribute fees.
```solidity
    function _transferFees(
        address curvesTokenSubject,
        bool isBuy,
        uint256 price,
        uint256 amount,
        uint256 supply
    ) internal {
        (uint256 protocolFee, uint256 subjectFee, uint256 referralFee, uint256 holderFee, ) = getFees(price);
        {
            bool referralDefined = referralFeeDestination[curvesTokenSubject] != address(0);
            {
                address firstDestination = isBuy ? feesEconomics.protocolFeeDestination : msg.sender;
                uint256 buyValue = referralDefined ? protocolFee : protocolFee + referralFee;
                uint256 sellValue = price - protocolFee - subjectFee - referralFee - holderFee;
                (bool success1, ) = firstDestination.call{value: isBuy ? buyValue : sellValue}("");
                if (!success1) revert CannotSendFunds();
            }
            {
                (bool success2, ) = curvesTokenSubject.call{value: subjectFee}("");
                if (!success2) revert CannotSendFunds();
            }
            {
                (bool success3, ) = referralDefined
                    ? referralFeeDestination[curvesTokenSubject].call{value: referralFee}("")
                    : (true, bytes(""));
                if (!success3) revert CannotSendFunds();
            }


            if (feesEconomics.holdersFeePercent > 0 && address(feeRedistributor) != address(0)) {
                feeRedistributor.onBalanceChange(curvesTokenSubject, msg.sender);
                feeRedistributor.addFees{value: holderFee}(curvesTokenSubject);
            }
        }
        emit Trade(
            msg.sender,
            curvesTokenSubject,
            isBuy,
            amount,
            price,
            protocolFee,
            subjectFee,
            isBuy ? supply + amount : supply - amount
        );
    }
```
If `holdersFeePercent` is greater than 0, `onBalanceChange` function of `FeeSplitter` contract is called.
```solidity
    function onBalanceChange(address token, address account) public onlyManager {
        TokenData storage data = tokensData[token];
        data.userFeeOffset[account] = data.cumulativeFeePerToken;
        if (balanceOf(token, account) > 0) userTokens[account].push(token);
    }
```
In `onBalanceChange` function, `data.userFeeOffset[account]` is updated as current value of `data.cumulativeFeePerToken`. Therefore, the holder lose the past `cumulativeFeePerToken` unless he claims it in advance before buy/sell the curves token.

## Tools Used
VS Code

## Recommended Mitigation Steps
The code line that updates `userFeeOffset[account]` in `onBalanceChange` function should be removed.
```diff
    function onBalanceChange(address token, address account) public onlyManager {
        TokenData storage data = tokensData[token];
-       data.userFeeOffset[account] = data.cumulativeFeePerToken;
        if (balanceOf(token, account) > 0) userTokens[account].push(token);
    }
```

# Title 7: When a user tries to deploy new curves token with default name and symbol, the malicious user can make it would be reverted.

## Severity
Medium

## Impact
When subjects try to deploy their curves token by using the default name and symbol, a malicious user can make it would be reverted.

## Proof of Concept
When a subject tries to deploy a new own curves token with the name and symbol, `_deployERC20` function of `Curves.sol` contract is called and checks if the passed symbol alreday exist.
```solidity
    function _deployERC20(
        address curvesTokenSubject,
        string memory name,
        string memory symbol
    ) internal returns (address) {
        // If the token's symbol is CURVES, append a counter value
        if (keccak256(bytes(symbol)) == keccak256(bytes(DEFAULT_SYMBOL))) {
            _curvesTokenCounter += 1;
            name = string(abi.encodePacked(name, " ", Strings.toString(_curvesTokenCounter)));
            symbol = string(abi.encodePacked(symbol, Strings.toString(_curvesTokenCounter)));
        }


@>      if (symbolToSubject[symbol] != address(0)) revert InvalidERC20Metadata();


        address tokenContract = CurvesERC20Factory(curvesERC20Factory).deploy(name, symbol, address(this));


        externalCurvesTokens[curvesTokenSubject].token = tokenContract;
        externalCurvesTokens[curvesTokenSubject].name = name;
        externalCurvesTokens[curvesTokenSubject].symbol = symbol;
        externalCurvesToSubject[tokenContract] = curvesTokenSubject;
        symbolToSubject[symbol] = curvesTokenSubject;


        emit TokenDeployed(curvesTokenSubject, tokenContract, name, symbol);
        return address(tokenContract);
    }
```
Also, if a subject tries to deploy new curves token without setting its name and symbol in advance by calling `mint` function, the default name and symbol would be used. And this can be a problem.

Let me explain with an example. Malicious users can perform the following steps to occur problem:

1. First, a malicious user get the vlaue of state `_curvesTokenCounter`. (Assume that this value is 3). Getting this private value is not impossible.

2. Next, the malicious user calls `buyCurvesTokenWithName` function with params of `name` as `string(abi.encodePacked(DEFAULT_NAME, " ", Strings.toString(4)))` and `symbol` as `string(abi.encodePacked(DEFAULT_SYMBOL, Strings.toString(4)))`.

    ```solidity
        function buyCurvesTokenWithName(
            address curvesTokenSubject,
            uint256 amount,
            string memory name,
            string memory symbol
        ) public payable {
            uint256 supply = curvesTokenSupply[curvesTokenSubject];
            if (supply != 0) revert CurveAlreadyExists();

            _buyCurvesToken(curvesTokenSubject, amount);
            _mint(curvesTokenSubject, name, symbol);
        }
    ```

3. As the result, `_mint` function is called and then `_deployERC20` function is called with the `name` and `symbol`. The passed symbol is different from `DEFAULT_SYMBOL` and it allows the malicious user own the curves token with symbol without increasing the `_curvesTokenCounter`.
    ```solidity
        if (keccak256(bytes(symbol)) == keccak256(bytes(DEFAULT_SYMBOL))) {
            _curvesTokenCounter += 1;
            name = string(abi.encodePacked(name, " ", Strings.toString(_curvesTokenCounter)));
            symbol = string(abi.encodePacked(symbol, Strings.toString(_curvesTokenCounter)));
        }
    ```

In this situation, when another user tries to deploy new curves token with default name and symbol, it will always be reverted because `symbolToSubject[string(abi.encodePacked(DEFAULT_SYMBOL, Strings.toString(4)))]` is already set as `true` in above steps by following codeline.

```solidity
    if (symbolToSubject[symbol] != address(0)) revert InvalidERC20Metadata();
```
In `withdraw` function, the same problem may occur.

## Tools Used
VS Code

## Recommended Mitigation Steps
Move the codeline that increasing `_curvesTokenCounter` as follow:
```diff
    function _deployERC20(
        address curvesTokenSubject,
        string memory name,
        string memory symbol
    ) internal returns (address) {
        // If the token's symbol is CURVES, append a counter value
+       _curvesTokenCounter += 1;
        if (keccak256(bytes(symbol)) == keccak256(bytes(DEFAULT_SYMBOL))) {
-           _curvesTokenCounter += 1;
            name = string(abi.encodePacked(name, " ", Strings.toString(_curvesTokenCounter)));
            symbol = string(abi.encodePacked(symbol, Strings.toString(_curvesTokenCounter)));
        }

        if (symbolToSubject[symbol] != address(0)) revert InvalidERC20Metadata();
```


# Title 8: Because the `transferCurvesToken` function doesn't check if `amount` is greater than 0, a malicious user can permanently block other users from buying new curves tokens and depositing.

## Severity
Medium

## Impact
A malicious user can increase another user's number of owned curves token massively by using `transferCurvesToken` function and this may occur DoS when the victim tries to buy new curves token or call `deposit` function.

## Proof of Concept
Users can transfer their tokens to the others by using `transferCurvesToken` function.
But, as you can see, this function doesn't check if `amount` is greater than 0.
```solidity
    function transferCurvesToken(address curvesTokenSubject, address to, uint256 amount) external {
        if (to == address(this)) revert ContractCannotReceiveTransfer();
        _transfer(curvesTokenSubject, msg.sender, to, amount);
    }

    // ...

    function _transfer(address curvesTokenSubject, address from, address to, uint256 amount) internal {
        if (amount > curvesTokenBalance[curvesTokenSubject][from]) revert InsufficientBalance();

        // If transferring from oneself, skip adding to the list
        if (from != to) {
            _addOwnedCurvesTokenSubject(to, curvesTokenSubject);
        }

        curvesTokenBalance[curvesTokenSubject][from] = curvesTokenBalance[curvesTokenSubject][from] - amount;
        curvesTokenBalance[curvesTokenSubject][to] = curvesTokenBalance[curvesTokenSubject][to] + amount;

        emit Transfer(curvesTokenSubject, from, to, amount);
    }

    // Internal function to add a curvesTokenSubject to the list if not already present
    function _addOwnedCurvesTokenSubject(address owner_, address curvesTokenSubject) internal {
        address[] storage subjects = ownedCurvesTokenSubjects[owner_];
        for (uint256 i = 0; i < subjects.length; i++) {
            if (subjects[i] == curvesTokenSubject) {
                return;
            }
        }
        subjects.push(curvesTokenSubject);
    }
```
Therefore, a malicious user can increase another user's `ownedCurvesTokenSubjects` by calling this `transferCurvesToken` function with params of `curvesTokenSubject` as an arbitrary address, `to` as the victim's address and `amount` as 0.
If the malicious user repeats above action massively with different `curvesTokenSubject` param each time, it is possible that reach to the block gas limit due to following for loop.
```solidity
    function _addOwnedCurvesTokenSubject(address owner_, address curvesTokenSubject) internal {
        address[] storage subjects = ownedCurvesTokenSubjects[owner_];
@>        for (uint256 i = 0; i < subjects.length; i++) {
            if (subjects[i] == curvesTokenSubject) {
                return;
            }
        }
        subjects.push(curvesTokenSubject);
    }
```
Since `_buyCurvesToken` and `_transfer` use this `_addOwnedCurvesTokenSubject` function, the victim cannot buy new curves token and deposit.
```solidity
    function _buyCurvesToken(address curvesTokenSubject, uint256 amount) internal {
        uint256 supply = curvesTokenSupply[curvesTokenSubject];
        if (!(supply > 0 || curvesTokenSubject == msg.sender)) revert UnauthorizedCurvesTokenSubject();

        uint256 price = getPrice(supply, amount);
        (, , , , uint256 totalFee) = getFees(price);

        if (msg.value < price + totalFee) revert InsufficientPayment();

        curvesTokenBalance[curvesTokenSubject][msg.sender] += amount;
        curvesTokenSupply[curvesTokenSubject] = supply + amount;
        _transferFees(curvesTokenSubject, true, price, amount, supply);

        // If is the first token bought, add to the list of owned tokens
        if (curvesTokenBalance[curvesTokenSubject][msg.sender] - amount == 0) {
            _addOwnedCurvesTokenSubject(msg.sender, curvesTokenSubject);
        }
    }

    // ..

    function deposit(address curvesTokenSubject, uint256 amount) public {
        if (amount % 1 ether != 0) revert NonIntegerDepositAmount();

        address externalToken = externalCurvesTokens[curvesTokenSubject].token;
        uint256 tokenAmount = amount / 1 ether;

        if (externalToken == address(0)) revert TokenAbsentForCurvesTokenSubject();
        if (amount > CurvesERC20(externalToken).balanceOf(msg.sender)) revert InsufficientBalance();
        if (tokenAmount > curvesTokenBalance[curvesTokenSubject][address(this)]) revert InsufficientBalance();

        CurvesERC20(externalToken).burn(msg.sender, amount);
        _transfer(curvesTokenSubject, address(this), msg.sender, tokenAmount);
    }
```

## Tools Used
VS Code

## Recommended Mitigation Steps
You should revert if `amount` is not greater than 0, in `_transfer` function.
```diff
    function _transfer(address curvesTokenSubject, address from, address to, uint256 amount) internal {
+       if (!(amount > 0)) revert; 
        if (amount > curvesTokenBalance[curvesTokenSubject][from]) revert InsufficientBalance();

        // If transferring from oneself, skip adding to the list
        if (from != to) {
            _addOwnedCurvesTokenSubject(to, curvesTokenSubject);
        }

        curvesTokenBalance[curvesTokenSubject][from] = curvesTokenBalance[curvesTokenSubject][from] - amount;
        curvesTokenBalance[curvesTokenSubject][to] = curvesTokenBalance[curvesTokenSubject][to] + amount;

        emit Transfer(curvesTokenSubject, from, to, amount);
    }
```

# Title 9: Because the `transferCurvesToken` function doesn't check if `amount` is greater than 0, a malicious user can permanently block other users from buying new curves tokens and depositing.

## Severity
Medium

## Impact
A malicious user can increase another user's number of owned curves token massively by using `transferCurvesToken` function and this may occur DoS when the victim tries to buy new curves token or call `deposit` function.

## Proof of Concept
Users can transfer their tokens to the others by using `transferCurvesToken` function.
But, as you can see, this function doesn't check if `amount` is greater than 0.
```solidity
    function transferCurvesToken(address curvesTokenSubject, address to, uint256 amount) external {
        if (to == address(this)) revert ContractCannotReceiveTransfer();
        _transfer(curvesTokenSubject, msg.sender, to, amount);
    }

    // ...

    function _transfer(address curvesTokenSubject, address from, address to, uint256 amount) internal {
        if (amount > curvesTokenBalance[curvesTokenSubject][from]) revert InsufficientBalance();

        // If transferring from oneself, skip adding to the list
        if (from != to) {
            _addOwnedCurvesTokenSubject(to, curvesTokenSubject);
        }

        curvesTokenBalance[curvesTokenSubject][from] = curvesTokenBalance[curvesTokenSubject][from] - amount;
        curvesTokenBalance[curvesTokenSubject][to] = curvesTokenBalance[curvesTokenSubject][to] + amount;

        emit Transfer(curvesTokenSubject, from, to, amount);
    }

    // Internal function to add a curvesTokenSubject to the list if not already present
    function _addOwnedCurvesTokenSubject(address owner_, address curvesTokenSubject) internal {
        address[] storage subjects = ownedCurvesTokenSubjects[owner_];
        for (uint256 i = 0; i < subjects.length; i++) {
            if (subjects[i] == curvesTokenSubject) {
                return;
            }
        }
        subjects.push(curvesTokenSubject);
    }
```
Therefore, a malicious user can increase another user's `ownedCurvesTokenSubjects` by calling this `transferCurvesToken` function with params of `curvesTokenSubject` as an arbitrary address, `to` as the victim's address and `amount` as 0.
If the malicious user repeats above action massively with different `curvesTokenSubject` param each time, it is possible that reach to the block gas limit due to following for loop.
```solidity
    function _addOwnedCurvesTokenSubject(address owner_, address curvesTokenSubject) internal {
        address[] storage subjects = ownedCurvesTokenSubjects[owner_];
@>        for (uint256 i = 0; i < subjects.length; i++) {
            if (subjects[i] == curvesTokenSubject) {
                return;
            }
        }
        subjects.push(curvesTokenSubject);
    }
```
Since `_buyCurvesToken` and `_transfer` use this `_addOwnedCurvesTokenSubject` function, the victim cannot buy new curves token and deposit.
```solidity
    function _buyCurvesToken(address curvesTokenSubject, uint256 amount) internal {
        uint256 supply = curvesTokenSupply[curvesTokenSubject];
        if (!(supply > 0 || curvesTokenSubject == msg.sender)) revert UnauthorizedCurvesTokenSubject();

        uint256 price = getPrice(supply, amount);
        (, , , , uint256 totalFee) = getFees(price);

        if (msg.value < price + totalFee) revert InsufficientPayment();

        curvesTokenBalance[curvesTokenSubject][msg.sender] += amount;
        curvesTokenSupply[curvesTokenSubject] = supply + amount;
        _transferFees(curvesTokenSubject, true, price, amount, supply);

        // If is the first token bought, add to the list of owned tokens
        if (curvesTokenBalance[curvesTokenSubject][msg.sender] - amount == 0) {
            _addOwnedCurvesTokenSubject(msg.sender, curvesTokenSubject);
        }
    }

    // ..

    function deposit(address curvesTokenSubject, uint256 amount) public {
        if (amount % 1 ether != 0) revert NonIntegerDepositAmount();

        address externalToken = externalCurvesTokens[curvesTokenSubject].token;
        uint256 tokenAmount = amount / 1 ether;

        if (externalToken == address(0)) revert TokenAbsentForCurvesTokenSubject();
        if (amount > CurvesERC20(externalToken).balanceOf(msg.sender)) revert InsufficientBalance();
        if (tokenAmount > curvesTokenBalance[curvesTokenSubject][address(this)]) revert InsufficientBalance();

        CurvesERC20(externalToken).burn(msg.sender, amount);
        _transfer(curvesTokenSubject, address(this), msg.sender, tokenAmount);
    }
```

## Tools Used
VS Code

## Recommended Mitigation Steps
You should revert if `amount` is not greater than 0, in `_transfer` function.
```diff
    function _transfer(address curvesTokenSubject, address from, address to, uint256 amount) internal {
+       if (!(amount > 0)) revert; 
        if (amount > curvesTokenBalance[curvesTokenSubject][from]) revert InsufficientBalance();

        // If transferring from oneself, skip adding to the list
        if (from != to) {
            _addOwnedCurvesTokenSubject(to, curvesTokenSubject);
        }

        curvesTokenBalance[curvesTokenSubject][from] = curvesTokenBalance[curvesTokenSubject][from] - amount;
        curvesTokenBalance[curvesTokenSubject][to] = curvesTokenBalance[curvesTokenSubject][to] + amount;

        emit Transfer(curvesTokenSubject, from, to, amount);
    }
```
