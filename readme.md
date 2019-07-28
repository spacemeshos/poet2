# POET2 - Incremental proofs of elapsed time MVP

## Overview
This repository provides the specs for an MVP implementing the main single-threaded construction defined in section 4 of this paper [Incremental proofs sequential work](https://eprint.iacr.org/2019/650]). For brevity, we call this construction `POET2` and refer to this paper in this readme as `POET2 paper`.
POET2 is based on the basic construction defined in the paper [simple proofs of sequential work](https://eprint.iacr.org/2018/183). We refer to it in this doc as the `POET PAPER`.
Please read both papers first to get familiar with the protocol. As a reference, please also see this [reference go implementation](https://github.com/spacemeshos/poet) of a non-incremental POET construction.


## Constants
- w:uint. A statistical security parameter. Shared between prover and verifiers. For the MVP we set it to 256. Note that this is denoted as λ in the POET2 paper
- H: A cryptographic hash function. For the MVP we set it to sha256
- t:uint a statistical security param which is a power of 2. For the MVP we set to 32.

The constants are fixed and shared between the Prover and the Verifier. Values shown here are just an example and may be set differently for different POET server instances.

## Parameters
- n: An unsigned int time parameter
- N: Number of iterations. N := 2^(n+1) - 1 for some unsigned integer n > 0

### Protocol Participants
The following entities execute POET2 by sending messages between them:
- Prover: The service providing incremental proofs for verifiers
- Verifier: A client using the prover by providing the input statement x, and verifying the prover provided proofs

## Definitions
- DAG(n): The core direct acyclic graph data structure used by the verifier. Referred in the paper as CP(n). The depth of DAG(n) is n, and the total number of nodes in Dag(n) is N where N=2^(n+1). Dag(n) has 2^n leaves
- x: {0,1}^w = rnd_in[0, 2^w - 1) - verifier provided input statement (commitment)
- Hx: (0, 1)^{<= w(n+1)} => (0, 1)^w . Hx is constructed in the following way: Hx(i) = H(x,i) where H() is sha256()

## MVP main use case
The following steps describe basic incremental POET2 protocol execution between a prover and a verifier as defined in section 4 of the POET2 paper. The happy path for the use case is for the verifier to complete the protocol. e.g. not abort it in any step. The MVP should implement this use case.

Prover and verifiers agree on an initial constants w=256, t= 32, and H = sha256()

1. Verifier generates a random commitment x (w bits long) and sends it to the prover
2. Prover computes proof `p(x, n)` by executing `Prove(x, n)`
3. Prover sends `p(x, n)` to the Verifier
4. Prover increments the proof by executing `Inc(x, p, n, n')` where `n' = n + 1` to generate `p1(x, n')`
5. Verifier verifies the `p(x, n)` by executing `Verify(x, n)` and aborts if verification fails
6. Prover sends the proof `p1(x, n')` to the verifier
7. Verifier verifies the proof p1 by executing `Verify(x, n')` and aborts the protocol if verification fails

## MVP Submission Guidelines
- Implement a prover and verifier where the prover implements Prove(x, n), Inc(x, p, n, n') and the verifier implements Verify(x, n).
- Use the DAG construction and the schemes described in section 4 of the POET2 paper (Main Construction)
- Implement a simple API between the prover and verifier to execute the main uses case between them
- Write a test of the main use case running with n = 24 and n' = 25 and verify that your test passes

## Implementation Tips for Bonus Points
- Optimize proof size by only including a leaf value once in a proof
- Don't use more than w(n+1) bits of memory in Prove(n)
- Implement the most efficient verifier in terms of computation complexity. See section 4.3 of the POET2 paper

### Theoretical background and context
- [1] https://eprint.iacr.org/2019/650.pdf
- [2] https://eprint.iacr.org/2011/553.pdf
- [3] https://eprint.iacr.org/2018/183.pdf
- [4] https://spacemesh.io/whitepaper1/
- [5] https://pdfs.semanticscholar.org/b904/6d002da153a6fe9b06d469da4efffdfcb9c6.pdf

### Related work
- [6] https://github.com/spcemeshos/poet
- [7] https://github.com/avive/slow-time-functions

---

### Tips & Tricks

#### DAG(n) Construction (See section 4, Lemma 3 in the POET paper)

The following is a possible algorithm that satisfies these requirements. However, any implementation that satisfies them (with equivalent or better computational complexity) is also acceptable.

Recursive computation of the labels of DAG(n):

1. Compute the labels of the left subtree (tree with root l0)
2. Keep the label of l0 in memory and discard all other computed labels from memory
3. Compute the labels of the right subtree (tree with root l1) - using l0
4. Once l1 is computed, discard all other computed labels from memory and keep l1
5. Compute the root label le = Hx("", l0, l1)

- When a label value is computed by the algorithm, store it in persistent storage
- Note that this works because only l0 is needed for computing labels in the tree rooted in l1. All of the additional edges to nodes in the tree rooted at l1 start at l0.
