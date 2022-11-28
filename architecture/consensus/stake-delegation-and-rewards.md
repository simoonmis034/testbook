# Stake Delegation and Rewards

Stakers are rewarded for helping to validate the ledger.&#x20;

They do this by delegating their stake to validator nodes. Those validators do the legwork of replaying the ledger and sending votes to a per-node vote account to which stakers can delegate their stakes.&#x20;

The rest of the cluster uses those stake-weighted votes to select a block when forks arise. Both the validator and staker need some economic incentive to play their part.&#x20;

The validator needs to be compensated for its hardware and the staker needs to be compensated for the risk of getting its stake slashed.&#x20;

The economics are covered instaking rewards. This section, on the other hand, describes the underlying mechanics of its implementation.

## Basic Design

The general idea is that the validator owns a Vote account.&#x20;

The Vote account tracks validator votes, counts validator generated credits, and provides any additional validator specific state.&#x20;

The Vote account is not aware of any stakes delegated to it and has no staking weight.A separate Stake account (created by a staker) names a Vote account to which the stake is delegated.&#x20;

Rewards generated are proportional to the amount of lamports staked. The Stake account is owned by the staker only.&#x20;

Some portion of the lamports stored in this account are the stake.

## Passive Delegation

Any number of Stake accounts can delegate to a single Vote account without an interactive action from the identity controlling the Vote account or submitting votes to the account.

The total stake allocated to a Vote account can be calculated by the sum of all the Stake accounts that have the Vote account pubkey as the StakeState::Stake::voter\_pubkey.

## Vote and Stake accounts

​The rewards process is split into two on-chain programs.&#x20;

The Vote program solves the problem of making stakes slashable.&#x20;

The Stake program acts as custodian of the rewards pool and provides for passive delegation.&#x20;

The Stake program is responsible for paying rewards to staker and voter when shown that a staker's delegate has participated in validating the ledger.

### VoteState#​

VoteState is the current state of all the votes the validator has submitted to the network.&#x20;

VoteState contains the following state information:

&#x20; votes - The submitted votes data structure.

&#x20; credits - The total number of rewards this Vote program has generated over its lifetime.

&#x20; root\_slot - The last slot to reach the full lockout commitment necessary for rewards.

&#x20; commission - The commission taken by this VoteState for any rewards claimed by staker's Stake accounts. This is the percentage ceiling of the reward.

&#x20; Account::lamports - The accumulated lamports from the commission. These do not count as stakes.

&#x20; authorized\_voter - Only this identity is authorized to submit votes. This field can only modified by this identity.

&#x20; node\_pubkey - The Solana node that votes in this account.

&#x20; authorized\_withdrawer - the identity of the entity in charge of the lamports of this account, separate from the account's address and the authorized vote signer.

### VoteInstruction::Initialize(VoteInit)#​

&#x20; account\[0] - RW - The VoteState.

&#x20; VoteInit carries the new vote account's node\_pubkey, authorized\_voter, authorized\_withdrawer, and commission.other VoteState members defaulted.



### VoteInstruction::Authorize(Pubkey, VoteAuthorize)

Updates the account with a new authorized voter or withdrawer, according to the VoteAuthorize parameter (Voter or Withdrawer).&#x20;

The transaction must be signed by the Vote account's current authorized\_voter or authorized\_withdrawer.

&#x20; account\[0] - RW - The VoteState.&#x20;

VoteState::authorized\_voter or authorized\_withdrawer is set to Pubkey.



### VoteInstruction::AuthorizeWithSeed(VoteAuthorizeWithSeedArgs)

Updates the account with a new authorized voter or withdrawer, according to the VoteAuthorize parameter (Voter or Withdrawer).&#x20;

Unlike VoteInstruction::Authorize this instruction is for use when the Vote account's current authorized\_voter or authorized\_withdrawer is a derived key.&#x20;

The transaction must be signed by someone who can sign for the base key of that derived key.

&#x20; account\[0] - RW - The VoteState. VoteState::authorized\_voter or authorized\_withdrawer is set to Pubkey.

### VoteInstruction::Vote(Vote)

&#x20; account\[0] - RW - The VoteState. VoteState::lockouts and VoteState::credits are updated according to voting lockout rules seeTower BFT.

&#x20; account\[1] - RO - sysvar::slot\_hashes A list of some N most recent slots and their hashes for the vote to be verified against.

&#x20; account\[2] - RO - sysvar::clock The current network time, expressed in slots, epochs.



### StakeState

A StakeState takes one of four forms, StakeState::Uninitialized, StakeState::Initialized, StakeState::Stake, and StakeState::RewardsPool.&#x20;

Only the first three forms are used in staking, but only StakeState::Stake is interesting.&#x20;

All RewardsPools are created at genesis.

### StakeState::Stake

StakeState::Stake is the current delegation preference of the staker and contains the following state information:

&#x20; Account::lamports - The lamports available for staking.

&#x20; stake - the staked amount (subject to warmup and cooldown) for generating rewards, always less than or equal to Account::lamports.

&#x20; voter\_pubkey - The pubkey of the VoteState instance the lamports are delegated to.

&#x20; credits\_observed - The total credits claimed over the lifetime of the program.

&#x20; activated - the epoch at which this stake was activated/delegated. The full stake will be counted after warmup.

&#x20; deactivated - the epoch at which this stake was de-activated, some cooldown epochs are required before the account is fully deactivated, and the stake available for withdrawal.

&#x20; authorized\_staker - the pubkey of the entity that must sign delegation, activation, and deactivation transactions.

&#x20; authorized\_withdrawer - the identity of the entity in charge of the lamports of this account, separate from the account's address, and the authorized staker.

### StakeState::RewardsPool​

To avoid a single network-wide lock or contention in redemption, 256 RewardsPools are part of genesis under pre-determined keys, each with std::u64::MAX credits to be able to satisfy redemptions according to point value.

The Stakes and the RewardsPool are accounts that are owned by the same Stake program.

### StakeInstruction::DelegateStake

The Stake account is moved from Initialized to StakeState::Stake form, or from a deactivated (i.e. fully cooled-down) StakeState::Stake to activated StakeState::Stake.&#x20;

This is how stakers choose the vote account and validator node to which their stake account lamports are delegated.&#x20;

The transaction must be signed by the stake's authorized\_staker.

&#x20; account\[0] - RW - The StakeState::Stake instance. StakeState::Stake::credits\_observed is initialized to VoteState::credits, StakeState::Stake::voter\_pubkey is initialized to account\[1].&#x20;

If this is the initial delegation of stake, StakeState::Stake::stake is initialized to the account's balance in lamports, StakeState::Stake::activated is initialized to the current Bank epoch, and StakeState::Stake::deactivated is initialized to std::u64::MAX

&#x20; account\[1] - R - The VoteState instance.

&#x20; account\[2] - R - sysvar::clock account, carries information about current Bank epoch.

&#x20; account\[3] - R - sysvar::stakehistory account, carries information about stake history.

&#x20; account\[4] - R - stake::Config account, carries warmup, cooldown, and slashing configuration.

### StakeInstruction::Authorize(Pubkey, StakeAuthorize)

Updates the account with a new authorized staker or withdrawer, according to the StakeAuthorize parameter (Staker or Withdrawer).&#x20;

The transaction must be by signed by the Stakee account's current authorized\_staker or authorized\_withdrawer.&#x20;

Any stake lock-up must have expired, or the lock-up custodian must also sign the transaction.

&#x20; account\[0] - RW - The StakeState.

&#x20;   StakeState::authorized\_staker or authorized\_withdrawer is set to to Pubkey.

### StakeInstruction::Deactivate

​A staker may wish to withdraw from the network.&#x20;

To do so he must first deactivate his stake, and wait for cooldown.&#x20;

The transaction must be signed by the stake's authorized\_staker.

&#x20;  account\[0] - RW - The StakeState::Stake instance that is deactivating.

&#x20; account\[1] - R - sysvar::clock account from the Bank that carries current epoch.



StakeState::Stake::deactivated is set to the current epoch + cooldown.&#x20;

The account's stake will ramp down to zero by that epoch, and Account::lamports will be available for withdrawal.

### StakeInstruction::Withdraw(u64)

​Lamports build up over time in a Stake account and any excess over activated stake can be withdrawn.&#x20;

The transaction must be signed by the stake's authorized\_withdrawer.

&#x20; account\[0] - RW - The StakeState::Stake from which to withdraw.

&#x20; account\[1] - RW - Account that should be credited with the withdrawn lamports.

&#x20; account\[2] - R - sysvar::clock account from the Bank that carries current epoch, to calculate stake.

&#x20; account\[3] - R - sysvar::stake\_history account from the Bank that carries stake warmup/cooldown history.



## Benefits of the design

Single vote for all the stakers.

Clearing of the credit variable is not necessary for claiming rewards.

Each delegated stake can claim its rewards independently.

Commission for the work is deposited when a reward is claimed by the delegated stake.

## Example Callflow

​​Passive Staking Callflow

## Staking Rewards#​

The specific mechanics and rules of the validator rewards regime is outlined here.

&#x20;Rewards are earned by delegating stake to a validator that is voting correctly.&#x20;

Voting incorrectly exposes that validator's stakes toslashing.

### Basics​

The network pays rewards from a portion of networkinflation.&#x20;

The number of lamports available to pay rewards for an epoch is fixed and must be evenly divided among all staked nodes according to their relative stake weight and participation.&#x20;

The weighting unit is called apoint.

Rewards for an epoch are not available until the end of that epoch.

At the end of each epoch, the total number of points earned during the epoch is summed and used to divide the rewards portion of epoch inflation to arrive at a point value.&#x20;

This value is recorded in the bank in asysvarthat maps epochs to point values.

During redemption, the stake program counts the points earned by the stake for each epoch, multiplies that by the epoch's point value, and transfers lamports in that amount from a rewards account into the stake and vote accounts according to the vote account's commission setting.

### Economics

Point value for an epoch depends on aggregate network participation.&#x20;

If participation in an epoch drops off, point values are higher for those that do participate.

### Earning credits\#

Validators earn one vote credit for every correct vote that exceeds maximum lockout, i.e. every time the validator's vote account retires a slot from its lockout list, making that vote a root for the node.

Stakers who have delegated to that validator earn points in proportion to their stake. Points earned is the product of vote credits and stake.

### Stake warmup, cooldown, withdrawal

Stakes, once delegated, do not become effective immediately. They must first pass through a warmup period.&#x20;

During this period some portion of the stake is considered "effective", the rest is considered "activating".&#x20;

Changes occur on epoch boundaries.

The stake program limits the rate of change to total network stake, reflected in the stake program's `config::warmup_rate` (set to 25% per epoch in the current implementation).

The amount of stake that can be warmed up each epoch is a function of the previous epoch's total effective stake, total activating stake, and the stake program's configured warmup rate.

Cooldown works the same way. Once a stake is deactivated, some part of it is considered "effective", and also "deactivating".&#x20;

As the stake cools down, it continues to earn rewards and be exposed to slashing, but it also becomes available for withdrawal.

Bootstrap stakes are not subject to warmup.

Rewards are paid against the "effective" portion of the stake for that epoch.

### **Warmup example**

Consider the situation of a single stake of 1,000 activated at epoch N, with network warmup rate of 20%, and a quiescent total network stake at epoch N of 2,000.

At epoch N+1, the amount available to be activated for the network is 400 (20% of 2000), and at epoch N, this example stake is the only stake activating, and so is entitled to all of the warmup room available.

| epoch | effective | activating | total effective | total activating |
| ----- | --------- | ---------- | --------------- | ---------------- |
| N-1   | ​         | ​          | 2,000           | 0                |
| N     | 0         | 1,000      | 2,000           | 1,000            |
| N+1   | 400       | 600        | 2,400           | 600              |
| N+2   | 880       | 120        | 2,880           | 120              |
| N+3   | 1000      | 0          | 3,000           | 0                |

Were 2 stakes (X and Y) to activate at epoch N, they would be awarded a portion of the 20% in proportion to their stakes.&#x20;

At each epoch effective and activating for each stake is a function of the previous epoch's state.

| epoch | X eff | X act | Y eff | Y act | total effective | total activating |
| ----- | ----- | ----- | ----- | ----- | --------------- | ---------------- |
| N-1   | ​     | ​     | ​     | ​     | 2,000           | 0                |
| N     | 0     | 1,000 | 0     | 200   | 2,000           | 1,200            |
| N+1   | 333   | 667   | 67    | 133   | 2,400           | 800              |
| N+2   | 733   | 267   | 146   | 54    | 2,880           | 321              |
| N+3   | 1000  | 0     | 200   | 0     | 3,200           | 0                |

### **Withdrawal**

Only lamports in excess of effective+activating stake may be withdrawn at any time.&#x20;

This means that during warmup, effectively no stake can be withdrawn.&#x20;

During cooldown, any tokens in excess of effective stake may be withdrawn (activating == 0).&#x20;

Because earned rewards are automatically added to stake, withdrawal is generally only possible after deactivation.

### **Lock-up**

Stake accounts support the notion of lock-up, wherein the stake account balance is unavailable for withdrawal until a specified time.&#x20;

Lock-up is specified as an epoch height, i.e. the minimum epoch height that must be reached by the network before the stake account balance is available for withdrawal, unless the transaction is also signed by a specified custodian.&#x20;

This information is gathered when the stake account is created, and stored in the Lockup field of the stake account's state.&#x20;

Changing the authorized staker or withdrawer is also subject to lock-up, as such an operation is effectively a transfer.
