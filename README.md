# researchoor_academy_progress

## Strengths and Weakness
Strengths
- ambitious
- design knowledge
- I have a intermediate understanding of smart contracts
- I have a strong gut feeling and I can spot where things can go wrong
- If given enough time I can grasp a particular topic

Weakness
- Lazy
- Scared to dive deeper
- Unclear long term goal
- scattered focus
- lack of discipline
- drawn towards external validation
- no commitment to a particular goal
- inconsistency
- lack of execution
- too emotional, I don't take decision based on what I need to do. I do what I feel like at the moment.
- no specialization on something and not good enough at everything
- overthinking

web3 sec specific weaknesses
- I can't write PoCs
- Lack of depth knowledge in major protocol types
- Lack of knowledge even in common & intermediate ones. That is why even if something is a vuln and easy to spot I can't find it. 
- I can't write good reports. 
- I lack basic development knowledge of smart contracts, which makes auditing even more difficult for me.
- I don't stay upto date with the updates happening in the space.

## Liquidation vuln

**What is liquidation?**
Being able to force close someone's position in defi protocol
- Borrowing/Lending
- PnL(Perps and option)

### Borrow/ Lending protocol
- Put down collateral borrow against it
- value of collateral * LTV > value of borrow
- LTV : loan to value ratio (always less than 1)

#### Insolvency VS Liquidatable

**Liquidatable** 
value of collateral * LTV < value of borrow

**Insolvent position** : a position that does not have enough collateral to repay the borrow amount that it has. Very bad for the protocol, leads to losses. This also called as bad debt.
Value of your collateral < value of borrow

### PnL (Perps and options)

**Liquidatable** 
value of collateral + PnL goes below X amount

**Insolvent**
value of collateral + PnL goes below zero

### Vulns and edge cases 

1. Liquidation reverts due to blacklisted tokens
2. Liquidation reverts due to zero transfers
3. is liquidatable does not account for fees
4. liquidation should be collecting latet fees
5. liquidation reverts when position becomes insolvent
6. Liquidation reverts when a position cannot cover the fees
7. Liquidation revrets due to accounting error

# Previous audit report revisit

### Interpol
# High
---
#### Title : Lack of validation leads to premature withdrawal of erc721 tokens
#### Vulnerability detail
The migrate function lacks checks which would allow users to immediately withdraw erc721 tokens before the lock period ends in `honeylocker::withdrawERC721()` function

In my understanding the reason why this vuln is high is because : this is breaking a core functionality of the protocol i.e. token lockup. Although there is no monetary incentive tied to this. Withdrawing tokens before completion of expiration duration takes away the economic incentive that comes with locking up the token in the pool.

However, this could be easily spotted. The reason why I missed it is I didn't study the protocol documentation properly and checked the code with enough concentration. 

How not to miss it next time? 
If tokens are supposed to be locked up, check if someway tokens can be withdrawn before lockup period ends. 
Read documentation properly. 

#### Mitigation
Add the following check to ensure expiration duration has passed for the token.
```diff
function withdrawERC721(address _token, uint256 _id) external onlyUnblockedTokens(_token) onlyOwnerOrOperator {
+++ if (expirations[_token] != 0) revert CannotBeLPToken();
    ERC721(_token).safeTransferFrom(address(this), recipient(), _id);
}
```


#### Title: Lock Period Bypass Through Recipient-Configurable Unstaking Calls
#### Vulnerability details
I do not understand what is the exact issue, but it is related to bypassing lock period with certain user actions that breaks protocols core functionality.

https://cantina.xyz/code/55023131-27df-44e4-af46-bec298d0fa8e/findings?status=confirmed&severity=medium,high&finding=140

#### Title: Users can migrate gauge NFTs to unlock LP tokens early
#### Vulnerability details

- When user stakes LP tokens in gauge they receive a gauge NFT representing their position.
- The migrate function allows user to migrate any token including gauge NFTs from one honeylocker to another.
1. User locks LP tokens
2. User stakes the locked LP tokens in a gauge, and the `HoneyLocker` contract receives an NFT representing the gauge position
3. User migrates the gauge NFT to the new `HoneyLocker`
4. User unstakes the tokens from the gauge in the new `HoneyLocker`, withdrawing the LP tokens to the new `HoneyLocker`
5. User can utilize the [`depositAndLock`](https://github.com/0xHoneyJar/interpol/blob/main/src/HoneyLocker.sol#L255) function to deposit 1 LP token with an expiration time in the past
6. User can freely withdraw the LP token, which would otherwise still be locked in the original `HoneyLocker`, using the [`withdrawLPToken`](https://github.com/0xHoneyJar/interpol/blob/main/src/HoneyLocker.sol#L191) function

Users can alternatively skip steps 5 and 6 in the new vault and simply call [`withdrawERC20`](https://github.com/0xHoneyJar/interpol/blob/main/src/HoneyLocker.sol#L299) to withdraw the LP tokens. This would, however, incur a 2% fee.

Why I missed it?
This is a unique high finding, again belonging to the same vulnerability type of reedeming tokens before expiration/lock period. But this vulnerability takes a complex path to perform, I do not have the technical know how to unveil this vuln. 

How can I not miss it the next time?
It would take time to understand different flows between different function that could ultimately lead to breaking of a functionality. It might take time but it is not impossible.

# Medium
---
#### Title: Migration for LP tokens of type ERC721 fails
#### Vulnerability detail
The migrate function for migrating erc721 tokens will fail due to erc20 cast.
The tokens are migration logic is as follows : 
```js
ERC20(_LPTokens[i]).approve(address(_newHoneyLocker), _amountsOrIds[i]);
```
The approve function of erc721 works differently than erc20's approve function. Erc721 approve function doesn't return a boolean but erc20 function does.

This function would work for erc20 tokens but when erc721 tokens are approve it would revert as the approve function won't return any boolean value.

Why I missed?
I didn't knew the specifics about different functioning of erc20 and erc721's approve function. Although when I look at it now, the issue seems very obvious as single line of code is being used for migrating erc20 and erc721 tokens. And at the end it is being casted to erc20, so it would obviously fail if it is a erc721 token.

How to not miss it the next time?
Remember that approve function from solady's library function differently for erc721 and erc20.

Solady ERC20 [`approve()`](https://github.com/Vectorized/solady/blob/main/src/tokens/ERC20.sol#L186) function returns a bool.
```js
function approve(address spender, uint256 amount) public virtual returns (bool)
```

 Solady ERC721 [`approve()`](https://github.com/Vectorized/solady/blob/main/src/tokens/ERC721.sol#L202) function does not return anything.
```js
function approve(address account, uint256 id) public payable virtual
```
#### Impact
ERC721 token migration will fail
#### Mitigation
Instead of using ERC20's `approve` function, the ERC721's approve function should be used

#### Title : Fees distributed to a referrer may be more/less than expected

#### Vulnerability details 
I don't understand what this vulnerability is about I need to read it properly to understand it. 
https://cantina.xyz/code/55023131-27df-44e4-af46-bec298d0fa8e/findings?status=confirmed&severity=medium,high&finding=43

### Fjord

# High
---
#### Title : Loss of funds for a user due to incorrect updating of state while unstaking
#### Vulnerability detail
If an user tries to unstake tokens in the current epoch, which are stakes in a previous epoch the `unredeemedEpoch` variable is updated wrongly. 
The protocol doesn't differentiate between unstaking in current epoch and previous epochs, leading to unintended update of `unredeemedEpoch` variable. The users rewards will be lost.

The problem flow is 

1. Alex deposits 10tokens in epoch 5. `unredeemedEpoch` = 5 (as current epoch is 5)
2. He deposits 12tokens more in epoch 10. `unredeemedEpoch` = 10 (as current epoch is 10)
3. He unstakes 10 tokens deposited previously in epoch 5. Now `unredeedmedEpoch` is updated to 0. But it should not be zero. 
   If it is set to zero the rewards of tokens staked in epoch 10 will be lost. Basically he would get no rewards.
#### Impact
user rewards are lost
#### Mitigation
Add checks that will take into account that `unredeemedEpoch` should be set to 0 only if the `currentEpoch == _epoch`

**Key takeaway** : The attack vector is valid if the user first stakes new tokens then unstakes tokens from the previous epoch, in the current epoch. The way this works is interesting. I should pay more details about different time variations. This is not a very complex issue, it is very evident when we read but works in a particular condition. Being able to imagine that condition in which this would be valid is important.

# Medium
---
#### Title : Incorrect `block.timestamp` check allows user to bid after the auction is finished, which enables user to claim more token than they are eligible.
#### Vulnerability detail
In the `auctionEnd()` function a variable `multiplier` is calculated on the basis of total bids, if this value is inflated it allows user to claim more tokens than they are eligible. 
```js
multiplier = totalTokens.mul(PRECISION_18).div(totalBids);
```

```js
    function bid(uint256 amount) external {
    @>  if (block.timestamp > auctionEndTime) {
            revert AuctionAlreadyEnded();
        }

        bids[msg.sender] = bids[msg.sender].add(amount);
    @>  totalBids = totalBids.add(amount);

        fjordPoints.transferFrom(msg.sender, address(this), amount);
        emit BidAdded(msg.sender, amount);
    }
```

The `if` statement in `bid()` function performs a wrong check, it only checks if the `block.timestamp` is greater than the end time. Which would allow the user to bid even after the auction has ended i.e. when `block.timestamp = auctionEndTime`.

This would increase the `totalBids` . Which in turn would decrease the value of `multiplier`, as we can see from above. 

```js
        function claimTokens() external {
        if (!ended) {
            revert AuctionNotYetEnded();
        }

        uint256 userBids = bids[msg.sender];
        if (userBids == 0) {
            revert NoTokensToClaim();
        }

@>        uint256 claimable = userBids.mul(multiplier).div(PRECISION_18);
        bids[msg.sender] = 0;

@>        auctionToken.transfer(msg.sender, claimable);
        emit TokensClaimed(msg.sender, claimable);
    }
```

So lower `multiplier` means higher `claimable` amount. Hence user is able to earn more than they are eligible.

#### Impact
User can claim more tokens than they are eligible
#### Mitigation
update the `if` statement in `bid()` and `unbid()` function to :
```js
if(block.timestamp >= auctionEndTime){
	revert AuctionAlreadyEnded();
}
```

**Key takeaway** : It's so easy to spot, and it's interesting to see how just a change from `>` to `>=` can change the whole thing. Attention has to be paid at granular level.


#### Title : Auction tokens will be lost forever in case the auction ends without any bids.
#### Vulnerability detail

The person who deploys `FjordAuctionFactory.sol` contract i.e. `msg.sender` becomes the owner of the contract.
```js
    constructor(address _fjordPoints) {
        if (_fjordPoints == address(0)) revert InvalidAddress();

        fjordPoints = _fjordPoints;
        owner = msg.sender;
    }
```

The `FjordAuctionFactory.sol` contract deploys a new instance of `FjordAuction` contract to create a new auction in the `createAuction()` function. And the specified amount of auction tokens are transferred from the `msg.sender` to `auctionAddress`
```js
    function createAuction(
        address auctionToken,
        uint256 biddingTime,
        uint256 totalTokens,
        bytes32 salt
    ) external onlyOwner {
        address auctionAddress = address(
            new FjordAuction{ salt: salt }(fjordPoints, auctionToken, biddingTime, totalTokens)
        );

        // Transfer the auction tokens from the msg.sender to the new auction contract
        IERC20(auctionToken).transferFrom(msg.sender, auctionAddress, totalTokens);

        emit AuctionCreated(auctionAddress);
    }

```

Someone calls `auctionEnd()` function, in case of no bids the function will execute this `if` statement :
```js
        if (totalBids == 0) {
            auctionToken.transfer(owner, totalTokens);
            return;
        }

```

Here we can see that the tokens are transferred to the `owner` address. But the `owner` of this contract is `FjordAuctionFactory.sol` and that contract do not have any method to transfer the funds to the actual deployer of the contract. Hence auction tokens are locked.
#### Impact
Auction tokens are locked in `FjordAuctionFactory` contract, in case there are no bids.
#### Mitigation
Implement a method in the `FjordAuctionFactory` contract so that the owner can withdraw their tokens.

**Key takeaway** again this occurs in a specific condition where there are no bids, which leads to a series of malfunctioning of different functions. Ultimately leading to locking of funds. A lot of people found it, which shows that it is pretty easy to track but they were able to think keeping the precondition of no-bid situation in mind which lead to discovering of the issue. I need to check functions in different scenarios in mind. 

#### Title Epoch mismatch in FjordToken and FjordStaking leads to unfair reward distribution
#### Vulnerability detail
The `FjordStaking` contract depends on the address of `FjordToken` contract for deployment. Both the contracts can be deployed at seperate times leading to mismatch in epochs, which leads to unfair reward calculation for user. 

The users won't have to stake their tokens for the entire period of time but they will receive full rewards for that time period, which is not intended.
#### Mitigation
Both the contracts have to be deployed in a single transaction, `FjordToken` first because `FjordStaking` depends on the other contract's address for deployment.
or
Have the same time variable for both of the contracts.

**Key takeaways** : as easy at it looks this fetched $95 as reward. The vulnerability lies in just a line which leads to a series of malfunctions. The researcher whose report got selected went a step further to show that this issue actually exists in the current test net deployment of the protocol. Both the contracts are actually deployed 10 blocks apart. This wasn't a unique precondition or an edge case this was pure logic. 

#### Title https://codehawks.cyfrin.io/c/2024-08-fjord/results?lt=contest&sc=reward&sj=reward&page=1&t=report
#### Vulnerability detail

#### Impact
#### Mitigation

I do not have proper knowledge of sablier and it's working, the finding is related to the protocols misconfiguration with sablier, which leads to unintended situations. But the finding is written beautifully, great explanation and poc.

# Low
---
#### Title Wrong parameter used in event 
#### Vulnerability detail
According to the documentation, the first argument of the event should be 'the total number of points distributed.' But the event emitted in the function uses `pointsPerEpoch` as the first parameter which is incorrect.

```js
FjordPoints::distributePoints()

emit PointsDistributed(pointsPerEpoch, pointsPerToken);
```

#### Impact
The front-end might show wrong values.
#### Mitigation
use `totalPoints` instead of `pointsPerEpoch`

**Key takeaway** can you believe this finding fetched $795 as reward? the interesting part is only 4 people found it. And there is only one Low finding in this contest. This one low paid more than all the highs and mediums combined. And what is the finding? wrong event parameter. All I can say is shame on me. To win at this this is how deep one should go. 

### Alchemix transmuter

# Medium
---
#### Title Incorrect total asset calculation in `harverstAndReport()` leading to inflation of share value and irredeemable assets
#### Vulnerability detail
Underlying asset is the asset(WETH) which is deposited by users to receive alAssets.
The underlying asset is directly deposited to the strategy, accounting it with the total profit inflates the value of profit for users in `_harvestAndReport()`
The `claimAndSwap()` function is designed to allow users to claim WETH from transmuter and swap it to alAssets. However the implementation only accounts for newly claimed WETH not the total balance of the underlying asset which is already in the contract. This makes the assets irredeemable.
#### Impact
1. Existing shares are diluted as it accounts for directly transferred weth as profit 
2. Dormant weth in the contract becomes irredeemable
3. Which also affects protocol fee distribution
#### Mitigation
4. Account for total balance of underlying assets in `claimAndSwap` function
5. remove accounting of WETH in `harvestAndreport`.

Why did I missed it? 
6. Not looking deep enough
7. Not understanding connection between different assets and their accounting
8. Not understanding true purpose of the affected functions, meaning I didn't understood at depth the real purpose of the function hence I couldn't find out if it is doing something wrong.
How can I not miss it the next time?
9. Giving enough time
10. Trusting my gut, even if I don't know the exact reason still figure it out and submit
11. Understanding true purpose of the function and different assets in the protocol.

This finding wasn't hard, it was just a mistake in accounting of different assets. Small mistakes here and there which would lead to major issues like locked assets and inflation of rewards/ fees. 


#### Title # Incorrect Calculation of `_totalAssets` in `_harvestAndReport` Function
#### Vulnerability detail
The `harvestAndReport` function includes the underlying asset directly without converting it into the asset. It lead to incorrect calculation of total balance of assets.

Instead of adding the underlying token balance directly it should be converted to the asset token then it should be added for the total balance accounting.
#### Impact
Incorrect calculation will lead to improper behavior of the functions depending on `harvestAndReport`
#### Mitigation
```js
_totalAssets = unexchanged + asset.balanceOf(address(this)) + _swapUnderlyingToAsset(underlying.balanceOf(address(this)));
```

Extremely simple finding, this is the most common medium finding. I didn't paid enough attention that's why I couldn't find it. 

#### Title Inflated `totalAssets` accounting in `StrategyMainnet`, `StrategyArb`, and `StrategyOp` contracts.
#### Vulnerability detail

Similar issue as mentioned in M2, this time it is happening in `harvestAndReport` function.
#### Impact
#### Mitigation


# Low
---
#### Title : Hardcoded router address prevents future updates.
#### Vulnerability detail
The curve router address is hardcoded, which prevents the contract to be upgraded in future. 

Again this is something I suspected but I didn't had the knowledge or it didn't strike my head that this could happen. 
#### Impact

#### Mitigation

Make a upgradable only management function which would ensure upgradability.

#### Title Old router retains token allowance after upgraded
#### Vulnerability details

```js
function setRouter(address _router) external onlyManagement {
    router = _router;
    underlying.safeApprove(router, type(uint256).max);
}
```

If the router is changed, the old router will still have the token allowance. In case it is compromised it can lead to unwanted situations. 

When the router is upgraded the old router token allowance permission should be revoked.

```js
function setRouter(address _router) external onlyManagement {
    // Revoke allowance for the old router
   + underlying.safeApprove(router, 0);

    // Set new router and approve maximum allowance
    router = _router;
    underlying.safeApprove(router, type(uint256).max);
}
```

Why I missed this? 
I didn't had the knowledge that this could happen, hence I missed it. 

How not to miss it the next time?
Remember that when routers are upgraded token allowance permissions should be revoked, otherwise malicious actors can do something vulnerable that might affect the protocol.

This is the most reported issue, I don't know how I missed it. People got $0 for this as so many dups are there.

#### Title : Missing Deadline Checks in `claimAndSwap` Function
#### Vulnerability detail
Missing deadline checks leads to slippage. The tx and sit in a mempool for long time it's parameters like minout amount will get stale. This creates opportunities for miners to frontrun/ sandwich attack.

Again simple issue but lack of knowledge and indepth check made me miss it.

# Audit progress

**Current** : RAAC on Codehawks
**Previous** : 
Dahlia 
- Valid bug but invalidated due to not being able to provide PoC
DAAO_AI
- 1 Med validated, but 50+ dups

# Notes

### Liquidation & related vulns

**What is liquidation?**
Being able to force close someone's position in defi protocol
- Borrowing/Lending
- PnL(Perps and option)

### Borrow/ Lending protocol
- Put down collateral borrow against it
- value of collateral * LTV > value of borrow
- LTV : loan to value ratio (always less than 1)

#### Insolvency VS Liquidatable

**Liquidatable** 
value of collateral * LTV < value of borrow

**Insolvent position** : a position that does not have enough collateral to repay the borrow amount that it has. Very bad for the protocol, leads to losses. This also called as bad debt.
Value of your collateral < value of borrow

### PnL (Perps and options)

**Liquidatable** 
value of collateral + PnL goes below X amount

**Insolvent**
value of collateral + PnL goes below zero

### Vulns and edge cases 

1. Liquidation reverts due to blacklisted tokens
2. Liquidation reverts due to zero transfers
3. is liquidatable does not account for fees
4. liquidation should be collecting latet fees
5. liquidation reverts when position becomes insolvent
6. Liquidation reverts when a position cannot cover the fees
7. Liquidation reverts due to accounting error

