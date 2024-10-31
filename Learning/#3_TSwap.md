![](../logo.png)
![](../img/t-swap.png)

# Security Report for the TSwap Protocol **learning*

**Source:** https://github.com/Cyfrin/5-t-swap-audit

------------------------------------------------------

## High Issues

### [H-1] Incorrect fee calculation in `TSwapPool::getInputAmountBasedOnOutput()` causes protocol to take too many tokens from users, resulting in lost fees

**Decsription:**

The `getInputAmountBasedOnOutput()` function is intended to calculate the amount of tokens a user should deposit based on an amount of output tokens. However this function currently miscalculates the resulting amount. When calculating the fee, it scales the amount by `10_000` instead of `1_000`.

**Impact:**

Protocol takes more fees than expected from users.

**Proof of Concept:**

```solidity
    function testFlaw() public {
        uint256 initLiquidity = 100e18;
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), initLiquidity);
        poolToken.approve(address(pool), initLiquidity);

        pool.deposit({
            wethToDeposit: initLiquidity,
            minimumLiquidityTokensToMint: 0,
            maximumPoolTokensToDeposit: initLiquidity,
            deadline: uint64(block.timestamp)
        });
        vm.stopPrank();

        // Creat a new user and mint them 11 pool tokens
        address Alice = makeAddr("Alice");
        uint256 initPoolTokenBal = 11e18;
        poolToken.mint(Alice, initPoolTokenBal);

        // Alice buys 1 WETH from the pool and pays with pool tokens
        vm.startPrank(Alice);
        poolToken.approve(address(pool), type(uint256).max);
        pool.swapExactOutput({
            inputToken: poolToken,
            outputToken: weth,
            outputAmount: 1e18,
            deadline: uint64(block.timestamp)
        });

        // initial liquidity was 1:1, so Alice should have paid ~1 pool token
        // However, much more than that was spent. Alice started with 11 pool tokens, yet is ending up with way less
        // than expected amount of pool tokens
        assertLt(poolToken.balanceOf(Alice), 1e18);
        vm.stopPrank();

        //  liquidity provider can rug all funds from the pool, including those deposited by Alice
        vm.startPrank(liquidityProvider);
        pool.withdraw({
            liquidityTokensToBurn: pool.balanceOf(liquidityProvider),
            minWethToWithdraw: 1e18,
            minPoolTokensToWithdraw: 1e18,
            deadline: uint64(block.timestamp)
        });

        assert(weth.balanceOf(address(pool)) == 0);
        assert(poolToken.balanceOf(address(pool)) == 0);
    }
```

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

### [H-2] `TSwapPool::deposit()` is missing deadline check causing transactions to complete even after the deadline

**Description:**

The `deposit()` function accepts a `deadline` parameter, which according to documentation is the *"deadline for the transaction to be completed by"*. However, because this parameter is never used, transactions that deposit liquidity to the pool could be executed at unexpected times. Especially in market conditions where the deposit rate is unfavourable.

**Impact:**

Transactions could take place when market conditions are unfavourable for them, despite a `deadline` parameter being inputted.

**Tools Used:**

- Solidity Compiler

**Recommended Mitigation:**

Implement the `TSwapPool::revertIfDeadlinePassed()` modifier

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

# [H-3] No slippage protection in `TSwapPool::swapExactOutput()` will cause users to receive way fewer tokens

**Description:**

The `swapExactOutput()` function does not include any sort of slippage protection. This function has somewhat similar logic to `TSwapPool::swapExactInput()`where that particular function has `minOutputAmount` specified. In like manner, the `swapExactOutput()` function should also have a `maxInputAmount` specified.

**Impact:**

If market conditions change before the transaction changes, user could get a much worse swap.

**Proof of Concept:**

- Assume the price of WETH is currently 5_000 USDC
- User calls `swapExactOutput()` looking for exactly 1 WETH
  - inputToken: USDC
  - outputToken: WETH
  - outputAmount: 1
  - deadline: whatever
- The function does not offer a `maxInputAmount`
- The transaction is pending in the mempool. Market changes!!!
- Assume price moves: 1 WETH => 20_000 USDC
- The transaction completes, but the user ensd up sending the protocol 20_000 USDC instead of the expected 5_000 USDC

**Recommended Mitigation:**

A `maxInputAmount` should be included so that the user only has to spend up to a specific amount, and can predict how much they will spend on a `swap`.

```diff
    function swapExactOutput(
        IERC20 inputToken,
        IERC20 outputToken,
        uint256 outputAmount,
+       uint256 maxInputAmount,
        uint64 deadline
    )
        public
        revertIfZero(outputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 inputAmount)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

+       if(inputAmount > maxInputAmount) {
+           revert();
+       }
        inputAmount = getInputAmountBasedOnOutput(outputAmount, inputReserves, outputReserves);

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```

# [H-4] Error in function call within `TSwapPool::sellPoolTokens()` function leads to users receiving wrong amount of tokens

**Description::**

The `sellPoolTokens()` function is intended to allow users to easily pool tokens -by specifying the amount of tokens they want to sell in the `poolTokenAmount` parameter-, and receive WETh in exchange. However the function currently miscalculates the swapped amount.

This is due to the fact that the `TSwapPool::swapExactOutput()` function is called within the `sellPoolTokens()` function. The `TSwapPool::swapExactInput()` is the right function to be called, and this is because users specify the exact amounts of input tokens they want to sell.

**Impact:**

Users will swap the wrong amount of tokens, which is a sever disruption of protocol functionality.

**Recommended Mitigation:**

Consider changing the implementation to use `swapExactInput()` instead of `swapExactOutput()`. Note that this would also require changing the `sellPoolTokens()` function to accept a new parameter (ie `minWethToReceive` to be passed to `swapExactInput()`)

```diff
    function sellPoolTokens(
    uint256 poolTokenAmount,
+   uint256 minWethToReceive,    
    ) external returns (uint256 wethAmount) {
-       return swapExactOutput(i_poolToken, i_wethToken, poolTokenAmount, uint64(block.timestamp));
+       return swapExactInput(i_poolToken, poolTokenAmount, i_wethToken, minWethToReceive, uint64(block.timestamp));
    }
```

# [H-5] The extra tokens given to users after every `swap_count` in the `TSwapPool::_swap()` breaks the protocol invariant of `x * y = k`

**Description:**

The protocol follows a strict invariant of `x * y = k`, however this invariant is broken due to the extra incentive in the `_swap()`. Meaning that over time, the protocol funds will be drained.

Guilty block of code:

```solidity
    swap_count++;
        if (swap_count >= SWAP_COUNT_MAX) {
            swap_count = 0;
            outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
        }
```

**Impact:**

A user could malicioulsy drain the protocol of funds by doing a lot of swaps and collecting the extra incentive given out by the protocol. Thus, breaking the protocol's core invariant.

**Proof of Concept:**

1. A user swaps 10 times and collects all the extra incentive of `1_000_000_000_000_000_000`
2. User continues to swap until all protocol funds are drained

**Proof of Code:**

Place the following into `TSwapPool.t.sol`

```solidity
    function testInvariantBroken() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        uint256 outputWeth = 1e17;

        vm.startPrank(user);
        poolToken.approve(address(pool), type(uint256).max);
        poolToken.mint(user, 100e18);
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));

        int256 startingY = int256(weth.balanceOf(address(pool)));
        int256 expectedDeltaY = int256(-1) * int256(outputWeth);

        pool.swapExactOutput(poolToken, weth, outputWeth, uint64(block.timestamp));
        vm.stopPrank();

        uint256 endingY = weth.balanceOf(address(pool));
        int256 actualDeltaY = int256(endingY) - int256(startingY);
        assertEq(actualDeltaY, expectedDeltaY);
    }
```

**Recommended Mitigation:**

Remove the extra incentive mechanism. If you want to keep this in, we should account for the change in the x * y = k protocol invariant. Or, we should set aside tokens in the same way we do with fees.

```diff
-   swap_count++;
-       if (swap_count >= SWAP_COUNT_MAX) {
-           swap_count = 0;
-           outputToken.safeTransfer(msg.sender, 1_000_000_000_000_000_000);
-       }
```


## Medium Issues

### [M-1] Rebase, fee-on-transfer, and ERC777 tokens break protocol invariant

I need to check out Solodit on tips for better understanding

## Low Issues

### [L-1] `TSwapPool::_addLiquidityMintAndTransfer()` emits event in the wrong order

**Description:**

When the `TSwapPool::LiquidityAdded()` event is emitted in the `_addLiquidityMintAndTransfer()` function, valuea are logged in the wrong order.

**Impact:**

Event emission is incorrect, and this leads to malfunction in off-chain functionality.

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

### [L-2] `Default value returned by `TSwapPool::swapExactInput()` results in incorrect return value given

**Description:**

The `swapExactInput()` function is expected to return the actual amount of tokens bought by the caller. However, while it declares the named return value `output`, it is never assigned a value nor use an explicit return statement.

**Impact:**

The `return` value will always be `0`, giving incorrect information to the `caller`.

**Tools Used:**

- Solidity Compiler
- Manual Review

**Mitigation:**

```diff
    function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
        public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 output)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

-       uint256 outputAmount = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves); 
+       output = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);

-       if (outputAmount < minOutputAmount) { 
-           revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount); 
-       } 
+       if (output < minOutputAmount) {
+           revert TSwapPool__OutputTooLow(output, minOutputAmount);
+       }

-       _swap(inputToken, inputAmount, outputToken, outputAmount); 
+       _swap(inputToken, inputAmount, outputToken, output);
    }
```



## Gas Issues

### [G-1] `TSwapPool::swapExactInput()` function should be made `external` since it is not called internally

```diff
    function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
-       public 
+       external
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (
            uint256 output // @BUG -- waste of gas.. possible LOW
        )
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        uint256 outputAmount = getOutputAmountBasedOnInput(inputAmount, inputReserves, outputReserves);

        if (outputAmount < minOutputAmount) {
            revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);
        }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```



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

