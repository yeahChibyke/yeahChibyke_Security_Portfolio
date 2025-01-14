![](../logo.png)
<img src="../img/mafia-takedown.png" alt="Mafia Takedown" height="320" />

# Security Report for the Mafia Takedown CodeHawks FirstFlight Contest \*_shadow audit_

## High Issues

### [H-1] Lack of access control in `Laundrette::depositTheCrimeMoneyInATM()` allows anyone to move USDC from another user's wallet

**Description:**

In the `depositTheCrimeMoneyInATM()` function, there are 3 parameters to be filled;
- `account`
- `to`
- `amount`

an external caller can set the `account` parameter to be that of another user, and transfer USDC from that account.

**Impact:**

A malicious user can call the `depositTheCrimeMoneyInATM()` function, set the `account` parameter to be any address with remaining USDC allowance, and set `to` to be himself, to steal USDC from `account` address. 

**PoC:**

Add the following test to the `Laundrette.t.sol` test file:

```solidity
    function test_can_steal_user_usdc() public {
        address chile = makeAddr("chile");
        address merlin = makeAddr("attacker");

        vm.prank(godFather);
        usdc.transfer(chile, 100e6);
        vm.prank(chile);
        usdc.approve(address(moneyShelf), 100e6); // chile grants approval to moneyshelf, but doesn't deposit into ATM

        vm.prank(merlin);
        laundrette.depositTheCrimeMoneyInATM(chile, merlin, 100e6); // merlin deposits usdc from chile's account because
            // there is still allowance

        assert(usdc.balanceOf(address(moneyShelf)) == 100e6);
        assert(crimeMoney.balanceOf(merlin) == 100e6);

        //unaware chile tries to deposit money into the ATM
        vm.prank(chile);
        vm.expectRevert(); // it fails
        laundrette.depositTheCrimeMoneyInATM(chile, chile, 100e6);
    }
```

**Recommended Mitigation:**

Add a `require()` in `depositTheCrimeMoneyInATM()` function that ensures that the address inputted in the `account` parameter is the same as the `msg.sender`.

```diff
    function depositTheCrimeMoneyInATM(address account, address to, uint256 amount) external {
+       require(account == msg.sender, "You cannot transfer money from this account modafucka!!!");
        moneyShelf.depositUSDC(account, to, amount);
    }
```

## Medium Issues

### [M-1] Malicious gangmember can revoke the gang membership of another gang member by calling `Laundrette:quitTheGang()` on said gang member

**Description**:

`quitTheGang()` function allows any gang member to revoke their gang membership and quit the gang, but because of no checks, a malicious gang member can revoke the gang membership of another gang member.

**PoC:**

Add the following test to the `Base.t.sol` test file:

```solidity
    function addToGang(address _account) internal {
        vm.prank(godFather);
        laundrette.addToTheGang(_account);
    }
```

And then this, to the `Laundrette.t.sol` test file:

```solidity
    function test_can_kick_out_another_gang_member() public {
        address chile = makeAddr("chill_gang_member");
        address merlin = makeAddr("malicious_gang_member");

        // chile and merlin become gang members
        joinGang(chile);
        addToGang(merlin);

        assert(kernel.hasRole(chile, Role.wrap("gangmember")) == true);
        assert(kernel.hasRole(merlin, Role.wrap("gangmember")) == true);

        // merlin kicks chile out of the gang
        vm.prank(merlin);
        laundrette.quitTheGang(chile);

        assert(kernel.hasRole(chile, Role.wrap("gangmember")) == false);
    }
```

**Recommended Mitigation:**

Add a `require()` check to ensure that the caller of the `quitTheGang()` function must also be the inputted `account` parameter, of be the `godFather`.

```diff
    function quitTheGang(address account) external onlyRole("gangmember") {
+       require(msg.sender == account || msg.sender == kernel.executor(), "Modafucka!!! You have no right to call this function");
        kernel.revokeRole(Role.wrap("gangmember"), account);
    }
```
