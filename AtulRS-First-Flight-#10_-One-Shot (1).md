# First Flight #10: One Shot - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. H-4: Missing cred token balance check for challenger in _battle function, allowing challengers to participate with zero cred balance.](#H-01)
    - ### [H-02. H-3: Missing Challenger's NFT Ownership Verification in _battle Function, allowing attackers to participate as a challenger in battles with NFTs they don't own](#H-02)
    - ### [H-03. H-2: Function "goOnStageOrBattle" can be frontrun by the defender to avoid losing Cred Tokens.](#H-03)
    - ### [H-04. H-1: Source of Randomness can be manipulated resulting unfair advantage for malicious participants. ](#H-04)

- ## Low Risk Findings
    - ### [L-01. L-1: No duplicate checks for tokenId in  goOnStageOrBattle function ](#L-01)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: First Flight #10

### Dates: Feb 22nd, 2024 - Feb 29th, 2024

[See more contest details here](https://www.codehawks.com/contests/clstf5qd2000rakskkj0lkm33)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 4
   - Medium: 0
   - Low: 1


# High Risk Findings

## <a id='H-01'></a>H-01. H-4: Missing cred token balance check for challenger in _battle function, allowing challengers to participate with zero cred balance.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L38

## Summary
There is no logic in internal function _battle that checks whether the challenger has enough cred balance to put on bet in a rap battle, allowing challenger to participate in rap battle with zero cred tokens and also potentially win defender's bet.

## Vulnerability Details
A challenger can participate in rap battle without owning any cred token. If he wins, he takes the money. if he loses, the transaction reverts due to insufficient balance.

#POC
function testChallengerCanRapBattleWithoutCred() public{

        vm.startPrank(user);
        oneShot.mintRapper();
        oneShot.approve(address(streets), 0);
        streets.stake(0);
        vm.stopPrank();

        vm.warp(4 days + 1);

        vm.startPrank(user);
        streets.unstake(0);
        vm.stopPrank();

        vm.startPrank(challenger);
        oneShot.mintRapper();
        
        vm.startPrank(user);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), 3);
        rapBattle.goOnStageOrBattle(0, 3);
        vm.stopPrank();

        console.log(cred.balanceOf(challenger));
        console.log(cred.balanceOf(user));

        vm.startPrank(challenger);
        oneShot.approve(address(rapBattle), 1);
        cred.approve(address(rapBattle), 3);
        rapBattle.goOnStageOrBattle(1, 3);
        vm.stopPrank();

        
        console.log(cred.balanceOf(challenger));
        console.log(cred.balanceOf(user));
   
}

## Impact
Challenger will never loose money. if he wins, he wins all. if he loses, transaction reverts. 

## Tools Used
Manual review

## Recommendations
```diff
  function _battle(uint256 _tokenId, uint256 _credBet) internal {
+ require(cred.balanceOf(msg.sender) >= _credBet);
        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);

```
## <a id='H-02'></a>H-02. H-3: Missing Challenger's NFT Ownership Verification in _battle Function, allowing attackers to participate as a challenger in battles with NFTs they don't own            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L38

## Summary
Description:  The _battle function lacks validation of NFT ownership, allowing attackers to participate in battles with NFTs they don't own.
```javascript
 function _battle(uint256 _tokenId, uint256 _credBet) internal {
        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");
        uint256 defenderRapperSkill = getRapperSkill(defenderTokenId);
        uint256 challengerRapperSkill = getRapperSkill(_tokenId);
        uint256 totalBattleSkill = defenderRapperSkill + challengerRapperSkill;
        uint256 totalPrize = defenderBet + _credBet;

        uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;

        // Reset the defender
        defender = address(0);
        emit Battle(msg.sender, _tokenId, random < defenderRapperSkill ? _defender : msg.sender);

        // If random <= defenderRapperSkill -> defenderRapperSkill wins, otherwise they lose
        if (random <= defenderRapperSkill) {
            // We give them the money the defender deposited, and the challenger's bet
            credToken.transfer(_defender, defenderBet);
            credToken.transferFrom(msg.sender, _defender, _credBet);
        } else {
            // Otherwise, since the challenger never sent us the money, we just give the money in the contract
            credToken.transfer(msg.sender, _credBet);
        }
        totalPrize = 0;
        // Return the defender's NFT
        oneShotNft.transferFrom(address(this), _defender, defenderTokenId);
    }
```

## Vulnerability Details
Missing Checks:  The _battle function doesn't include require statements or other mechanisms to verify that the  challenger address actually own the _tokenId they are using in the battle.

Exploitation: An attacker could utilize any arbitrary valid NFT token IDs of anyone (even that of defender's) who owns a Rapper NFT, within the _battle function to participate in Rap Battle and earn cred token. 

#POC

The following POC shows how an attacker can participate in rap battle without owning the Rapper NFT and earn cred token. In below example, attacker is using defender's tokenID to participate in the battle and can potentially win the rap battle.




```

    function testUserCanRapBattleWithoutNFT() public twoSkilledRappers{
        address attacker = makeAddr("Attacker");
//defender locking his NFT for Rap Battle
        vm.startPrank(user);
        oneShot.approve(address(rapBattle), 0);
        cred.approve(address(rapBattle), 3);
        rapBattle.goOnStageOrBattle(0, 3);
        vm.stopPrank();
//Attacker using defender's tokenID to participate in the Rap Battle
        vm.startPrank(attacker);
        rapBattle.goOnStageOrBattle(0, 3);
        vm.stopPrank();

        console.log(cred.balanceOf(attacker));
        console.log(cred.balanceOf(user));
}

```
## Impact
Compromised Fairness: The battle system's integrity is undermined as attackers can use NFTs they don't own and earn unlimited cred bets. 

## Tools Used
Manual Review

## Recommendations
```diff

  function _battle(uint256 _tokenId, uint256 _credBet) internal {

+ require(msg.sender == oneShotNft.ownerOf(_tokenId));

        address _defender = defender;
        require(defenderBet == _credBet, "RapBattle: Bet amounts do not match");

```
## <a id='H-03'></a>H-03. H-2: Function "goOnStageOrBattle" can be frontrun by the defender to avoid losing Cred Tokens.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L38C14-L38C31

## Summary
In "RapBattle.sol", the "goOnStageOrBattle" function randomly picks a Rap Battle"s winner between the challenger and the defender. However, the function is vulnerable to front-running. The defender can manipulate the battle outcome through a combination of mempool observation and front-running to become both the challenger and defender, guaranteeing a win.

## Vulnerability Details

1: Random Number Visibility: The _battle function's random number relies on values visible in the mempool (block.timestamp, block.prevrandao, msg.sender).

2: Defender's Observation: The defender (potentially working with a validator/miner) can monitor the mempool for incoming challenges and predict the random number.

3: Front-Running the Challenger: If the calculated random number is unfavorable for the defender, they can:

4: Front-run the original challenger's transaction by paying a higher gas fee.
They can Immediately challenge themselves as a challenger with any valid tokenId (the protocol also lacks validation to check for the ownership of tokenId)  and the same _credBet.

5: Guaranteed Win: By effectively replacing the original challenger with itself using another EOA, the defender now controls both sides of the battle and will win regardless of the random number outcome.

## Impact
Severe Compromise of Fairness: This exploit completely breaks the fairness of the battle system.
Financial Gain for Attacker: The defender can manipulate the system to guarantee winnings, essentially avoid loosing any funds to any different person.

## Tools Used
Manual Review

## Recommendations

## <a id='H-04'></a>H-04. H-1: Source of Randomness can be manipulated resulting unfair advantage for malicious participants.             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L62

## Summary
In "RapBattle.sol", the function "goOnStageOrBattle" implements an internal function "_battle" which uses an unfair source of randomness to generate a random number and pick a random winner between the "defender" and "challenger".

## Vulnerability Details
```javascript
uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;
```
The function "_battle" uses value like block.timestamp, block.prevrandao, msg.sender to generate a random number which can be either be influenced or viewed by the validators or miners on the mempools to get a more favorable outcome. A malicious participant with the help of a validator or a miner can Intentionally delay or expedite block production to get a slightly more favorable block.timestamp and win the RapBattle.

## Impact
A malicious participant (defender or challenger) with the help of malicious validator with  enough stake or with the ability to collude with other validators can influence the random number and win the Rap Battle unfairly. 

## Tools Used
Manual review

## Recommendations
Use Chainlink VRF  or any trusted decentralized oracle network to generate a provable fair random number. 
		


# Low Risk Findings

## <a id='L-01'></a>L-01. L-1: No duplicate checks for tokenId in  goOnStageOrBattle function             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2024-02-one-shot/blob/47f820dfe0ffde32f5c713bbe112ab6566435bf7/src/RapBattle.sol#L62

## Summary
 The  _battle function allows the same NFT tokenId to be used as both the defender and the challenger, potentially leading to unfair advantages and system logic disruption.

## Vulnerability Details
Missing Validation:  There's no check to prevent identical tokenId values from being used in a single battle by the defender and challenger.

## Impact
Potential for DoS: An attacker can disrupt the availability of the battle system for others.

## Tools Used
Manual review

## Recommendations
```diff
+ require(defenderTokenId != _tokenId, "Cannot use the same NFT as defender and challenger");
```


