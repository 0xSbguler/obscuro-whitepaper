## Proof of Block Inclusion

Obscuro uses a novel decentralised round-based consensus protocol based on a fair lottery and on synchronisation with the L1, specifically designed for L2 rollups, called _Proof Of Block Inclusion_ (POBI). It solves, among others, the fair leader election problem, which is a fundamental issue that all decentralised rollup solutions have to address. POBI is inspired by [Proof Of Elapsed Time](https://sawtooth.hyperledger.org/docs/core/releases/1.0/architecture/poet.html).

### High Level Description

The high level goals of the POBI protocol are:

1. Each round, to distribute the sequencer function fairly among all the active registered aggregators.
2. To synchronise the L2 round duration to L1 rounds. Because the L1 is the source of truth, the finality of the L2 transactions is dependent on the finality of the L1 rollup transaction that includes them, which means there is no advantage in publishing multiple rollups in a single L1 block. It is not possible to decrease the finality time below that of the L1. On the other hand, publishing L2 rollups less frequently means that L2 finality will be unnecessarily long. The optimum frequency is to publish one rollup per L1 block.

To achieve fairness, the PoBI protocol states that each round the TEE can generate one random nonce and the winner of a round is the aggregator whose TEE generated the lowest random number from the group. The TEEs will generate these numbers independently and then will gossip them. The aggregators who didn't win the round, similar to L1 miners, will respect this decision because it is the rational thing to do based on the incentive mechanism. If they don't want to respect the protocol, they are free to submit a losing rollup to the L1, but it will be ignored by all compliant aggregators, which means such an aggregator has to pay L1 gas and not get any useful reward.

We achieve the second goal by linking the random nonce generation which terminates a round to the Merkle proof of inclusion of the parent rollup in a L1 block. This property is what gives the name of the protocol. This means that an aggregator is able to obtain a signed rollup from the TEE only if it is able to present a Merkle proof of block inclusion. This feature links the creation of L2 rollup to an L1 block, thus synchronising their cadence.

A party wishing to increase its chances of winning rounds will have to register multiple aggregators and pay the stake for each. The value of the stake needs to be calculated in such a way as to achieve a right decentralisation and practicality balance. This will be discussed further in the incentives section.

It is very easy for all the other aggregators to verify which rollup is the winner, by just comparing the nonces and checking that the rollup signature is from an approved aggregator.

Note that the L1 management contract is not checking the nonces of the submitted rollups, but it checks that the block inclusion proof is valid. The L1 contract will reject rollups generated using a proof of inclusion that is not an ancestor of the current block.

A further issue to solve is to ensure that the host will not be able to repeatedly submit the proof to the TEE to get a new random number.

### Typical Scenario

1. A new round starts from the point of view of an aggregator when it decides that someone has gossiped a winning rollup. At that point it creates a new empty rollup structure, points it to the previous one, and starts adding transactions to it (which are being received from users or by gossip).
2. In the meantime it is closely monitoring the L1 by being directly connected to a L1 node.
3. As soon as the previous rollup was added to a mined L1 block, the aggregator takes that Merkle proof, feeds it to the TEE who replies with a signed rollup containing a random nonce generated inside the enclave.
4. All the other aggregators will be doing roughly the same thing at the same time.
5. At this point (which happens immediately after the successful publishing of the previous rollup in the L1), every aggregator will have a signed rollup with a random nonce which they will gossip between them. The party with the lowest nonce wins. All the aggregators know this, and a new round starts.
6. The winning aggregator has to create an ethereum transaction that publishes this rollup to L1.

Note that by introducing the requirement for proof of inclusion in the L1, the cadence of publishing the rollups to the block times is synchronised. Also, note that the hash of the L1 block used to prove to the TEE that the previous rollup was published will be added to the current rollup such that the management contract, and the other aggregators know whether this rollup was actually generated correctly.

This sequence is depicted in the following diagram:
![node-processing](./images/node-processing.png)

### Notation

There are four elements which define a rollup :

- the parent
- the generation
- the aggregator who generated it
- the generation of the L1 block used as proof
- the generation of the L1 block that includes this rollup.
- the nonce

The notation is the following: _R_$Rollup_Generation[$Aggregator, $L1_Proof_Generation, $L1_Block_Generation, $Nonce]_

Note that: L1_Proof_Generation < L1_Block_Generation .

Example: _R_15[Alice, 100, 102, 20]_ means the generation is 15, the aggregator is _Alice_, the generation of the L1 bock used as proof is 100, the generation of the L1 bock that included the rollup is 102, and the nonce equals 20.

### The canonical chain

The POBI protocol allows any aggregator to publish rollups to the management contract. This means that short-lived forks are a normal part of the protocol. The ObscuroVM running inside the TEE of every node must be able to deterministically select one of the forks as the canonical chain and only append a rollup on top of that. This means that the TEE must receive all the relevant content of the L1 blocks, and the logic must be identical on all nodes.

The rules for the canonical chain:

1. The genesis rollup is part of the canonical chain, and will be included in a block by the first aggregator.

2. If an L1 block contains a single rollup whose parent is the head rollup of the canonical chain included in a previous L1 block, it will also be on the canonical chain if no other rollup with the same parent was included in an earlier block. Any other sibling rollup included in a later block is not on the canonical chain. This is the _Primogeniture_ rule, where a rollup is born when included in an L1 block.

3. If an L1 block contains multiple sibling rollups created in the same round using the same L1 proof, the one with the lower nonce will be on the canonical chain.

4. If an L1 block contains multiple sibling rollups created using different L1 proofs, the one created more recently is on the canonical chain.

[comment]: <> ([TODO - diagram depicting these scenarios])

Using the notation, for the same _Rollup_Generation_, the rollup that will be on the canonical chain is the one with:

- the lowest L1_Block_Generation
- in case there are multiple use the highest L1_Proof_Generation
- in case there are multiple use the lowest Nonce

Given that the nonce is a random number with sufficient entropy, we assume there cannot be a collision at this point.

### Preventing Repeated Random Nonce Generation

In phase 3 of the protocol, the TEE of each aggregator generates a random nonce which determines the winner of the protocol. This introduces the possibility of gaming the system by restarting the TEE, and attempting to generate multiple numbers.

The solution proposed by Obscuro is to introduce a timer upon startup of the enclave, in the constructor. A conventional timer, based on the clock of the computer is not very effective since it can be gamed by the host.

The solution is for the enclave to serially (on a single thread) calculate a large enough number of SHA256 hashes which it wouldn't be able to do faster than an average block time on even a very fast CPU.

This solution is effective, since the code is attested, and does not rely on any input from the host.

A node operator wanting to cheat would restart the enclave, and quickly feed it the proof of inclusion, but the enclave will only process it after 15 seconds, which means the operator has already missed the time window for that rollup.

This built-in startup delay is also useful in preventing other real time side channel attacks, which could be used for MEV.

### Aggregator Incentives

Compared to a typical L1 protocol, there is an additional complexity. In a L1 like Bitcoin or Ethereum, once a node gossips a valid block, all the other nodes are incentivised to use it as a parent, because they know everyone will do that as well. In a L2 decentralised protocol like POBI, there is an additional step, which is the publication of the rollup to L1, which can fail for multiple reasons.

The high level goal is to keep the system functioning as smoothly as possible, and be resistant to random failures or malicious behaviour, while not penalising Obscuro nodes for not being available.

These are the aggregator rewarding rules:

1. The first aggregator that has successfully published a rollup without competition in the current or next L1 block, will get the full reward. This is the most efficient case that is encouraged.
_Note: Competition means a rollup from the same generation._ 

2. If multiple rollups generated with the same L1 proof and different nonces are published in the same block, the one with the lowest nonce is on the canonical chain, but the reward is split between them in a proportion of 80/20 (this ratio is indicative only). The reason for this rule is that it incentivises aggregators to gossip their winning rollup such that no one else publishes at the same time.
    - There is no incentive for the losing aggregator to publish as 20% of the reward will not even cover the cost of gas.
    - There is an incentive for the winning aggregator to gossip to everyone else to avoid having this competition.
    - In case of a genuine failure in gossip, the losing aggregator will receive something. This is to reduce the risk
      of aggregators just waiting more than necessary to receive messages from all the other aggregators.

3. If multiple sibling rollups generated using different L1 blocks as proof are included in the same block, the one created with the most recent proof will receive the full reward. 
 The original winning rollup that didn't get published immediately will not receive any reward since there was more recent competition. This rule is designed to encourage publishing with enough gas, such that there is no risk of competition in a further block. The rule also encourages aggregators to not wait for rollups published with insufficient gas or not at all. As soon as there is a new L1 block, the round resets, and the reward is up for grabs. An actor controlling multiple aggregators with malicious irrational behaviour will only be able to slow the ledger down, because the rational actors will publish the rounds they win.

4. If two consecutive L1 blocks include each a rollup from the same generation created from the same L1 proof, but the rollup from the second block has a lower nonce, split the reward evenly between the two aggregators. 
 Note that the rollup with the higher nonce will be on the canonical chain. 
 The reason for this rule is that this scenario is possibly the result of rollup front-running, which is thus discouraged as the frontrunner is consuming precious Ethereum gas.
 The even splitting of the reward also encourages the aggregator that wins a nonce generation round to publish as soon as possible, because publishing a block later will at best result in a small loss.

6. If two rollups created from the same L1 proof, are published more than one block apart, where the first published rollup has a higher nonce, then pay the reward in full to the first published rollup. The reason for this rule is that the winner most likely added far too little gas, and someone else spotted the opportunity and contributed to earlier finality, which is rewarded.

7. Do not pay rewards in any other case.

The reward rules are depicted in the following diagram:
![L1 rewarding](./images/block-rewarding.png)

The rules in the case of front-running are depicted in the following diagram:
![L1 front running](./images/block-frontrunning.png)

There are four types of rewards:

- _Full Reward_ (FR)
- _Minimal Reward_ (MR - which covers the cost of gas)
- _Split_Reward_ (SR - a percentage of FR)

Using the notation, this is how to calculate the rewards that can be claimed by an _Aggregator_ for a _Rollup_Generation_.

This is python-like pseudocode. Note that it is not comprehensive.

```python
# The rollup generation for which we calculate the rewards
generation = N

# 'genN_L1_Blocks' is a list of all L1 blocks starting with the _L1_Block_Generation_ of the head 
# of the canonical chain of the previous generation, until the block where you encounter the 
# first valid rollup of _Rollup_Generation_ plus one extra L1 block.
genN_L1_Blocks = calculateBlocks()

# List of rollups of generation N found in the last block
rollups_in_last_block = genN_L1_Blocks[-1].rollups.filter(lambda r: r.generation == generation)

# List of rollups of generation N found in the target block
rollups_in_target_block = genN_L1_Blocks[-2].rollups.filter(lambda r: r.generation == generation)

if rollups_in_target_block.size == 1 and rollups_in_last_block.size == 0:

    # There is no competition for the target rollup
    fullRewardTo(rollups_in_target_block[0].aggregator)

elif rollups_in_target_block.size == 1 and rollups_in_last_block.size == 1:

    # There is competition for the target rollup in the next rollup
    # Which means there is suspicion of frontrunning
    target_rollup = rollups_in_target_block[0]
    competition_rollup = rollups_in_last_block[0]
   
    if competition_rollup.L1_Proof_Generation == target_rollup.L1_Proof_Generation and competition_rollup.nonce < target_rollup.nonce:
        # This is possibly front-running or failure to gossip
        # All parties involved in this will make a small loss
        partialRewardTo(target_rollup.aggregator, '50%')
        partialRewardTo(competition_rollup.aggregator, '50%')
    else:
        # The target has the lower nonce or is generated with a different proof
        fullRewardTo(target_rollup.aggregator)

elif rollups_in_target_block.size == 2:
    # Two competing rollups in the target block
    # This is not a frunt-running situation, so eventual rollups published in the next block don't matter
    rollup1 = rollups_in_target_block[0]
    rollup2 = rollups_in_target_block[1]

    if rollup1.L1_Proof_Generation == rollup2.L1_Proof_Generation:

        # According to rule #2 the competing rollups will split the reward 
        if rollup1.nonce < rollup2.nonce:
            partialRewardTo(rollup1.aggregator, '80%')
            partialRewardTo(rollup2.aggregator, '20%')
        else:
            partialRewardTo(rollup2.aggregator, '80%')
            partialRewardTo(rollup1.aggregator, '20%')

    elif rollup1.L1_Proof_Generation > rollup2.L1_Proof_Generation:

        # According to rule #3 the rollup generated with the more recent proof gets the reward 
        fullRewardTo(rollup1.aggregator)
   
    else:
        # According to rule #3 the rollup generated with the more recent proof gets the reward 
        fullRewardTo(rollup2.aggregator)
        
else:
    pass
```

_Note that these rules are subject to tweaking based on real life experience._

### Failure Scenarios

The next sections will analyze different failure scenarios and how the incentive rules ensure the good functioning of the protocol.

#### 1. The winning sequencer does not publish

The winning aggregator is incentivised to publish the rollup in order to receive the reward. This scenario should only occur infrequently, if the aggregator crashes or malfunctions. In case it happens, it will only be detected by the other aggregators when the round is supposed to end, and the next rollup is ready to publish. They will notice that the winning rollup was not added to the L1 block.

In this situation, every aggregator will:

* Discard the current rollup.
* Unseal the previous rollup.
* Add all current transactions to it.
* Then seal it using the last empty block.
* Gossip it.

In effect this means that the previous round is replayed. The winning aggregator of this replayed round has priority over the reward in case the previous winner is eventually added in the same block.

#### 2. The winning sequencer adds too little gas, and the rollup just sits in the mempool unconfirmed

This scenario has the exact same effect as the previous one and can be handled in the same way. If the rollup is not in the next block, the round is replayed.

Publishing with insufficient gas is in effect punished by the protocol, because it means that on top of missing the rollup reward, the aggregator will also pay the L1 gas fee, and there is no guarantee that she will receive the reward.

### Competing L1 Blockchain Forks

In theory, different L2 aggregators could be connected to L1 nodes that have different views of the L1 ledger. This will be visible in the L2 network as rollups being gossiped that point to different L1 forked blocks. Each aggregator will have to make a bet and continue working on the L1 fork which it considers to have the best chance. This is the same behaviour as any L1 node.

This is depicted in [Rollup Data Structure](detailed-design#rollup-data-structure).

In case it proves that the decision was wrong it has to roll back the state to a checkpoint and replay the winning rollups.