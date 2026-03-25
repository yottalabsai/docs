# Parallelism Strategies

## Pipeline Parallelism

Model layers are divided into sequential “stages,” each assigned to a different device. Data flows through the stages like an assembly line, with micro‑batches interleaving forward and backward passes to keep GPUs busy. Diagram 2 illustrates this: multiple GPUs form pipeline stages, with staggered execution across micro‑batches to overlap computation and communication

## Data parallelism

Every device holds a complete model replica and processes a distinct portion of the input batch in parallel. After the backward pass, gradients are synchronized across devices—usually via an all‑reduce—to ensure model consistency

## Tensor Parallelism

Large tensors (e.g., weight matrices) within a layer are split and distributed across devices. Each device computes part of the operation—like matrix multiplication—and then uses collectives such as all‑gather or reduce‑scatter to assemble outputs. This is well illustrated in diagrams 1 and 3

## Sequence Parallelism

An extension of tensor parallelism, sequence parallelism shards along the sequence dimension (e.g., time steps) across devices. It’s especially useful for processing long input sequences, splitting activations across devices for operations that aren’t tensor‑friendly (like LayerNorm and non‑linearities)

## Expert Parallelism

In Mixture-of-Experts (MoE) models, only a subset of expert sub-networks is activated per token. Expert parallelism distributes these experts across devices—each GPU holds a subset of experts and processes tokens routed to them, using all‑to‑all communication to handle routing

