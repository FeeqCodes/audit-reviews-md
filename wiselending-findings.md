### [H-1] Repetitive \_safeTransfer call leading to loss of funds

## Description:

withdrawAmount is been withdrawn twice in `PendlePowerFarm.sol::_manuallyWithdrawShares` internal function. It makes a `_safeTransfer` with a value to the msg.sender in `withdrawExactShares` and a `_safeTransfer` is being called in the same code block right after.

## Impact:

This would Lead to a loss of funds to the protocol since users can now withdraw from their deposited shares twice in a single transaction.

## Tools Used

Manual Review

## Proof Of Concept:

```javascript
     function _manuallyWithdrawShares(
        uint256 _nftId,
        uint256 _withdrawShares
    )
        internal
    {
        uint256 withdrawAmount = WISE_LENDING.cashoutAmount(
            PENDLE_CHILD,
            _withdrawShares
        );

        // @audit-
        withdrawAmount = WISE_LENDING.withdrawExactShares(
            _nftId,
            PENDLE_CHILD,
            _withdrawShares
        );


        // @audit-
        _safeTransfer(
            PENDLE_CHILD,
            msg.sender,
            withdrawAmount
        );
    }

```

## Recommended Mitigation Steps:

Change the code logic to allow for only a single withdrawAmount in the `_manuallyWithdrawShares` internal function

```diff
    function _manuallyWithdrawShares(
        uint256 _nftId,
        uint256 _withdrawShares
    )
        internal
    {
        uint256 withdrawAmount = WISE_LENDING.cashoutAmount(
            PENDLE_CHILD,
            _withdrawShares
        );

        withdrawAmount = WISE_LENDING.withdrawExactShares(
            _nftId,
            PENDLE_CHILD,
            _withdrawShares
        );

-        _safeTransfer(
-            PENDLE_CHILD,
-            msg.sender,
-            withdrawAmount
-        );
    }

```

### [H-2] Break of protocol logic, specific function can't be called upon deployment

## Description:

" NOTE - The protocol will be deployed on ETH and Arbitrum. On ETH AaveHub will NOT be used.."
The above plan by the protocol will lead to users unable to call the `WiseLending.sol::borrowOnBehalfExactAmount` due the `onlyAaveHub` access control being set

## Impact:

This would lead to a break in protocols logic and waste of gas.

## Tools Used

Manual Review

## Proof Of Concept:

```javascript
      function borrowOnBehalfExactAmount(
        uint256 _nftId,
        address _poolToken,
        uint256 _amount
    )
        external
        onlyAaveHub
        syncPool(_poolToken)
        healthStateCheck(_nftId)
        returns (uint256)
    {
        uint256 shares = _handleBorrowExactAmount({
            _nftId: _nftId,
            _poolToken: _poolToken,
            _amount: _amount,
            _onBehalf: true
        });

        _safeTransfer(_poolToken, msg.sender, _amount);

        return shares;
    }

```

## Recommended Mitigation Steps:

Change the protocol logic to either accept Aave or discard the overall function




### [M-4] Misinformation of events for critical parameters

## Description:

Event is currently being emitted before the transfer has occured. This leads to misinformation of critical parameters

## Impact:

Misinformation of events for critical parameters

## Tools Used

Manual Review

## Proof Of Concept:

```javascript
       function solelyDeposit(
        uint256 _nftId,
        address _poolToken,
        uint256 _amount
    ) public syncPool(_poolToken) {
        _handleSolelyDeposit(msg.sender, _nftId, _poolToken, _amount);

        _emitFundsSolelyDeposited(msg.sender, _nftId, _poolToken, _amount);

        _safeTransferFrom(_poolToken, msg.sender, address(this), _amount);
    }

```

## Recommended Mitigation Steps:

Function should emit event after safe transfer

```diff
    function solelyDeposit(
        uint256 _nftId,
        address _poolToken,
        uint256 _amount
    ) public syncPool(_poolToken) {
        _handleSolelyDeposit(msg.sender, _nftId, _poolToken, _amount);

-     _emitFundsSolelyDeposited(msg.sender, _nftId, _poolToken, _amount);
-     _safeTransferFrom(_poolToken, msg.sender, address(this), _amount);

+     _safeTransferFrom(_poolToken, msg.sender, address(this), _amount);
+     _emitFundsSolelyDeposited(msg.sender, _nftId, _poolToken, _amount);
    }

```





### [M-5] Missing Important event logs after sensitive actions

## Description:

Function is missing important event logs, sensitive actions is performed but there are no events being emitted.

## Impact:

Lack of sensitive informations by users.

## Tools Used

Manual Review

## Proof Of Concept:

```javascript
    function collateralizeDeposit(
        uint256 _nftId,
        address _poolToken

    ) external syncPool(_poolToken) {

        WISE_SECURITY.checksCollateralizeDeposit(
            _nftId,
            msg.sender,
            _poolToken
        );

        userLendingData[_nftId][_poolToken].unCollateralized = false;
    }
```

## Recommended Mitigation Steps:

Emit an eventLog upon successfull deposit

```diff
+    event DepositCollateralized(
+        address indexed sender,
+        uint256 indexed nftId,
+        address indexed token,
+        uint256 amount,
+        uint256 shares,
+        uint256 timestamp
+    );

    function collateralizeDeposit(
        uint256 _nftId,
        address _poolToken

    ) external syncPool(_poolToken) {

        WISE_SECURITY.checksCollateralizeDeposit(
            _nftId,
            msg.sender,
            _poolToken
        );

        userLendingData[_nftId][_poolToken].unCollateralized = false;

+       _emitDepositCollateralized(msg.sender, _nftId, _poolToken);
    }

```





### [H-2] Fee manager's funds gets stick in contract

## Description:

Fee manager's funds get stuck in smart contract after the first transfer.

## Impact:

Loss of funds meant to be paid as fees to the `feeManager`

## Tools Used

Manual Review

## Proof Of Concept:

Upon declaration the `feeManger` address is equal to zero, which passes the "if" check and sends the `feeManager` a `FEE_MANAGER_NFT`. subsequent transfer will fail the "if" check and revert "NotPermitted()" because the `feeManager` address will now be initialized with the `_feeManagerContract` which is a dynamic address passed as a parameter to the `forwardFeeManagerNFT.

```javascript
    function forwardFeeManagerNFT(
        address _feeManagerContract
    )
        external
        onlyMaster
    {
        if (feeManager > ZERO_ADDRESS) {
            revert NotPermitted();
        }

        feeManager = _feeManagerContract;

        _transfer(
            address(this),
            _feeManagerContract,
            FEE_MANAGER_NFT
        );
    }

```

## Recommended Mitigation Steps:

Re configure the logic and set `feeManager` address back to zero after every successfull `_transfer()`.

```diff
    function forwardFeeManagerNFT(
        address _feeManagerContract
    )
        external
        onlyMaster
    {
        if (feeManager > ZERO_ADDRESS) {
            revert NotPermitted();
        }

        feeManager = _feeManagerContract;

        _transfer(
            address(this),
            _feeManagerContract,
            FEE_MANAGER_NFT
        );

+       feeManager = address(0);

    }
```

### [H-3] Share's health factor not checked leading to wrong positions health state.

## Description:

Function Interface `IWiseLending::withdrawExactShares` does not apply essential modifiers and checks for `syncPool()` and `healthStateCheck()`. hence, functions inheriting the interface like `PendlePowerFarm::_manuallyWithdrawShares` are not checked to know if positions are healthy or not and might lead to wrong values in ascertaining positions that should be liquidated.

## Impact:

Break in core contract logic in asscertaining the health factor of positions

## Tools Used

Manual Review

## Proof Of Concept:

```javascript
    function _manuallyWithdrawShares(
        uint256 _nftId,
        uint256 _withdrawShares
    )
        internal
    {
        uint256 withdrawAmount = WISE_LENDING.cashoutAmount(
            PENDLE_CHILD,
            _withdrawShares
        );

        withdrawAmount = WISE_LENDING.withdrawExactShares(
            _nftId,
            PENDLE_CHILD,
            _withdrawShares
        );

        _safeTransfer(
            PENDLE_CHILD,
            msg.sender,
            withdrawAmount
        );
    }

```

## Recommended Mitigation Steps:

Create an interface in the `IWiseLending` with an external function for

```javascript
    function healthStateCheck() external
```

Add additional checks to check for the health State of positions in `PendlePowerFarm::_manuallyWithdrawShares` using the `_healthStateCheck()`

```diff
     function _manuallyWithdrawShares(
        uint256 _nftId,
        uint256 _withdrawShares
    )
        internal
    {
        uint256 withdrawAmount = WISE_LENDING.cashoutAmount(
            PENDLE_CHILD,
            _withdrawShares
        );

+       withdrawAmount = WISE_LENDING.healthStateCheck(nftId)

        withdrawAmount = WISE_LENDING.withdrawExactShares(
            _nftId,
            PENDLE_CHILD,
            _withdrawShares
        );

        _safeTransfer(
            PENDLE_CHILD,
            msg.sender,
            withdrawAmount
        );
    }
```

### [M-1] Malicious user can dynamically delete availableNFT

## Description:

When a user enters the farm with 0 eth, the `getWiseLendingNFT()`decreases and reverts at `_openPostion()` with - "AmountTooSmall()". but the `availableNFT` will be reduced already

## Impact:

reduction in `availableNFT`

## Tools Used

Manual Review

## Proof Of Concept:

```javascript
    function enterFarm(
        bool _isAave,
        uint256 _amount,
        uint256 _leverage,
        uint256 _allowedSpread
    )
        external
        isActive
        updatePools
        returns (uint256)
    {
        uint256 wiseLendingNFT = _getWiseLendingNFT();

        _safeTransferFrom(
            WETH_ADDRESS,
            msg.sender,
            address(this),
            _amount
        );

        _openPosition(
            _isAave,
            wiseLendingNFT,
            _amount,
            _leverage,
            _allowedSpread
        );

        uint256 keyId = _reserveKey(
            msg.sender,
            wiseLendingNFT
        );

        isAave[keyId] = _isAave;

        emit FarmEntry(
            keyId,
            wiseLendingNFT,
            _leverage,
            _amount,
            block.timestamp
        );

        return keyId;
    }

```

```javascript
     function _getWiseLendingNFT()
        internal
        returns (uint256)
    {
        if (availableNFTCount == 0) {

            uint256 nftId = POSITION_NFT.mintPosition();

            _registrationFarm(
                nftId
            );

            POSITION_NFT.approve(
                AAVE_HUB_ADDRESS,
                nftId
            );

            return nftId;
        }

        // @audit - this reduces the list of available NFTs
        return availableNFTs[
            availableNFTCount--
        ];
    }
```

## Recommended Mitigation Steps:

Make a check to ensure `_amount` is greater than zero

```diff
+   error AmountLessThanZero()

    function enterFarm(
        bool _isAave,
        uint256 _amount,
        uint256 _leverage,
        uint256 _allowedSpread
    )
        external
        isActive
        updatePools
        returns (uint256)
    {

+       if ( _amount < 0) {
+           revert AmountLessThanZero()
+       }

        uint256 wiseLendingNFT = _getWiseLendingNFT();

        _safeTransferFrom(
            WETH_ADDRESS,
            msg.sender,
            address(this),
            _amount
        );

        _openPosition(
            _isAave,
            wiseLendingNFT,
            _amount,
            _leverage,
            _allowedSpread
        );

        uint256 keyId = _reserveKey(
            msg.sender,
            wiseLendingNFT
        );

        isAave[keyId] = _isAave;

        emit FarmEntry(
            keyId,
            wiseLendingNFT,
            _leverage,
            _amount,
            block.timestamp
        );

        return keyId;
    }
```










### [M-2] Users can withdraw shares before required checks revert

## Description:
Not following CEI. users an withdraw shares before the following check revert `_checkDebtRatio(wiseLendingNFT) == false` in `PendlePowerManager::manuallyWithdrawShares`

## Impact:
This could lead to loss of funds since the user can withdraw funds before the the collateral debt ratio get checked. Users with bad debts can withdraw their funds wich is against the protocol logic.

## Tools Used
Manual Review

## Proof Of Concept:

```javascript
    function manuallyWithdrawShares(
        uint256 _keyId,
        uint256 _withdrawShares
    )
        external
        updatePools
        onlyKeyOwner(_keyId)
    {
        uint256 wiseLendingNFT = farmingKeys[
            _keyId
        ];

        _manuallyWithdrawShares(
            wiseLendingNFT,
            _withdrawShares
        );

        if (_checkDebtRatio(wiseLendingNFT) == false) {
            revert DebtRatioTooHigh();
        }

        emit ManualWithdrawShares(
            _keyId,
            wiseLendingNFT,
            _withdrawShares,
            block.timestamp
        );
    }
```

## Recommended Mitigation Steps:
Check the debt ratio before allowing manual withdrawals

```diff
    function manuallyWithdrawShares(
        uint256 _keyId,
        uint256 _withdrawShares
    )
        external
        updatePools
        onlyKeyOwner(_keyId)
    {
        uint256 wiseLendingNFT = farmingKeys[
            _keyId
        ];

-        _manuallyWithdrawShares(
-            wiseLendingNFT,
-            _withdrawShares
-        );

-        if (_checkDebtRatio(wiseLendingNFT) == false) {
-            revert DebtRatioTooHigh();
-        }

+        if (_checkDebtRatio(wiseLendingNFT) == false) {
+            revert DebtRatioTooHigh();
+        }

+        _manuallyWithdrawShares(
+            wiseLendingNFT,
+            _withdrawShares
+        );


        emit ManualWithdrawShares(
            _keyId,
            wiseLendingNFT,
            _withdrawShares,
            block.timestamp
        );
    }

```






### [M-3] farmContract address can become unchangeable

## Description:
If `setFarmContract::farmContract` address is mistakenly set to a different address, it becomes unchangeable since we require to pass a check of `farmContract == ZERO_ADDRESS` before we can change the address which in this is going to fail.

## Impact:
we cant change farmContract address if it is not the zero address and if there is a need to change, the protocol will hav to redeployt the contract

## Tools Used
Manual Review

## Proof Of Concept:

```javascript
    function setFarmContract(
        address _farmContract
    )
        external
        onlyMaster
    {
        if (farmContract == ZERO_ADDRESS) {
            farmContract = _farmContract;
        }
    }
```

## Recommended Mitigation Steps:
Change the `setFarmContract` logic


