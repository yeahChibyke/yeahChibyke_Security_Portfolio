![](../logo.png)
<img src="../img/oneshot.png" alt="OneShot" height="320" />

# Security Report for the OneShot CodeHawks FirstFlight Contest **shadow audit*

## High Issues

### It is possible to challenge with another rapper's NFT in `RapBattle::goOnStageOrBattle()` function

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
