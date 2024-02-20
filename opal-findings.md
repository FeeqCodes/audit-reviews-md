### [L-1] The depositFor function does'nt include a check for the contract address(0). This could break the deposit logic

**Description:** The `Omnipool::depositFor` function doesn't include a check for the `_depositFor` parameter in the code. Instead, the check is placed in the `BPTOracle::getUSDPrice` This could lead to a user depositing on behalf of our contracts address(0) which breaks the protocol logic and more gas fees since we are now calling two functions - `Omnipool::depositFor` and `BPTOracle::getUSDPrice` instead of just one.

**Impact:** This can lead to a break in protocol structure and more gas usage.

**Proof Of Concept:** (Proof of concept)

```javascrypt
     function depositFor(uint256 _amountIn, address _depositFor, uint256 _minLpReceived) public {

        if (lastTransactionBlock[_depositFor] == block.number) {
            revert CantDepositAndWithdrawSameBlock();
        }

        if (lastTransactionBlock[_depositFor] == block.number) {
            revert CantDepositAndWithdrawSameBlock();
        }

        underlyingToken.forceApprove(address(auraRewardPoolDepositWrapper), _amountIn);

        (
            uint256 beforeTotalUnderlying,
            uint256 beforeAllocatedBalance,
            uint256[] memory beforeAllocatedPerPool
        ) = _getTotalAndPerPoolUnderlying(underlyingPrice);

        uint256 exchangeRate = _exchangeRate(beforeTotalUnderlying);

        underlyingToken.safeTransferFrom(msg.sender, address(this), _amountIn);

        _depositToAura(beforeAllocatedBalance, beforeAllocatedPerPool, _amountIn);

        (uint256 afterTotalUnderlying,, uint256[] memory afterAllocatedPerPool) =
            _getTotalAndPerPoolUnderlying(underlyingPrice);

        uint256 underlyingBalanceIncrease = afterTotalUnderlying - beforeTotalUnderlying;
        uint256 mintableUnderlyingAmount = _min(_amountIn, underlyingBalanceIncrease);
        uint256 lpReceived = mintableUnderlyingAmount.divDown(exchangeRate);
        require(lpReceived >= _minLpReceived, "too much slippage");

        lpToken.mint(_depositFor, lpReceived);

        totalDeposited += _amountIn;

        _handleRebalancingRewards(
            msg.sender,
            beforeTotalUnderlying,
            afterTotalUnderlying,
            beforeAllocatedPerPool,
            afterAllocatedPerPool
        );
        lastTransactionBlock[_depositFor] = block.number;

        // emit
        emit result(_amountIn);
    }
```

and

```javscript
    function getUSDPrice(address token) public view returns (uint256 priceInUSD) {
        if (tokenHeartbeat[token] == 0) {
            revert HeartbeatNotSet();
        }

        IOracle priceFeed = IPriceFeed(priceFeedAddress).getPriceFeedFromAsset(token);
        if (address(priceFeed) == address(0)) revert PriceFeedNotFound();
        (, int256 priceInUSDInt,, uint256 updatedAt,) = priceFeed.latestRoundData();
        if (updatedAt + tokenHeartbeat[token] < block.timestamp) revert StalePrice();
        // Oracle answer are normalized to 8 decimals
        uint256 newPrice = _normalizeAmount(uint256(priceInUSDInt), 8);
        return newPrice;
    }
```

**Recommended Mitigation:**

```diff
+   event CantDepositToContractAddress(address depositFor);

   function getUSDPrice(address token) public view returns (uint256 priceInUSD) public {
-    if (address(priceFeed) == address(0)) revert PriceFeedNotFound();
   }

  function depositFor(uint256 _amountIn, address _depositFor, uint256 _minLpReceived) public {
+   if (_depositFor == address(0)) revert CantDepositToContractAddress();
    }
```



### [I-1] Two similar reverts statement in a function leading to more gas usage.

**Description:** The `Omnipool::depositFor` is calling two similar revert statements and this could lead to a higher gas usage.

Both revert statements are doing the same thing, except they are been called twice which is't necessary.

**Impact:** This leads to excesive gas usage which is'nt ideal for the protocol.

The below function shows the `Omnipool::depositFor::cantDepositAndWithdrawSameBlock` is been called twice.

**Proof Of Concept:** (Proof of concept)

```javascrypt
     function depositFor(uint256 _amountIn, address _depositFor, uint256 _minLpReceived) public {

        if (lastTransactionBlock[_depositFor] == block.number) {
            revert CantDepositAndWithdrawSameBlock();
        }

        if (lastTransactionBlock[_depositFor] == block.number) {
            revert CantDepositAndWithdrawSameBlock();
        }

        underlyingToken.forceApprove(address(auraRewardPoolDepositWrapper), _amountIn);

        (
            uint256 beforeTotalUnderlying,
            uint256 beforeAllocatedBalance,
            uint256[] memory beforeAllocatedPerPool
        ) = _getTotalAndPerPoolUnderlying(underlyingPrice);

        uint256 exchangeRate = _exchangeRate(beforeTotalUnderlying);

        // @auditL check for reentrancy. not following CEI principle
        // Transfer underlying token to this contract
        underlyingToken.safeTransferFrom(msg.sender, address(this), _amountIn);

        _depositToAura(beforeAllocatedBalance, beforeAllocatedPerPool, _amountIn);

        (uint256 afterTotalUnderlying,, uint256[] memory afterAllocatedPerPool) =
            _getTotalAndPerPoolUnderlying(underlyingPrice);

        uint256 underlyingBalanceIncrease = afterTotalUnderlying - beforeTotalUnderlying;
        uint256 mintableUnderlyingAmount = _min(_amountIn, underlyingBalanceIncrease);
        uint256 lpReceived = mintableUnderlyingAmount.divDown(exchangeRate);
        require(lpReceived >= _minLpReceived, "too much slippage");

        lpToken.mint(_depositFor, lpReceived);

        totalDeposited += _amountIn;

        _handleRebalancingRewards(
            msg.sender,
            beforeTotalUnderlying,
            afterTotalUnderlying,
            beforeAllocatedPerPool,
            afterAllocatedPerPool
        );
        lastTransactionBlock[_depositFor] = block.number;

        // emit
        emit result(_amountIn);
    }
```

**Recommended Mitigation:**

```diff
-   if (lastTransactionBlock[_depositFor] == block.number) {
-            revert CantDepositAndWithdrawSameBlock();
-   }

+
```


**Description:** The `Omnipool::desactive` function is spelt wrongly. The function is also a access controls  

Both revert statements are doing the same thing, except they are been called twice which is't necessary.

**Impact:** This leads to excesive gas usage which is'nt ideal for the protocol.

The below function shows the `Omnipool::depositFor::cantDepositAndWithdrawSameBlock` is been called twice.

**Proof Of Concept:** (Proof of concept)

```javascrypt
     function depositFor(uint256 _amountIn, address _depositFor, uint256 _minLpReceived) public {

        if (lastTransactionBlock[_depositFor] == block.number) {
            revert CantDepositAndWithdrawSameBlock();
        }

        if (lastTransactionBlock[_depositFor] == block.number) {
            revert CantDepositAndWithdrawSameBlock();
        }

        underlyingToken.forceApprove(address(auraRewardPoolDepositWrapper), _amountIn);

        (
            uint256 beforeTotalUnderlying,
            uint256 beforeAllocatedBalance,
            uint256[] memory beforeAllocatedPerPool
        ) = _getTotalAndPerPoolUnderlying(underlyingPrice);

        uint256 exchangeRate = _exchangeRate(beforeTotalUnderlying);

        // @auditL check for reentrancy. not following CEI principle
        // Transfer underlying token to this contract
        underlyingToken.safeTransferFrom(msg.sender, address(this), _amountIn);

        _depositToAura(beforeAllocatedBalance, beforeAllocatedPerPool, _amountIn);

        (uint256 afterTotalUnderlying,, uint256[] memory afterAllocatedPerPool) =
            _getTotalAndPerPoolUnderlying(underlyingPrice);

        uint256 underlyingBalanceIncrease = afterTotalUnderlying - beforeTotalUnderlying;
        uint256 mintableUnderlyingAmount = _min(_amountIn, underlyingBalanceIncrease);
        uint256 lpReceived = mintableUnderlyingAmount.divDown(exchangeRate);
        require(lpReceived >= _minLpReceived, "too much slippage");

        lpToken.mint(_depositFor, lpReceived);

        totalDeposited += _amountIn;

        _handleRebalancingRewards(
            msg.sender,
            beforeTotalUnderlying,
            afterTotalUnderlying,
            beforeAllocatedPerPool,
            afterAllocatedPerPool
        );
        lastTransactionBlock[_depositFor] = block.number;

        // emit
        emit result(_amountIn);
    }
```

**Recommended Mitigation:**

```diff
-   if (lastTransactionBlock[_depositFor] == block.number) {
-            revert CantDepositAndWithdrawSameBlock();
-   }

+
```