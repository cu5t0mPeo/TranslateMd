# TranslateMd
Feature: Secondary Rewarder
https://github.com/notional-finance/contracts-v3/pull/21/commits/3903761e2a03c8cebf575bdca3efaf0795b48783

Currently, NOTE tokens are given out as incentives for providing liquidity (minting nTokens) using the incentive calculation in Incentives.sol.

We are extending this functionality to allow nTokens to be incentivized by a secondary reward token. On Arbitrum, this will be ARB as a result of the ARB STIP grant. In the future, this may be any arbitrary ERC20 token. The Secondary Rewarder contract will be deployed externally to the Notional system and whitelisted via `setSecondaryIncentiveRewarder` in Treasury Action.

Once whitelisted, the Secondary Rewarder will be called after NOTE tokens have been claimed. The secondary rewarder does the following:

Calculates the total reward token accumulated to the account and transfers the reward tokens to the account.
Allows the Notional owner to set a new emission rate and a timestamp when that emission rate will stop accruing.
Allows the Notional owner to recover any tokens sent to the address or any reward tokens remaining on the contract in excess of the total rewards emitted.
Allows the incentive rewarder to be “detached” from the Notional contract.
Once the rewarder is detached, the owner will publish a merkle root which contains the nToken balance of every valid account at the point of detachment. Accounts will claim their final incentives directly against the Rewarder. Effectively, the rewarder turns into an airdrop contract at this point.
This design ensures that the rewarder uses the accurate point in time nToken balance when claiming rewards once detached.

Invariants:
The rewarder must be called on every nToken balance change.
The rewarder must emit a total of `emissionRatePerYear` tokens pro rata over any given time period, regardless of the number of nToken balance changes or the nToken total supply.
The rewarder’s emissions can no longer increase after it is detached.


Feature: External Lending Rebalancing
https://github.com/notional-finance/contracts-v3/pull/21/commits/0f58c24019f8162776a5d94d7748ab38e801a35a

Notional V3 includes a feature where underlying tokens held by the contract which are unused may be lent out to other lending protocols such as Aave or Compound for increased yield. This feature extends the rebalancing code to add additional safeguards for this lending activity.

There are two parameters that ensure sufficient liquidity for withdraws and liquidations:

Target Utilization: the amount of the total prime supply that Notional will lend to external money markets. If the target utilization is 80% and the current prime cash market utilization is at 70%, that means 10% of the total prime supply is eligible for external lending. If the market utilization is above the target utilization then external lending will be reduced to zero. This setting ensures that prime cash holders have sufficient liquidity due to utilization on Notional itself.

External Withdraw Threshold: ensures that Notional has sufficient liquidity to withdraw from an external lending market. If Notional has 1000 units of underlying lent out on Aave, it requires 1000 * externalWithdrawThreshold units of underlying to be available on Aave for withdraw. This ensures there is sufficient buffer to process the redemption of Notional funds. If available liquidity on Aave begins to drop due to increased utilization, Notional will automatically begin to withdraw its funds from Aave to ensure that they are available for withdrawal on Notional itself.


The PrimeCashHoldingsOracle manages all the interface code between Notional and the target External Lending market. Currently there is one, the AaveV3HoldingsOracle. This contract generates the necessary calldata for redemptions, deposits and rebalancing.

Currently, the rebalancing functionality is limited to a single external lending market by design.

Rebasing Tokens
Notional tracks the “stored token balance” of all of its holdings to prevent donation attacks that could inflate the value of prime cash. When an external lending market uses rebasing tokens (such as AaveV3), the `balanceOf` selector returns the current value of principal plus interest the address holds in terms of underlying tokens.

At the point of deposit, this `balanceOf` selector returns the principal balance (with the exception of potential rounding issues for tokens with 6 decimals like USDC). Since Notional uses the difference between two successive `balanceOf` calls before and after a deposit or redeem on an external lending market, the “stored token balance” only ever tracks changes to the principal held by the Notional contract. 

Effectively, any interest is treated as a “donation” to the protocol and is not recognized in the total underlying balance available to prime cash holders. The Treasury Action contract exposes a method that allows the Notional treasury to redeem this accrued interest as profits to the DAO.
Non Rebasing Tokens
If the external lending market uses non-rebasing tokens, the `balanceOf` selector returns some other denomination that can only be converted to underlying value via an exchange rate. That exchange rate would be applied in the PrimeCashHoldingsOracle via the `getUnderlyingValueStateful` method. As underlying value increases, it increases the value of the underlying balance available to prime cash holders. Therefore, interest accrued is credited to prime cash holders via the `underlyingScalar` in PrimeCashExchangeRate. The DAO would have no claim over this interest accrued.


It should be noted that external lending protocol tokens may be wrapped and unwrapped between rebasing and non-rebasing versions by a PrimeCashHoldingsOracle, allowing Notional to flexibly allocate this external lending interest between the protocol DAO and prime cash holders.

Feature: Prime Debt Cap
https://github.com/notional-finance/contracts-v3/pull/21/commits/80d3e9383fee7921f5978bf699a927f673fc4a9b

Prevents accounts from borrowing at a variable rate above some percentage of the total supply cap. The purpose of this change is to prevent the following situation:
Users deposit up to the max supply.
Users borrow all the cash in the lending market and push utilization to 100%.
At this point, prime cash is not redeemable. This is dangerous and makes prime cash un-liquidatable. Notional’s protection against this situation is to increase interest rates substantially as utilization increases. But if we’re already at the max supply, new lenders can’t deposit which means that it is much harder for Notional to ensure redeemability of prime cash. . 

Variable debt is represented by a negative cash balance and can be created the following ways in Notional V3:

[Prime Debt Cap Enforced] Accounts withdraw to a negative cash balance, this is the equivalent of borrowing at a variable rate. Accounts can withdraw funds out of Notional or use their debt to provide liquidity or lend fixed. These two scenarios affect the prime debt cap differently, see the table below.
[Prime Debt Cap Enforced] Accounts borrow at a variable rate in leveraged vaults.
Accounts are liquidated by a collateral currency liquidation and can incur a negative cash balance because they are holding fCash collateral. This is the equivalent of a forced interest rate swap.
Accounts are holding matured fCash debts (negative fCash balances) and then they are settled to negative cash balances. This is a conversion from fixed rate debt to variable rate debt at maturity.
Accounts borrow at a fixed rate in leveraged vaults and then reach maturity, they are then settled to a variable rate debt in leveraged vaults. This is the same as the above scenario, but for leveraged vault accounts.

This feature adds a setting called maxPrimeDebtUtilization which is defined as a percentage from 0 to 100 of the maxUnderlyingSupply, which is the supply cap for a given currency. The prime debt cap is only enforced in situations #1 and #2. Situations #3, #4, #5 must always be allowed to proceed in order for the system to remain functional and solvent.

It is understood that due to the nature of settlement, accounts may generate short dated fCash and push the system above the debt cap at maturity. This would be alleviated by tracking overall fCash debt outstanding, however, this adds significantly more complexity to the system and makes the debt cap more difficult to administer. If this does become an issue a mitigation would be a temporary increase in the supply cap. If this is a persistent issue, a more permanent mitigation would include a more comprehensive tracking and limiting of total fCash debts.

Situation
Total Prime Supply
Total Prime Debt
Total Underlying
Supply Cap
Debt Cap
Borrow Variable and Withdraw
Increase
Increase
Decrease
Enforced
Enforced
Deposit and Repay Variable Debt
Decrease
Decrease
Increase
Not Enforced
Not Enforced
Deposit and Lend Variable
Increase
No Change
Increase
Enforced
Not Enforced
Withdraw Variable Lend
Decrease
No Change
Decrease
Not Enforced
Not Enforced
Deposit and Lend Fixed or Provide Liquidity
Increase
No Change
Increase
Enforced
Not Enforced
Borrow Fixed and Withdraw
Decrease
No Change
Decrease
Not Enforced
Not Enforced
Borrow Variable and Lend Fixed, Provide Liquidity or Leveraged Vault
Increase
Increase
No Change
Enforced
Enforced
Borrow Fixed and Lend Variable, Provide Liquidity or Leveraged Vault
No Change
No Change
No Change
Not Enforced
Not Enforced
Settle Fixed Lends and Borrows
Increase
Increase
No Change
Not Enforced
Not Enforced



Feature: nToken Mint Deviation
https://github.com/notional-finance/contracts-v3/pull/21/commits/c8c7b516d20ecc3240a21817c231e3beedffb518

The nToken is a unique liquidity provider that holds assets in three categories. nToken mechanics are further described here: https://docs.notional.finance/notional-v3/ntokens/ntoken-mechanics

A prime cash balance that is not allocated to any of the active fCash markets. This is a result of fee collection from leveraged vaults and repurchase of nToken residuals. This balance can be reallocated to fCash markets using the `sweepCashIntoMarkets` function.
One set of liquidity tokens per fCash market. The nToken is the only account that is allowed to hold liquidity tokens, it is the sole liquidity provider in the fCash markets. Each fCash market has a balance of cash and positive fCash associated with it.
A balance of fCash on a number of fCash maturities. These are typically negative when associated with active fCash markets and may be positive or negative if they are “idiosyncratic”. Adding this fCash balance with the positive fCash in the fCash market produces the “net fCash” balance for the nToken.

Typically speaking, if there has been more lending than borrowing, the net fCash figure is negative – the nToken as a market maker has been doing more net borrowing. Conversely, if there is more borrowing then the net fCash figure is positive.

When the value of an nToken is calculated, the cash balances are added up at their face value and the net fCash figure is discounted to present value using the oracle interest rate. This oracle interest rate is a TWAP (time weighted average interest rate) which converges to the current interest rate over some number of hours. This value is used when minting nTokens and when calculating the collateral value of nTokens.

However, when nTokens are redeemed, the account redeeming will receive their share of the net fCash amount pro-rata to their share of nTokens and then sell them back to the existing fCash market at the current market rate (which may deviate significantly from the oracle rate). This creates a valuation discrepancy. The mint and redeem nToken values can differ. This creates the potential for an MEV opportunity at certain edge conditions where accounts may sandwich a trade by minting nTokens at the favorable oracle rate and then selling them back at the market interest rate by redeeming them. 

This opportunity would only be profitable and exist when many factors coincide, however, out of an abundance of caution we are changing the nToken mint logic to use the current interest rates instead of the oracle rates to calculate mint values. This brings the mint and redeem values in line and shuts down any potential MEV edge case scenario.

We’re also adding a max price deviation limit to prevent minting of nTokens when the market price and oracle price of the nToken deviate above some percentage. This ensures that a user can’t mint nTokens and then immediately be credited with collateral value greater than the amount they deposited (because nToken collateral value is determined using oracle rates and not market rates). Again, this is an edge case scenario but for completeness we want to protect against it using this price deviation limit.

Feature: Prime Cash ERC4626
https://github.com/notional-finance/contracts-v3/pull/21/commits/4ce37e9f98c7dc61000d1763ad8e6306557cff01

Updates the PrimeCashProxy contract to support ERC4626 compatible mint, deposit, redeem and withdraw functionality. Prime Cash is represented as an ERC4626 contract but these methods are currently disabled.

On Mint and Deposit:

Account will call mint or deposit on the BeaconProxy
BeaconProxy will use the PrimeCashProxy implementation
BeaconProxy will transferFrom `asset` tokens from the account
BeaconProxy will call AccountAction.depositUnderlyingToken with the account specified.
NotionalProxy will transfer underlying tokens and mint into the account specified.

NOTE:
Prime Cash ETH will use WETH in the BeaconProxy and unwrap it and forward native ETH to the NotionalProxy
In AccountAction.depositUnderlyingToken a restriction where msg.sender == account is removed. It should be noted that this check was used to prevent potential donation issues for integrating protocols, however, it is always possible to simply mint prime cash and use an ERC20 transfer it to any account so the restriction is ineffective.

On Redeem and Withdraw:

BeaconProxy will call AccountAction.withdrawViaProxy method on the NotionalProxy
msg.sender must equal the PrimeCashProxy address for the matching currencyId
Allowances are checked if account is not the same as the spender.
Withdrawals are restricted to the balance of the account, i.e. the account cannot borrow via ERC4626 methods.
The prime cash is deducted from the account and the underlying tokens are sent to the receiver.
A free collateral check is processed on the account if it has debt balances.

NOTE:
WETH is sent from the NotionalProxy, not ETH due to the `withdrawWrapped` flag set to true on `finalizeWithWithdrawReceiver`

Bugfix: Free Collateral Dust Rounding
https://github.com/notional-finance/contracts-v3/pull/21/commits/f217109e854bef795c95bc701d37b0e59b5c9208


This is a minor fix to ensure that dust debt balances return a -1 value in the case that they round down to zero. This fix is applied to three places where debt balances are used in collateral valuations:

Discounting fCash to present value
Prime Cash to Underlying conversion
Underlying to ETH conversion

Feature: Wrapped fCash
https://github.com/notional-finance/wrapped-fcash/pull/12

These are a set of changes to the Wrapped fCash contract to support Notional V3. The changes include:

Removing the ability to mint and redeem via “Asset Cash” which is deprecated in Notional V3.
Adding the ability to “mint” wrapped fCash when the fCash interest rate is near zero. 
Adding the ability to “withdraw” wrapped fCash when it has previously lent at zero interest rates.

Lending at zero interest rates allows wrapped fCash to continue to work in all scenarios which may be important for certain integrations, (e.g. Cross Currency fCash Leveraged Vaults). 

Minting at Zero Interest
When lending fCash, the user is actually purchasing an fCash token on an AMM. Lending a sufficiently large position at low interest rates may require purchasing more fCash than exists on the AMM. Instead of reverting, the wrapped fCash contract will simply deposit the funds into Notional to be minted as “Prime Cash” which is lending at a variable rate.

The net effect is that the wrapped fCash lender lends at a zero interest rate (i.e. 1 USDC = 1 fUSDC) and the excess variable rate interest will simply accrue to the protocol.

Redeeming Prior to Maturity
Redemptions prior to maturity will sell fCash on the market and return the resulting cash to the account. If the wrapped fCash contract has minted at zero interest, at some point there will be insufficient fCash to sell on the market. In this case, we calculate using the oracle interest rate how much the fCash would sell for and transfer that amount to the account.

Because the account which lent at zero interest purchased fCash at its maximum price (1 to 1), we can be assured that there will be sufficient cash to transfer back to the account. fCash will always trade at a discount prior to maturity.

Redeeming Post Maturity
Post maturity, fCash will settle to a cash balance. Each account will receive the value of their wrapped fCash shares converted to a prime cash balance. We do not use a pro rata share of the prime cash held by the contract since that may result in a windfall for wrapped fCash holders and encourage MEV type activity.
