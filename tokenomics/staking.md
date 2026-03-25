# Staking

## Introduction

Staking is a core initiative of Yotta Protocol, bringing a new layer of cryptoeconomic security to the Yotta Network. Staking enables ecosystem participants, such as broker node operators and community members, to back the performance of decentralized GPU services with staked YOT tokens and earn rewards for helping secure the network.

## Staking Mechanism

The staking feature will be architected in a fully modular, extensible, and upgradable fashion. Users will be able to stake their tokens on broker nodes to support the network's decentralized GPU infrastructure and efficient scheduling of AI workload to GPUs. By staking tokens, users essentially vouch for the reliability and performance of the broker nodes they support. This process helps to ensure that only trustworthy and well-performing broker nodes are properly rewarded to actively participating in the network's operations.

### Early Access and General Access

Early access will be provided to YOT token holders who meet at least one predefined criterion on the updated Early Access Eligibility List (TBD). These holders will have the opportunity to stake YOT up to the per-wallet maximum. After Early Access ends, the staking pool will open to General Access, at which point anyone will have the chance to stake up to the per-wallet maximum, provided that the pool has not yet been filled.

## Reward Distribution

### Allocation and Delegation

The token pool will be capped at a certain size, which is to be determined. The pool will be allocated to community members and broker node operators. A portion of the rewards earned by Community Stakers will be made available to Broker Node Stakers as `Delegation Rewards` in exchange for providing services on behalf of Community Stakers.

### Staking Rewards

The base floor reward rate will be 5% per year in YOT tokens for helping secure the Yotta Network. This rate will be the same across community stakers and broker node stakers. However, due to the delegation reward from the Community Stakers to Broker Node Stakers, the effective base floor rewards rate will be 4.8% on an annualized basis. This base floor reward rate was determined by the Yotta DAO based on the need to balance the network’s economic sustainability over time so that stakers are fairly rewarded for their work securing the network.

Initially, the base floor reward will be guaranteed by tokens from the Yotta Ecosystem fund. With the ramp-up of the network's adoption, the sources of staking rewards will shift to the protocol fees, which will sustainably support the operation of the Yotta Network in a cryptoeconomic secure manner. This transition ensures that staking rewards are maintained through the network's growth and operational success, providing long-term incentives for both community stakers and broker node operators.

### Long-term Staking Incentives

Stakers can withdraw their staked tokens at any point. However, if they lock their tokens for a selected period, they will receive a higher reward rate. Once the period ends, all staked tokens will enter the unlock status, and stakers can withdraw at any time again. If a staker changes their mind and would like to unlock the tokens before the end of the previously selected period, they can request an unlock with a penalty of the staking reward. This will initiate a linear unlock of the staked tokens over a 7-day progressive cooldown period.

### Performance and Reliability

Broker nodes will be evaluated based on their performance and reliability. Broker nodes that consistently deliver high-quality service (e.g. scheduling, verification) and maintain robust performance will receive higher rewards, which are then distributed to the users who have staked their tokens on these nodes. On the contrary, when a broker node is not doing a great job, other broker nodes can raise a challenge. When a valid challenge is raised and a slashing condition is met, these Broker Node Stakers will see a portion of their staked YOT tokens slashed as a penalty for failing to meet performance requirements and those who raised the challenge will received the extra challenge reward. Over time, the conditions around raising challenges and slashing amounts are expected to evolve and determined by the Yotta DAO. This creates a feedback loop that encourages the maintenance of high standards within the network.

## Overview of Staking Parameters

The below table provides an overview of parameters in the staking of Yotta Protocol. We will update the parameters when more details are ready to be revealed.



|          Parameter          |  Value |
| :-------------------------: | :----: |
|       Total Pool Size       |   TBD  |
|     Community Allocation    |   85%  |
|    Broker Node Allocation   |   15%  |
|      Per-wallet Maximum     |   TBD  |
|      Per-wallet Minimum     |   TBD  |
|        Slash Penalty        |   TBD  |
|       Challenge Reward      |   TBD  |
|     Maximum Lock Period     |   TBD  |
| Progressive Cooldown Period | 7 days |
|       Base Reward Rate      |   5%   |
|       Delegation Rate       |   4%   |

