# Since the typehash of Rent Payload is not derived correctly, the signer address can be calculated incorrectly in `validateOrder` function of `Create` policy.

## Severity
High

## Impact
Since deriving typehash of `RentPayload` struct is not EIP712 compliant, the `signer` address is calculated incorrectly in `validateOrder` function of `Create` policy.

## Proof of Concept
In `validateOrder` function, they decode the signed rental zone payload and use it to calculate `signer` address.
```solidity
    function validateOrder(
        ZoneParameters calldata zoneParams
    ) external override onlyRole("SEAPORT") returns (bytes4 validOrderMagicValue) {
        // Decode the signed rental zone payload from the extra data.
@>      (RentPayload memory payload, bytes memory signature) = abi.decode(
            zoneParams.extraData,
            (RentPayload, bytes)
        );

        // Create a payload of seaport data.
        SeaportPayload memory seaportPayload = SeaportPayload({
            orderHash: zoneParams.orderHash,
            zoneHash: zoneParams.zoneHash,
            offer: zoneParams.offer,
            consideration: zoneParams.consideration,
            totalExecutions: zoneParams.totalExecutions,
            fulfiller: zoneParams.fulfiller,
            offerer: zoneParams.offerer
        });

        // Check: The signature from the protocol signer has not expired.
        _validateProtocolSignatureExpiration(payload.expiration);

        // Check: The fulfiller is the intended fulfiller.
        _validateFulfiller(payload.intendedFulfiller, seaportPayload.fulfiller);

        // Recover the signer from the payload.
@>      address signer = _recoverSignerFromPayload(
            _deriveRentPayloadHash(payload),
            signature
        );

        // Check: The data matches the signature and that the protocol signer is the one that signed.
        if (!kernel.hasRole(signer, toRole("CREATE_SIGNER"))) {
            revert Errors.CreatePolicy_UnauthorizedCreatePolicySigner();
        }

        // Initiate the rental using the rental manager.
        _rentFromZone(payload, seaportPayload);

        // Return the selector of validateOrder as the magic value.
        validOrderMagicValue = ZoneInterface.validateOrder.selector;
    }
```
To do this, they derive hash of `payload` which has type of `RentPayload` struct based on EIP-712.
```solidity
    function _deriveRentPayloadHash(
        RentPayload memory payload
    ) internal view returns (bytes32) {
        // Derive and return the rent payload hash as specified by EIP-712.
        return
            keccak256(
                abi.encode(
                    _RENT_PAYLOAD_TYPEHASH,
                    _deriveOrderFulfillmentHash(payload.fulfillment),
                    _deriveOrderMetadataHash(payload.metadata),
                    payload.expiration,
                    payload.intendedFulfiller
                )
            );
    }
```
However, `_RENT_PAYLOAD_TYPEHASH` that is used to get hash of rent payload is not derived correctly as specified EIP-712.
```solidity
        // Derive name and version hashes alongside required EIP-712 typehashes.
        (
            _ITEM_TYPEHASH,
            _HOOK_TYPEHASH,
            _RENTAL_ORDER_TYPEHASH,
            _ORDER_FULFILLMENT_TYPEHASH,
            _ORDER_METADATA_TYPEHASH,
@>          _RENT_PAYLOAD_TYPEHASH
        ) = _deriveRentalTypehashes();
```
```solidity
    function _deriveRentalTypehashes()
        internal
        pure
        returns (
            bytes32 itemTypeHash,
            bytes32 hookTypeHash,
            bytes32 rentalOrderTypeHash,
            bytes32 orderFulfillmentTypeHash,
            bytes32 orderMetadataTypeHash,
            bytes32 rentPayloadTypeHash
        )
    {
        // ..

        {
            // Construct the OrderFulfillment type string.
            bytes memory orderFulfillmentTypeString = abi.encodePacked(
                "OrderFulfillment(address recipient)"
            );

            // Construct the OrderMetadata type string.
            bytes memory orderMetadataTypeString = abi.encodePacked(
                "OrderMetadata(uint8 orderType,uint256 rentDuration,Hook[] hooks,bytes emittedExtraData)"
            );

            // Construct the RentPayload type string.
            bytes memory rentPayloadTypeString = abi.encodePacked(
                "RentPayload(OrderFulfillment fulfillment,OrderMetadata metadata,uint256 expiration,address intendedFulfiller)"
            );

            // Derive RentPayload type hash via combination of relevant type strings.
@>          rentPayloadTypeHash = keccak256(
                abi.encodePacked(
                    rentPayloadTypeString,
                    orderMetadataTypeString,
                    orderFulfillmentTypeString
                )
            );
            
            // ..
        }
    }
```
According to EIP-712, the structure [typehash is defined as](https://eips.ethereum.org/EIPS/eip-712#definition-of-hashstruct): `typeHash = keccak256(encodeType(typeOf(s)))`.

And [definition of `encodeType`](https://eips.ethereum.org/EIPS/eip-712#definition-of-encodetype) is as follow:

>The type of a struct is encoded as `name ‖ "(" ‖ member₁ ‖ "," ‖ member₂ ‖ "," ‖ … ‖ memberₙ ")"` where each member is written as `type ‖ " " ‖ name`. For example, the above `Mail` struct is encoded as `Mail(address from,address to,string contents)`.
>
>If the struct type references other struct types (and these in turn reference even more struct types), then the set of referenced struct types is collected, sorted by name and appended to the encoding. An example encoding is `Transaction(Person from,Person to,Asset tx)Asset(address token,uint256 amount)Person(address wallet,string name)`.

As described in above quote, if the struct type references other struct types within it, the referenced struct types should be sorted by their names and appended to the encoding.
`RentPayload` struct references two structs - `OrderMedata` and `OrderFulfillment`. Therefore, by alphabetical orders of their names, `orderFulfillmentTypeString` should be appended first and `orderMetadataTypeString` last. But `rentPaylodTypehash` is not derived in that way.
```solidity
    rentPayloadTypeHash = keccak256(
        abi.encodePacked(
            rentPayloadTypeString,
@>          orderMetadataTypeString,
@>          orderFulfillmentTypeString
        )
    );
```
This makes hash of rent paylod is derived incorrectly. Therefore, `signer` address is decoded incorrectly.
```solidity
    function _recoverSignerFromPayload(
        bytes32 payloadHash,
        bytes memory signature
    ) internal view returns (address) {
        // Derive original EIP-712 digest using domain separator and order hash.
        bytes32 digest = _DOMAIN_SEPARATOR.toTypedDataHash(payloadHash);

        // Recover the signer address of the signature.
        return digest.recover(signature);
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Derive the typehash of `RentPayload` correctly as specified EIP-712.
```diff
    rentPayloadTypeHash = keccak256(
        abi.encodePacked(
            rentPayloadTypeString,
+           orderFulfillmentTypeString,
+           orderMetadataTypeString
-           orderMetadataTypeString,
-           orderFulfillmentTypeString
        )
    );
```
