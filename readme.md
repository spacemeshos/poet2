# POET2 - Incremental proofs of elapsed time MVP

## Overview
This repository provides the specs for an MVP implementing the main construction defined in section 4 of this paper [Incremental proofs sequential work](https://eprint.iacr.org/2019/650]). For brevity, we call this construction POET2 and refer to this paper in this readme as `POET2 paper`.
POET2 is based on the basic construction defined in the paper [simple proofs of sequential work](https://eprint.iacr.org/2018/183). We refer to it in this doc as the 'POET PAPER'.
Please read both papers first to get familiar with the protocol. As a reference, please also see this [reference go implementation](https://github.com/spacemeshos/poet) of a non-incremental POET construction.


## Constants
- w:int. A statistical security parameter. Shared between prover and verifiers. For the MVP we set it to 256
- H - A cryptographic hash function. For the MVP, we set it to sha256

The constants are fixed and shared between the Prover and the Verifier. Values shown here are just an example and may be set differently for different POET server instances.

## Parameters
- n: An unsigned int time parameter
- N: Number of iterations. N := 2^(n+1) - 1 for some unsigned integer n > 0

### Protocol Participants
The following entities execute POET2 by sending messages between them:
- Prover: The service providing incremental proofs for verifiers
- Verifier: A client using the prover by providing the input statement x, and verifying the prover provided proofs

## Definitions
- DAG(n): The core direct acyclic graph data structure used by the verifier. Referred in the paper as CP(n). The depth of DAG(n) is n, and the total number of nodes in Dag(n) is N where N=2^(n+1). Dag(n) has 2^n leaves.
- x: {0,1}^w = rnd_in[0, 2^w - 1) - verifier provided input statement (commitment)
- Hx: (0, 1)^{<= w(n+1)} => (0, 1)^w . Hx is constructed in the following way: Hx(i) = H(x,i) where H() is a cryptographic hash function. The implementation should use a macro or inline function for H(), and should support a command-line switch that allows it to run with either H()=sha3() or H=sha256().

todo: define the proofs here from the paper

## MVP main use case
The following steps describe basic incremental POET2 protocol execution between a prover and a verifier as defined in section 4 of the POET2 paper. The happy path for the use case is for the verifier to complete the protocol. e.g. not abort it in any step.

1. Prover and verifiers agree on an initial constants n, H, and  w
2. Verifier generates a random commitment x (w bits long) and sends it to the prover
3. Prover computes proof `p(x,n)` by executing `Prove(x,n)`
4. Prover sends `p(x,n)` to the Verifier
5. Prover increments the proof by executing `Inc(p, n')` where `n' = n + 1` to generate `p1(x,n')`
6. Verified verifies the `p(x,n)` by executing `Verify(x,n)` and aborts if verification fails
7. Prover sends the proof `p1(x,n')` to the verifier
8. Verifier verifies the proof p1 by executing `Verify(x,n')` and aborts the protocol if verification fails

### Theoretical background and context
- [1] https://eprint.iacr.org/2019/650.pdf
- [2] https://eprint.iacr.org/2011/553.pdf
- [3] https://eprint.iacr.org/2018/183.pdf
- [4] https://spacemesh.io/whitepaper1/
- [5] https://pdfs.semanticscholar.org/b904/6d002da153a6fe9b06d469da4efffdfcb9c6.pdf

### Related work
- [5] https://github.com/spcemeshos/poet
- [6] https://github.com/avive/slow-time-functions

### Implementation Guidelines

##### DAG Definition
- We start with B(n) - `the complete binary tree of depth n` where all edges go from leaves up the tree to the root. B(n) has 2^n leaves and 2^n -1 internal nodes. We add edges to the n leaves in the following way:
    - For each leaf i of the 2^n leaves, we add an edge to the leaf from all the direct siblings of the nodes on the path from the leaf to the root node
    - In other words, for every leaf u, we add an edge to u from node v_{b-1}, iff v_{b} is an ancestor of u and nodes v_{b-1}, v_{b} are direct siblings

- Each node in the DAG is identified by a binary string in the form `0`, `01`, `0010` based on its location in Bn. This is the node id
- The root node at height 0 is identified by the empty string ""
- The nodes at height 1 (l0 and l1) are labeled `0` and `1`. The nodes at height 2 are labeled `00`, `01`, `10` and `11`, etc... So for each height h, node's id is an h bits binary number that uniquely defines the location of the node in the DAG
- We say node u is a parent of node v if there's a direct edge from u to v in the DAG (based on its construction)
- Each node has a label. The label li of node i (the node with id i) is defined as:

```
li = Hx(i,lp1,...,lpd)` where `(p1,...,pd) = parents(i)
```

For example, the root node's label is `lε = Hx(bytes(""), l0, l1)` as it has 2 only parents l0 and l1 and its id is the empty string "".

##### DAG Construction (See section 4, Lemma 3)

The following is a possible algorithm that satisfies these requirements. However, any implementation that satisfies them (with equivalent or better computational complexity) is also acceptable.

Recursive computation of the labels of DAG(n):

1. Compute the labels of the left subtree (tree with root l0)
2. Keep the label of l0 in memory and discard all other computed labels from memory
3. Compute the labels of the right subtree (tree with root l1) - using l0
4. Once l1 is computed, discard all other computed labels from memory and keep l1
5. Compute the root label le = Hx("", l0, l1)

- When a label value is computed by the algorithm, store it in persistent storage
- Note that this works because only l0 is needed for computing labels in the tree rooted in l1. All of the additional edges to nodes in the tree rooted at l1 start at l0.


### Data Types

#### Commitment X
arbitrary length bytes. Verifier implementation should just use H(commitment) to create a commitment that is in range (0, 1)^w . So the actually commitment can be sha256(commitment) when w=256.

#### Proof (See section 5.2)
A proof needs includes the following data:
1. φ - the label of the root node.
2. For each identifier i in a challenge (0 <= i < t), an ordered list of labels which includes:
   2.1 li - The label of the node i
   2.2 An ordered list of the labels of the sibling node of each node on the path to the parent node, omitting siblings that were already included in previous siblings list int he proof

So, for example for DAG(4) and for a challenge identifier `0101` - The labels that should be included in the list are: 0101, 0100, 011, 00 and 1. This is basically an opening of a Merkle tree commitment.

The complete proof data can be encoded in a tuple where the first value is φ and the second value is a list with t entries. Each of the t entries is a list starting with the node with identifier_t label, and a node for each sibling on the path to the root from node identifier_t:

{ φ, {list_of_siblings_on_path_to_root_from_0}, .... {list_of_siblings_on_path_to_root_from_t} }

Note that we don't need to include identifier_t in the proof as the identifiers needs to be computed by the verifier.

Also note that the proof should omit from the siblings list labels that were already included previously once in the proof. The verifier should create a dictionary of label values keyed by their node id, populate it from the siblings list it receives, and use it to get label values omitted from siblings lists - as the verifier knows the ids of all siblings on the path to the root from a given node.

### About NIPs

a NIP is a proof created by computing openH for the challenge γ := (Hx(φ,1),...Hx(φ,t)). e.g. without receiving γ from the verifier. Verifier asks for the NIP and verifies it like any other openH using verifyH. Note that the prover generates a NIP using only Hx(), t (shared security param) and φ (generated by PoSW(n)). To verify a NIP, a verifier generates the same challenge γ and verifies the proof using this challenge.

Hx(φ,i) output is `w bits` but each challenge identifier should be `n bits` long. To create an identifier from Hx(φ,i) we take `the leftmost t bits` - starting from the most significant bit. So, for example, for n == 3 and for Hx(φ,1) = 010011001101..., the identifier will be `010`.

### GetProof(challenge: Challenge) notes
The verifier only stores up to m layers of the DAG (from the root at height 0, up to height m) and the labels of the n leaves.
Generating a proof involves computing the labels of the siblings on the path from a leaf to the DAG root where some of these siblings are in DAG layers with height > m. These labels are not stored by the prover and need to be computed when a proof is generated. The following algorithm describes how to compute these siblings:

1. For each node_id included in the input challenge, compute the node id of the node n. The node on the path from node_id to the root at DAG level m

2. Construct the DAG rooted at node n. When the label of a sibling on the path from node_id to the root is computed as part of the DAG construction, add it to the list of sibling labels on the path from node_id to the root
