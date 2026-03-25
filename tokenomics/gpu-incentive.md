# GPU Incentive

## Introduction

The purpose of this incentive plan is to reward GPU providers for their contributions to the network by distributing tokens from a weekly token pool. The size of the token pool is dynamically adjusted based on projected demand to encourage an adequate supply of GPUs. Rewards are distributed based on several factors including the compute capability, availability, GPU type, and networking conditions such as bandwidth and latency.

## Weekly Token Pool

Each week, a total token pool will be determined and allocated for all GPU providers participated in the network in the next week. The size of this pool will be adjusted based on the projected demand of GPU usages to ensure there is sufficient GPU supply to meet network needs. This dynamic adjustment helps to balance supply and demand, providing incentives for new GPUs to join the network.

## Factors for Reward Distribution

The rewards will be distributed based on the following factors:

1. **Compute Capability**: GPUs with higher compute capabilities will have a higher weight. Compute capability is evaluated based on metrics such as FLOPS (Floating Point Operations Per Second).
2. **Availability**: GPUs that are available for more hours in a week will receive higher rewards. Availability is tracked to ensure that providers who offer consistent and reliable access to their GPUs are rewarded.
3. **GPU Type**: Different types of GPUs will be weighted based not only on their performance characteristics but also on their current demand and scarcity within the network. GPUs that are in higher demand or less abundant will receive higher weights to incentivize their availability.
4. **Networking Conditions**: Networking conditions such as bandwidth and latency will be factored in. GPUs with higher bandwidth and lower latency connections will receive higher rewards. This ensures that GPUs with better network performance contribute more effectively to the network's efficiency.

## Weight Calculation

Each GPU will be assigned a weight based on the following formula:

$$
Weight=(W_{Compute Capability} + W_{GPUType}+W_{AvgBandwidth}+W_{AvgLatency})×Available Hours
$$

Where:

* $$W_{ComputeCapability}$$ is the constant that can be adjust to fine-tune the reward distribution based on related to compute capability
* $$W_{GPU Type}$$ is a score assigned to the type of GPU based on its performance characteristics, economic contribution and scarcity in the network.
* $$W_{Avg Bandwidth}$$ and $$W_{Avg Latency}$$ are constants related to the networking conditions.
* $$Available Hours$$ is the number of hours the GPU was available during the week.

## Reward Distribution

Once the weights are calculated for each GPU, the rewards will be distributed proportionally based on these weights. The formula for the reward distribution is as follows:

$$
Reward_{i}=(\frac{Weight_{i}}{\sum{Weight_{i}}})×Total Token Pool
$$

Where:

* $$Reward_i$$ is the reward for $$GPU_i$$
* $$Weight_i$$ is the weight assigned to $$GPU_i$$.
* $$\sum{Weight_i}$$ is the sum of the weights of all GPUs in the pool.
* $$Total Token Pool$$ is the total number of tokens allocated for distribution that week.

## Token Pool Adjustment

The size of the token pool will be reviewed and adjusted weekly based on the following factors:

1. **Projected Demand**: Analysis of the demand for GPU resources will help determine if the token pool needs to be increased or decreased.
2. **Network Growth**: The number of new GPUs joining the network and the overall growth of the network will influence the size of the token pool.
3. **Market Conditions**: Market conditions and token value will be considered to ensure that rewards remain attractive to GPU providers.

## Conclusion

This token incentive plan is designed to ensure a balanced and fair distribution of rewards to GPU providers based on their contributions to the network. By dynamically adjusting the token pool and considering various performance factors, we aim to maintain a robust and scalable supply of GPU resources to meet the network's demands.
