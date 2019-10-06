% date: 'October 4, 2019'
% title: Polkadot Runtime Environment Protocol Specification
% based_on_commit (master): 6d9d4b542e7a8bd8ab55fd2d95db1daf5745b928

# Background
## Introduction

Formally, Polkadot is a replicated sharded state machine designed to
resolve the scalability and interoperability among blockchains. In
Polkadot vocabulary, shards are called *parachains* and Polkadot *relay
chain* is part of the protocol ensuring global consensus among all the
parachains. The Polkadot relay chain protocol, henceforward called
*Polkadot protocol*, can itself be considered as a replicated state
machine on its own. As such, the protocol can be specified by
identifying the state machine and the replication strategy.

From a more technical point of view, the Polkadot protocol has been
divided into two parts, the *Runtime* and the *Runtime environment*
(RE). The Runtime comprises most of the state transition logic for the
Polkadot protocol and is designed and expected to be upgradable as part
of the state transition process. The Runtime environment consists of
parts of the protocol,shared mostly among peer-to-peer decentralized
cryptographically-secured transaction systems, i.e. blockchains whose
consensus system is based on the proof-of-stake. The RE is planned to be
stable and static for the lifetime duration of the Polkadot protocol.

With the current document, we aim to specify the RE part of the Polkadot
protocol as a replicated state machine. After defining the basic terms
in Chapter 1, we proceed to specify the representation of a valid state
of the Protocol in Chapter [2](#chap-state-spec){reference-type="ref"
reference="chap-state-spec"}. In Chapter
[3](#chap-state-transit){reference-type="ref"
reference="chap-state-transit"}, we identify the protocol states, by
explain the Polkadot state transition and discussing the detail based on
which Polkadot RE interacts with the state transition function, i.e.
Runtime. Following, we specify the input messages triggering the state
transition and the system behaviour. In Chapter
[5](#chap-consensu){reference-type="ref" reference="chap-consensu"}, we
specify the consensus protocol, which is responsible for keeping all the
replica in the same state. Finally, the initial state of the machine is
identified and discussed in Appendix
[8](#sect-genisis-block){reference-type="ref"
reference="sect-genisis-block"}. A Polkadot RE implementation which
conforms with this part of the specification should successfully be able
to sync its states with the Polkadot network.

## Definitions and Conventions {#sect-defn-conv}

[\[defn-state-machine\]]{#defn-state-machine
label="defn-state-machine"}A **Discrete State Machine (DSM)** is a state
transition system whose set of states and set of transitions are
countable and admits a starting state. Formally, it is a tuple of
$$(\Sigma, S, s_0, \delta)$$ where

-   $\Sigma$ is the countable set of all possible transactions.

-   $S$ is a countable set of all possible states.

-   $s_0 \in S$ is the initial state.

-   $\delta$ is the state-transition function, known as
    [\[defn-runtime\]]{#defn-runtime label="defn-runtime"}**Runtime** in
    the Polkadot vocabulary, such that
    $$\delta : S \times \Sigma \rightarrow S$$

[\[defn-path-graph\]]{#defn-path-graph label="defn-path-graph"}A **path
graph** or a **path** of *n* nodes formally referred to as $P_n$, is
a tree with two nodes of vertex degree 1 and the other n-2 nodes of
vertex degree 2. Therefore, $P_n$ can be represented by sequences of
$(v_1, \ldots, v_n)$ where $e_i = (v_i, v_{i + 1})$ for
$1 \leqslant i \leqslant n - 1$ is the edge which connect $v_i$ and
$v_{i + 1}$.

[\[defn-radix-tree\]]{#defn-radix-tree label="defn-radix-tree"}**Radix-r
tree** is a variant of  a trie in which:

-   Every node has at most $r$ children where $r = 2^x$ for some $x$;

-   Each node that is the only child of a parent, which does not
    represent a valid key is merged with its parent.

As a result, in a radix tree, any path whose interior vertices all have
only one child and does not represent a valid key in the data set, is
compressed into a single edge. This improves space efficiency when the
key space is sparse.

By a **sequences of bytes** or a **byte array**, $b$, of length $n$, we
refer to
$$b :=(b_0, b_1, ..., b_{n - 1})  \text{such that } 0 \leqslant b_i
     \leqslant 255$$ We define $\mathbb{B}_n$ to be the **set of all
byte arrays of length $n$**. Furthermore, we define:
$$\mathbb{B} :=\bigcup^{\infty}_{i = 0} \mathbb{B}_i$$

We represent the concatenation of byte arrays $a :=(a_0, \ldots, a_n)$
and $b :=(b_0, \ldots, b_m)$ by:
$$a || b :=(a_0, \ldots, a_n, b_0, \ldots, b_m)$$

[\[defn-bit-rep\]]{#defn-bit-rep label="defn-bit-rep"}For a given byte
$b$ the **bitwise representation** of $b$ is defined as
$$b :=b^7 \ldots b^0$$ where
$$b = 2^0 b^0 + 2^1 b^1 + \cdots + 2^7 b^7$$

[\[defn-little-endian\]]{#defn-little-endian
label="defn-little-endian"}By the **little-endian** representation of a
non-negative integer, I, represented as $$I = (B_n \ldots B_0)_{256}$$
in base 256, we refer to a byte array $B = (b_0, b_1, \ldots, b_n)$ such
that $$b_i :=B_i$$ Accordingly, define the function
$\operatorname{Enc}_{\operatorname{LE}}$:
$$\begin{array}{llll}
       \operatorname{Enc}_{\operatorname{LE}} : & \mathbb{Z}^+ & \rightarrow & \mathbb{B}\\
       & (B_n \ldots B_0)_{256} & \mapsto & (B_{0,} B_1, \ldots_{}, B_n)
     \end{array}$$

By []{.smallcaps} we refer to a non-negative integer stored in a byte
array of length 4 using little-endian encoding format.

A **blockchain** $C$ is a directed path graph. Each node of the graph is
called **Block** and indicated by **$B$**. The unique sink of $C$ is
called **Genesis Block**, and the source is called the **Head** of C.
For any vertex $(B_1, B_2)$ where $B_1
  \rightarrow B_2$ we say $B_2$ is the **parent** of $B_1$ and we
indicate it by $$B_2 :=P (B_1)$$

[\[defn-unix-time\]]{#defn-unix-time label="defn-unix-time"}By **UNIX
time**, we refer to the unsigned, little-endian encoded 64-bit integer
which stores the number of **milliseconds** that have elapsed since the
Unix epoch, that is the time 00:00:00 UTC on 1 January 1970, minus leap
seconds. Leap seconds are ignored, and every day is treated as if it
contained exactly 86400 seconds.

### Block Tree

In the course of formation of a (distributed) blockchain, it is possible
that the chain forks into multiple subchains in various block positions.
We refer to this structure as a *block tree:*

[\[defn-block-tree\]]{#defn-block-tree label="defn-block-tree"}The
**block tree** of a blockchain, denoted by
$\operatorname{BT}$ is the union of all different versions
of the blockchain observed by all the nodes in the system such as every
such block is a node in the graph and $B_1$ is connected to $B_2$ if
$B_1$ is a parent of $B_2$.

When a block in the block tree gets finalized, there is an opportunity
to prune the block tree to free up resources into branches of blocks
that do not contain all of the finalized blocks or those that can never
be finalized in the blockchain. For a definition of finality, see
Section [5.2](#sect-finality){reference-type="ref"
reference="sect-finality"}.

[\[defn-pruned-tree\]]{#defn-pruned-tree label="defn-pruned-tree"}By
**Pruned Block Tree**, denoted by $\operatorname{PBT}$, we
refer to a subtree of the block tree obtained by eliminating all
branches which do not contain the most recent finalized blocks, as
defined in Definition
[\[defn-finalized-block\]](#defn-finalized-block){reference-type="ref"
reference="defn-finalized-block"}. By **pruning**, we refer to the
procedure of $\operatorname{BT} \leftarrow
  \operatorname{PBT}$. When there is no risk of ambiguity
and is safe to prune BT, we use $\operatorname{BT}$ to
refer to $\operatorname{PBT}$.

Definition
[\[defn-chain-subchain\]](#defn-chain-subchain){reference-type="ref"
reference="defn-chain-subchain"} gives the means to highlight various
branches of the block tree.

[\[defn-chain-subchain\]]{#defn-chain-subchain
label="defn-chain-subchain"}Let $G$ be the root of the block tree and
$B$ be one of its nodes. By [**Chain($B$)**,]{.smallcaps} we refer to
the path graph from $G$ to $B$ in (P)$\operatorname{BT}$.
Conversely, for a chain $C$=[Chain(B)]{.smallcaps}, we define **the head
of $C$** to be $B$, formally noted as $B :=$[Head($C$)]{.smallcaps}. We
define $| C |$, the length of $C$as a path graph. If $B'$ is another
node on [Chain($B$)]{.smallcaps}, then by
[SubChain($B', B$)]{.smallcaps} we refer to the subgraph of
[Chain($B$)]{.smallcaps} path graph which contains both $B$ and $B'$.
Accordingly, $\mathbb{C}_{B'} ((P) \operatorname{BT})$ is
the set of all subchains of $(P) \operatorname{BT}$ rooted
at $B'$. The set of all chains of $(P)
  \operatorname{BT}$,
$\mathbb{C}_G ((P) \operatorname{BT})$ is denoted by
$\mathbb{C}$((P)BT) or simply $\mathbb{C}$, for the sake of brevity.

[\[defn-longest-chain\]]{#defn-longest-chain
label="defn-longest-chain"}We define the following complete order over
$\mathbb{C}$ such that for $C_1, C_2 \in \mathbb{C}$ if
$| C_1 | \neq | C_2
  |$ we say $C_1 > C_2$ if and only if $| C_1 | > | C_2 |$.

If $| C_1 | = | C_2 |$ we say $C_1 > C_2$ if and only if the block
arrival time of $\operatorname{Head} (C_1)_{}$ is less than
the block arrival time of $\operatorname{Head} (C_2)$ as
defined in Definition
[\[defn-block-time\]](#defn-block-time){reference-type="ref"
reference="defn-block-time"}. We define the
**[Longest-Chain($\operatorname{BT}$)]{.smallcaps}** to be
the maximum chain given by this order.

[Longest-Path($\operatorname{BT}$)]{.smallcaps} returns the
path graph of $(P)
  \operatorname{BT}$ which is the longest among all paths
in $(P) \operatorname{BT}$ and has the earliest block
arrival time as defined in Definition
[\[defn-block-time\]](#defn-block-time){reference-type="ref"
reference="defn-block-time"}.
[Deepest-Leaf($\operatorname{BT}$)]{.smallcaps} returns the
head of [Longest-Path($\operatorname{BT}$)]{.smallcaps}
chain.

Because every block in the blockchain contains a reference to its
parent, it is easy to see that the block tree is de facto a tree. A
block tree naturally imposes partial order relationships on the blocks
as follows:

We say **B is descendant of $B'$**, formally noted as **$B
  > B'$** if $B$ is a descendant of $B'$ in the block tree.

State Specification {#chap-state-spec}
===================

State Storage and Storage Trie
------------------------------

For storing the state of the system, Polkadot RE implements a hash table
storage where the keys are used to access each data entry. There is no
assumption either on the size of the key nor on the size of the data
stored under them, besides the fact that they are byte arrays with
specific upper limits on their length. The limit is imposed by the
encoding algorithms to store the key and the value in the storage trie.

### Accessing System Storage 

Polkadot RE implements various functions to facilitate access to the
system storage for the runtime. Section
[\[sect-runtime-api\]](#sect-runtime-api){reference-type="ref"
reference="sect-runtime-api"} lists all of those functions. Here we
formalize the access to the storage when it is being directly accessed
by Polkadot RE (in contrast to Polkadot runtime).

[\[defn-stored-value\]]{#defn-stored-value label="defn-stored-value"}The
**StoredValue** function retrieves the value stored under a specific key
in the state storage and is formally defined as : $$\begin{array}{cc}
       \operatorname{StoredValue} : & \mathcal{K} \rightarrow \mathcal{V}\\
       & k \mapsto \left\{ \begin{array}{cc}
         v & \text{if (k,v) exists in state storage}\\
         \phi & \operatorname{otherwise}
       \end{array} \right.
     \end{array}$$ where $\mathcal{K} \subset \mathbb{B}$ and
$\mathcal{V} \subset \mathbb{B}$ are respectively the set of all keys
and values stored in the state storage.

 

### The General Tree Structure

In order to ensure the integrity of the state of the system, the stored
data needs to be re-arranged and hashed in a *modified Merkle Patricia
Tree*, which hereafter we refer to as the ***Trie***. This rearrangment
is necessary to be able to compute the Merkle hash of the whole or part
of the state storage, consistently and efficiently at any given time.

The Trie is used to compute the *state root*, $H_r$, (see Definition
[\[defn-block-header\]](#defn-block-header){reference-type="ref"
reference="defn-block-header"}), whose purpose is to authenticate the
validity of the state database. Thus, Polkadot RE follows a rigorous
encoding algorithm to compute the values stored in the trie nodes to
ensure that the computed Merkle hash, $H_r$, matches across the Polkadot
RE implementations.

The Trie is a *radix-16* tree as defined in Definition
[\[defn-radix-tree\]](#defn-radix-tree){reference-type="ref"
reference="defn-radix-tree"}. Each key value identifies a unique node in
the tree. However, a node in a tree might or might not be associated
with a key in the storage.

When traversing the Trie to a specific node, its key can be
reconstructed by concatenating the subsequences of the key which are
stored either explicitly in the nodes on the path or implicitly in their
position as a child of their parent.

To identify the node corresponding to a key value, $k$, first we need to
encode $k$ in a consistent with the Trie structure way. Because each
node in the trie has at most 16 children, we represent the key as a
sequence of 4-bit nibbles:

For the purpose of labeling the branches of the Trie, the key $k$ is
encoded to $k_{\operatorname{enc}}$ using KeyEncode
functions:
$$k_{\operatorname{enc}} :=(k_{\operatorname{enc}_1}, \ldots, k_{\operatorname{enc}_{2 n}})
    :=\operatorname{KeyEncode} (k) key-encode-in-trie$$
such that:
$$\operatorname{KeyEncode} (k) : \left\{ \begin{array}{lll}
       \mathbb{B}^{} & \rightarrow & \operatorname{Nibbles}^4\\
       k :=(b_1, \ldots, b_n) :=& \mapsto & (b^1_1, b^2_1, b_2^1,
       b^2_2, \ldots, b^1_n, b^2_n)\\
       &  & :=(k_{\operatorname{enc}_1}, \ldots, k_{\operatorname{enc}_{2 n}})
     \end{array} \right.$$ where $\operatorname{Nibble}^4$
is the set of all nibbles of 4-bit arrays and $b^1_i$ and $b^2_i$ are
4-bit nibbles, which are the big endian representations of $b_i$:
$$(b^1_i, b^2_i) :=(b_i / 16, b_i \operatorname{mod} 16)$$
, where mod is the remainder and / is the integer division operators.

By looking at $k_{\operatorname{enc}}$ as a sequence of
nibbles, one can walk the radix tree to reach the node identifying the
storage value of $k$.

### Trie Structure

In this subsection, we specify the structure of the nodes in the Trie as
well as the Trie structure:

We refer to the **set of the nodes of Polkadot state trie** by
$\mathcal{N}.$ By $N \in \mathcal{N}$ to refer to an individual node in
the trie.

[\[defn-nodetype\]]{#defn-nodetype label="defn-nodetype"}The State Trie
is a radix-16 tree. Each Node in the Trie is identified with a unique
key $k_N$ such that:

-   $k_N$ is the shared prefix of the key of all the descendants of $N$
    in the Trie.

and, at least one of the following statements holds:

-   $(k_N, v)$ corresponds to an existing entry in the State Storage.

-   N has more than one child.

Conversely, if $(k, v)$ is an entry in the State Trie then there is a
node $N \in \mathcal{N}$ such that $k_N$=k.

A **branch** node is a node which has one child or more. A branch node
can have at most 16 children. A **leaf** node is a childless node.
Accordingly: $$\begin{array}{c}
       \mathcal{N}_b :=\left\{ N \in \mathcal{N}|N \text{is a branch
       node} \right\}\\
       \mathcal{N}_l :=\left\{ N \in \mathcal{N}|N \text{is a leaf node}
       \right\}
     \end{array}$$

For each Node, part of $k_N$ is built while the trie is traversed from
root to $N$ part of $k_N$ is stored in $N$ as formalized in Definition
[\[defn-node-key\]](#defn-node-key){reference-type="ref"
reference="defn-node-key"}.

[\[defn-node-key\]]{#defn-node-key label="defn-node-key"}For any
$N \in \mathcal{N}$, its key $k_N$ is divided into an **aggregated
prefix key**,
**$\operatorname{pk}_N^{\operatorname{Agr}}$**,
aggregated by Algorithm
[\[algo-aggregate-key\]](#algo-aggregate-key){reference-type="ref"
reference="algo-aggregate-key"} and a **partial key**,
**$\operatorname{pk}_N$** of length
$0 \leqslant l_{\operatorname{pk}_N} \leqslant
  65535$ in nibbles such that:
$$\operatorname{pk}_N :=(k_{\operatorname{enc}_i}, \ldots, k_{\operatorname{enc}_{i +
     l_{\operatorname{pk}_N}}})$$ where
$\operatorname{pk}_N$ is a suffix subsequence of $k_N$; $i$
is the length of
$\operatorname{pk}_N^{\operatorname{Agr}}$ in
nibbles and so we have:
$$\operatorname{KeyEncode} (k_N) = \operatorname{pk}_N^{\operatorname{Agr}} | | \operatorname{pk}_N =
     (k_{\operatorname{enc}_1}, \ldots, k_{\operatorname{enc}_{i - 1}}, k_{\operatorname{enc}_i},
     k_{\operatorname{enc}_{i + l_{\operatorname{pk}_N}}})$$

Part of
$\operatorname{pk}_N^{\operatorname{Agr}}$ is
explicitly stored in $N$'s ancestors. Additionally, for each ancestor, a
single nibble is implicitly derived while traversing from the ancestor
to its child included in the traversal path using the
$\operatorname{Index}_N$ function defined in Definition
[\[defn-index-function\]](#defn-index-function){reference-type="ref"
reference="defn-index-function"}.

[\[defn-index-function\]]{#defn-index-function
label="defn-index-function"}For $N \in \mathcal{N}_b$ and $N_c$ child of
N, we define **$\operatorname{Index}_N$** function as:
$$\begin{array}{cc}
       \operatorname{Index}_N : & \left\{ N_c \in \mathcal{N}|N_c  \text{is a child of
       N} \right\} \rightarrow \operatorname{Nibbles}^4_1\\
       & N_c \mapsto i_{}
     \end{array}$$ such that
$$k_{N_c} = k_N | | i | | \operatorname{pk}_{N_c}$$

Assuming that $P_N$ is the path (see Definition
[\[defn-path-graph\]](#defn-path-graph){reference-type="ref"
reference="defn-path-graph"}) from the Trie root to node $N$, Algorithm
[\[algo-aggregate-key\]](#algo-aggregate-key){reference-type="ref"
reference="algo-aggregate-key"} rigorously demonstrates how to build
$\operatorname{pk}^{\operatorname{Agr}}_N$
while traversing $P_N$.

**Algorithm**

[\[algo-aggregate-key\]]{#algo-aggregate-key
label="algo-aggregate-key"}[Aggregate-Key]{.smallcaps}$(P_N : =
      (\operatorname{TrieRoot} = N_1, \ldots, N_j = N))$

[\[defn-node-value\]]{#defn-node-value label="defn-node-value"}A node
$N \in \mathcal{N}$ stores the **node value**, **$v_N$**, which consists
of the following concatenated data: $$\begin{array}{|l|l|l|}
       \hline
       \operatorname{Node} \operatorname{Header} & \operatorname{Partial} \operatorname{key} & \operatorname{Node}
       \operatorname{Subvalue}\\
       \hline
     \end{array}$$ Formally noted as:
$$v_N :=\operatorname{Head}_N | | \operatorname{Enc}_{\operatorname{HE}} (\operatorname{pk}_N) | |
     \operatorname{sv}_N$$ where
$\operatorname{Head}_N$,
$\operatorname{pk}_N$,
$\operatorname{Enc}_{\operatorname{nibbles}}$
and $\operatorname{sv}_N$ are defined in Definitions
[\[defn-node-header\]](#defn-node-header){reference-type="ref"
reference="defn-node-header"},
[\[defn-node-key\]](#defn-node-key){reference-type="ref"
reference="defn-node-key"},
[\[defn-hex-encoding\]](#defn-hex-encoding){reference-type="ref"
reference="defn-hex-encoding"} and
[\[defn-node-subvalue\]](#defn-node-subvalue){reference-type="ref"
reference="defn-node-subvalue"}, respectively.

[\[defn-node-header\]]{#defn-node-header label="defn-node-header"}The
**node header** of node $N$, $\operatorname{Head}_N$,
consists of $l + 1 \geqslant 1$ bytes
$\operatorname{Head}_{N, 1},
  \ldots, \operatorname{Head}_{N, l + 1}$ such that:

$$\begin{array}{ll}
       \hline
       \operatorname{Node} \operatorname{Type} & \operatorname{pk} \operatorname{length}\\
       \hline
       \operatorname{Head}_{N, 1}^{6 - 7}_{} & \operatorname{Head}_{N, 1}^{0 - 5}_{}
     \end{array}  \begin{array}{|l|}
       \hline
       \operatorname{pk} \operatorname{length} \operatorname{extra} \operatorname{byte} 1\\
       \hline
       \operatorname{Head}_{N, 2}\\
       \hline
     \end{array}  \begin{array}{|l|}
       \hline
       \operatorname{pk} \operatorname{key} \operatorname{length} \operatorname{extra} \operatorname{byte} 2\\
       \hline
       \ldots .\\
       \hline
     \end{array} \ldots \begin{array}{|l|}
       \hline
       \operatorname{pk} \operatorname{length} \operatorname{extra} \operatorname{byte} l\\
       \hline
       \operatorname{Head}_{N, l + 1}^{}_{}\\
       \hline
     \end{array}$$

In which $\operatorname{Head}_{N, 1}^{6 - 7}_{}$, the two
most significant bits of the first byte of
$\operatorname{Head}_N$ are determined as follows:
$$\operatorname{Head}_{N, 1}^{6 - 7}_{} :=\left\{ \begin{array}{ll}
       00 & \operatorname{Special} \operatorname{case}\\
       01 & \operatorname{Leaf} \operatorname{Node}\\
       10 & \operatorname{Branch} \operatorname{Node} \operatorname{with} k_N \not\in\mathcal{K}\\
       11 & \operatorname{Branch} \operatorname{Node} \operatorname{with} k_N \in \mathcal{K}
     \end{array} \right.$$ where $\mathcal{K}$ is defined in Definition
[\[defn-stored-value\]](#defn-stored-value){reference-type="ref"
reference="defn-stored-value"}.

$\operatorname{Head}_{N, 1}^{0 - 5}_{}$, the 6 least
significant bits of the first byte of
$\operatorname{Head}_N$ are defined to be:
$$\operatorname{Head}_{N, 1}^{0 - 5}_{} :=\left\{ \begin{array}{ll}
       \| \operatorname{pk}_N \|_{\operatorname{nib}} & \| \operatorname{pk}_N \|_{\operatorname{nib}} < 63\\
       63 & \| \operatorname{pk}_N \|_{\operatorname{nib}} \geqslant 63
     \end{array} \right.$$ In which
**$\| \operatorname{pk}_N \|_{\operatorname{nib}}$**
is the length of $\operatorname{pk}_N$ in number nibbles.
$\operatorname{Head}_{N, 2}, \ldots,
  \operatorname{Head}_{N, l + 1}$ bytes are determined by
Algorithm [\[algo-pk-length\]](#algo-pk-length){reference-type="ref"
reference="algo-pk-length"}.

**Algorithm**

[\[algo-pk-length\]]{#algo-pk-length
label="algo-pk-length"}[Partial-Key-Length-Encoding$(\operatorname{Head}_{N,
      1}^{6 - 7}_{}, \operatorname{pk}_N)$]{.smallcaps}

### Merkle Proof {#sect-merkl-proof}

To prove the consistency of the state storage across the network and its
modifications both efficiently and effectively, the Trie implements a
Merkle tree structure. The hash value corresponding to each node needs
to be computed rigorously to make the inter-implementation data
integrity possible.

The Merkle value of each node should depend on the Merkle value of all
its children as well as on its corresponding data in the state storage.
This recursive dependancy is encompassed into the subvalue part of the
node value which recursively depends on the Merkle value of its
children.

We use the auxilary function introduced in Definition
[\[defn-children-bitmap\]](#defn-children-bitmap){reference-type="ref"
reference="defn-children-bitmap"} to encode and decode information
stored in a branch node.

[\[defn-children-bitmap\]]{#defn-children-bitmap
label="defn-children-bitmap"}Suppose $N_b, N_c \in \mathcal{N}$ and
$N_c$ is a child of $N_b$. We define where bit $b_i : = 1$ if $N$ has a
child with partial key $i$, therefore we define **ChildrenBitmap**
functions as follows: $$\begin{array}{cc}
       \operatorname{ChildrenBitmap} : & \mathcal{N}_b \rightarrow \mathbb{B}_2\\
       & N \mapsto (b_{15}, \ldots, b_8, b_7, \ldots b_0)_2
     \end{array}$$ where $$b_i :=\left\{ \begin{array}{cc}
       1 & \exists N_c \in \mathcal{N}: k_{N_c} = k_{N_b} | | i | |
       \operatorname{pk}_{N_c}\\
       0 & \text{otherwise}
     \end{array} \right.$$

[\[defn-node-subvalue\]]{#defn-node-subvalue
label="defn-node-subvalue"}For a given node $N$, the **subvalue** of
$N$, formally referred to as $\operatorname{sv}_N$, is
determined as follows: in a case which:

$$\begin{array}{l}
         \operatorname{sv}_N :=\\
         \left\{ \begin{array}{cc}
           \operatorname{Enc}_{\operatorname{SC}} (\operatorname{StoredValue} (k_N)) & \text{N is a
           leaf node}\\
           \operatorname{ChildrenBitmap} (N)\| \operatorname{Enc}_{\operatorname{SC}} (H
           (N_{C_1})) \ldots \operatorname{Enc}_{\operatorname{SC}} (H (N_{C_n})) | |
           \operatorname{Enc}_{\operatorname{SC}} (\operatorname{StoredValue} (k_N))  & \text{N is a
           branch node}
         \end{array} \right.
       \end{array}$$

Where $N_{C_1} \ldots N_{C_n}$ with $n \leqslant 16$ are the children
nodes of the branch node $N$ and Enc$_{\textrm{SC}}$,
$\operatorname{StoredValue}$, $H$, and
$\operatorname{ChildrenBitmap} (N)$ are defined in
Definitions [7.1](#sect-scale-codec){reference-type="ref"
reference="sect-scale-codec"},
[\[defn-stored-value\]](#defn-stored-value){reference-type="ref"
reference="defn-stored-value"},
[\[defn-merkle-value\]](#defn-merkle-value){reference-type="ref"
reference="defn-merkle-value"} and
[\[defn-children-bitmap\]](#defn-children-bitmap){reference-type="ref"
reference="defn-children-bitmap"} respectively.

 

The Trie deviates from a traditional Merkle tree where node value, $v_N$
(see Definition
[\[defn-node-value\]](#defn-node-value){reference-type="ref"
reference="defn-node-value"}) is presented instead of its hash if it
occupies less space than its hash.

[\[defn-merkle-value\]]{#defn-merkle-value label="defn-merkle-value"}For
a given node $N$, the **Merkle value** of $N$, denoted by $H (N)$ is
defined as follows: $$\begin{array}{ll}
       & H : \mathbb{B} \rightarrow \mathbb{B}_{32}\\
       & H (N) : \left\{ \begin{array}{lcl}
         v_N &  & \|v_N \|< 32\\
         \operatorname{Blake} 2 b (v_N) &  & \|v_N \| \geqslant 32
       \end{array} \right.
     \end{array}$$ Where $v_N$ is the node value of $N$ defined in
Definition [\[defn-node-value\]](#defn-node-value){reference-type="ref"
reference="defn-node-value"} and $0_{32 - \| v_N \|}$ an all zero byte
array of length $32 - | | v_N | |$. The **Merkle hash** of the Trie is
defined as: $$\operatorname{Blake} 2 b (H (R))$$ Where $R$
is the root of the Trie.

State Transition {#chap-state-transit}
================

Like any transaction-based transition system, Polkadot state changes via
an executing ordered set of instructions. These instructions are known
as *extrinsics*. In Polkadot, the execution logic of the
state-transition function is encapsulated in Runtime as defined in
Definition
[\[defn-state-machine\]](#defn-state-machine){reference-type="ref"
reference="defn-state-machine"}. Runtime is presented as a Wasm blob
in(if?) ordered be easily upgradable. Nonetheless, the Polkadot Runtime
Environment needs to be in constant interaction with Runtime. The detail
of such interaction is further described in Section
[3.1](#sect-entries-into-runtime){reference-type="ref"
reference="sect-entries-into-runtime"}.

In Section [3.2](#sect-extrinsics){reference-type="ref"
reference="sect-extrinsics"}, we specify the procedure of the process
where the extrinsics are submitted, pre-processed and validated by
Runtime and queued to be applied to the current state.

Polkadot, likewise most prominent distributed ledger systems that make
state replication feasible, journals and batches a series of extrinsics
together in a structure knows as a *block* before propagating to the
other nodes. The specification of the Polkadot block as well the process
of verifying its validity are both explained in Section
[3.3](#sect-state-replication){reference-type="ref"
reference="sect-state-replication"}.

Interactions with Runtime {#sect-entries-into-runtime}
-------------------------

Runtime as defined in Definition
[\[defn-runtime\]](#defn-runtime){reference-type="ref"
reference="defn-runtime"} is the code implementing the logic of the
chain. This code is decoupled from the Polkadot RE to make the Runtime
easily upgradable without the need to upgrade the Polkadot RE itself.
The general procedure to interact with Runtime is described in Algorithm
[\[algo-runtime-interaction\]](#algo-runtime-interaction){reference-type="ref"
reference="algo-runtime-interaction"}.

**Algorithm**

[\[algo-runtime-interaction\]]{#algo-runtime-interaction
label="algo-runtime-interaction"}[Interact-With-Runtime]{.smallcaps}($F$:
the runtime entry,

$H_b (B)$: Block hash indicating the state at the end of $B$,

$A_1, A_2, \ldots, A_n$: arguments to be passed to the runtime entry)

In this section, we describe the details upon which the Polkadot RE is
interacting with the Runtime. In particular,
[Storage-At-State]{.smallcaps} and [Call-Runtime-Entry]{.smallcaps}
procedures called in Algorithm
[\[algo-runtime-interaction\]](#algo-runtime-interaction){reference-type="ref"
reference="algo-runtime-interaction"} are explained in Notation
[\[nota-call-into-runtime\]](#nota-call-into-runtime){reference-type="ref"
reference="nota-call-into-runtime"} and Definition
[\[defn-storage-at-state\]](#defn-storage-at-state){reference-type="ref"
reference="defn-storage-at-state"} respectively. $R_B$ is the Runtime
code loaded from $\mathcal{S}_B$, as described in Notation
[\[nota-runtime-code-at-state\]](#nota-runtime-code-at-state){reference-type="ref"
reference="nota-runtime-code-at-state"}, and $\mathcal{R}\mathcal{E}_B$
is the Polkadot RE API, as described in Notation
[\[nota-re-api-at-state\]](#nota-re-api-at-state){reference-type="ref"
reference="nota-re-api-at-state"}.

### Loading the Runtime Code    {#sect-loading-runtime-code}

Polkadot RE expects to receive the code for the Runtime of the chain as
a compiled WebAssembly (Wasm) Blob. The current runtime is stored in the
state database under the key represented as a byte array:
$$b :=\text{3A,63,6F,64,65}$$ which is the byte array of ASCII
representation of string ":code" (see Section
[9](#sect-predef-storage-keys){reference-type="ref"
reference="sect-predef-storage-keys"}). For any call to the Runtime,
Polkadot RE makes sure that it has the Runtime corresponding to the
state in which the entry has been called. This is, in part, because the
calls to Runtime have potentially the ability to change the Runtime code
and hence Runtime code is state sensitive. Accordingly, we introduce the
following notation to refer to the Runtime code at a specific state:

[\[nota-runtime-code-at-state\]]{#nota-runtime-code-at-state
label="nota-runtime-code-at-state"}By $R_B$, we refer to the Runtime
code stored in the state storage whose state is set at the end of the
execution of block $B$.

The initial runtime code of the chain is embedded as an extrinsics into
the chain initialization JSON file and is submitted to Polkadot RE (see
Section [8](#sect-genisis-block){reference-type="ref"
reference="sect-genisis-block"}).

Subsequent calls to the runtime have the ability to call the storage API
(see Section
[\[sect-runtime-api\]](#sect-runtime-api){reference-type="ref"
reference="sect-runtime-api"}) to insert a new Wasm blob into runtime
storage slot to upgrade the runtime.

### Code Executor

Polkadot RE provides a Wasm Virtual Machine (VM) to run the Runtime. The
Wasm VM exposes the Polkadot RE API to the Runtime, which, on its turn,
executes a call to the Runtime entries stored in the Wasm module. This
part of the Runtime environment is referred to as the ***Executor**.*

Definition
[\[nota-call-into-runtime\]](#nota-call-into-runtime){reference-type="ref"
reference="nota-call-into-runtime"} introduces the notation for calling
the runtime entry which is used whenever an algorithm of Polkadot RE
needs to access the runtime.

[\[nota-call-into-runtime\]]{#nota-call-into-runtime
label="nota-call-into-runtime"} By
$$\text{{\textsc{Call-Runtime-Entry}}} \left( R, \mathcal{R}\mathcal{E},
     \text{{\ttfamily{Runtime-Entry}}}, A, A_{\operatorname{len}} \right)$$
we refer to the task using the executor to invoke the while passing an
$A_1, \ldots, A_n$ argument to it and using the encoding described in
Section
[\[sect-send-args-to-runtime\]](#sect-send-args-to-runtime){reference-type="ref"
reference="sect-send-args-to-runtime"}.

In this section, we specify the general setup for an Executor call into
the Runtime. In Section [12](#sect-runtime-entries){reference-type="ref"
reference="sect-runtime-entries"} we specify the parameters and the
return values of each Runtime entry separately.

#### Access to Runtime API

When Polkadot RE calls a Runtime entry it should make sure Runtime has
access to the all Polkadot Runtime API functions described in Appendix
[\[sect-runtime-api\]](#sect-runtime-api){reference-type="ref"
reference="sect-runtime-api"}. This can be done for example by loading
another Wasm module alongside the runtime which imports these functions
from Polkadot RE as host functions.

#### Sending Arguments to Runtime  {#sect-runtime-send-args-to-runtime-enteries}

In general, all data exchanged between Polkadot RE and the Runtime is
encoded using SCALE codec described in Section
[7.1](#sect-scale-codec){reference-type="ref"
reference="sect-scale-codec"}. As a Wasm function, all runtime entries
have the following identical signatures:

 

 

In each invocation of a Runtime entry, the argument(s) which are
supposed to be sent to the entry, need to be encoded using SCALE codec
into a byte array $B$ using the procedure defined in Definition
[7.1](#sect-scale-codec){reference-type="ref"
reference="sect-scale-codec"}.

The Executor then needs to retrieve the Wam memory buffer of the Runtime
Wasm module and extend it to fit the size of the byte array. Afterwards,
it needs to copy the byte array $B$ value in the correct offset of the
extended buffer. Finally, when the Wasm method , corresponding to the
entry is invoked, two UINT32 integers are sent to the method as
arguments. The first argument is set to the offset where the byte array
$B$ is stored in the Wasm the extended shared memory buffer. The second
argument sets the length of the data stored in $B$., and the second one
is the size of $B$.

#### The Return Value from a Runtime Entry {#sect-runtime-return-value}

The value which is returned from the invocation is an integer,
representing two consecutive integers in which the least significant one
indicates the pointer to the offset of the result returned by the entry
encoded in SCALE codec in the memory buffer. The most significant one
provides the size of the blob.

In the case that the runtime entry is returning a boolean value, then
the SCALEd (boolean) value returns in the least significant byte and all
other bytes are set to zero.

Extrinsics {#sect-extrinsics}
----------

The block body consists of an array of extrinsics. Nonetheless, Polkadot
RE does not specify or limit the internals of each extrinsics. From
Polkadot RE point of view, each extrinsics is simply a SCALE-encoded
byte array (see Definition
[\[defn-scale-byte-array\]](#defn-scale-byte-array){reference-type="ref"
reference="defn-scale-byte-array"}).

### Preliminaries

[\[defn-account-key\]]{#defn-account-key
label="defn-account-key"}**Account key
$(\operatorname{sk}^a,
  \operatorname{pk}^a)$** is a pair of Ristretto SR25519
used to sign extrinsics among other accounts and blance-related
functions.

### Extrinsics Submission

Extrinsic submission is made by sending a *Transactions* network
message. The structure of this message is specified in Section
[10.1.5](#sect-msg-transactions){reference-type="ref"
reference="sect-msg-transactions"}. Upon receiving a Transactions
message, Polkadot RE decodes the transaction and calls runtime function,
defined in Section
[12.2.7](#sect-rte-validate-transaction){reference-type="ref"
reference="sect-rte-validate-transaction"}, to check the validity of the
extrinsic. If considers the submitted extrinsics as a valid one,
Polkadot RE makes the extrinsics available for the consensus engine for
inclusion in future blocks.

### Transaction Queue

A Block producer node should listen to all transaction messages. This is
because the transactions are submitted to the node through the
*transactions* network message specified in Section
[10.1.5](#sect-msg-transactions){reference-type="ref"
reference="sect-msg-transactions"}. Upon receiving a transactions
message, Polkadot RE separates the submitted transactions in the
transactions message into individual extrinsics and passes them to the
Runtime by executing Algorithm
[\[algo-validate-transactions\]](#algo-validate-transactions){reference-type="ref"
reference="algo-validate-transactions"} to validate and store them for
inclusion into future blocks. To that aim, Polkodot RE should keep a
*transaction pool* and a *transaction queue* defined as follows:

The **Transaction Queue** of a block producer node, formally referred to
as $\operatorname{TQ}$ is a data structure which stores the
transactions ready to be included in a block sorted according to their
priorities. The **Transaction Pool**, formally referred to as
$\operatorname{TP}$, is a hash table in which Polkadot RE
keeps the list of all valid transactions not in the transaction queue.  

Algorithm
[\[algo-validate-transactions\]](#algo-validate-transactions){reference-type="ref"
reference="algo-validate-transactions"} updates the transaction pool and
the transaction queue according to the received message:

**Algorithm**

[\[algo-validate-transactions\]]{#algo-validate-transactions
label="algo-validate-transactions"}[Validate-Extrinsics-and-Store]{.smallcaps}($M_T
      :$Transaction Message)

In which

-   [Longest-Chain]{.smallcaps} is defined in Definition
    [\[defn-longest-chain\]](#defn-longest-chain){reference-type="ref"
    reference="defn-longest-chain"}.

-   is a Runtime entry specified in Section
    [12.2.7](#sect-rte-validate-transaction){reference-type="ref"
    reference="sect-rte-validate-transaction"} and Requires(R),
    Priority(R) and Propagate(R) refer to the corresponding fields in
    the tuple returned by the entry when it deems that $T$ is valid.

-   [Provided-Tags]{.smallcaps}(T) is the list of tags that transaction
    $T$ provides. Polkadot RE needs to keep track of tags that
    transaction $T$ provides as well as requires after validating it.

-   [Insert-At(]{.smallcaps}$\operatorname{TQ}, T, \operatorname{Requires} (R),
      \operatorname{Priority} (R)$) places $T$ into
    $\operatorname{TQ}$ approperietly such that the
    transactions providing the tags which $T$ requires or have higher
    priority than $T$ are ahead of $T$.

-   [Maintain-Transaction-Pool]{.smallcaps} is described in Algorithm
    [\[algo-maintain-transaction-pool\]](#algo-maintain-transaction-pool){reference-type="ref"
    reference="algo-maintain-transaction-pool"}.

-   [Propagate(]{.smallcaps}$T$) include $T$ in the next *transactions
    message* sent to all peers of Polkadot RE node.

**Algorithm**

[\[algo-maintain-transaction-pool\]]{#algo-maintain-transaction-pool
label="algo-maintain-transaction-pool"}[Maintain-Transaction-Pool]{.smallcaps}

State Replication {#sect-state-replication}
-----------------

Polkadot nodes replicate each other's state by syncing the history of
the extrinsics. This, however, is only practical if a large set of
transactions are batched and synced at the time. The structure in which
the transactions are journaled and propagated is known as a block (of
extrinsics).

### Block Format {#sect-block-format}

In Polkadot RE, a block is made of two main parts, namely the *block
header* and the *list of extrinsics*. *The Extrinsics* represent the
generalization of the concept of *transaction*, containing any set of
data that is external to the system, and which the underlying chain
wishes to validate and keep track of.

#### Block Header {#block}

The block header is designed to be minimalistic in order to boost the
efficiency of the light clients. It is defined formally as follows:

[\[defn-block-header\]]{#defn-block-header label="defn-block-header"}The
**header of block B**, **$\operatorname{Head} (B)$** is a
5-tuple containing the following elements:

-   **[parent\_hash:]{.sans-serif}** is the 32-byte Blake2b hash (see
    Section [6.2](#sect-blake2){reference-type="ref"
    reference="sect-blake2"}) of the header of the parent of the block
    indicated henceforth by **$H_p$**.

-   **[number:]{.sans-serif}** formally indicated as **$H_i$** is an
    integer, which represents the index of the current block in the
    chain. It is equal to the number of the ancestor blocks. The genesis
    block has number 0.

-   **[state\_root:]{.sans-serif}** formally indicated as **$H_r$** is
    the root of the Merkle trie, whose leaves implement the storage for
    the system.

-   **[extrinsics\_root:]{.sans-serif}** is the field which is reserved
    for the Runtime to validate the integrity of the extrinsics
    composing the block body. For example, it can hold the root hash of
    the Merkle trie which stores an ordered list of the extrinsics being
    validated in this block. The [extrinsics\_root]{.sans-serif} is set
    by the runtime and its value is opaque to Polkadot RE. This element
    is formally referred to as **$H_e$**.

-   **[digest:]{.sans-serif}** this field is used to store any
    chain-specific auxiliary data, which could help the light clients
    interact with the block without the need of accessing the full
    storage. Polkadot RE does not impose any limitation or specification
    for this field. Essentially, it can be a byte array of any length.
    This field is indicated as **$H_d$**

[\[defn-block-header-hash\]]{#defn-block-header-hash
label="defn-block-header-hash"}The **Block Header Hash of Block $B$**,
**$H_h (B)$**, is the hash of the header of block $B$ encoded by simple
codec:"
$$H_h (B) :=\operatorname{Blake} 2 b (\operatorname{Enc}_{\operatorname{SC}} (\operatorname{Head}
     (B)))$$

#### Justified Block Header

The Justified Block Header is provided by the consensus engine and
presented to the Polkadot RE, for the block to be appended to the
blockchain. It contains the following parts:

-   **[**block\_header**]{.sans-serif}** the complete block header as
    defined in Section [3.3.1.1](#block){reference-type="ref"
    reference="block"} and denoted by
    $\operatorname{Head} (B)$.

-   **[justification]{.sans-serif}**: as defined by the consensus
    specification indicated by $\operatorname{Just} (B)$ .

-   **[authority Ids]{.sans-serif}**: This is the list of the Ids of
    authorities, which have voted for the block to be stored and is
    formally referred to as $A (B)$. An authority Id is 32bit.

#### Block Inherent Data

Block inherent data represent the totality of extrinsics included in
each block. In general, these data are collected or generated by
Polkadot RE and handed to Runtime for inclusion in the block. Table
[\[tabl-inherent-data\]](#tabl-inherent-data){reference-type="ref"
reference="tabl-inherent-data"} lists these inherent data, their
identifiers, and types.

  Identifier   Type   Description
  ------------ ------ -------------------------------------------------------------------------------------------------------------
  timstap0     u64    Unix epoch time in number of seconds
  babeslot     u64    Babe Slot Number^[\[defn-epoch-slot\]](#defn-epoch-slot){reference-type="ref" reference="defn-epoch-slot"}^

  : [\[tabl-inherent-data\]]{#tabl-inherent-data
  label="tabl-inherent-data"}List of inherent data

[\[defn-func-inherent-data\]]{#defn-func-inherent-data
label="defn-func-inherent-data"}The function
[Block-Inherents-Data($B_n$)]{.smallcaps} return the inherent data
defined in Table
[\[tabl-inherent-data\]](#tabl-inherent-data){reference-type="ref"
reference="tabl-inherent-data"} corresponding to Block $B$ as a SCALE
encoded dictionary as defined in Definition
[\[defn-scale-list\]](#defn-scale-list){reference-type="ref"
reference="defn-scale-list"}.

#### Block Body {#sect-block-body}

The Block Body consists of array extrinsics each encoded as a byte
array. The internal of extrinsics is completely opaque to Polkadot RE.
As such, it forms the point of Polkadot RE, and is simply a SCALE
encoded array of byte arrays. Formally:

[\[defn-block-body\]]{#defn-block-body label="defn-block-body"}The
**body of Block** $B$ represented as
**$\operatorname{Body} (B)$** is defined to be
$$\operatorname{Body} (B) :=\operatorname{Enc}_{\operatorname{SC}} (E_1, \ldots, E_n)$$
Where each $E_i \in \mathbb{B}$ is a SCALE encoded extrinsic.

### Block Submission {#sect-block-submission}

Block validation is the process by which the client asserts that a block
is fit to be added to the blockchain. This means that the block is
consistent with the world state and transitions from the state of the
system to a new valid state.

Blocks can be handed to the Polkadot RE both from the network stack for
example by means of Block response network message (see Section
[10.1.3](#sect-msg-block-response){reference-type="ref"
reference="sect-msg-block-response"} ) and from the consensus engine.

### Block Validation {#sect-block-validation}

Both the Runtime and the Polkadot RE need to work together to assure
block validity. A block is deemed valid if the block author had the
authorship right for the slot during which the slot was built as well as
if the transactions in the block constitute a valid transition of
states. The former criterion is validated by Polkadot RE according to
the block production consensus protocol. The latter can be verified by
Polkadot RE invoking entry into the Runtime as a part of the validation
process.

Polkadot RE implements the following procedure to assure the validity of
the block:

**Algorithm**

[Import-and-Validate-Block($B, \operatorname{Just} (B)$)]{.smallcaps}

For the definition of the finality and the finalized block see Section
[5.2](#sect-finality){reference-type="ref" reference="sect-finality"}.
$\operatorname{PBT}$ is the pruned block tree defined in
Definition [\[defn-block-tree\]](#defn-block-tree){reference-type="ref"
reference="defn-block-tree"}. [Verify-Authorship-Right]{.smallcaps} is
part of the block production consensus protocol and is described in
Algorithm
[\[algo-verify-authorship-right\]](#algo-verify-authorship-right){reference-type="ref"
reference="algo-verify-authorship-right"}.

Network Protocol[\[sect-network-interactions\]]{#sect-network-interactions label="sect-network-interactions"} {#network-protocol}
=============================================================================================================

Polkadot network protocol is work-in-progress. The API specification and
usage may change in future.

This chapter offers a high-level description of the network protocol
based on [@parity_technologies_substrate_2019]. Polkadot network
protocol relies on *libp2p*. Specifically, the following libp2p modules
are being used in the Polkadot Networking protocol:

-   [mplex.]{.sans-serif}

-   [yamux]{.sans-serif}

-   [secio]{.sans-serif}

-   [noise]{.sans-serif}

-   [kad]{.sans-serif} (kademlia)

-   [identity]{.sans-serif}

-   [ping]{.sans-serif}

For more detailed specification of these modules and the Peer-to-Peer
layer see libp2p specification document [@protocol_labs_libp2p_2019].

Node Identities and Addresses
-----------------------------

Similar to other decentralized networks, each Polkadot RE node possesses
a network private key and a network public key representing an ED25519
key pair [@liusvaara_edwards-curve_2017].

**Peer Identity**, formally noted by
$P_{\operatorname{id}}$ is derived from the node's public
key as follows:

 and uniquely identifies a node on the network.

Because the $P_{\operatorname{id}}$ is derived from the
node's public key, running two or more instances of Polkadot network
using the same network key is contrary to the Polkadot protocol.

All network communications between nodes on the network use encryption
derived from both sides' keys.

Discovery Mechanisms
--------------------

In order for a Polkadot node to join a peer-to-peer network, it has to
know a list of Polkadot nodes that already take part in the network.
This process of building such a list is referred to as *Discovery*. Each
element of this list is a pair consisting of the peer's node identities
and their addresses.

Polkadot discovery is done through the following mechanisms:

-   *Bootstrap nodes*: These are hard-coded node identities and
    addresses passed alongside with the network configuration.

-   *mDNS*, performing a UDP broadcast on the local network. Nodes that
    listen may respond with their identity as described in the mDNS
    section of [@protocol_labs_libp2p_2019]. (Note: mDNS can be disabled
    in the network configuration.)

-   *Kademlia random walk*. Once connected to a peer node, a Polkadot
    node can perform a random Kademlia 'FIND\_NODE' requests for the
    nodes to respond by propagating their view of the network.

     

Transport Protocol
------------------

A Polkadot node can establish a connection with nodes in its peer list.
All the connections must always use encryption and multiplexing. While
some nodes' addresses (eg. addresses using '/quic') already imply the
encryption and/or multiplexing to use, for others the
"multistream-select" protocol is used in order to negotiate an
encryption layer and/or a multiplexing layer.

The following transport protocol is supported by a Polkadot node:

-   *TCP/IP* for addresses of the form '/ip4/1.2.3.4/tcp/5'. Once the
    TCP connection is open, an encryption and a multiplexing layers are
    negotiated on top.

-   *WebSockets* for addresses of the form '/ip4/1.2.3.4/tcp/5/ws'. A
    TC/IP connection is open and the WebSockets protocol is negotiated
    on top. Communications then happen inside WebSockets data frames.
    Encryption and multiplexing are additionally negotiated again inside
    this channel.

-   DNS for addresses of the form '/dns4/example.com/tcp/5' or
    '/dns4/example.com/tcp/5/ws'. A node's address can contain a domain
    name.

### Encryption

The following encryption protocols from libp2p are supported by Polkadot
protocol:

**Secio**: A TLS-1.2-like protocol but without certificates
[@protocol_labs_libp2p_2019]. Support for secio will likely to be
deprecated in the future.

**Noise**: Noise is a framework for crypto protocols based on the
Diffie-Hellman key agreement [@perrin_noise_2018]. Support for noise is
experimental and details may change in the future.

### Multiplexing

The following multiplexing protocols are supported:

-   **Mplex**: Support for mplex will be deprecated in the future.

-   **Yamux**.

Substreams
----------

Once a connection has been established between two nodes and is able to
use multiplexing, substreams can be opened. When a substream is open,
the *multistream-select* protocol is used to negotiate which protocol to
use on that given substream.

### Periodic Ephemeral Substreams

A Polkadot RE node should open several substreams. In particular, it
should periodically open ephemeral substreams in order to:

-   ping the remote peer and check whether the connection is still
    alive. Failure for the remote peer to reply leads to a
    disconnection. This uses the libp2p *ping* protocol specified in
    [@protocol_labs_libp2p_2019].

-   ask information from the remote. This is the *identity* protocol
    specified in [@protocol_labs_libp2p_2019].

-   send Kademlia random walk queries. Each Kademlia query is done in a
    new separate substreams. This uses the libp2p *Kademlia* protocol
    specified in [@protocol_labs_libp2p_2019].

### Polkadot Communication Substream

For the purposes of communicating Polkadot messages, the dailer of the
connection opens a unique substream. Optionally, the node can keep a
unique substream alive for this purpose. The name of the protocol
negotiated is based on the *protocol ID* passed as part of the network
configuration. This protocol ID should be unique for each chain and
prevents nodes from different chains to connect to each other.

The structure of SCALE encoded messages sent over the unique Polkadot
communication substream is described in Appendix
[10](#sect-network-messages){reference-type="ref"
reference="sect-network-messages"}.

Once the substream is open, the first step is an exchange of a *status*
message from both sides described in Section
[\[sect-net-msg-status\]](#sect-net-msg-status){reference-type="ref"
reference="sect-net-msg-status"}.

Communications within this substream include:

-   Syncing. Blocks are announced and requested from other nodes.

-   Gossiping. Used by various subprotocols such as GRANDPA.

-   Polkadot Network Specialization: .

Consensus {#chap-consensu}
=========

Consensus in Polkadot RE is achieved during the execution of two
different procedures. The first procedure is block production and the
second is finality. Polkadot RE must run these procedures, if and only
if it is running on a validator node.

Block Production {#sect-babe}
----------------

[\[sect-block-production\]]{#sect-block-production
label="sect-block-production"}

Polkadot RE uses BABE protocol [@w3f_research_group_blind_2019] for
block production designed based on Ouroboros praos
[@david_ouroboros_2018]. BABE execution happens in sequential
non-overlapping phases known as an ***epoch***. Each epoch on its turn
is divided into a predefined number of slots. All slots in each epoch
are sequentially indexed starting from 0. At the beginning of each
epoch, the BABE node needs to run Algorithm
[\[algo-block-production-lottery\]](#algo-block-production-lottery){reference-type="ref"
reference="algo-block-production-lottery"} to find out in which slots it
should produce a block and gossip to the other block producers. In turn,
the block producer node should keep a copy of the block tree and grow it
as it receives valid blocks from other block producers. A block producer
prunes the tree in parallel using Algorithm
[\[algo-block-tree-prunning\]](#algo-block-tree-prunning){reference-type="ref"
reference="algo-block-tree-prunning"}.

### Preliminaries

A **block producer**, noted by $\mathcal{P}_j$, is a node running
Polkadot RE which is authorized to keep a transaction queue and which
gets a turn in producing blocks.

**Block authoring session key pair $(\operatorname{sk}^s_j,
  \operatorname{pk}^s_j)$** is an SR25519 key pair which
the block producer $\mathcal{P}_j$ signs by their account key (see
Definition
[\[defn-account-key\]](#defn-account-key){reference-type="ref"
reference="defn-account-key"}) and is used to sign the produced block as
well as to compute its lottery values in Algorithm
[\[algo-block-production-lottery\]](#algo-block-production-lottery){reference-type="ref"
reference="algo-block-production-lottery"}.

[\[defn-epoch-slot\]]{#defn-epoch-slot label="defn-epoch-slot"}A block
production **epoch**, formally referred to as $\mathcal{E}$ is a period
with pre-known starting time and fixed length during which the set of
block producers stays constant. Epochs are indexed sequentially, and we
refer to the $n^{\operatorname{th}}$ epoch since genesis by
$\mathcal{E}_n$. Each epoch is divided into equal length periods known
as block production **slots**, sequentially indexed in each epoch. The
index of each slot is called **slot number**. Each slot is awarded to a
subset of block producers during which they are allowed to generate a
block.

[\[note-slot\]]{#note-slot label="note-slot"}We refer to the number of
slots in epoch $\mathcal{E}_n$ by $\operatorname{sc}_n$.
$\operatorname{sc}_n$ is set to the field in the returned
data from the call of the Runtime entry (see
[12.2.5](#sect-rte-babeapi-epoch){reference-type="ref"
reference="sect-rte-babeapi-epoch"}) at the beginning of each epoch. For
a given block $B$, we use the notation **$s_B$** to refer to the slot
during which $B$ has been produced. Conversely, for slot $s$,
$\mathcal{B}_s$ is the set of Blocks generated at slot $s$.

Definition
[\[defn-epoch-subchain\]](#defn-epoch-subchain){reference-type="ref"
reference="defn-epoch-subchain"} provides an iterator over the blocks
produced during an specific epoch.

[\[defn-epoch-subchain\]]{#defn-epoch-subchain
label="defn-epoch-subchain"} By [SubChain($\mathcal{E}_n$)]{.smallcaps}
for epoch $\mathcal{E}_n$, we refer to the path graph of
$\operatorname{BT}$ which contains all the blocks generated
during the slots of epoch $\mathcal{E}_n$. When there is more than one
block generated at a slot, we choose the one which is also on
[Longest-Chain($\operatorname{BT}$)]{.smallcaps}.

### Block Production Lottery

[\[defn-winning-threshold\]]{#defn-winning-threshold
label="defn-winning-threshold"}**Winning threshold** denoted by
**$\tau$** is the threshold which is used alongside with the result of
Algorirthm
[\[algo-block-production-lottery\]](#algo-block-production-lottery){reference-type="ref"
reference="algo-block-production-lottery"} to decide if a block producer
is the winner of a specific slot. $\tau$ is set to result of call into
runtime entry.

A block producer aiming to produce a block during $\mathcal{E}_n$ should
run Algorithm
[\[algo-block-production-lottery\]](#algo-block-production-lottery){reference-type="ref"
reference="algo-block-production-lottery"} to identify the slots it is
awarded. These are the slots during which the block producer is allowed
to build a block. The $\operatorname{sk}$ is the block
producer lottery secret key and $n$ is the index of epoch for whose
slots the block producer is running the lottery.

**Algorithm**

[\[algo-block-production-lottery\]]{#algo-block-production-lottery
label="algo-block-production-lottery"}[Block-production-lottery]{.smallcaps}($\operatorname{sk}
      :$session secret key of the producer,

$n :$epoch index)

For any slot $i$ in epoch $n$ where $d < \tau$, the block producer is
required to produce a block. For the definitions of
[Epoch-Randomness]{.smallcaps} and *[VRF]{.smallcaps}* functions, see
Algorithm
[\[algo-epoch-randomness\]](#algo-epoch-randomness){reference-type="ref"
reference="algo-epoch-randomness"} and Section
[6.4](#sect-vrf){reference-type="ref" reference="sect-vrf"}
respectively.

### Slot Number Calculation

It is essential for a block producer to calculate and validate the slot
number at a certain point in time. Slots are dividing the time continuum
in an overlapping interval. At a given time, the block producer should
be able to determine the set of slots which can be associated to a valid
block generated at that time. We formalize the notion of validity in the
following definitions:

[\[slot-time-cal-tail\]]{#slot-time-cal-tail
label="slot-time-cal-tail"}The **slot tail**, formally referred to by
$\operatorname{SlTl}$ represents the number of on-chain
blocks that are used to estimate the slot time of a given slot. This
number is set to be 1200.

Algorithm [\[algo-slot-time\]](#algo-slot-time){reference-type="ref"
reference="algo-slot-time"} determines the slot time for a future slot
based on the *block arrival time* associated with blocks in the slot
tail defined in Definition
[\[defn-block-time\]](#defn-block-time){reference-type="ref"
reference="defn-block-time"}.

[\[defn-block-time\]]{#defn-block-time label="defn-block-time"}The
**block arrival time** of block $B$ for node $j$ formally represented by
**$T^j_B$** is the local time of node $j$ when node $j$ has received the
block $B$ for the first time. If the node $j$ itself is the producer of
$B$, $T_B^j$ is set equal to the time that the block is produced. The
index $j$ in $T^j_B$ notation may be dropped and B's arrival time is
referred to by $T_B$ when there is no ambiguity about the underlying
node.

In addition to the arrival time of block $B$, the block producer also
needs to know how many slots have passed since the arrival of $B$. This
value is formalized in Definition
[\[defn-slot-offset\]](#defn-slot-offset){reference-type="ref"
reference="defn-slot-offset"}.

[\[defn-slot-offset\]]{#defn-slot-offset label="defn-slot-offset"}Let
$s_i$ and $s_j$ be two slots belonging to epochs $\mathcal{E}_k$ and
$\mathcal{E}_l$. By [Slot-Offset]{.smallcaps}$(s_i, s_j)$ we refer to
the function whose value is equal to the number of slots between $s_i$
and $s_j$ (counting $s_j$) on time continuum. As such, we have
[Slot-Offset]{.smallcaps}$(s_i, s_i) = 0$.

**Algorithm**

[\[algo-slot-time\]]{#algo-slot-time
label="algo-slot-time"}[Slot-Time]{.smallcaps}($s$: the slot number of
the slot whose time needs to be determined)

### Block Production {#block-production}

At each epoch, each block producer should run Algorithm
[\[algo-block-production\]](#algo-block-production){reference-type="ref"
reference="algo-block-production"} to produce blocks during the slots it
has been awarded during that epoch. The produced blocks need to be
broadcasted alongside with the *babe header* defined in Definition
[\[defn-babe-header\]](#defn-babe-header){reference-type="ref"
reference="defn-babe-header"}.

The [\[defn-babe-header\]]{#defn-babe-header
label="defn-babe-header"}**Babe Header** of block B, referred to
formally by **$H_{\operatorname{Babe}} (B)$** is a tuple
that consists of the following components: $$(\pi, d, j, s, w)$$ in
which:

  ----------- -----------------------------------------------------------------------
    $\pi, d$: are the results of the block lottervrf\_output, vrfy for slot s.
         $j$: is the SR25519 session public key associated with the block producer.
           s: is the slot at which the block is produced.
            w reserved
  ----------- -----------------------------------------------------------------------

 

The block producer includes $H_{\operatorname{Babe}} (B)$
as a log in $H_d (B)$ and sign $\operatorname{Head} (B)$ as
defined in Definition
[\[defn-block-signature\]](#defn-block-signature){reference-type="ref"
reference="defn-block-signature"}

[\[block-signature\]]{#block-signature label="block-signature"}The
**Block Signature** noted by $S_B$ is computed as
$\operatorname{Sig}_{\operatorname{SR} 25519, \operatorname{sk}^s_j}
  (\operatorname{Enc}_{\operatorname{SC}} (\operatorname{Black} 2 s (\operatorname{Head} (B_{}))))$

 

**Algorithm**

[\[algo-block-production\]]{#algo-block-production
label="algo-block-production"}[Invoke-Block-Authoring]{.smallcaps}($\operatorname{sk}$,
pk, $n$,
$\operatorname{BT} : \operatorname{Current} \operatorname{Block} \operatorname{Tree}$)

### Epoch Randomness {#sect-epoch-randomness}

At the end of epoch $\mathcal{E}_n$, each block producer is able to
compute the randomness seed it needs in order to participate in the
block production lottery in epoch $\mathcal{E}_{n + 2}$. The computation
of the seed is described in Algorithm
[\[algo-epoch-randomness\]](#algo-epoch-randomness){reference-type="ref"
reference="algo-epoch-randomness"} which uses the concept of epoch
subchain described in Definition
[\[defn-epoch-subchain\]](#defn-epoch-subchain){reference-type="ref"
reference="defn-epoch-subchain"}.

**Algorithm**

[\[algo-epoch-randomness\]]{#algo-epoch-randomness
label="algo-epoch-randomness"}[Epoch-Randomness]{.smallcaps}($n > 2 :$epoch
index)

In which value $d_B$ is the VRF output computed for slot $s_B$ by
running Algorithm
[\[algo-block-production-lottery\]](#algo-block-production-lottery){reference-type="ref"
reference="algo-block-production-lottery"}.

 

### Verifying Authorship Right {#sect-verifying-authorship}

Seal $D_s$

When a Polkadot node receives a produced block, it needs to verify if
the block producer was entitled to produce the block in the given slot
by running Algorithm
[\[algo-verify-authorship-right\]](#algo-verify-authorship-right){reference-type="ref"
reference="algo-verify-authorship-right"} where:

-   T$_B$ is $B$'s arrival time defined in Definition
    [\[defn-block-time\]](#defn-block-time){reference-type="ref"
    reference="defn-block-time"}.

-   $H_d (B)$ is the digest sub-component of
    $\operatorname{Head} (B)$ defined in Definition
    [\[defn-block-header\]](#defn-block-header){reference-type="ref"
    reference="defn-block-header"}.

-   $\operatorname{AuthorityDirectory}^{\mathcal{E}_c}$ is
    the set of Authority ID for block producers of epoch
    $\mathcal{E}_c$.

-   [verify-Slot-Winner]{.smallcaps} is defined in Algorithm
    [\[algo-verify-slot-winner\]](#algo-verify-slot-winner){reference-type="ref"
    reference="algo-verify-slot-winner"}.

**Algorithm**

[\[algo-verify-authorship-right\]]{#algo-verify-authorship-right
label="algo-verify-authorship-right"}[Verify-Authorship-Right]{.smallcaps}($\operatorname{Head}_s
      (B)$: The header of the block being verified)

Algorithm
[\[algo-verify-slot-winner\]](#algo-verify-slot-winner){reference-type="ref"
reference="algo-verify-slot-winner"} is run as a part of the
verification process, when a node is importing a block, in which:

-   [Epoch-Randomness]{.smallcaps} is defined in Algorithm
    [\[algo-epoch-randomness\]](#algo-epoch-randomness){reference-type="ref"
    reference="algo-epoch-randomness"}.

-   $H_{\operatorname{BABE}} (B)$ is the BABE header
    defined in Definition
    [\[defn-babe-header\]](#defn-babe-header){reference-type="ref"
    reference="defn-babe-header"}.

-   [Verify-VRF]{.smallcaps} is described in Section
    [6.4](#sect-vrf){reference-type="ref" reference="sect-vrf"}.

-   $\tau$ is the winning threshold defined in
    [\[defn-winning-threshold\]](#defn-winning-threshold){reference-type="ref"
    reference="defn-winning-threshold"}.

**Algorithm**

[\[algo-verify-slot-winner\]]{#algo-verify-slot-winner
label="algo-verify-slot-winner"}[Verify-Slot-Winner]{.smallcaps}($B$:
the block whose winning status to be verified)

$(d_B, \pi_B)$: Block Lottery Result for Block $B$,

$s_n$: the slot number,

$n$: Epoch index

AuthorID: The public session key of the block producer

### Blocks Building Process {#sect-block-building}

The blocks building process is triggered by Algorithm
[\[algo-block-production\]](#algo-block-production){reference-type="ref"
reference="algo-block-production"} of the consensus engine which runs
Alogrithm [\[algo-build-block\]](#algo-build-block){reference-type="ref"
reference="algo-build-block"}.

**Algorithm**

[\[algo-build-block\]]{#algo-build-block
label="algo-build-block"}[Build-Block]{.smallcaps}($C_{\operatorname{Best}}$:
The chain where at its head, the block to be constructed,

s: Slot number)

$\operatorname{Head} (B)$ is defined in Definition
[\[defn-block-header\]](#defn-block-header){reference-type="ref"
reference="defn-block-header"}. [Block-Inherents-Data]{.smallcaps},
[Inherents-Queue]{.smallcaps}, [Block-Is-Full]{.smallcaps} and
[Next-Ready-Extrinsic]{.smallcaps} are defined in Definition

Finality {#sect-finality}
--------

Polkadot RE uses GRANDPA Finality protocol [@stewart_grandpa:_2019] to
finalize blocks. Finality is obtained by consecutive rounds of voting by
validator nodes. Validators execute GRANDPA finality process in parallel
to Block Production as an independent service. In this section, we
describe the different functions that GRANDPA service is supposed to
perform to successfully participate in the block finalization process.

### Preliminaries

A **GRANDPA Voter**, $v$, is represented by a key pair
$(k^{\operatorname{pr}}_v, v_{\operatorname{id}})$
where $k_v^{\operatorname{pr}}$ represents its private key
which is an $\operatorname{ED} 25519$ private key, is a
node running GRANDPA protocol, and broadcasts votes to finalize blocks
in a Polkadot RE - based chain. The **set of all GRANDPA voters** is
indicated by $\mathbb{V}$. For a given block B, we have
$$\mathbb{V}_B = \text{{\ttfamily{grandpa\_authorities}}} (B)$$ where
$\mathtt{grandpa\_authorities}$ is the entry into runtime described in
Section [12.2.6](#sect-rte-grandpa-auth){reference-type="ref"
reference="sect-rte-grandpa-auth"}.

**GRANDPA state**, $\operatorname{GS}$, is defined as
$$\operatorname{GS} :=\{\mathbb{V}, \operatorname{id}_{\mathbb{V}}, r\}$$
where:

$\mathbb{V}$: is the set of voters.

**$\mathbb{V}_{\operatorname{id}}$**: is an incremental
counter tracking membership, which changes in V.

**r**: is the voting round number.

Now we need to define how Polkadot RE counts the number of votes for
block $B$. First a vote is defined as:

[\[defn-vote\]]{#defn-vote label="defn-vote"}A **GRANDPA vote** or
simply a vote for block $B$ is an ordered pair defined as
$$V_{} (B) :=(H_h (B), H_i (B))$$ where $H_h (B)$ and $H_i (B)$ are the
block hash and the block number defined in Definitions
[\[defn-block-header\]](#defn-block-header){reference-type="ref"
reference="defn-block-header"} and
[\[defn-block-header-hash\]](#defn-block-header-hash){reference-type="ref"
reference="defn-block-header-hash"} respectively.

Voters engage in a maximum of two sub-rounds of voting for each round
$r$. The first sub-round is called **pre-vote** and the second sub-round
is called **pre-commit**.

By **$V_v^{r, \operatorname{pv}}$** and
**$V_v^{r, \operatorname{pc}}$** we refer to the vote cast
by voter $v$ in round $r$ (for block $B$) during the pre-vote and the
pre-commit sub-round respectively.

The GRANDPA protocol dictates how an honest voter should vote in each
sub-round, which is described in Algorithm
[\[algo-grandpa-round\]](#algo-grandpa-round){reference-type="ref"
reference="algo-grandpa-round"}. After defining what constitues a vote
in GRANDPA, we define how GRANDPA counts votes.

Voter $v$ **equivocates** if they broadcast two or more valid votes to
blocks not residing on the same branch of the block tree during one
voting sub-round. In such a situation, we say that $v$ is an
**equivocator** and any vote
$V_v^{r, \operatorname{stage}} (B)$ cast by $v$ in that
round is an **equivocatory vote** and
$$\mathcal{E}^{r, \operatorname{stage}}$$ represents the
set of all equivocators voters in sub-round
"$\operatorname{stage}$" of round $r$. When we want to
refer to the number of equivocators whose equivocation has been observed
by voter $v$ we refer to it by:
$$\mathcal{E}^{r, \operatorname{stage}}_{\operatorname{obs} (v)}$$

A vote $V_v^{r, \operatorname{stage}} = V (B)$ is
**invalid** if

-   $H (B)$ does not correspond to a valid block;

-   $B$ is not an (eventual) descendant of a previously finalized block;

-   $M^{r, \operatorname{stage}}_v$ does not bear a valid
    signature;

-   $\operatorname{id}_{\mathbb{V}}$ does not match the
    current $\mathbb{V}$;

-   If $V_v^{r, \operatorname{stage}}$ is an equivocatory
    vote.

For validator v, **the set of observed direct votes for Block $B$ in
round $r$**, formally denoted by
$\operatorname{VD}^{r, \operatorname{stage}}_{\operatorname{obs}
  (v)}^{}_{} (B)$ is equal to the union of:

-   set of valid votes $V^{r, \operatorname{stage}}_{v_i}$
    cast in round $r$ and received by v such that
    $V^{r, \operatorname{stage}}_{v_i} = V (B)$.

We refer to **the set of total votes observed by voter $v$ in sub-round
"$\operatorname{stage}$" of round $r$** by **$V^{r,
  \operatorname{stage}}_{\operatorname{obs} (v)}^{}_{}$**.

The **set of all observed votes by $v$ in the sub-round stage of round
$r$ for block $B$**,
**$V^{r, \operatorname{stage}}_{\operatorname{obs} (v)}
  (B)$** is equal to all of the observed direct votes casted for block
$B$ and all of the $B$'s descendents defined formally as:
$$V^{r, \operatorname{stage}}_{\operatorname{obs} (v)} (B) :=\bigcup_{v_i \in
     \mathbb{V}, B \geqslant B'} \operatorname{VD}^{r, \operatorname{stage}}_{\operatorname{obs} (v)}
     (B')_{}^{}_{}$$ The **total number of observed votes for Block $B$
in round $r$** is defined to be the size of that set plus the total
number of equivocators voters:
$$\#V^{r, \operatorname{stage}}_{\operatorname{obs} (v)} (B) = |V^{r,
     \operatorname{stage}}_{\operatorname{obs} (v)} (B) | + | \mathcal{E}^{r,
     \operatorname{stage}}_{\operatorname{obs} (v)} |$$

The current **pre-voted** block
$B^{r, \operatorname{pv}}_v$ is the block with
$$H_n (B^{r, \operatorname{pv}}_v) = \operatorname{Max} (H_n (B) | \forall B :
     \#V_{\operatorname{obs} (v)}^{r, \operatorname{pv}} (B) \geqslant 2 / 3|\mathbb{V}|)$$

Note that for genesis block $\operatorname{Genesis}$ we
always have $\#V_{\operatorname{obs}
(v)}^{r, \operatorname{pv}} (B) = | \mathbb{V} |$.

 

Finally, we define when a voter $v$ see a round as completable, that is
when they are confident that $B_v^{r, \operatorname{pv}}$
is an upper bound for what is going to be finalised in this round.  

[\[defn-grandpa-completable\]]{#defn-grandpa-completable
label="defn-grandpa-completable"}We say that round $r$ is
**completable** if
$|V^{r, \operatorname{pc}}_{\operatorname{obs} (v)} |
  +\mathcal{E}^{r, \operatorname{pc}}_{\operatorname{obs} (v)} > \frac{2}{3} \mathbb{V}$
and for all $B' > B_v^{r, \operatorname{pv}}$:
$$\begin{array}{l}
       |V^{r, \operatorname{pc}}_{\operatorname{obs} (v)} | -\mathcal{E}^{r,
       \operatorname{pc}}_{\operatorname{obs} (v)} - |V^{r, \operatorname{pc}}_{\operatorname{obs}
       (v)_{}} (B') | > \frac{2}{3} |\mathbb{V}|
     \end{array}$$

Note that in practice we only need to check the inequality for those
$B' >
B_v^{r, \operatorname{pv}}$ where
$|V^{r, \operatorname{pc}}_{\operatorname{obs} (v)_{}} (B')
| > 0$.

 

### Voting Messages Specification

Voting is done by means of broadcasting voting messages to the network.
Validators inform their peers about the block finalized in round $r$ by
broadcasting a finalization message (see Algorithm
[\[algo-grandpa-round\]](#algo-grandpa-round){reference-type="ref"
reference="algo-grandpa-round"} for more details). These messages are
specified in this section.

A vote casted by voter $v$ should be broadcasted as a **message
$M^{r, \operatorname{stage}}_v$** to the network by voter
$v$ with the following structure:
$$M^{r, \operatorname{stage}}_v :=\operatorname{Enc}_{\operatorname{SC}} (r,
     \operatorname{id}_{\mathbb{V}}, \operatorname{Enc}_{\operatorname{SC}} (\operatorname{stage}, V_v^{r,
     \operatorname{stage}}, \operatorname{Sig}_{\operatorname{ED} 25519} (\operatorname{Enc}_{\operatorname{SC}}
     (\operatorname{stage}, V_v^{r, \operatorname{stage}}, r, V_{\operatorname{id}}), v_{\operatorname{id}})$$
Where:

[\[defn-grandpa-justification\]]{#defn-grandpa-justification
label="defn-grandpa-justification"}The **justification for block B in
round $r$** of GRANDPA protocol defined $J^r (B)$ is a vector of pairs
of the type:
$$(V (B'), (\operatorname{Sign}^{r, \operatorname{pc}}_{v_i} (B'), v_{\operatorname{id}}))$$
in which either $$B' \geqslant B$$ or
$V^{r, \operatorname{pc}}_{v_i} (B')$ is an equivocatory
vote.

In all cases,
$\operatorname{Sign}^{r, \operatorname{pc}}_{v_i} (B')$
is the signature of voter $v_i$ broadcasted during the pre-commit
sub-round of round r.

We say $J^r (B)$ **justifies the finalization** of $B$ if the number of
valid signatures in $J^r (B)$ is greater than $\frac{2}{3}
  |\mathbb{V}_B |$.

**$\operatorname{GRANDPA}$ finalizing message for block $B$
in round $r$** represented as
**$M_v^{r, \operatorname{Fin}}$(B)** is a message
broadcasted by voter $v$ to the network indicating that voter $v$ has
finalized block $B$ in round $r$. It has the following structure:
$$M^{r, \operatorname{Fin}}_v (B) :=\operatorname{Enc}_{\operatorname{SC}} (r, V (B), J^r
     (B))$$ in which $J^r (B)$ in the justification defined in
Definition
[\[defn-grandpa-justification\]](#defn-grandpa-justification){reference-type="ref"
reference="defn-grandpa-justification"}.

### Initiating the GRANDPA State

A validator needs to initiate its state and sync it with other
validators, to be able to participate coherently in the voting process.
In particular, considering that voting is happening in different rounds
and each round of voting is assigned a unique sequential round number
$r_v$, it needs to determine and set its round counter $r$ in accordance
with the current voting round $r_n$, which is currently undergoing in
the network.

As instructed in Algorithm
[\[alg-join-leave-grandpa\]](#alg-join-leave-grandpa){reference-type="ref"
reference="alg-join-leave-grandpa"}, whenever the membership of GRANDPA
voters changes, $r$ is set to 0 and $V_{\operatorname{id}}$
needs to be incremented.

**Algorithm**

[\[alg-join-leave-grandpa\]]{#alg-join-leave-grandpa
label="alg-join-leave-grandpa"}[Join-Leave-Grandpa-Voters]{.smallcaps}
($\mathcal{V}$)

### Voting Process in Round $r$

For each round $r$, an honest voter $v$ must participate in the voting
process by following Algorithm
[\[algo-grandpa-round\]](#algo-grandpa-round){reference-type="ref"
reference="algo-grandpa-round"}.

**Algorithm**

[\[algo-grandpa-round\]]{#algo-grandpa-round
label="algo-grandpa-round"}[Play-Grandpa-round]{.smallcaps}$(r)$

The condition of *completablitiy* is defined in Definition
[\[defn-grandpa-completable\]](#defn-grandpa-completable){reference-type="ref"
reference="defn-grandpa-completable"}.
[Best-Final-Candidate]{.smallcaps} function is explained in Algorithm
[\[algo-grandpa-best-candidate\]](#algo-grandpa-best-candidate){reference-type="ref"
reference="algo-grandpa-best-candidate"} and
[[Attempt-To-Finalize-Round]{.smallcaps}($r$)]{.smallcaps} is described
in Algorithm
[\[algo-attempt-tofinalize\]](#algo-attempt-tofinalize){reference-type="ref"
reference="algo-attempt-tofinalize"}.

**Algorithm**

[\[algo-grandpa-best-candidate\]]{#algo-grandpa-best-candidate
label="algo-grandpa-best-candidate"}[Best-Final-Candidate]{.smallcaps}($r$)

**Algorithm**

[\[algo-attempt-tofinalize\]]{#algo-attempt-tofinalize
label="algo-attempt-tofinalize"}[Attempt-To-Finalize-Round]{.smallcaps}($r$)

Block Finalization {#sect-block-finalization}
------------------

[\[defn-finalized-block\]]{#defn-finalized-block
label="defn-finalized-block"}A Polkadot relay chain node n should
consider block $B$ as **finalized** if any of the following criteria
holds for $B' \geqslant B$:

-   $V^{r, \operatorname{pc}}_{\operatorname{obs} (n)}^{}_{} (B') > 2
        / 3 |\mathbb{V}_{B'} |$.

-   it receives a $M_v^{r, \operatorname{Fin}} (B')$
    message in which $J^r (B)$ justifies the finalization (according to
    Definition
    [\[defn-grandpa-justification\]](#defn-grandpa-justification){reference-type="ref"
    reference="defn-grandpa-justification"}).

-   it receives a block data message for $B'$ with
    $\operatorname{Just} (B')$ defined in Section
    [\[sect-justified-block-header\]](#sect-justified-block-header){reference-type="ref"
    reference="sect-justified-block-header"} which justifies the
    finalization.

for

-   any round $r$ if the node $n$ is *not* a GRANDPA voter.

-   only for rounds $r$ for which the the node $n$ has invoked Algorithm
    [\[algo-grandpa-round\]](#algo-grandpa-round){reference-type="ref"
    reference="algo-grandpa-round"} if $n$ is a GRANDPA voter.

Note that all Polkadot relay chain nodes are supposed to listen to
GRANDPA finalizing messages regardless if they are GRANDPA voters.

Cryptographic Algorithms
========================

Hash Functions {#sect-hash-functions}
--------------

BLAKE2 {#sect-blake2}
------

BLAKE2 is a collection of cryptographic hash functions known for their
high speed. their design closely resembles BLAKE which has been a
finalist in SHA-3 competition.

Polkadot is using Blake2b variant which is optimized for 64bit
platforms. Unless otherwise specified, Blake2b hash function with 256bit
output is used whenever Blake2b is invoked in this document. The
detailed specification and sample implementations of all variants of
Blake2 hash functions can be found in RFC 7693 [@saarinen_blake2_2015].

Randomness {#sect-randomness}
----------

VRF {#sect-vrf}
---

Auxiliary Encodings {#sect-encoding}
===================

SCALE Codec {#sect-scale-codec}
-----------

Polkadot RE uses *Simple Concatenated Aggregate Little-Endian" (SCALE)
codec* to encode byte arrays as well as other data structures. SCALE
provides a canonical encoding to produce consistent hash values across
their implementation, including the Merkle hash proof for the State
Storage.

[\[defn-scale-byte-array\]]{#defn-scale-byte-array
label="defn-scale-byte-array"}The **SCALE codec** for **Byte array** $A$
such that $$A :=b_1 b_2 \ldots b_n$$ such that $n < 2^{536}$ is a byte
array refered to
$\operatorname{Enc}_{\operatorname{SC}}
  (A)$ and defined as:
$$\operatorname{Enc}_{\operatorname{SC}} (A) :=\operatorname{Enc}^{\operatorname{Len}}_{\operatorname{SC}}
     (\| A \|) | | A$$ where
$\operatorname{Enc}_{\operatorname{SC}}^{\operatorname{Len}}$
is defined in Definition
[\[defn-sc-len-encoding\]](#defn-sc-len-encoding){reference-type="ref"
reference="defn-sc-len-encoding"}.

[\[defn-scale-tuple\]]{#defn-scale-tuple label="defn-scale-tuple"}The
**SCALE codec** for **Tuple** $T$ such that: $$T :=(A_1, \ldots, A_n)$$
Where $A_i$'s are values of **different types**, is defined as:
$$\operatorname{Enc}_{\operatorname{SC}} (T) :=\operatorname{Enc}_{\operatorname{SC}} (A_1) | |
     \operatorname{Enc}_{\operatorname{SC}} (A_2) | | \ldots | | \operatorname{Enc}_{\operatorname{SC}} (A_n)$$

In case of a tuple (or struct), the knowledge of the shape of data is
not encoded even though it is necessary for decoding. The decoder needs
to derive that information from the context where the encoding/decoding
is happenning.

[\[defn-varrying-data-type\]]{#defn-varrying-data-type
label="defn-varrying-data-type"}We define a **varying data** type to be
an ordered set of data types $$\mathcal{T}= \{ T_1, \ldots, T_n \}$$ A
value $\boldsymbol{A}$ of varying date type is a pair
$(A_{\operatorname{Type}},
  A_{\operatorname{Value}})$ where
$A_{\operatorname{Type}} = T_i$ for some $T_i \in
  \mathcal{T}$ and $A_{\operatorname{Value}}$ is its value
of type $T_i$. We define
$\operatorname{idx} (T_i) = i - 1.$

In particular, we define **optional type** to be $\mathcal{O}= \{
  \operatorname{None}, T_2 \}$ for some data type $T_2$
where $\operatorname{idx}
  (\operatorname{None}) = 0$
$(\operatorname{None}, \phi)$ is the only possible value,
when the data is of type None and a codec value is one byte of 0 value.

[\[defn-scale-variable-type\]]{#defn-scale-variable-type
label="defn-scale-variable-type"}Scale coded for value **$A =
  (A_{\operatorname{Type}}, A_{\operatorname{Value}})$
of varying data type** $\mathcal{T}= \{
  T_1, \ldots, T_n \}$
$$\operatorname{Enc}_{\operatorname{SC}} (A) :=\operatorname{Enc}_{\operatorname{SC}} (\operatorname{Idx}
     (A_{\operatorname{Type}})) | | \operatorname{Enc}_{\operatorname{SC}} (A_{\operatorname{Value}})$$
Where $\operatorname{Idx}$ is encoded in a fixed length
integer determining the type of $A$.

In particular, for the optional type defined in Definition
[\[defn-varrying-data-type\]](#defn-varrying-data-type){reference-type="ref"
reference="defn-varrying-data-type"}, we have:
$$\operatorname{Enc}_{\operatorname{SC}} ((\operatorname{None}, \phi)) :=0_{\mathbb{B}_1}$$

SCALE codec does not encode the correspondence between the value of
$\operatorname{Idx}$ defined in Definition
[\[defn-scale-variable-type\]](#defn-scale-variable-type){reference-type="ref"
reference="defn-scale-variable-type"} and the data type it represents;
the decoder needs prior knowledge of such correspondence to decode the
data.

[\[defn-scale-list\]]{#defn-scale-list label="defn-scale-list"}The
**SCALE codec** for **sequence** $S$ such that: $$S :=A_1, \ldots, A_n$$
where $A_i$'s are values of **the same type** (and the decoder is unable
to infer value of $n$ from the context) is defined as:
$$\operatorname{Enc}_{\operatorname{SC}} (S) :=\operatorname{Enc}^{\operatorname{Len}}_{\operatorname{SC}}
     (\| S \|) \operatorname{Enc}_{\operatorname{SC}} (A_1) | \operatorname{Enc}_{\operatorname{SC}} (A_2) |
     \ldots | \operatorname{Enc}_{\operatorname{SC}} (A_n)$$
where
$\operatorname{Enc}_{\operatorname{SC}}^{\operatorname{Len}}$
is defined in Definition
[\[defn-sc-len-encoding\]](#defn-sc-len-encoding){reference-type="ref"
reference="defn-sc-len-encoding"}. SCALE codec for **dictionary** or
**hashtable** D with key-value pairs $(k_i, v_i)$s such that:
$$D :=\{ (k_1, v_1), \ldots, (k_1, v_n) \}$$ is defined the SCALE codec
of $D$ as a sequence of key value pairs (as tuples):
$$\operatorname{Enc}_{\operatorname{SC}} (D) :=\operatorname{Enc}^{\operatorname{Len}}_{\operatorname{SC}}
     (\| D \|) \operatorname{Enc}_{\operatorname{SC}} ((k_1, v_1)_{}) | \operatorname{Enc}_{\operatorname{SC}}
     ((k_2, v_2)) | \ldots | \operatorname{Enc}_{\operatorname{SC}} ((k_n, v_n))$$
$$\$$

The **SCALE codec** for **boolean value** $b$ defined as a byte as
follows: $$\begin{array}{ll}
       \operatorname{Enc}_{\operatorname{SC}} : & \{ \operatorname{False}, \operatorname{True} \} \rightarrow
       \mathbb{B}_1\\
       & b \rightarrow \left\{ \begin{array}{lcl}
         0 &  & b = \operatorname{False}\\
         1 &  & b = \operatorname{True}
       \end{array} \right.
     \end{array}$$

The **SCALE codec,
$\operatorname{Enc}_{\operatorname{SC}}$** for
other types such as fixed length integers not defined here otherwise, is
equal to little endian encoding of those values defined in Definition
[\[defn-little-endian\]](#defn-little-endian){reference-type="ref"
reference="defn-little-endian"}.

### Length Encoding {#sect-int-encoding}

*SCALE Length encoding* is used to encode integer numbers of variying
sizes prominently in an encoding length of arrays:

[\[defn-sc-len-encoding\]]{#defn-sc-len-encoding
label="defn-sc-len-encoding"}**SCALE Length Encoding,
$\operatorname{Enc}^{\operatorname{Len}}_{\operatorname{SC}}$**
also known as compact encoding of a non-negative integer number $n$ is
defined as follows: $$\begin{array}{ll}
       \operatorname{Enc}^{\operatorname{Len}}_{\operatorname{SC}} : & \mathbb{N} \rightarrow
       \mathbb{B}\\
       & n \rightarrow b :=\left\{ \begin{array}{lll}
         l^{}_1 &  & 0 \leqslant n < 2^6\\
         i^{}_1 i^{}_2 &  & 2^6 \leqslant n < 2^{14}\\
         j^{}_1 j^{}_2 j_3 &  & 2^{14} \leqslant n <
         2^{30}\\
         k_1^{} k_2^{} \ldots k_m^{}  &  & 2^{30}
         \leqslant n
       \end{array} \right.
     \end{array}$$ in where the least significant bits of the first byte
of byte array b are defined as follows: $$\begin{array}{lcc}
       l^1_1 l_1^0 & = & 00\\
       i^1_1 i_1^0 & = & 01\\
       j^1_1 j_1^0 & = & 10\\
       k^1_1 k_1^0 & = & 11
     \end{array}$$ and the rest of the bits of $b$ store the value of
$n$ in little-endian format in base-2 as follows:
$$\left. \begin{array}{lll}
       l^7_1 \ldots l^3_1 l^2_1 &  & n < 2^6\\
       i_2^7 \ldots i_2^0 i_1^7 \ldots i^2_1^{} &  & 2^6 \leqslant n
       < 2^{14}\\
       j_4^7 \ldots j_4^0 j_3^7 \ldots j_1^7 \ldots j^2_1 &  & 2^{14}
       \leqslant n < 2^{30}\\
       k_2 + k_3 2^8 + k_4 2^{2 \cdot 8} + \cdots + k_m 2^{(m - 2) 8} &  &
       2^{30} \leqslant n
     \end{array} \right\} :=n$$ such that:
$$k^7_1 \ldots k^3_1 k^2_1 : = m - 4$$

Frequently SCALEd Object
------------------------

In this section, we will specify the objects which are frequently used
in transmitting data between PDRE,  Runtime and other clients and their
SCALE encodings.

### Result

### Error

Hex Encoding
------------

Practically, it is more convenient and efficient to store and process
data which is stored in a byte array. On the other hand, the Trie keys
are broken into 4-bits nibbles. Accordingly, we need a method to encode
sequences of 4-bits nibbles into byte arrays canonically:

[\[defn-hex-encoding\]]{#defn-hex-encoding
label="defn-hex-encoding"}Suppose that
$\operatorname{PK} = (k_1, \ldots, k_n)$ is a sequence of
nibbles, then

l
$\operatorname{Enc}_{\operatorname{HE}} (\operatorname{PK}) :=$\
$\left\{ \begin{array}{lll}
      \operatorname{Nibbles}_4 & \rightarrow & \mathbb{B}\\
      \operatorname{PK} = (k_1, \ldots, k_n) & \mapsto & \left\{ \begin{array}{l}
        \begin{array}{ll}
          (16 k_1 + k_2, \ldots, 16 k_{2 i - 1} + k_{2 i}) & n = 2 i\\
          (k_1, 16 k_2 + k_3, \ldots, 16 k_{2 i} + k_{2 i + 1}) & n = 2 i + 1
        \end{array}
      \end{array} \right.
    \end{array} \right.$

Genesis Block Specification {#sect-genisis-block}
===========================

Predefined Storage Keys {#sect-predef-storage-keys}
=======================

Network Messages {#sect-network-messages}
================

In this section, we will specify various types of messages which
Polkadot RE receives from the network. Furthermore, we also explain the
appropriate responses to those messages.

A **network message** is a byte array, **$M$** of length $\| M \|$ such
that:

$$\begin{array}{cc}
       M_1 & \operatorname{Message} \operatorname{Type} \operatorname{Indicator}\\
       M_2 \ldots M_{\| M \|} & \operatorname{Enc}_{\operatorname{SC}} (\operatorname{MessageBody})
     \end{array}$$

The body of each message consists of different components based on its
type. The different possible message types are listed below in Table
[\[tabl-message-types\]](#tabl-message-types){reference-type="ref"
reference="tabl-message-types"}. We describe the sub-components of each
message type individually in Section
[10.1](#sect-message-detail){reference-type="ref"
reference="sect-message-detail"}.

   $M_1$        Message Type                                                 Description
  ------- ------------------------- ----------------------------------------------------------------------------------------------
     0             Status                    [10.1.1](#sect-msg-status){reference-type="ref" reference="sect-msg-status"}
     1          Block Request         [10.1.2](#sect-msg-block-request){reference-type="ref" reference="sect-msg-block-request"}
     2         Block Response        [10.1.3](#sect-msg-block-response){reference-type="ref" reference="sect-msg-block-response"}
     3         Block Announce        [10.1.4](#sect-msg-block-announce){reference-type="ref" reference="sect-msg-block-announce"}
     4          Transactions           [10.1.5](#sect-msg-transactions){reference-type="ref" reference="sect-msg-transactions"}
     5            Consensus               [10.1.6](#sect-msg-consensus){reference-type="ref" reference="sect-msg-consensus"}
     6       Remote Call Request    
     7      Remote Call Response    
     8       Remote Read Request    
     9      Remote Read Response    
    10      Remote Header Request   
    11     Remote Header Response   
    12     Remote Changes Request   
    13     Remote Changes Response  
    14      FinalityProofRequest    
    15      FinalityProofResponse   
    255        Chain Specific       

  : [\[tabl-message-types\]]{#tabl-message-types
  label="tabl-message-types"}List of possible network message types.

Detailed Message Structure {#sect-message-detail}
--------------------------

This section disucsses the detailed structure of each network message.

### Status Message {#sect-msg-status}

A *Status* Message represented by $M_S$ is sent after a connection with
a neighbouring node is established and has the following structure:
$$M^{}_S :=\operatorname{Enc}_{\operatorname{SC}} (v, r, N_B, \operatorname{Hash}_B,
   \operatorname{Hash}_G, C_S)$$ Where:

In which, Role is a bitmap value whose bits represent different roles
for the sender node as specified in Table
[\[tabl-node-role\]](#tabl-node-role){reference-type="ref"
reference="tabl-node-role"}:

 

\setlength{\tmfloatwidth}{\widthof{\tmfloatcontents}+1in}
\ifthenelse{\equal{small}{small}}{\setlength{\tmfloatwidth}{0.45\linewidth}}{\setlength{\tmfloatwidth}{\linewidth}}
\captionof{table}{\label{tabl-node-role}Node role representation in the status message.}
### Block Request Message {#sect-msg-block-request}

A Block request message, represented by
$M_{\operatorname{BR}}$, is sent to request block data for
a range of blocks from a peer and has the following structure:
$$M^{}_{\operatorname{BR}} :=\operatorname{Enc}_{\operatorname{SC}} (\operatorname{id}, A_B, S_B,
   \operatorname{Hash}_E, d, \operatorname{Max})$$
where:

 

in which

-   $A_B$, the requested data, is a bitmap value, whose bits represent
    the part of the block data requested, as explained in Table
    [\[tabl-block-attributes\]](#tabl-block-attributes){reference-type="ref"
    reference="tabl-block-attributes"}:

\setlength{\tmfloatwidth}{\widthof{\tmfloatcontents}+1in}
\ifthenelse{\equal{small}{small}}{\setlength{\tmfloatwidth}{0.45\linewidth}}{\setlength{\tmfloatwidth}{\linewidth}}
\captionof{table}{\label{tabl-block-attributes}Bit values for block attribute $A_B$, to
-   $S_B$ is SCALE encoded varying data type (see Definition
    [\[defn-scale-variable-type\]](#defn-scale-variable-type){reference-type="ref"
    reference="defn-scale-variable-type"}) of either $\mathbb{B}_{32}$
    representing the block hash, $H_B$, or
    $64 \operatorname{bit}$ integer representing the block
    number of the starting block of the requested range of blocks.

-   $\operatorname{Hash}_E$ is optionally the block hash of
    the last block in the range.

-   $d$ is a flag; it defines the direction on the block chain where the
    block range should be considered (starting with the starting block),
    as follows $$d = \left\{ \begin{array}{cc}
           0 & \operatorname{child} \operatorname{to} \operatorname{parent} \operatorname{direction}\\
           1 & \operatorname{parent} \operatorname{to} \operatorname{child} \operatorname{direction}
         \end{array} \right.$$

Optional data type is defined in Definition
[\[defn-varrying-data-type\]](#defn-varrying-data-type){reference-type="ref"
reference="defn-varrying-data-type"}.

### Block Response Message {#sect-msg-block-response}

A *block response message* represented by
$M_{\operatorname{BS}}$ is sent in a response to a
requested block message (see Section
[10.1.2](#sect-msg-block-request){reference-type="ref"
reference="sect-msg-block-request"}). It has the following structure:
$$M^{}_{\operatorname{BS}} :=\operatorname{Enc}_{\operatorname{SC}} (\operatorname{id}, D)$$
where:

 

In which block data is defined in Definition
[\[defn-block-data\]](#defn-block-data){reference-type="ref"
reference="defn-block-data"}.

[\[defn-block-data\]]{#defn-block-data label="defn-block-data"}**Block
Data** is defined as the follownig tuple:

$$(H_B, \operatorname{Header}_B, \operatorname{Body}, \operatorname{Receipt}, \operatorname{MessageQueue},
   \operatorname{Justification})$$ Whose elements, with the
exception of $H_B$, are all of the following *optional type* (see
Definition
[\[defn-varrying-data-type\]](#defn-varrying-data-type){reference-type="ref"
reference="defn-varrying-data-type"}) and are defined as follows:

### Block Announce Message {#sect-msg-block-announce}

A *block announce message* represented by
$M_{\operatorname{BA}}$ is sent when a node becomes aware
of a new complete block on the network and has the following structure:
$$M_{\operatorname{BA}} :=\operatorname{Enc}_{\operatorname{SC}} (\operatorname{Header}_B)$$
Where:

### Transactions {#sect-msg-transactions}

   The transactions Message is represented by $M_T$ and is defined as
follows:
$$M_T :=\operatorname{Enc}_{\operatorname{SC}} (C_1, \ldots, C_n)$$
in which:
$$C_i :=\operatorname{Enc}_{\operatorname{SC}} (E_i)$$
Where each $E_i$ is a byte array and represents a sepearate extrinsic.
Polkadot RE is indifferent about the content of an extrinsic and treats
it as a blob of data.

### Consensus Message {#sect-msg-consensus}

A *consensus message* represented by $M_C$ is sent to communicate
messages related to consensus process:
$$M_C :=\operatorname{Enc}_{\operatorname{SC}} (E_{\operatorname{id}}, D)$$
Where:

in which
$$E_{\operatorname{id}} :=\left\{ \begin{array}{ccc}
     '' \operatorname{BABE}'' &  & \operatorname{For} \operatorname{messages} \operatorname{related} \operatorname{to}
     \operatorname{BABE} \operatorname{protocol}\\
     '' \operatorname{FRNK}'' &  & \operatorname{For} \operatorname{messages} \operatorname{related} \operatorname{to}
     \operatorname{GRANDPA} \operatorname{protocol}
   \end{array} \right.$$

The network agent should hand over $D$ to approperiate consensus engine
which identified by $E_{\operatorname{id}}$.

Runtime Environment API[\[sect-re-api\]]{#sect-re-api label="sect-re-api"}
==========================================================================

The Runtime Environment API is a set of functions that Polkadot RE
exposes to Runtime to access external functions needed for various
reasons, such as the Storage of the content, access and manipulation,
memory allocation, and also efficiency. We introduce Notation
[\[nota-re-api-at-state\]](#nota-re-api-at-state){reference-type="ref"
reference="nota-re-api-at-state"} to emphasize that the result of some
of the API functions depends on the content of state storage.

[\[nota-re-api-at-state\]]{#nota-re-api-at-state
label="nota-re-api-at-state"}By $\mathcal{R}\mathcal{E}_B$ we refer to
the API exposed by Polkadot RE which interact, manipulate and response
based on the state storage whose state is set at the end of the
execution of block $B$.

The functions are specified in each subsequent subsection for each
category of those functions.

Storage
-------

### 

Sets the value of a specific key in the state storage.

**Prototype:**

    (func $ext_storage
      (param $key_data i32) (param $key_len i32) (param $value_data i32)                           (param $value_len i32))

**Arguments**:

-   : a pointer indicating the buffer containing the key.

-   : the key length in bytes.

-   : a pointer indicating the buffer containing the value to be stored
    under the key.

-   :  the length of the value buffer in bytes.

### 

Retrieves the root of the state storage.

 

**Prototype:**

    (func $ext_storage_root
      (param $result_ptr i32))

**Arguments**:

-   : a memory address pointing at a byte array which contains the root
    of the state storage after the function concludes.

#### 

Given an array of byte arrays, it arranges them in a Merkle trie,
defined in Section [2.1.4](#sect-merkl-proof){reference-type="ref"
reference="sect-merkl-proof"}, where the key under which the values are
stored is the 0-based index of that value in the array. It computes and
returns the root hash of the constructed trie.

 

**Prototype:**

    (func $ext_blake2_256_enumerated_trie_root
          (param $values_data i32) (param $lens_data i32) (param $lens_len i32) 
          (param $result i32))

**Arguments**:

-   : a memory address pointing at the buffer containing the array where
    byte arrays are stored consecutively.

-   : an array of elements each stores the length of each byte array
    stored in .

-   s\_len: the number of elements in .

-   : a memory address pointing at the beginning of a 32-byte byte array
    containing the root of the Merkle trie corresponding to elements of
    .

### 

Given a byte array, this function removes all storage entries whose key
matches the prefix specified in the array.

 

**Prototype:**

    (func $ext_clear_prefix
          (param $prefix_data i32) (param $prefix_len i32))

**Arguments**:

-   : a memory address pointing at the buffer containing the byte array
    containing the prefix.

-   : the length of the byte array in number of bytes.

### 

Given a byte array, this function removes the storage entry whose key is
specified in the array.

 

**Prototype:**

    (func $ext_clear_storage
          (param $key_data i32) (param $key_len i32))

**Arguments**:

-   : a memory address pointing at the buffer containing the byte array
    containing the key value.

-   : the length of the byte array in number of bytes.

#### 

Given a byte array, this function checks if the storage entry
corresponding to the key specified in the array exists.

 

**Prototype:**

    (func $ext_exists_storage
          (param $key_data i32) (param $key_len i32) (result i32)
        )

**Arguments**:

-   : a memory address pointing at the buffer containing the byte array
    containing the key value.

-   : the length of the byte array in number of bytes.

-   : An integer which is equal to 1 verifies if an entry with the given
    key exists in the storage or 0 if the key storage does not contain
    an entry with the given key.

### 

Given a byte array, this function allocates a large enough buffer in the
memory and retrieves the value stored under the key that is specified in
the array. Then, it stores it in the allocated buffer if the entry
exists in the storage.

 

**Prototype:**

        (func $get_allocated_storage
          (param $key_data i32) (param $key_len i32) (param $written_out i32) (result i32))

**Arguments**:

-   : a memory address pointing at the buffer containing the byte array
    containing the key value.

-   : the length of the byte array in number of bytes.

-   : the function stores the length of the retrieved value in number of
    bytes if the enty exists. If the entry does not exist, it returns
    $2^{32} - 1$.

-   : A pointer to the buffer in which the function allocates and stores
    the value corresponding to the given key if such an entry exist;
    otherwise it is equal to 0.

### 

Given a byte array, this function retrieves the value stored under the
key specified in the array and stores a specified chunk of it in the
provided buffer, if the entry exists in the storage.

 

**Prototype:**

        (func $ext_get_storage_into 
          (param $key_data i32) (param $key_len i32) (param $value_data i32)
          (param $value_len i32) (param $value_offset i32) (result i32))

**Arguments**:

-   : a memory address pointing at the buffer containing the byte array
    containing the key value.

-   : the length of the byte array in number of bytes.

-   : a pointer to the buffer in which the function stores the chunk of
    the value it retrieves.

-   : the (maximum) length of the chunk in bytes the function will read
    of the value and will store in the buffer.

-   : the offset of the chunk where the function should start storing
    the value in the provided buffer, i.e. the number of bytes the
    functions should skip from the retrieved value before storing the
    data in the in number of bytes.

-   : The number of bytes the function writes in if the value exists or
    $2^{32} - 1$ if the entry does not exist under the specified key.

### To Be Specced

-   -   -   -   -   -   -   

### Memory

#### 

Allocates memory of a requested size in the heap.

 

**Prototype**:

    (func $ext_malloc
      (param $size i32) (result i32))

**Arguments**:

-   the size of the buffer to be allocated in number of bytes.

**Result**:

a memory address pointing at the beginning of the allocated buffer.

#### 

Deallocates a previously allocated memory.

 

**Prototype**:

    (func $ext_free
          (param $addr i32))

**Arguments:**

-   : a 32bit memory address pointing at the allocated memory.

#### Input/Output

-   -   -   

### Cryptograhpic Auxiliary Functions

#### 

Computes the Blake2b 256bit hash of a given byte array.

 

**Prototype:**

    (func (export "ext_blake2_256")
          (param $data i32) (param  $len i32) (param $out i32))

**Arguments**:

-   : a memory address pointing at the buffer containing the byte array
    to be hashed.

-   : the length of the byte array in bytes.

-   : a memory address pointing at the beginning of a 32-byte byte array
    contanining the Blake2b hash of the data.

#### 

Computes the Keccak-256 hash of a given byte array.

 

**Prototype:**

    (func $ext_keccak_256
          (param $data i32) (param $len i32) (param $out i32))

**Arguments**:

-   : a memory address pointing at the buffer containing the byte array
    to be hashed.

-   : the length of the byte array in bytes.

-   : a memory address pointing at the beginning of a 32-byte byte array
    contanining the Keccak-256 hash of the data.

#### 

Computes the *xxHash64* algorithm (see [@collet_extremely_2019]) twice
initiated with seeds 0 and 1 and applied on a given byte array and
outputs the concatenated result.

 

**Prototype:**

    (func $ext_twox_128
           (param $data i32) (param $len i32) (param $out i32))

**Arguments**:

-   : a memory address pointing at the buffer containing the byte array
    to be hashed.

-   : the length of the byte array in bytes.

-   : a memory address pointing at the beginning of a 16-byte byte array
    containing  *$\text{xxhash}
      64_0$*()() where *$\text{xxhash} 64_i$* is the xxhash64 function
    initiated with seed $i$ as a 64bit unsigned integer.

#### 

Given a message signed by the ED25519 signature algorithm alongside with
its signature and the allegedly signer public key, it verifies the
validity of the signature by the provided public key.

 

**Prototype:**

    (func $ext_ed25519_verify
          (param $msg_data i32) (param $msg_len i32) (param $sig_data i32)
          (param $pubkey_data i32) (result i32))

**Arguments**:

-   : a pointer to the buffer containing the message body.

-   : an integer indicating the size of the message buffer in bytes.

-   : a pointer to the 64 byte memory buffer containing the ED25519
    signature corresponding to the message.

-   : a pointer to the 32 byte buffer containing the public key and
    corresponding to the secret key which has signed the message.

-   :  an integer value equal to 0 indicating the validity of the
    signature or a nonzero value otherwise.

#### 

Given a message signed by the SR25519 signature algorithm alongside with
its signature and the allegedly signer public key, it verifies the
validity of the signature by the provided public key.

 

**Prototype:**

    (func $ext_sr25519_verify
          (param $msg_data i32) (param $msg_len i32) (param $sig_data i32)
          (param $pubkey_data i32) (result i32))

**Arguments**:

-   : a pointer to the buffer containing the message body.

-   : an integer indicating the size of the message buffer in bytes.

-   : a pointer to the 64 byte memory buffer containing the SR25519
    signature corresponding to the message.

-   : a pointer to the 32 byte buffer containing the public key and
    corresponding to the secret key which has signed the message.

-   :  an integer value equal to 0 indicating the validity of the
    signature or a nonzero value otherwise.

#### To be Specced

-   

### Offchain Worker 

The Offchain Workers allow the execution of long-running and possibly
non-deterministic tasks (e.g. web requests, encryption/decryption and
signing of data, random number generation, CPU-intensive computations,
enumeration/aggregation of on-chain data, etc.) which could otherwise
require longer than the block execution time. Offchain Workers have
their own execution environment. This separation of concerns is to make
sure that the block production is not impacted by the long-running
tasks.

 

As Offchain Workers run on their own execution environment they have
access to their separate storage. There are two different types of
storage available as defined in Definitions
[\[defn-offchain-persistent-storage\]](#defn-offchain-persistent-storage){reference-type="ref"
reference="defn-offchain-persistent-storage"} and
[\[defn-offchain-local-storage\]](#defn-offchain-local-storage){reference-type="ref"
reference="defn-offchain-local-storage"}.

[\[defn-offchain-persistent-storage\]]{#defn-offchain-persistent-storage
label="defn-offchain-persistent-storage"}**Persistent**storage**** is
non-revertible and not fork-aware. It means that any value set by the
offchain worker is persisted even if that block (at which the worker is
called) is reverted as non-canonical (meaning that the block was
surpassed by a longer chain). The value is available for the worker that
is re-run at the new (different block with the same block number) and
future blocks. This storage can be used by offchain workers to handle
forks and coordinate offchain workers running on different forks.

[\[defn-offchain-local-storage\]]{#defn-offchain-local-storage
label="defn-offchain-local-storage"}**Local storage** is revertible and
fork-aware. It means that any value set by the offchain worker triggered
at a certain block is reverted if that block is reverted as
non-canonical. The value is NOT available for the worker that is re-run
at the next or any future blocks.

[\[defn-http-return-value\]]{#defn-http-return-value
label="defn-http-return-value"}**HTTP status codes** that can get
returned by certain Offchain HTTP functions.

-   **0**: the specified request identifier is invalid.

-   **10**: the deadline for the started request was reached.

-   **20**: an error has occurred during the request, e.g. a timeout or
    the remote server has closed the connection. On returning this error
    code, the request is considered destroyed and must be reconstructed
    again.

-   **100**..**999**: the request has finished with the given HTTP
    status code.

 

#### 

Returns if the local node is a potential validator. Even if this
function returns 1, it does not mean that any keys are configured and
that the validator is registered in the chain.

 

**Prototype:**

    (func $ext_is_validator
          (result i32))

**Arguments**:

-   :  an i32 integer which is equal to 1 if the local node is a
    potential validator or a equal to 0 if it is not.

#### 

Given an extrinsic as a SCALE encoded byte array, the system decodes the
byte array and submits the extrinsic in the inherent pool as an
extrinsic to be included in the next produced block.

 

**Prototype:**

    (func $ext_submit_transaction
          (param $data i32) (param $len i32) (result i32))

**Arguments**:

-   : a pointer to the buffer containing the byte array storing the
    encoded extrinsic.

-   : an integer indicating the size of the encoded extrinsic.

-   : an integer value equal to 0 indicates that the extrinsic is
    successfully added to the pool or a nonzero value otherwise.

#### 

Returns opaque information about the local node's network state.

 

**Prototype:**

    (func $ext_network_state
          (param $written_out i32)(result i32))

**Arguments**:

-   : a pointer to the 4-byte buffer where the size of the opaque
    network state gets written to.

-   : a pointer to the buffer containing the SCALE encoded network
    state.

#### 

Returns current timestamp.

 

**Prototype:**

    (func $ext_timestamp
          (result i64))

**Arguments**:

-   : an i64 integer indicating the current UNIX timestamp as defined in
    Definition
    [\[defn-unix-time\]](#defn-unix-time){reference-type="ref"
    reference="defn-unix-time"}.

#### 

Pause the execution until 'deadline' is reached.

 

**Prototype:**

    (func $ext_sleep_until
          (param $deadline i64))

**Arguments**:

-   : an i64 integer specifying the UNIX timestamp as defined in
    Definition
    [\[defn-unix-time\]](#defn-unix-time){reference-type="ref"
    reference="defn-unix-time"}.

#### 

Generates a random seed. This is a truly random non deterministic seed
generated by the host environment.

 

**Prototype:**

    (func $ext_random_seed
          (param $seed_data i32))

**Arguments**:

-   : a memory address pointing at the beginning of a 32-byte byte array
    containing the generated seed.

#### 

Sets a value in the local storage. This storage is not part of the
consensus, it's only accessible by the offchain worker tasks running on
the same machine and is persisted between runs.

 

**Prototype:**

    (func $ext_local_storage_set
          (param $kind i32) (param $key i32) (param $key_len i32)
          (param $value i32) (param $value_len i32))

**Arguments**:

-   : an i32 integer indicating the storage kind. A value equal to 1 is
    used for a persistent storage as defined in Definition
    [\[defn-offchain-persistent-storage\]](#defn-offchain-persistent-storage){reference-type="ref"
    reference="defn-offchain-persistent-storage"} and a value equal to 2
    for local storage as defined in Definition
    [\[defn-offchain-local-storage\]](#defn-offchain-local-storage){reference-type="ref"
    reference="defn-offchain-local-storage"}.

-   : a pointer to the buffer containing the key.

-   : an i32 integer indicating the size of the key.

-   : a pointer to the buffer containg the value.

-   : an i32 integer indicating the size of the value.

#### 

Sets a new value in the local storage if the condition matches the
current value.

 

**Prototype:**

    (func $ext_local_storage_compare_and_set 
          (param $kind i32) (param $key i32) (param $key_len i32)
          (param $old_value i32) (param $old_value_len) (param $new_value i32)
          (param $new_value_len) (result i32))

**Arguments**:

-   : an i32 integer indicating the storage kind. A value equal to 1 is
    used for a persistent storage as defined in Definition
    [\[defn-offchain-persistent-storage\]](#defn-offchain-persistent-storage){reference-type="ref"
    reference="defn-offchain-persistent-storage"} and a value equal to 2
    for local storage as defined in Definition
    [\[defn-offchain-local-storage\]](#defn-offchain-local-storage){reference-type="ref"
    reference="defn-offchain-local-storage"}.

-   : a pointer to the buffer containing the key.

-   : an i32 integer indicating the size of the key.

-   : a pointer to the buffer containing the current value.

-   : an i32 integer indicating the size of the current value.

-   : a pointer to the buffer containing the new value.

-   : an i32 integer indicating the size of the new value.

-   : an i32 integer equal to 0 if the new value has been set or a value
    equal to 1 if otherwise.

#### 

Gets a value from the local storage.

 

**Prototype:**

    (func $ext_local_storage_set
          (param $kind i32) (param $key i32) (param $key_len i32)
          (param $value_len i32) (result i32))

**Arguments**:

-   : an i32 integer indicating the storage kind. A value equal to 1 is
    used for a persistent storage as defined in Definition
    [\[defn-offchain-persistent-storage\]](#defn-offchain-persistent-storage){reference-type="ref"
    reference="defn-offchain-persistent-storage"} and a value equal to 2
    for local storage as defined in Definition
    [\[defn-offchain-local-storage\]](#defn-offchain-local-storage){reference-type="ref"
    reference="defn-offchain-local-storage"}.

-   : a pointer to the buffer containing the key.

-   : an i32 integer indicating the size of the key.

-   : an i32 integer indicating the size of the value.

-   : a pointer to the buffer in which the function allocates and stores
    the value corresponding to the given key if such an entry exist;
    otherwise it is equal to 0.

#### 

Initiates a http request given by the HTTP method and the URL. Returns
the id of a newly started request.

 

**Prototype:**

    (func $ext_http_request_start
          (param $method i32) (param $method_len i32) (param $url i32)
          (param $url_len i32) (param $meta i32) (param $meta_len i32) (result i32))

**Arguments**:

-   : a pointer to the buffer containing the key.

-   : an i32 integer indicating the size of the method.

-   : a pointer to the buffer containing the url.

-   : an i32 integer indicating the size of the url.

-   : a future-reserved field containing additional, SCALE encoded
    parameters.

-   : an i32 integer indicating the size of the parameters.

-   : an i32 integer indicating the ID of the newly started request.

#### 

Append header to the request. Returns an error if the request identifier
is invalid, has already been called on the specified request identifier,
the deadline is reached or an I/O error has happened (e.g. the remote
has closed the connection).

 

**Prototype:**

    (func $ext_http_request_add_header
          (param $request_id i32) (param $name i32) (param $name_len i32)
          (param $value i32) (param $value_len i32) (result i32))

**Arguments**:

-   : an i32 integer indicating the ID of the started request.

-   : a pointer to the buffer containing the header name.

-   : an i32 integer indicating the size of the header name.

-   : a pointer to the buffer containing the header value.

-   : an i32 integer indicating the size of the header value.

-   : an i32 integer where the value equal to 0 indicates if the header
    has been set or a value equal to 1 if otherwise.

#### 

Writes a chunk of the request body. Writing an empty chunk finalises the
request. Returns a non-zero value in case the deadline is reached or the
chunk could not be written.

 

**Prototype:**

    (func $ext_http_request_write_body
          (param $request_id i32) (param $chunk i32) (param $chunk_len i32)
          (param $deadline i64) (result i32))

**Arguments**:

-   : an i32 integer indicating the ID of the started request.

-   : a pointer to the buffer containing the chunk.

-   : an i32 integer indicating the size of the chunk.

-   : an i64 integer specifying the UNIX timestamp as defined in
    Definition
    [\[defn-unix-time\]](#defn-unix-time){reference-type="ref"
    reference="defn-unix-time"}. Passing '0' will block indefinitely.

-   : an i32 integer where the value equal to 0 indicates if the header
    has been set or a non-zero value if otherwise.

#### 

Blocks and waits for the responses for given requests. Returns an array
of request statuses (the size is the same as number of IDs).

 

**Prototype:**

    (func $ext_http_response_wait
          (param $ids i32) (param $ids_len i32) (param $statuses i32)
          (param $deadline i64))

**Arguments**:

-   : a pointer to the buffer containing the started IDs.

-   : an i32 integer indicating the size of IDs.

-   : a pointer to the buffer where the request statuses get written to
    as defined in Definition
    [\[defn-http-return-value\]](#defn-http-return-value){reference-type="ref"
    reference="defn-http-return-value"}. The lenght is the same as the
    length of .

-   : an i64 integer indicating the UNIX timestamp as defined in
    Definition
    [\[defn-unix-time\]](#defn-unix-time){reference-type="ref"
    reference="defn-unix-time"}. Passing '0' as deadline will block
    indefinitely.

#### 

Read all response headers. Returns a vector of key/value pairs. Response
headers must be read before the response body.

 

**Prototype:**

    (func $ext_http_response_headers
          (param $request_id i32) (param $written_out i32) (result i32))

**Arguments**:

-   : an i32 integer indicating the ID of the started request.

-   : a pointer to the buffer where the size of the response headers
    gets written to.

-   : a pointer to the buffer containing the response headers.

#### 

Reads a chunk of body response to the given buffer. Returns the number
of bytes written or an error in case a deadline is reached or the server
closed the connection. If '0' is returned it means that the response has
been fully consumed and the is now invalid. This implies that response
headers must be read before draining the body.

 

**Prototype:**

    (func $ext_http_response_read_body
          (param $request_id i32) (param $buffer i32) (param $buffer_len)
          (param $deadline i64) (result i32))

**Arguments**:

-   : an i32 integer indicating the ID of the started request.

-   : a pointer to the buffer where the body gets written to.

-   : an i32 integer indicating the size of the buffer.

-   : an i64 integer indicating the UNIX timestamp as defined in
    Definition
    [\[defn-unix-time\]](#defn-unix-time){reference-type="ref"
    reference="defn-unix-time"}. Passing '0' will block indefinitely.

-   : an i32 integer where the value equal to 0 indicateds a fully
    consumed response or a non-zero value if otherwise.

### Sandboxing

#### To be Specced

-   -   -   -   -   -   -   

### Auxillary Debugging API

#### 

Prints out the content of the given buffer on the host's debugging
console. Each byte is represented as a two-digit hexadecimal number.

 

**Prototype:**

        (func $ext_print_hex
          (param $data i32) (parm $len i32))

**Arguments**:

-   : a pointer to the buffer containing the data that needs to be
    printed.

-   : an integer indicating the size of the buffer containing the data
    in bytes.

#### 

Prints out the content of the given buffer on the host's debugging
console. The buffer content is interpreted as a UTF-8 string if it
represents a valid UTF-8 string, otherwise does nothing and returns.

**Prototype:**o

        (func $ext_print_utf8
          (param $utf8_data i32) (param $utf8_len i32))

**Arguments**:

-   : a pointer to the buffer containing the utf8-encoded string to be
    printed.

-   : an integer indicating the size of the buffer containing the UTF-8
    string in bytes.

### Misc

#### To be Specced

-   

### Block Production {#block-production-1}

Validation
----------

 

Runtime Entries {#sect-runtime-entries}
===============

List of Runtime Entries {#sect-list-of-runtime-entries}
-----------------------

Polkadot RE assumes that at least the following functions are
implemented in the Runtime Wasm blob and have been exported as shown in
Snippet
[\[snippet-runtime-enteries\]](#snippet-runtime-enteries){reference-type="ref"
reference="snippet-runtime-enteries"}:

\setlength{\tmfloatwidth}{\widthof{\tmfloatcontents}+1in}
\ifthenelse{\equal{small}{small}}{\setlength{\tmfloatwidth}{0.45\linewidth}}{\setlength{\tmfloatwidth}{\linewidth}}
      (export "Core_version" (func $Core_version))
      (export "Core_execute_block" (func $Core_execute_block))
      (export "Core_initialize_block" (func $Core_initialize_block))
      (export "Metadata_metadata" (func $Metadata_metadata))
      (export "BlockBuilder_apply_extrinsic" (func $BlockBuilder_apply_extrinsic))
      (export "BlockBuilder_finalize_block" (func $BlockBuilder_finalize_block))
      (export "BlockBuilder_inherent_extrinsics" 
              (func $BlockBuilder_inherent_extrinsics))
      (export "BlockBuilder_check_inherents" (func $BlockBuilder_check_inherents))
      (export "BlockBuilder_random_seed" (func $BlockBuilder_random_seed))
      (export "TaggedTransactionQueue_validate_transaction" 
              (func $TaggedTransactionQueue_validate_transaction))
      (export "OffchainWorkerApi_offchain_worker" 
              (func $OffchainWorkerApi_offchain_worker))
      (export "ParachainHost_validators" (func $ParachainHost_validators))
      (export "ParachainHost_duty_roster" (func $ParachainHost_duty_roster))
      (export "ParachainHost_active_parachains" 
              (func $ParachainHost_active_parachains))
      (export "ParachainHost_parachain_status" (func $ParachainHost_parachain_status))
      (export "ParachainHost_parachain_code" (func $ParachainHost_parachain_code))
      (export "ParachainHost_ingress" (func $ParachainHost_ingress))
      (export "GrandpaApi_grandpa_pending_change" 
              (func $GrandpaApi_grandpa_pending_change))
      (export "GrandpaApi_grandpa_forced_change" 
              (func $GrandpaApi_grandpa_forced_change))
      (export "GrandpaApi_grandpa_authorities" (func $GrandpaApi_grandpa_authorities))
      (export "BabeApi_startup_data" (func $BabeApi_startup_data))
      (export "BabeApi_epoch" (func $BabeApi_epoch))
      (export "SessionKeys_generate_session_keys" 
              (func $SessionKeys_generate_session_keys))

\captionof{figure}{\label{snippet-runtime-enteries}Snippet to export entries into
The following sections describe the standard based on which Polkadot RE
communicates with each runtime entry.

Argument Specification
----------------------

As a wasm functions, all runtime entries have the following prototype
signature:

        (func $generic_runtime_entry
          (param $data i32) (parm $len i32) (reslut i64))

where points to the SCALE encoded paramaters sent to the function and is
the length of the data. can similarly either point to the SCALE encoded
data the function returns or represent a boolean value (See Sections
[3.1.2.2](#sect-runtime-send-args-to-runtime-enteries){reference-type="ref"
reference="sect-runtime-send-args-to-runtime-enteries"} and
[3.1.2.3](#sect-runtime-return-value){reference-type="ref"
reference="sect-runtime-return-value"}).

In this section, we describe the function of each of the entries
alongside with the details of the SCALE encoded arguments and the return
values for each one of these enteries.

### 

This entry receives no argument; it returns the version data encoded in
ABI format described in Section
[3.1.2.3](#sect-runtime-return-value){reference-type="ref"
reference="sect-runtime-return-value"} containing the following
information:

 

\setlength{\tmfloatwidth}{\widthof{\tmfloatcontents}+1in}
\ifthenelse{\equal{small}{small}}{\setlength{\tmfloatwidth}{0.45\linewidth}}{\setlength{\tmfloatwidth}{\linewidth}}
  Name   Type      Description
  ------ --------- -------------------------------------------
         String    Runtime identifier
         String    the name of the implementation (e.g. C++)
         UINT32    the version of the authorship interface
         UINT32    the version of the Runtime specification
         UINT32    the version of the Runtime implementation
         ApisVec   List of supported AP

\captionof{table}{Detail of the version data type returns from runtime{\ttfamily{version}} function.}
### 

This entry is responsible for executing all extrinsics in the block and
reporting back if the block was successfully executed.

**Arguments**:

-   The entry accepts the *block data* defined in Definition
    [\[defn-block-data\]](#defn-block-data){reference-type="ref"
    reference="defn-block-data"} as the only argument.

**Return**:

A Boolean value indicates if the execution was successful.

### 

###  {#sect-rte-hash-and-length}

An auxilarry function which returns hash and encoding length of an
extrinsics.

**Arguments**:

-   A SCALE encoded blob of an extrinsic.

**Return**:

Pair of Blake2Hash of the blob as element of $\mathbb{B}_{32}$ and its
length as 64 bit integer.

###  {#sect-rte-babeapi-epoch}

This entry is called to obtain the current configuration of BABE
consensus protocol.

**Arguments**:

-   $H_n (B)$: the block number at whose final state the epoch
    configuration should be obtained.

**Return**:

A tuple
$$(\mathcal{E}_n, s^n_0, \operatorname{sc}_n, A, \rho, \operatorname{Sec})$$

where:

in which:

###  {#sect-rte-grandpa-auth}

This entry is to report the set of GRANDPA voters at a given block. It
receives as an argument; it returns an array of 's.

###  {#sect-rte-validate-transaction}

This entry is invoked against extrinsics submitted through the
Transaction network message
[10.1.5](#sect-msg-transactions){reference-type="ref"
reference="sect-msg-transactions"} and indicates if the submitted blob
represents a valid extrinsics applied to the specified block.

**Arguments**:

-   $H_n (B)$: the block number whose final state is where the
    transaction should apply the system state.

-   UTX: A byte array that contains the SCALE encoded transaction.

**Return**:

A varying type Result object which has type of *TransactionValidity* in
case no error occurs in course of its execution. TransactionValidity is
of varying type described in the Table
[\[tabl-transaction-validity\]](#tabl-transaction-validity){reference-type="ref"
reference="tabl-transaction-validity"}:

 

\setlength{\tmfloatwidth}{\widthof{\tmfloatcontents}+1in}
\ifthenelse{\equal{small}{small}}{\setlength{\tmfloatwidth}{0.45\linewidth}}{\setlength{\tmfloatwidth}{\linewidth}}
  ------------ -------------- ----------------------------------------------------------------------
  Type Index   Data type      Description
  0            Byte           Indicating invalid extrinsic and bearing the error code concerning
                              the cause of invalidity of the transaction.
  1            A Quin-tuple   Indicating whether the extrinsic is valid and providing guidance for
                              Polkadot RE on how to proceed with the extrinsic (see below)
  2            Byte           The Validity of the extrinsic cannot be determined
  ------------ -------------- ----------------------------------------------------------------------

\captionof{table}{\label{tabl-transaction-validity}Type variation for the return
 

In which the quintuple of type for valid extrinsics consists of the
following parts:
$$(\operatorname{priority}, \operatorname{requires}, \operatorname{provides}, \operatorname{longevity},
   \operatorname{propagate})$$

\setlength{\tmfloatwidth}{\widthof{\tmfloatcontents}+1in}
\ifthenelse{\equal{small}{small}}{\setlength{\tmfloatwidth}{0.45\linewidth}}{\setlength{\tmfloatwidth}{\linewidth}}
  ----------- -------------------------------------------------------------------------- ------------------
  Name        Description                                                                Type
  Priority    Determines the ordering of two transactions that have                      64bit integer
              all their dependencies (required tags) satisfied.                          
  Requires    List of tags specifying extrinsics which should be applied                 Array of
              before the current exrinsics can be applied.                               Transaction Tags
  Provides    Informs Runtime of the extrinsics depending on the tags in                 Array of
              the list that can be applied after current extrinsics are being applied.   Transaction Tags
              Describes the minimum number of blocks for the validity to be correct      
  Longevity   After this period, the transaction should be removed from the              64 bit integer
              pool or revalidated.                                                       
  Propagate   A flag indicating if the transaction should be propagated to               Boolean
              other peers.                                                               
  ----------- -------------------------------------------------------------------------- ------------------

\captionof{table}{The quintuple provided by{\ttfamily{TaggedTransactionQueue\_transaction\_validity}}
 

Note that if *Propagate* is set to the transaction will still be
considered for including in blocks that are authored on the current
node, but will never be sent to other peers.

DGKR18 Yann Collet. Extremely fast non-cryptographic hash algorithm.
Technical Report, -, <http://cyan4973.github.io/xxHash/>, 2019.

Bernardo David, Peter Gaži, Aggelos Kiayias, and Alexander Russell.
Ouroboros praos: An adaptively-secure, semi-synchronous proof-of-stake
blockchain. In *Annual International Conference on the Theory and
Applications of Cryptographic Techniques*, pages 66--98. Springer, 2018.

W3F Research Group. Blind Assignment for Blockchain Extension. Technical
, Web 3.0 Foundation,
<http://research.web3.foundation/en/latest/polkadot/BABE/Babe/>, 2019.

Protocol labs. Libp2p Specification. Technical Report, Protocol labs,
<https://github.com/libp2p/specs>, 2019.

Ilari Liusvaara and Simon Josefsson. Edwards-Curve Digital Signature
Algorithm (EdDSA). 2017.

Trevor Perrin. The Noise Protocol Framework. Technical Report,
<https://noiseprotocol.org/noise.html>, 2018.

Markku Juhani Saarinen and Jean-Philippe Aumasson. The BLAKE2
cryptographic hash and message authentication code (MAC). 7693, -,
<https://tools.ietf.org/html/rfc7693>, 2015.

Alistair Stewart. GRANDPA: A Byzantine Finality Gadgets. 2019.

Parity Technologies. Substrate Reference Documentation. Rust , Parity
Technologies, <https://substrate.dev/rustdocs/>, 2019.

\printindex
