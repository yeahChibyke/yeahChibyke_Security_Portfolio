![](../logo.png)
<img src="../img/oneshot.png" alt="OneShot" height="320" />

# Security Report for the OneShot CodeHawks FirstFlight Contest \*_shadow audit_

## High Issues

### [H-1] It is possible to challenge with another rapper's NFT in `RapBattle::goOnStageOrBattle()` function

**Description:**

The `goOnStageOrBattle()` function allows a rapper to battle rap, either as a defender or challenger. But, it is possible for a challenger to go up against a defender with the NFT of another rapper. And this is because for a challenger, there is no check for the ownership of the rapper NFT.

**Impact:**

A prankster can take advantage of another rapper's well-trained NFT and use it to challenge a defender and claim the CredTokens if they win.

**PoC:**

Add the following test to the `OneShotTest.t.sol` file:

```solidity
    function test_battle_prank() public {
        address Thug = makeAddr("Thugnificient");
        address Gangsta = makeAddr("Gangstalicious");
        address Riley = makeAddr("Riley");

        // First 2 rappers mint NFTs
        vm.prank(Thug);
        oneShot.mintRapper();
        vm.prank(Gangsta);
        oneShot.mintRapper();

        // Time for battle... Thug wants to battle Riley
        vm.startPrank(Thug);
        oneShot.approve(address(rapBattle), 0);
        rapBattle.goOnStageOrBattle(0, 0);
        vm.stopPrank();

        // Riley steps up to battle Thug, but he uses Gangsta's NFT instead of his
        vm.startPrank(Riley);
        rapBattle.goOnStageOrBattle(1, 0);
    }
```

**Recommended Mitigation:**

Add a check in the `goOnStageOrBattle()` function to ensure that the challenger is the owner of the supplied NFT

```diff
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        ...
        ...

        else {
            // credToken.transferFrom(msg.sender, address(this), _credBet);
+           require(msg.sender == oneShotNft.ownerOf(_tokenId));
            _battle(_tokenId, _credBet);
        }
    }
```

### [H-2] A rapper can avoid losing their `CredTokens`when they are the challengers in `RapBattle::goOnStageOrBattle()` by not approving

**Description:**

In the `goOnStageOrBattle()` function, a rapper is assigned as `challenger` if there is already a `defender`.  
If there is already a `defender` at the time a rapper calls the `goOnStageOrBattle()` function, the `RapBattle::_battle()` internal function, and it is in this internal function that the winner is determined.  
But because the `challenger` does not grant `CredToken` approval to the `Rapbattle` contract, the function ends up performing unexpectedly.

**Impact:**

A `challenger` suffers no risk to their `CredToken` when they call the `goOnStageorBattle()` function.  
A `defender` gets no winnings from a battle they win, instead they lose it to the `RapBattle` contract.

**PoC:**

Add the following test to the `OneShotTest.t.sol` file:

```solidity
    function test_token_prank() public twoSkilledRappers {
        uint256 userInitBal = cred.balanceOf(user);
        uint256 challengerInitBal = cred.balanceOf(challenger);
        uint256 rapBattleInitBal = cred.balanceOf(address(rapBattle));

        vm.startPrank(user);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), 10);
        rapBattle.goOnStageOrBattle(0, 3);
        vm.stopPrank();

        vm.startPrank(challenger);
        oneShot.approve(address(rapBattle), 1);

        vm.warp(1 days);
        vm.expectRevert();
        rapBattle.goOnStageOrBattle(1, 3);
        vm.stopPrank();

        uint256 userFinalBal = cred.balanceOf(user);
        uint256 challengerFinalBal = cred.balanceOf(challenger);
        uint256 rapBattleFinalBal = cred.balanceOf(address(rapBattle));

        assert(challengerFinalBal == challengerInitBal); // challenger's token balance remains unchanged
        assert(userInitBal > userFinalBal); // user loses tokens despite them being winners of rap battles
        assert(rapBattleFinalBal > rapBattleInitBal); // rap battle contract gains user's tokens
    }
```

**Recommended Mitigation:**

Ensure `challenger` gives approval before going into battle.  
This is done by calling `CredToken::transferFrom()` before `CredToken::_battle()`

```diff
    function goOnStageOrBattle(uint256 _tokenId, uint256 _credBet) external {
        ...
        ...
        else {
-           // credToken.transferFrom(msg.sender, address(this), _credBet);
+           credToken.transferFrom(msg.sender, address(this), _credBet);
            _battle(_tokenId, _credBet);
        }
    }
```

Then update the `CredToken::_battle()` function to transfer the `totalPrize` of tokens since both the `defender` and `challenger` would have already transferred their tokens

```diff
    function _battle(uint256 _tokenId, uint256 _credBet) internal {
        ...
        ...
        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
-           credToken.transfer(_defender, defenderBet);
-           credToken.transferFrom(msg.sender, _defender, _credBet);
+           credToken.transfer(_defender, totalPrize);
        } else {
-           // Otherwise, since the challenger never sent us the money, we just give the money in the contract
-           credToken.transfer(msg.sender, _credBet);
+           // Otherwise, send the totalPrize to the challenger
+           credToken.transfer(msg.sender, totalPrize);
        }
        totalPrize = 0;
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }
```

### [H-3] A user can mint multiple rapper NFTs and use them to game the system

**Description:**

According to the `README` and all available `NatSpec`, it can be deduced that the overall flow and features of the protocol is built on the assumption that users mint only one rapper NFT. But, since there is no restriction on how many NFTs a user can mint, a malicius user can mint as many NFTs as they want, and use them to game the system.

Also, it is possible for users to transfer their NFTs to another user.

**Impact:**

A user can mint as many NFTs as they want, stake them, and then use them to battle rap.... even against themselves. SInce it is a number's game, they stand to gain more than they lose.   
If this is a user who is aware of the other security risks that exist in the system, they can use this knowledge and their multiple NFTs for great ruin of the protocol.  
Also, since users can transfer NFTs, a malicious user can transfer their NFts to an accomplice to an accomplice, who can then go ahead to battle rap with this NFT, win some `CredToken`, and then send the NFT back to the owner.

**Recommended Mitigation:**

There would need to be a rethink of the protocol. Guidelines would have to be put in place to prevent against multiple minting as well as users being able to transfer minted NFTs to other users.

## Medium Issues

### [M-1] `Streets::unstake()` function mints incorrect amount of `CredToken` to stakers

**Description:**

The `unstake()` allows rappers to unstake their NFTs and get `CredToken` according to how many days they were staked... for a max of 4 days.   
But, whereas `CredToken` is an `ERC20` with decimal place of `18`, the `unstake()` function is rewarding the rappers `0.000000000000000001` `CredToken`, instead of `1e18` `CredToken`.

```solidity
    function unstake(uint256 tokenId) external {
        ...
        ...
        // Apply changes based on the days staked
        if (daysStaked >= 1) {
            stakedRapperStats.weakKnees = false;
-->         credContract.mint(msg.sender, 1);
        }
        if (daysStaked >= 2) {
            stakedRapperStats.heavyArms = false;
-->         credContract.mint(msg.sender, 1);
        }
        if (daysStaked >= 3) {
            stakedRapperStats.spaghettiSweater = false;
-->         credContract.mint(msg.sender, 1);
        }
        if (daysStaked >= 4) {
            stakedRapperStats.calmAndReady = true;
-->         credContract.mint(msg.sender, 1);
        }
        ...
        ...
    }
```

**Impact:**

Rappers are rewarded with way less amount of `CredToken` than they should be getting.

**PoC**:

Add the following test to the `OneShotTest.t.sol` file:

```solidity
    function test_token_error() public mintRapper {
        vm.startPrank(user);
        oneShot.approve(address(streets), 0);
        streets.stake(0);
        vm.stopPrank();
        vm.warp(1 days + 1);
        vm.startPrank(user);
        streets.unstake(0);

        assert(cred.balanceOf(user) == 1);
        assert(cred.balanceOf(user) != 1e18);
    }
```

**Recommended Mitigation:**

Refactor the reward maths of the `unstake()` function as so:

```diff
    function unstake(uint256 tokenId) external {
        ...
        ...
        // Apply changes based on the days staked
        if (daysStaked >= 1) {
            stakedRapperStats.weakKnees = false;
-           credContract.mint(msg.sender, 1);
+           credContract.mint(msg.sender, 1e18);
        }
        if (daysStaked >= 2) {
            stakedRapperStats.heavyArms = false;
-           credContract.mint(msg.sender, 1);
+           credContract.mint(msg.sender, 1e18);
        }
        if (daysStaked >= 3) {
            stakedRapperStats.spaghettiSweater = false;
-           credContract.mint(msg.sender, 1);
+           credContract.mint(msg.sender, 1e18);
        }
        if (daysStaked >= 4) {
            stakedRapperStats.calmAndReady = true;
-           credContract.mint(msg.sender, 1);
+           credContract.mint(msg.sender, 1e18);
        }
        ...
        ...
    }
```

