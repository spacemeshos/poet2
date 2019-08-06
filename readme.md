# POET2 - Incremental proofs of elapsed time MVP

## Introduction

A proof of elapsed time (POET, for short) is a technology that allows one party, a _prover_, to assert to another independent party, a _verifier_, that a minimum amount of time has elapsed since the prover generated and sent a random commitment to the verifier. The technology is used in the [Spacemesh](https://spacemesh.io) blockchain protocol to avoid the issues associated with [costless simulation, weak subjectivity, and long-range attacks](https://blog.positive.com/rewriting-history-a-brief-introduction-to-long-range-attacks-54e473acdba9) that proof of stake and a more naive proof of space protocol are inherently subject to. See this [whiteboard series video with Tal Moran](https://youtu.be/liNmlxrwrvI) for an overview of the Spacemesh protocol and how POET fits in.

This task is to implement an MVP of an incremental POET protocol in Rust.

## Overview
This repository provides the specs for an MVP (Minimal Viable Product) implementing the main single-threaded construction defined in section 4 of this paper [Incremental proofs sequential work](https://eprint.iacr.org/2019/650). For brevity, we call this construction `POET2` and refer to this paper in this readme as `POET2 paper`. POET2 is based on the basic construction defined in the paper [simple proofs of sequential work](https://eprint.iacr.org/2018/183). We refer to it in this document as the `POET PAPER`. Please skim both papers first to get familiar with the protocol. As a reference, please also see this [reference go implementation](https://github.com/spacemeshos/poet) of a non-incremental POET construction.

## Constants
- `w:uint`: A statistical security parameter. Shared between prover and verifiers. For the MVP we set it to 256. Note that this is denoted as λ in the POET2 paper.
- `H`: A cryptographic hash function. For the MVP we set it to sha256.
- `t:uint` a statistical security parameter which is a power of 2. For the MVP we set to 32.

The constants are fixed and shared between the Prover and the Verifier. Values shown here are just an example and may be set differently for different POET server instances.

## Parameters
- `n:` An unsigned positive integer time parameter
- `N:` Number of iterations. N := 2^(n+1) - 1 for some unsigned positive integer n

### Protocol Participants
The following entities execute POET2 by sending messages between them:
- Prover: The service providing incremental proofs for verifiers
- Verifier: A client using the prover by providing the input statement x, and verifying the prover provided proofs

## Definitions
- DAG(n): The core direct acyclic graph data structure used by the verifier. Referred to in the POET2 paper as CP(n). The depth of DAG(n) is n, and the total number of nodes in DAG(n) is N where N=2^(n+1). DAG(n) has 2^n leaves
- x: Verifier provided input statement (challenge). A w-bits long binary string
- Hx(): A hash function constructed in the following way: Hx(i) = H(x,i) where H() is sha256() and x is a challenge

## MVP main use case
The following steps describe a basic incremental POET2 protocol execution between a prover and a verifier as defined in section 4 of the POET2 paper. The happy-path for the use case is for the verifier to complete the protocol, e.g., not abort it in any step. The MVP should correctly implement this use case.

The prover and the verifier agree on initial constants w=256, t=32, and H=sha256().
They also need to agree on the binary encoding and decoding of the data exchanged between them.

1. Verifier generates a random challenge x (w-bits long) and sends it together with n to the prover
2. Prover computes a proof `p(x, n)` by executing `Prove(x, n)`
3. Prover sends `p(x, n)` to the Verifier
4. Prover increments the proof by executing `Inc(x, p, n, n')` where `n' = n + 1` to compute and generate `p1(x, n')`
5. Verifier verifies the `p(x, n)` by executing `Verify(x, p, n)` and aborts if verification fails
6. Prover sends the proof `p1(x, n')` to the verifier
7. Verifier verifies the proof p1 by executing `Verify(x, p1, n')` and aborts the protocol if verification fails

## MVP Requirements & Guidelines
- Implement a Prover and a Verifier where the Prover implements Prove(x, n), Inc(x, p, n, n') and the Verifier implements Verify(x, n).
- Use the DAG construction and the schemes described in section 4 of the POET2 paper (Main Construction)
- Implement a simple API between the Prover and the Verifier to execute the main uses case between them
- Write a test of the main use case running with n = 33 and n' = 34 and verify that your test pass
- Prover time complexity should be bounded by O(N) sequential calls to H()
- Clearly document all of your implementation modules, types, traits, functions and methods
- Use Cargo for module management and the latest stable release of Rust
- Your solution should work on a standard x86-64 Linux system with 18GB of RAM. Our reference hardware for grading performance is an AWS m5a.xlarge instance running Amazon Linux, an AMD EPYC CPU at 2.3 GHZ base frequency (with boosting to up to 2.6 GHZ), 16 GB of RAM and a 2.5TB SSD system volume.

## Implementation Tips for Bonus Points
- Optimize proof size by only including a leaf value once in a proof
- Don't use more than w(n+1) bits of memory in Prove(n)
- Verifier time complexity should be bounded by O(log(t\*n))
- See section 4.3 of the POET2 paper for more details

### Theoretical background, context and related work
- [1] https://eprint.iacr.org/2019/650.pdf
- [2] https://eprint.iacr.org/2011/553.pdf
- [3] https://eprint.iacr.org/2018/183.pdf
- [4] https://spacemesh.io/whitepaper1/
- [5] https://pdfs.semanticscholar.org/b904/6d002da153a6fe9b06d469da4efffdfcb9c6.pdf
- [6] https://github.com/spacemeshos/poet
