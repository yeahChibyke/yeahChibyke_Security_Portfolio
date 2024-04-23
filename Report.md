![](security_logo.jpg)

# Security Report for the PasswordStore CodeHawks FirstFlight Contest **learning*

**Source:** https://github.com/Cyfrin/2023-10-PasswordStore

### [H-1] Storing the password on-chain makes it visible to anyone, thus no longer private

**Description:** All data stored on-chain is visible to anyone and can be read directly from the blockchain. The `PasswordStore::s_password` variable is intended to be private, and accessible only by the owner of the contract through the `PasswordStore::getPassword()` function.

**Impact:** Anyone can read the supposed private password, thus breaking the functionality of the protocol.

**Proof of Concept:** The test case below shows how anyone can read the password directly from the blockchain.

   1. Create a locally running chain
      
      ``` bash
      make anvil
      ```

   2. Deploy the contract to the chain
      
      ```
      make deploy
      ```

   3. Run the storage tool

      We use `1` because that is the storage slot of `s_password` in the contract.

      ```
      cast storage <CONTRACT_ADDRESS_HERE> 1 --rpc-url http://127.0.0.1:8545
      ```

      You will get an output like this:

      `0x626a696f6d766f69666f756e756269757566766d75697366766e6975736d003c`

      You can then parse that hex to a string with:

      ```
      cast parse-bytes32-string 0x626a696f6d766f69666f756e756269757566766d75697366766e6975736d003c
      ```

      And get an output of:

      ```
      myPassword
      ```

**Recommended Mitigation:** Due to this, the overall architecture of the contract should be rethought. One could encrypt the password off-chain, and then store the encrypted password on-chain. This would require the user to remember another password off-chain to decrypt the stored password. However, you're also likely want to remove the view function as you wouldn't want the user to accidentally send a transaction with this decryption key.

### [H-2] `PasswordStore::setPassword()` has no access controls. Non-owner can change the password

**Description:** The `PasswordStore::setPassword()` function is set to be an `external` function, however the atspec and overall purpose of the contract is that `This function allows only the owner to set a new password`.

<details>
<summary>Code</summary>

   ```solidity
      //q? Can non-owner set password? Yes
      //q? Should non-owner be able to set password? No
      //f! Any user can set password. MISSING ACCESS CONTROL!!!
      function setPassword(string memory newPassword) external {         
         s_password = newPassword;
         emit SetNetPassword();
      }
   ```
</details>

**Impact:** Anyone can set/change the password of the contract, thus breaking the functionality of the contract.

**Proof of Concept:** Add the following to the `PasswordStore.t.sol` test file

<details>
<summary>Code</summary>

   ```solidity
      function test_anyone_can_set_password(address randomaddress) public {
         vm.assume(randomaddress != owner);
         vm.prank(randomaddress);
         string memory expectedPassword = "sdfghjnbchnuduwudnm";
         passwordStore.setPassword(expectedPassword);

         vm.prank(owner);
         string memory actualPassword = passwordStore.getPassword();

         assertEq(actualPassword, expectedPassword);
      }
   ```
</details>

**Recommended Mitigation:** Add an access control conditional to the `PasswordStore::setPassword()` function

<details>
<summary>Code</summary>

   ```solidity
      function setPassword(string memory newPassword) external {
@-->         if (msg.sender != s_owner) {
               revert PasswordStore__NotOwner();
         }
         s_password = newPassword;
         emit SetNetPassword();
      }
   ```

</details>

### [I-1] The NatSpec of the `PasswordStore::getPassword()` function indicates a parameter that doesn't exist, causing the NatSpec to be incorrect.

**Description:** 

<details>
<summary>Code</summary>

   ```solidity
      /*
      * @notice This allows only the owner to retrieve the password.
@-->  * @param newPassword The new password to set
      */
      function getPassword() external view returns (string memory) {}
   ```

</details>

The function signature of `PasswordStore::getPassword()` is `getPassword()` and not `getPassword(string)` as the NatSpec says.

**Impact:** The NatSpec is incorrect.

**Recommended Mitigation:** Remove the incorrect NatSpec line.

   ```diff
-     * @param newPassword The new password to set
   ```
