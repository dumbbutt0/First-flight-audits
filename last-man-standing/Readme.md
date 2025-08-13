# last man standing audit
The Last man standing contract is a "king of the hill" type competitive game where users fight for a title of `king` by paying an increasing fee. 
The game's core mechanic revolves around a grace period: if no new player claims the throne before this period expires, the current King wins the entire accumulated prize pot.

---
## Valid submissions

**1. Game Manipulation via declareWinner() Frontrunning. Severity: High**

The normal behavior of the declareWinner() function is to allow anyone to end the game and award the pot to the current king once the grace period has expired.

However, the function lacks a nonReentrant modifier and any caller-specific cooldown mechanism. 
As a result, a malicious actor can deploy a contract that programmatically front-runs the grace period check, repeatedly calling claimThrone() followed immediately by declareWinner() until timing lines up, effectively forcing an early win or stalling the game.
```solidity
function declareWinner() external gameNotEnded {
    require(currentKing != address(0), "Game: No one has claimed the throne yet.");
@>  require(
@>      block.timestamp > lastClaimTime + gracePeriod,
@>      "Game: Grace period has not expired yet."
@>  );
​
    gameEnded = true;
    pendingWinnings[currentKing] += pot;
    pot = 0;
​
    emit GameEnded(currentKing, pot, block.timestamp, gameRound);
}
```
#### Observation 1

***The contract exposes the exact time remaining in the gracePeriod using the public getRemainingTime() view function.
Combined with public variables like lastClaimTime and gracePeriod, this allows external actors to precisely predict when the game state will change, removing any uncertainty in timing.
This leads to automated sniping, where a bot or MEV agent can claim the throne just before declareWinner() becomes callable — defeating the game’s intent.***

```solidity
function getRemainingTime() public view returns (uint256) {
    if (block.timestamp >= lastClaimTime + gracePeriod) {
        return 0;
    }
    return (lastClaimTime + gracePeriod) - block.timestamp;
}
@> `getRemainingTime()` exposes real-time countdown
@> Relies on public `block.timestamp`, `lastClaimTime`, and `gracePeriod`
@> Allows exact prediction of when to front-run claim
```
**Likelihood: High**
- The grace period timer is fully observable and deterministic.
- Any external bot or contract with mempool access can calculate optimal sniping windows in real-time.

**Impact:**
- Premature game-ending grants unfair rewards to the attacker.
- This may cause loss of player trust and manipulation of on-chain game analytics.

**Recommended Mitigation**
```diff
- function declareWinner() external gameNotEnded {
+ function declareWinner() external gameNotEnded nonReentrant {
```
#### Observation 2

***This function may need to be completely recreated because the exposed timing and the effectiveness of bot sniping combined will be a severe issue even with reentrant guards***

  ---
## **Incorrect Access Control in `claimThrone()` Freezes Game**

The normal behavior of claimThrone() is to allow any new player to become the king by paying the required claimFee. The function should reject only redundant claims from the current king.

However, the current implementation mistakenly allows only the current king to call claimThrone(), freezing the game for all other players. This makes the game unplayable after the first claim.
```solidity
function claimThrone() external payable gameNotEnded nonReentrant {
    require(msg.value >= claimFee, "Game: Insufficient ETH sent to claim the throne.");
@>  require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");
    ...
}
```

**Likelihood:Always**

- This bug will occur immediately after the first claim, as the next player (a different address) will always fail the require(msg.sender == currentKing) check.

- Since the game requires multiple participants, this stops core game flow by design.

**Impact:**

- Players are permanently locked out after one person claims the throne.

- The declareWinner, claimFee increase, and platform fee logic will never execute again, rendering the game and economy non-functional.

**Recommended Mitigation**
```diff
- require(msg.sender == currentKing, "Game: You are already the king. No need to re-claim.");
+ require(msg.sender != currentKing, "Game: You are already the king. No need to re-claim.");
```

## Technical Appendix

**PoC 1 — Game Lock After First Claim**
```solidity
function test_GameLocksAfterFirstClaim() public {
    game.claimThrone{value: initialFee}();

    // Simulate a second player trying to claim
    vm.prank(address(0xBEEF));
    vm.deal(address(0xBEEF), 10 ether);
    vm.expectRevert("Game: You are already the king. No need to re-claim.");
    game.claimThrone{value: game.claimFee()}();
}
```
**PoC 2 — Automated Grace Period Exploit**
```solidity
contract ExploitBot {
    Game public game;

    constructor(address _game) {
        game = Game(_game);
    }

    function exploit() external payable {
        // Claim the throne shortly before the grace period could expire
        game.claimThrone{value: msg.value}();

        // Loop until grace period is expired and call declareWinner
        while (true) {
            try game.declareWinner() {
                break;
            } catch {}
        }
    }

    receive() external payable {}
}
```
