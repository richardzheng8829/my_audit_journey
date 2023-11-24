# General
1) Unchecked returned value of transfer of ERC20 token, Should use `SafeERC20.safeTransfer`.
```
-       collateralToken.transfer(msg.sender, amount); // @audit -unchecked return value of transfer
+       collateralToken.safeTransfer(msg.sender, amount);
```
2) Incompatible with fee-on-transfer or rebasing ERC20 tokens
3) Lack of `__Ownable_init()` and `__ReentrancyGuard_init()` at initialize(), constructor with `_disableInitializer()`
4) Should inherit upgradeable counterpart of base contracts for upgradeability

# Governance.sol

1) Malfunctioning vote counting mechanism, anyone with large balance can execute proposal irrespective of number of vote for proposal.
```
        if (balanceOf[msg.sender] >= quorumVotes) { // @audit - invalid vote counting, should not use executor's balance
```

2) Number of votes cast for proposal is not tracked in `vote` or `revokeVote`.

2) Total supply is not updated when balanceOf is updated at `createProposal()`, `vote()`, `revokeVote()`
```
balanceOf[msg.sender] = balanceOf[msg.sender].sub(proposalDeposit);
balanceOf[msg.sender] = balanceOf[msg.sender].sub(1);
balanceOf[msg.sender] = balanceOf[msg.sender].add(1);
```

4) Unreachable code
```
        if (block.timestamp >= proposal.votingEndTime) {
            // If voting ends, reset hasVoted status
            hasVoted[msg.sender] = false;
        }
```
5) Lack of storage gap

# LendingPool.sol
1) All borrowing token reserves can be drained by calling `borrow()` time and again with amount smaller than `collateralBalance / 1.5`.

- CR should be calculated as `Collateral / Debt` not `Collateral / Borrowing Amount`.

```
-       require(collateralBalance[msg.sender] >= amount.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
        uint256 interest = amount.mul(interestRate).div(100);
        uint256 totalRepayment = amount.add(interest);
        borrowBalance[msg.sender] = borrowBalance[msg.sender].add(totalRepayment); 
+       require(collateralBalance[msg.sender] >= borrowBalance.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
```

2) Collateral can be withdrawn without repaying remaining debt.
```
    function removeCollateral(uint256 amount) public nonReentrant {
        ...
        collateralBalance[msg.sender] = collateralBalance[msg.sender].sub(amount); // @audit - use unchecked to save gas
+       require(collateralBalance[msg.sender] >= borrowBalance.mul(minCollateralRatio).div(100), "Insufficient collateral"); 
```

3) Important operations like `addCollateral()`, `removeCollateral()`, `borrow()`, `repay()` are not pausable because of missing `whenNotPaused` modifier.

```
-    function removeCollateral(uint256 amount) public nonReentrant {
+    function removeCollateral(uint256 amount) public whenNotPaused nonReentrant {
```

4) Theres' no incentive for borrower to repay earlier because interest to be paid is calculated only once and does not grow over time.

```
        uint256 interest = amount.mul(interestRate).div(100);
        uint256 totalRepayment = amount.add(interest);
```

5) No upper limit for interest rate, owner can rug user by sandwiching `borrow()` with `setInterestRate()`.

6) Unused `collateralizationBonus` maybe for liquidator.

# LiquidityPool.sol
1) Unable to withdraw if `withdrawalCooldown` >= `withdrawalWindow` which is true by default
```
        uint256 cooldownEndTime = block.timestamp + withdrawalCooldown;
        if (block.timestamp < cooldownEndTime) {
            // @audit - unable to withdraw, if withdrawalCooldown >= withdrawalWindow 
            require(block.timestamp + withdrawalWindow >= cooldownEndTime, "Withdrawal window closed"); 
        } 
```
2) Wrong access control for `setWithdrawCooldown()`

```
    function setWithdrawalCooldown(uint256 cooldown) public {
-        require(msg.sender == owner() && isadmin(msg.sender), "Not the owner or admin"); // @audit - should be ||
+        require(msg.sender == owner() || isadmin(msg.sender), "Not the owner or admin"); 
        withdrawalCooldown = cooldown;
    }
```
3) Lack of storage gap

# Token.sol
1) If `burnFee` is greater than `transferFee`, it might revert because of lack of token balance.

- Let's say amount = balanceOf(msg.sender).

- After transfer to `to`, remaining balance will be feeAmount.
```
        super.transfer(to, netAmount);
```
- If burnFee is greater than tranferFee, burnAmount will be greater than remaining balance, thus `_burn()` will revert.
```
            uint256 burnAmount = amount.mul(burnFee).div(100);
            _burn(msg.sender, burnAmount); // @audit if fee amount is smaller than burn fee, it will create insolvency.
```

2) As transfer fee is not deducted from `msg.sender`, fee to be collected is transferrable to other addresses.

 - `transfer()` should send tranfer fee to treausry address, otherwise it can be transferred and fee to be collected are lost.
```
+       _transfer(msg.sender, feeTreasury, feeAmount);
        totalFees[msg.sender] = totalFees[msg.sender].add(feeAmount); // @audit - collected fee can be transfered.
```

3) Net transfer amount should be subtracted by burn amount.

```
-       net amount  = total amount - fee amount
+       net amount  = total amount - fee amount - burn amount
```
For every transfer, amount should be divided to three addresses.

- net amount for `to` address
- fee amount for `feeTreasury` address
- burn amount for `zero` address


```
    function transfer(address to, uint256 amount) public override returns (bool) {
        uint256 feeAmount = amount.mul(transferFee).div(100);
-       uint256 netAmount = amount.sub(feeAmount);
-       super.transfer(to, netAmount);
         ...
+       uint256 netAmount = amount.sub(feeAmount).sub(burnAmount);
+       super.transfer(to, netAmount);

        emit FeeCollected(msg.sender, feeAmount);
        return true;
    }
```