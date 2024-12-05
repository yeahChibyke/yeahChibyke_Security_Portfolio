![](../logo.png)
<img src="../img/soulmate.png" alt="Soulmate" height="320" />

# Security Report for the Soulmate CodeHawks FirstFlight Contest **shadow audit*

## High Issues

### [H-1] Message written by soulmates can be read by literally anyone. Including addresses that own no NFT

**Description:**

Per the NatSpec of the `Soulmate::readMessageInShredSpace()` function;

```solidity
    /// @notice Allows any soulmates with the same NFT ID to read in a shared space on blockchain.
    function readMessageInSharedSpace() external view returns (string memory) {
        // code
    } 
```

only soulmates with the same NFT id are able to read messages shared between themselves.

But apparently, this is not so. Any third party address can read the shared message, including addresses that are not even users of the protocol.

**Impact:**

The privacy of the shared messages is not acheieved as these messages are visible are to anyone. Content meant for only `soulmates` is now visible to all and sundry. 

**Proof of Concept:**

Add the following test to `SolmateTest.t.sol`:

```solidity
    function test_ThirdPartyCanReadWrittenMessageInSharedSpace() public {
        // soulmate1 writes message
        vm.prank(soulmate1);
        soulmateContract.writeMessageInSharedSpace("Sign up on Updraft");

        // thirdParty suucessfully reads message meant for soulmate2
        address thirdParty = makeAddr("thirdParty");

        vm.prank(thirdParty);
        string memory message = soulmateContract.readMessageInSharedSpace();

        string[4] memory possibleText =
            ["Sign up on Updraft, sweetheart", "Sign up on Updraft, darling", "Sign up on Updraft, my dear", "Sign up on Updraft, honey"];
        bool found;
        for (uint256 i; i < possibleText.length; i++) {
            if (compare(possibleText[i], message)) {
                found = true;
                break;
            }
        }
        console2.log(message);
        assertTrue(found);
    }
```

**Recommended Mitigation:**

*Will come back to you later*

### [H-2] Error in divorce check in `Airdrop::claim()` function

**Description:** 

There is a `soulmateContract.isDivorced()` check in the `claim()` function to ensure that divorced soulmates cannot claim `LoveToken` from the airdrop.

However, because the `Soulmate::isDivorced()` function returns the divorce status of the caller, it means that when called in the `claim()` function, it is checking the divorce status of the `Airdrop` contract address. And that address can never be in the `Soulmate::divorced[]` mapping because it is not a `soulmate`.

**Impact:**

A divorced `soulmate` can claim `LoveTokens`

**PoC:**

Add the following test to the `AirdropTest.t.sol` file:

```solidity
    function testCanClaimEvenWhenDivorced() public {
        _mintOneTokenForBothSoulmates();

        vm.startPrank(soulmate1);
        soulmateContract.getDivorced();
        assert(soulmateContract.isDivorced() == true);

        vm.warp(block.timestamp + 1000 days + 1 seconds);

        // attempt to claim airdrop despite being divorced
        airdropContract.claim();

        // confirm that claim was indeed successful
        assert(loveToken.balanceOf(soulmate1) == 1000e18);
        vm.stopPrank();

        // confirm that soulmate2 can also claim airdrop depsite being divorced
        vm.prank(soulmate2);
        airdropContract.claim();

        // confirm that claim was successful
        assert(loveToken.balanceOf(soulmate2) == 1000e18);
    }
```

**Recommended Mitigation:**

Modify the `Soulmate::isDivorced()` function to check the divorce status of an inputted address instead of the divorce status of the `msg.sender`:

```diff
-   function isDivorced() public view returns (bool) {
+   function isDivorced(address _soulmate) public view returns (bool) {
-       return divorced[msg.sender];
+       return divorced[_soulmate];
    } 
```

Modify the `ISoulmate` interface:

```diff
-   function isDivorced() external view returns (bool);
+   function isDivorced(address _soulmate) external view returns (bool);
```

Modify the `claim()` function of the `Airdrop` contract:

```diff
    function claim() public {
        // No LoveToken for people who don't love their soulmates anymore.
-       if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();
+       if (soulmateContract.isDivorced(msg.sender)) revert Airdrop__CoupleIsDivorced();

        // Calculating since how long soulmates are reunited
        uint256 numberOfDaysInCouple = (
            block.timestamp - soulmateContract.idToCreationTimestamp(soulmateContract.ownerToId(msg.sender))
        ) / daysInSecond;

        uint256 amountAlreadyClaimed = _claimedBy[msg.sender];

        if (amountAlreadyClaimed >= numberOfDaysInCouple * 10 ** loveToken.decimals()) {
            revert Airdrop__PreviousTokenAlreadyClaimed();
        }

        uint256 tokenAmountToDistribute = (numberOfDaysInCouple * 10 ** loveToken.decimals()) - amountAlreadyClaimed;

        // Dust collector
        if (tokenAmountToDistribute >= loveToken.balanceOf(address(airdropVault))) {
            tokenAmountToDistribute = loveToken.balanceOf(address(airdropVault));
        }
        _claimedBy[msg.sender] += tokenAmountToDistribute;

        emit TokenClaimed(msg.sender, tokenAmountToDistribute);

        loveToken.transferFrom(address(airdropVault), msg.sender, tokenAmountToDistribute);
    }
```

### [H-3] It is possible to claim airdrop without having a Soulbound NFT due to lack of validation in `Airdrop::claim()` function

**Description:**

There are no checks in the `Airdrop::claim()` function to prevent a user without a Soulbound NFT from claiming a `LoveToken`.  

The `numberOfDaysInCouple` is calculated by using the particular timestamp of when the NFT was minted, but for a user without a minted NFT, it returns the NFT of the first minted NFT.

```solidity
    function claim() public {
        // No LoveToken for people who don't love their soulmates anymore.
        if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();

        // Calculating since how long soulmates are reunited
-->     uint256 numberOfDaysInCouple = (
            block.timestamp - soulmateContract.idToCreationTimestamp(soulmateContract.ownerToId(msg.sender))
        ) / daysInSecond;

        ...
        ...
    }
```

**Impact:**

An attacker without a Soulbound NFT can claim rewards from the airdrop, because the function will think they have the first minted NFT.

**PoC:**

Add the following test to the `AirdropTest.t.sol` file:

```solidity
    function test_can_claim_without_soulboundnft() public {
        address bonnie = makeAddr("bonnie");

        console2.log("This is the balance of bonnie before airdrop claim: ", loveToken.balanceOf(bonnie));

        vm.warp(block.timestamp + 1 days + 1 seconds);

        vm.prank(bonnie);
        airdropContract.claim();

        console2.log("This is the balance of bonnie after airdrop claim: ", loveToken.balanceOf(bonnie) / 1e18);
    }
```

**Recommended Mitigation:**

Modify the `Airdrop::claim()` function to check that the caller indeed has a Soulbound NFT.

```diff
    function claim() public {
        // No LoveToken for people who don't love their soulmates anymore.
        if (soulmateContract.isDivorced()) revert Airdrop__CoupleIsDivorced();

+       require(soulmateContract.soulmateOf(msg.sender) != address(0), "Caller has not minted a Soulbound NFT");

        // Calculating since how long soulmates are reunited
        uint256 numberOfDaysInCouple = (
            block.timestamp - soulmateContract.idToCreationTimestamp(soulmateContract.ownerToId(msg.sender))
        ) / daysInSecond;

        ...
        ...
    }
```

*P.S.  Ensure that the mitigation involving divorce check is also implemented*

