![](logo.png)
![](./img/t-swap.png)

# Security Report for the TSwap Protocol **learning*

**Source:** https://github.com/Cyfrin/5-t-swap-audit

------------------------------------------------------

## High Issues

### [H-1] Incorrect fee calculation in `TSwapPool::getInputAmountBasedOnOutput()` causes protocol to take too many tokens from users, resulting in lost fees

**Decsription:**

The `getInputAmountBasedOnOutput()` function is intended to calculate the amount of tokens a user should deposit based on an amount of output tokens. However this function currently miscalculates the resulting amount. When calculating the fee, it scales the amount by `10_000` instead of `1_000`.

**Impact:**

Protocol takes more fees than expected from users.

**Recommended Mitigation:**

```diff
    function getInputAmountBasedOnOutput(
        uint256 outputAmount,
        uint256 inputReserves,
        uint256 outputReserves
    )
        public
        pure
        revertIfZero(outputAmount)
        revertIfZero(outputReserves)
        returns (uint256 inputAmount)
    {
-       return ((inputReserves * outputAmount) * 10000) / ((outputReserves - outputAmount) * 997); 
+       return ((inputReserves * outputAmount) * 1000) / ((outputReserves - outputAmount) * 997); 
    }
```



## Medium Issues

### [M-1] `TSwapPool::deposit()` is missing deadline checkcausing transactions to complete even after the deadline

**Description:**

The `TSwapPool::deposit()` function accepts a `deadline` parameter, which according to documentation is the "deadline for the transaction to be completed by". However, because this parameter is never used, transactions that liquidity to the pool could be executed at unexpected times. Especially in market conditions where the deposit rate is unfavourable.

**Impact:**

Transactions could take place when market conditions are unfavourable for them, despite a `deadline` parameter being inputted.

**Tools Used:**

- Solidity Compiler

**Recommended Mitigation:**

Implement the `revertIfDeadlinePassed()` modifier

```diff
        function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
        revertIfZero(wethToDeposit)
+       revertIfDeadlinePassed(deadline)
        returns (uint256 liquidityTokensToMint)
    {
        if (wethToDeposit < MINIMUM_WETH_LIQUIDITY) {
            revert TSwapPool__WethDepositAmountTooLow(MINIMUM_WETH_LIQUIDITY, wethToDeposit);
        }
        if (totalLiquidityTokenSupply() > 0) {
            uint256 wethReserves = i_wethToken.balanceOf(address(this));
            uint256 poolTokenReserves = i_poolToken.balanceOf(address(this)); 
            uint256 poolTokensToDeposit = getPoolTokensToDepositBasedOnWeth(wethToDeposit);
            if (maximumPoolTokensToDeposit < poolTokensToDeposit) {
                revert TSwapPool__MaxPoolTokenDepositTooHigh(maximumPoolTokensToDeposit, poolTokensToDeposit);
            }
            liquidityTokensToMint = (wethToDeposit * totalLiquidityTokenSupply()) / wethReserves;
            if (liquidityTokensToMint < minimumLiquidityTokensToMint) {
                revert TSwapPool__MinLiquidityTokensToMintTooLow(minimumLiquidityTokensToMint, liquidityTokensToMint);
            }
            _addLiquidityMintAndTransfer(wethToDeposit, poolTokensToDeposit, liquidityTokensToMint);
        } else {
            _addLiquidityMintAndTransfer(wethToDeposit, maximumPoolTokensToDeposit, wethToDeposit);
            liquidityTokensToMint = wethToDeposit;
        }
    }
```



## Low Issues

### [L-1] `TSwapPool::_addLiquidityMintAndTransfer()` emits event in the wrong order

**Description:**

When the `LiquidityAdded()` event is emitted in the `TSwapPool::_addLiquidityMintAndTransfer()` function, valuea are logged in the wrong order.

**Impact:**

Event emission is incorrect, and this leads to malfunction in oof-chain functionality.

**Tools used:**

- Manual Review

**Recommended Mitigation:**

Rearrange the values so that they can be logged correctly

```diff
    function _addLiquidityMintAndTransfer(
        uint256 wethToDeposit,
        uint256 poolTokensToDeposit,
        uint256 liquidityTokensToMint
    )
        private
    {
        _mint(msg.sender, liquidityTokensToMint);
-       emit LiquidityAdded(msg.sender, poolTokensToDeposit, wethToDeposit); 
+       emit LiquidityAdded(msg.sender, wethToDeposit, poolTokensToDeposit); 

        // Interactions
        i_wethToken.safeTransferFrom(msg.sender, address(this), wethToDeposit);
        i_poolToken.safeTransferFrom(msg.sender, address(this), poolTokensToDeposit);
    }
```

## Gas Issues



## Informational Issues

### [I-1] `PoolFactory::PoolFactory__PoolDoesNotExist()` is not used and should be removed

```diff
-   error PoolFactory__PoolDoesNotExist(address tokenAddress);
```

### [I-2] `PoolFactory::constructor()` lacks zero-address check

```diff
    constructor(address wethToken) {
+        require(wethToken != address(0));
        i_wethToken = wethToken; // @BUG -- lacking zero address check
    }
```

### [I-3] `PoolFactory::createPool()` should use `.symbol()` instead of `.name()`

```diff
    function createPool(address tokenAddress) external returns (address) {
        if (s_pools[tokenAddress] != address(0)) {
            revert PoolFactory__PoolAlreadyExists(tokenAddress);
        }
        string memory liquidityTokenName = string.concat("T-Swap ", IERC20(tokenAddress).name());
-        string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).name());
+        string memory liquidityTokenSymbol = string.concat("ts", IERC20(tokenAddress).symbol());
        TSwapPool tPool = new TSwapPool(tokenAddress, i_wethToken, liquidityTokenName, liquidityTokenSymbol);
        s_pools[tokenAddress] = address(tPool);
        s_tokens[address(tPool)] = tokenAddress;
        emit PoolCreated(tokenAddress, address(tPool));
        return address(tPool);
    }
```

