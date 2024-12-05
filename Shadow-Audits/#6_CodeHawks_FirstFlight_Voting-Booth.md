![](../logo.png)
<img src="../img/voting-booth.png" alt="Voting Booth" height="320" />

# Security Report for the Voting Booth CodeHawks FirstFlight Contest **shadow audit*

## High Issues

### [H-1] Calculation error in reward system

**Description:** 

A miscalculation in the reward system (`VotingBooth::_distributionRewards()`) results in an error when rewards are given to `voters` who voted in support of the proposal.

Implementation of `totalVotes` instead of `totalVotesFor` when calculating `rewardPerVoter` means there will always be an incorrect result when `totalVotesFor` != `totalVotes`.

```solidity
    uint256 rewardPerVoter = totalRewards / totalVotes;
```

**Impact:**

Incorrect calculation of `rewardPerVoter` inside `_distributeRewards()` function results in lower rewards to be awarded to `for` voters and also considerable leftover amount in the contract is stuck which can't be withdrawn now or transferred.


**Proof of Concept:**

An easy *proof of code* is to modify the `VotingBoothTest.t.sol::testVotePassesAndMoneyIsSent()` and so, and monitor the result logged unto the console:

```solidity
    function testVotePassesAndMoneyIsSent() public {
        console2.log("This is the balance of booth before voting: ", address(booth).balance);
        vm.prank(address(0x1));
        booth.vote(true);

        vm.prank(address(0x2));
        booth.vote(true);

        vm.prank(address(0x3));
        booth.vote(false);

        // assert(!booth.isActive() && address(booth).balance == 0);
        console2.log("This is the reward for addressOne: ", address(0x1).balance);
        console2.log("This is the reward for addressTwo: ", address(0x2).balance);
        console2.log("This is the reward for addressThree: ", address(0x3).balance);
        console2.log(
            "This is the balance of the booth after rewards have been shared to forVoters : ", address(booth).balance
        );
    }
```

and here are the results:

```bash
    Logs:
        This is the balance of booth before voting:  10000000000000000000
        This is the reward for addressOne:  3333333333333333333
        This is the reward for addressTwo:  3333333333333333334
        This is the reward for addressThree:  0
        This is the balance of the booth after rewards have been shared to forVoters :  3333333333333333333
```

From the logged results, we can see that if 3 voters register their votes -with a vote pattern of 2 `true` and 1 `false`, the reward `true` are rewarded from the pot.   
But instead of the balance of the pot to be sent to whoever initiated the proposal, it is stuck in the contract forever.   
This will always happen if the `totalVotes` != `totalVotesFor`.

Here is a more specific test for this finding: 

```solidity
    function testRewardSystemError() public {
        vm.prank(address(0x1));
        booth.vote(true);

        vm.prank(address(0x2));
        booth.vote(true);

        vm.prank(address(0x3));
        booth.vote(false);

        assert(!booth.isActive());
        assert(address(0x1).balance != 5e18); // The reward of addressOne should be 5e18 but it is not
        assert(address(0x2).balance != 5e18); // The reward of addressTwo should be 5e18 but it is not
        assert(address(booth).balance != 0); // The balance in the pot should be zero but it is not
    }
```

I saw this amazing [`fuzz test`](https://codehawks.cyfrin.io/c/2023-12-Voting-Booth/results?lt=contest&page=1&sc=xp&sj=reward&t=report) in the official competition report, it is amazing.

```solidity
    function testFuzz_RewardSystemError(bool vote1, bool vote2, bool vote3) public {
        vm.prank(address(0x1));
        booth.vote(vote1);

        vm.prank(address(0x2));
        booth.vote(vote2);

        vm.prank(address(0x3));
        booth.vote(vote3);

        assert(!booth.isActive() && address(booth).balance == 0);
    }
```

and here are the results:

```bash
    [FAIL: panic: assertion failed (0x01); counterexample: calldata=0xfb0b1328000000000000000000000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000001 args=[true, false, true]] testFuzz_RewardSystemError(bool,bool,bool) (runs: 2, μ: 232750, ~: 232750)
```

**Recommended Mitigation:**

Modify the `_distributeRewards()` function as so:

```diff
    function _distributeRewards() private {
        // get number of voters for & against
        uint256 totalVotesFor = s_votersFor.length;
        uint256 totalVotesAgainst = s_votersAgainst.length;
        uint256 totalVotes = totalVotesFor + totalVotesAgainst;

        // rewards to distribute or refund. This is guaranteed to be
        // greater or equal to the minimum funding amount by a check
        // in the constructor, and there is intentionally by design
        // no way to decrease or increase this amount. Any findings
        // related to not being able to increase/decrease the total
        // reward amount are invalid
        uint256 totalRewards = address(this).balance;

        // if the proposal was defeated refund reward back to the creator
        // for the proposal to be successful it must have had more `For` votes
        // than `Against` votes
        if (totalVotesAgainst >= totalVotesFor) {
            // proposal creator is trusted to create a proposal from an address
            // that can receive ETH. See comment before declaration of `s_creator`
            _sendEth(s_creator, totalRewards);
        }
        // otherwise the proposal passed so distribute rewards to the `For` voters
        else {
-           uint256 rewardPerVoter = totalRewards / totalVotes; 
+           uint256 rewardPerVoter = totalRewards / totalVotesFor; 

            for (uint256 i; i < totalVotesFor; ++i) {
                // proposal creator is trusted when creating allowed list of voters,
                // findings related to gas griefing attacks or sending eth
                // to an address reverting thereby stopping the reward payouts are
                // invalid. Yes pull is to be preferred to push but this
                // has not been implemented in this simplified version to
                // reduce complexity & help you focus on finding the
                // harder to find bug

                // if at the last voter round up to avoid leaving dust; this means that
                // the last voter can get 1 wei more than the rest - this is not
                // a valid finding, it is simply how we deal with imperfect division
                if (i == totalVotesFor - 1) {
-                   rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
+                   rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotesFor, Math.Rounding.Ceil);
                }
                _sendEth(s_votersFor[i], rewardPerVoter);
            }
        }
    }
```