![](../logo.png)
<img src="../img/santa-list.png" alt="Santa List" height="320" />

# Security Report for the Santa's List CodeHawks FirstFlight Contest **shadow audit*

## High Issues

### [H-1] `SantasList::buyPresent()` function  also burns tokens of the `presentReceiver` instead of tokens of the buyer

**Description:**

From the NatSpec of the `buyPresent()` function, anyone who has `santaTokens` can call this function to buy a SantasList NFT for someone else.   
But, from this line of code, it shows that the `buyPresent()` functions burns tokens of the `presentReceiver` as payment for the NFT bought, instead of burning tokens from the buyer.

```diff
    /*
     * @notice Buy a present for someone else. This should only be callable by anyone with SantaTokens.
     * @dev You'll first need to approve the SantasList contract to spend your SantaTokens.
     */
    function buyPresent(address presentReceiver) external {
--> i_santaToken.burn(presentReceiver);
        _mintAndIncrement();
    }
```

**Impact:**

The `buyPresent()` function will always revert if the `presentReceiver` does not have any `santaTokens`.   
And if the `presentReceiver` has `santaTokens`, it will be burned by the function, which is not the intention of the protocol.

**Proof of Concept:**

Add the following test to the `SantasListTest.t.sol` file:

```solidity
    function testBuyPresentBurnsTokensOfPresentReceiver() public {
        deal(address(santaToken), user, 1e18);
        address alice = makeAddr("alice");

        assertEq(santaToken.balanceOf(user), 1e18);
        assertEq(santaToken.balanceOf(alice), 0);

        vm.startPrank(user);
        santaToken.approve(address(santasList), type(uint256).max);
        vm.expectRevert();
        santasList.buyPresent(alice);
        vm.stopPrank();

        assertEq(santaToken.balanceOf(user), 1e18);

        // fund alice some santaTokens
        deal(address(santaToken), alice, 1e18);

        assertEq(santaToken.balanceOf(alice), 1e18);

        // attempt to buy present again
        vm.prank(user);
        santasList.buyPresent(alice);

        assertEq(santaToken.balanceOf(alice), 0); // The santa tokens of alice were burnt
    }
```

**Recommended Mitigation:**

The `buyPresent()` function should be modified to burn tokens of the buyer (i.e., the `msg.sender`) instead of the `presentReceiver`

```diff
    function buyPresent(address presentReceiver) external {
-       i_santaToken.burn(presentReceiver);
+       i_santaToken.burn(msg.sender);
        _mintAndIncrement();
    }
```

### [H-2] `SantasList::buyPresent()` mints the NFT to the caller of the function instead of the `presentReceiver` because of an error in the `SantasList::_mintAndIncrement()` function

**Description:**

The `buyPresent()` is intended to mint a `SantaList` NFT to the `presentReceiver`. It does this by calling the internal `_mintAndIncrement()` function, but an error in this function means that the intended purpose of the `buyPresent()` is not achieved.

```diff
    function _mintAndIncrement() private {
-->     _safeMint(msg.sender, s_tokenCounter++);
    }
```

**Impact:**

The caller of the `buyPresent()` will always end up with the bought NFT, instead of the `presentReceiver`.

**Proof of Concept:**

PS: *This PoC assumes that the recommended mitigation from **[H-1]** has been applied*

```solidity
    function testBuyPresentMintsNFTToBuyerInsteadOfReceiver() public {
        deal(address(santaToken), user, 1e18);
        address alice = makeAddr("alice");

        assert(santasList.balanceOf(user) == 0);
        assert(santasList.balanceOf(alice) == 0);

        vm.startPrank(user);
        santaToken.approve(address(santasList), type(uint256).max);
        santasList.buyPresent(alice);
        vm.stopPrank();

        assert(santasList.balanceOf(user) == 1); // NFT balance of user increased
        assert(santasList.balanceOf(alice) == 0); // NFT balance of alice never increased
    }
```

**Recommended Mitigation:**

Modify the `_mintAndIncrement()` as so 👇🏾,  such that it mints the present to the `presentReceiver` instead of the `msg.sender`

```diff
-   function _mintAndIncrement() private {
-       _safeMint(msg.sender, s_tokenCounter++);
-   }

+   function _mintAndIncrement(address _receiver) private {
+       _safeMint(_receiver, s_tokenCounter++);
+   }
```

P.S: *Modification of the `_mintAndIncrement()` function means that all functions where it is called will also have to be modified.*

Further P.S.: Judging could dispute that [H-1] and [H-2] are duplicates sort of, because root cause is very similar.

### [H-3] `SantasList::checkList()` is callable by anyone instead of by `onlySanta`

**Description:**

As per the NatSpec and project `README`, the `checkList()` function is used to mark a user as either `NAUGHTY` or `NICE`, and it should be callable only by `Santa`. But that is not the case as the function is missing the `onlySanta` modifier.

**Impact:**

Anybody can call the `checkList()` and maliciously set the status of any user to what they please; `NAUGHTY` or `NICE`.

**Proof of Concept:**

Add the following test to `SantasListTest.t.sol` file:

```solidity
    function testAnyoneCanCallCheckList() public {
        address alice = makeAddr("alice");
        address ben = makeAddr("ben");

        vm.prank(alice);
        // alice marks ben as NAUGHTY
        santasList.checkList(ben, SantasList.Status.NAUGHTY);
        assertEq(uint256(santasList.getNaughtyOrNiceOnce(ben)), uint256(SantasList.Status.NAUGHTY));
    }
```

**Recommended Mitigation:**

Implement the `onlySanta` modifier to make the `checkList()` function only callable by `Santa`

```diff
-   function checkList(address person, Status status) external {
+   function checkList(address person, Status status) external onlySanta {
        s_theListCheckedOnce[person] = status;
        emit CheckedOnce(person, status);
    }
```

### [H-4] `NICE` and `EXTRA_NICE` users can call `SantasList::collectPresent()` function multiple times with a nifty little trick

**Description:**

There is a check in the `collectPresent()` function to prevent any user from collecting a present more than once 👇🏾;

```solidity
    if (balanceOf(msg.sender) > 0) {
        revert SantasList__AlreadyCollected();
    } 
```

However this check can be gamed as a winner can transfer their `SantasList NFT` (and `SantaToken` if they wish) to another address, and then call the `collectPresent()` again.   
This by-pass can be performed as many times as possible, especially since the `SantasList NFT` doesn't have a `initialSupply`. So, there is a possibility for an infinite mint (of both `SantasList NFT` and `SantaTokens`) with this by-pass.

**Impact:**

Any `NICE` user can mint as much NFTs as they like.

Any `EXTRA_NICE` user can mint as much `SantaTokens` and `SantasList NFTs` as they like.

**Proof of Concept:**

Add the following test to the `SantaListTest.t.sol` file:

```solidity
    function testCanCollectPresentMultipleTimes() public {
        address bonnie = makeAddr("bonnie");
        address clyde = makeAddr("clyde");

        // by-pass as NICE
        vm.startPrank(santa);
        santasList.checkList(bonnie, SantasList.Status.NICE);
        santasList.checkTwice(bonnie, SantasList.Status.NICE);
        vm.stopPrank();

        vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

        // bonnie collects present as NICE
        vm.startPrank(bonnie);
        santasList.collectPresent();
        santasList.safeTransferFrom(bonnie, clyde, 0); // bonnie transfers NFT to clyde
        santasList.collectPresent(); // bonnie collects present again
        vm.stopPrank();

        assert(santasList.balanceOf(bonnie) == 1);
        assert(santasList.balanceOf(clyde) == 1);

        // bonnie transfers NFT to clyde so as to clear her NFT balance
        vm.prank(bonnie);
        santasList.safeTransferFrom(bonnie, clyde, 1);

        // by-pass as EXTRA_NICE
        vm.startPrank(santa);
        santasList.checkList(bonnie, SantasList.Status.EXTRA_NICE);
        santasList.checkTwice(bonnie, SantasList.Status.EXTRA_NICE);
        vm.stopPrank();

        vm.warp(santasList.CHRISTMAS_2023_BLOCK_TIME() + 1);

        // bonnie collects present as EXTRA_NICE
        vm.startPrank(bonnie);
        santasList.collectPresent();
        santasList.safeTransferFrom(bonnie, clyde, 2); // bonnie transfers NFT to clyde
        santasList.collectPresent(); // bonnie collects present again 
        vm.stopPrank();

        assert(santasList.balanceOf(clyde) == 3);
        assert(santasList.balanceOf(bonnie) == 1);
        assert(santaToken.balanceOf(bonnie) == 2e18);
    }
```

**Recommended Mitigation:**

Create an `address => bool` mapping that checks if the caller of the `collectPresent()` has collected present already, and implement it as a modifier in the `collectPresent()` function

```diff
    /*//////////////////////////////////////////////////////////////
                            STATE VARIABLES
    //////////////////////////////////////////////////////////////*/
.
.
.
+   mapping(address => bool) private s_hasCollected;   
.
.
.
    /*//////////////////////////////////////////////////////////////
                               MODIFIERS
    //////////////////////////////////////////////////////////////*/


+   modifier notCollected(address _user) {
+       if (s_hasCollected[_user]) {
+           revert SantasList__AlreadyCollected();
+       }
+       _;
+   }
.
.
.
    /*//////////////////////////////////////////////////////////////
                               FUNCTIONS
    //////////////////////////////////////////////////////////////*/
.
.
.
-   function collectPresent() external (msg.sender) {
+   function collectPresent() external notCollected(msg.sender) {
        if (block.timestamp < CHRISTMAS_2023_BLOCK_TIME) {
            revert SantasList__NotChristmasYet();
        }
-       if (balanceOf(msg.sender) > 0) {
-           revert SantasList__AlreadyCollected();
-       }
        if (s_theListCheckedOnce[msg.sender] == Status.NICE && s_theListCheckedTwice[msg.sender] == Status.NICE) {
+           s_hasCollected[msg.sender] = true;
            _mintAndIncrement();
            return;
        } else if (
            s_theListCheckedOnce[msg.sender] == Status.EXTRA_NICE
                && s_theListCheckedTwice[msg.sender] == Status.EXTRA_NICE
        ) {
+           s_hasCollected[msg.sender] = true;
            _mintAndIncrement();
            i_santaToken.mint(msg.sender);
            return;
        }
        revert SantasList__NotNice();
    }
```

### [H-5] Every user is `NICE` by default 😂😂😂

**Description:** 

Every user has a status of `NICE` by default. Not much to be said.

**Impact:**

Every benefit available to `NICE` users is automatically available to every user by default, without need for `onlySanta` to mark them as `NICE`.

**Proof of Concept:**

Add the following test to the `SantasListTest.t.sol` file:

```solidity
    function testEveryUserIsNICEByDefault() public {
        assertEq(uint256(santasList.getNaughtyOrNiceOnce(user)), uint256(SantasList.Status.NICE));
    }
```

**Recommended Mitigation:**

Modify the `Status` `enum` as so 👇🏾:

```diff
-   enum Status {
-       NICE,
-       EXTRA_NICE,
-       NAUGHTY,
-       NOT_CHECKED_TWICE
-   }

+   enum Status {
+       UNKNOWN,
+       NICE,
+       EXTRA_NICE,
+       NAUGHTY
+   }
```

## Medium Issues

### [M-1] Make sure all events are indexed appropriately
