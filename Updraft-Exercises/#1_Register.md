## Fuzzing

This is a short and simple exercise to start sharpening your smart contract fuzzing skills with Foundry.

The scenario is simple. There's a registry contract that allows callers to register by paying a fixed fee in ETH. If the caller sends too little ETH, execution should revert. If the caller sends too much ETH, the contract should give back the change.

Things look good according to the unit test we coded in the `Registry.t.sol` contract.

Your goal is to code at least one fuzz test for the `Registry` contract. By following the brief specification above, the test must be able to detect a bug in the `register` function.


### Source:

https://gist.github.com/tinchoabbate/67b195b95fe77a5b9e3c6cc4bf80b3f7


### Registry.sol:

```solidity
    // SPDX-License-Identifier: UNLICENSED
    pragma solidity ^0.8.13;

    contract Registry {
        error PaymentNotEnough(uint256 expected, uint256 actual);

        uint256 public constant PRICE = 1 ether;

        mapping(address account => bool registered) private registry;

        function register() external payable {
            if (msg.value < PRICE) {
                revert PaymentNotEnough(PRICE, msg.value);
            }

            registry[msg.sender] = true;
        }

        function isRegistered(address account) external view returns (bool) {
            return registry[account];
        }
    }

```


### Registry.t.sol:

```solidity
    // SPDX-License-Identifier: UNLICENSED
    pragma solidity ^0.8.13;

    import {Test, console2} from "forge-std/Test.sol";
    import {Registry} from "../src/Registry.sol";

    contract RegistryTest is Test {
        Registry registry;
        address alice;

        function setUp() public {
            alice = makeAddr("alice");
            
            registry = new Registry();
        }

        function test_register() public {
            uint256 amountToPay = registry.PRICE();
            
            vm.deal(alice, amountToPay);
            vm.startPrank(alice);

            uint256 aliceBalanceBefore = address(alice).balance;

            registry.register{value: amountToPay}();

            uint256 aliceBalanceAfter = address(alice).balance;
            
            assertTrue(registry.isRegistered(alice), "Did not register user");
            assertEq(address(registry).balance, registry.PRICE(), "Unexpected registry balance");
            assertEq(aliceBalanceAfter, aliceBalanceBefore - registry.PRICE(), "Unexpected user balance");
        }

        /** Code your fuzz test here */
    }
```


### Fuzz Tests:

I wrote the following tests to test the following scenarios:

- Will the function revert in scenarios where `msg.value` is less than 1 ETH (which is the required fee for registration)?

```solidity
    function testFuzz_registry_revert(uint256 lessAmount) public {
        vm.assume(lessAmount < registry.PRICE());
        vm.deal(alice, lessAmount);

        vm.startPrank(alice);
        vm.expectRevert();
        registry.register{value: lessAmount}();
        vm.stopPrank();

        assert(registry.isRegistered(alice) == false);
        assert(alice.balance == lessAmount);
    }
```

Yes... the function will revert if the `msg.value` is less than 1 ETH

------------

- In scenarios where `msg.value` is more than 1 ETH, will the contract give back the extra to the `msg.sender` (as stated in the documentation)?

```solidity
    function testFuzz_registry_does_not_pay_back_extra(uint256 extraAmount) public {
        vm.assume(extraAmount > registry.PRICE());
        vm.deal(alice, extraAmount);

        uint256 registryBalanceBefore = address(registry).balance;

        vm.startPrank(alice);

        uint256 aliceBalanceBefore = alice.balance;

        registry.register{value: extraAmount}();

        uint256 aliceBalanceAfter = alice.balance;

        vm.stopPrank();

        uint256 registryBalanceAfter = address(registry).balance;

        assert(registry.isRegistered(alice) == true);
        assert(registryBalanceBefore == 0);
        assert(aliceBalanceBefore == extraAmount);
        // The balance of the registry should be 1 eth and any extra given back to the user going by the documentation, but this assert proves it is not so
        assert(registryBalanceAfter == extraAmount); // which should not be so
        assert(aliceBalanceAfter == 0); // which should not be so
        assert(registryBalanceAfter != registry.PRICE()); // registry balance after registration should be only 1 eth
    }
```

No...the contract does not give back any extra change