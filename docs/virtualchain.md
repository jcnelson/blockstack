# Virtualchain

Virtualchain is a Python library that lets a distributed application implement replicated state machine logic
that uses pre-existing blockchain to host its input history.  This is useful for building **decentralized 
open-membership applications**, where anyone can run a node, but all correct nodes independently reach
agreement on state as long as they can read the shared blockchain.

## Why use a blockchain for RSMs?

Replicated state machines are not a new concept.  Traditionally, the RSM design pattern is used for reliability.
Instead of relying on a single faulty process for maintaining state, an application
can rely on a quorum of faulty processes that, in aggregate, are more reliable than a single
process and (with the right protocols) can resist Byzantine failures in up to *f* processes out of
*3f+1* total.

The problem we're interested in solving is building **open-membership** RSMs.  Anyone should be able
to join or leave the RSM process group at will and help maintain state, *even if they cannot synchronously
communicate with other processes* (i.e. they could be behind a NAT or a firewall).  At the same time,
all processes in the RSM should be able to reach consensus on state, regardless of how many or few of them
are online.

This is extremely difficult to do with conventional RSM designs, since conventional designs depend on a quorum of participants
admitting a new process or kicking out an existing process.  Moreover, in order to do so across
untrusted networks, conventional RSM processes additionally need a way to bootstrap trust
in one another (i.e. to communicate securely) and to survive unavailability, partitions, and
key compromises.

Blockchains--in particular, proof-of-work blockchains--have the useful property that anyone can write
to them (i.e. by submitting a transaction) while also guaranteeing that there is only one valid
transaction history.  By using a blockchain as an "input tape" shared by all RSM processes, we can
build open-membership RSMs where processes interact only by reading and writing to the blockchain.  The blockchain
provides the "ground truth" the order and history of submitted inputs, and each correct
RSM process interprets the input sequence to independently derive the same current state.

This use-case yields two important properties.  First, this **decouples liveness from application availability** since
as long as the *blockchain* is available, a correct process can both independently derive the latest state and submit new inputs without
communicating with other processes.  Second, this **decouples security from trust in applications** since the authenticity of
the input history is maintained by the blockchain's proof-of-work, instead of trust-by-decree in a set of application processes.
Only an attack on the blockchain itself (which is computationally expensive and easy to notice) can alter the application state.

## Background Theory

The high-level concept behind Virtualchain is that we can treat a blockchain as a fully-replicated linearizable log, where
each node usually sees the same order of writes (i.e. "transactions") after a certain number of leader-election rounds (i.e.
"confirmations").  We say "usually" since the consensus algorithm behind most blockchains admits multiple leaders,
which leads to blockchain *forks*, in which two or more sets of nodes see diverging write histories.

The proof-of-work (PoW) scheme in Bitcoin, Ethereum, and others gives the system
the ability to resolve these forks.  By selecting the longest valid fork with the most proof-of-work, nodes eventually agree
on the same write history.  In addition, PoW gives the system the
property that probability of a fork lasting longer than *K* rounds decreases exponentially with *K*
when the set of possible leaders (i.e. "miners") contains only honest nodes and there are no 
long-lasting network partitions.

In effect, the write history in a PoW blockchain after *K* blocks can be considered stable with 
high probability.  However, since the blockchain isn't aware of the application-specific consensus
rules, and since anyone (including other applications and malicious actors) can write transactions,
the application needs a way to **multiplex the blockchain** so that it will **select only its inputs**
and process them **in the same order** in each process.  This is where virtualchain comes in.

Virtualchain lets an application write a *fork * -consistent* input log to the blockchain,
which can then be fed into any of its processes.  Each process replays this input log to reach the
same state as its peers.

Fork * -consistency is described by [Li and Mazieres](https://www.scs.stanford.edu/~jinyuan/bft2f.pdf),
where a fork * -consistent RSM has the following properties:

* **Legitimate-request Property**:  *Every result accepted by a correct application process has a result list *L*
that contains only well-formed operations legitimately issued by other processes.*  Virtualchain allows the
application to define its own consensus rules in order to determine which transaction is legitimate.

* **Self-consistency Property**:  *Each honest process sees all past operations from itself.  That is,
an honest process sees all the past operations it committed in the order in which it committed them.*  
In addition, the blockchain guarantees that with high probability, all processes see the same history of past operations from all 
other processes up to the last *K* blocks.

* **Join-at-most-once Property**:  *If two result lists accepted by correct processes contain operations
*op'* and *op* from the same correct process, then both result lists will have the operations in the
same order.  If *op'* precedes *op*, then both result lists are identical up to *op'*.*  

By reading the same blockchain up to the last *K* blocks, correct processes will process the same operations
in the same order.  Applications choose *K* to select their desired security/liveness trade-off:  higher values
 of *K* decrease the probability of encountering a fork while also making inputs take longer to be processed.

### Possible Failure Modes

The catastrophic failure modes with a virtualchain-powered RSM are (1)
the blockchain encountering a "deep" fork lasting longer than *K* blocks, and (2)
the blockchain being successfully attacked.

We offer applications one of three recovery options in the first case:

* Processes with invalid state should revert to an earlier state and re-calculate the current state 
from the input history in the post-fork blockchain.  This is how [Blockstack](https://blockstack.org) works, for example.

* The application developer issues a "checkpoint" to declare after a blockchain fork what the resulting
application state ought to be post-fork.  Other processes are free to ignore it if they want, if they
do not wish to trust the checkpoint instruction.

* A single application deployment splits into two deployments:  one with a group of processes that keep
building state off of the pre-fork blockchain input history,
and one group that uses the post-fork blockchain input history.  These deployments do not consider each other's
inputs as valid, due to the join-at-most-once property.

In the second failure mode, applications can use a variant of the "checkpoint" feature to migrate from one blockchain
to another, more secure one.  The authors have already done this by migrating [Blockstack](https://blockstack.org) 
from the Namecoin blockchain to the Bitcoin blockchain.

For more background, please read the [virtualchain paper](https://blockstack.org/virtualchain.pdf), and see
the ([DCCL 2016](https://www.zurich.ibm.com/dccl/) slide deck [here](https://github.com/blockstack/blockstack/talks/virtualchain.pdf).

## How Virtualchain Works

Virtualchain works by crawling an existing blockchain's blocks in order from oldest to newest.
For each block, it selects transactions that
contain possible inputs, decodes them, and forwards them to the application process's consensus logic.
The application's consensus logic selects the valid inputs, executes the
corresponding state-transitions, and saves its current state.  Virtualchain then calculates a "consensus hash"
for that block, which is derived from both the block's inputs as well as a geometric series of prior consensus hashes.
The exact derivation can be found in the [virtualchain paper](https://blockstack.org/virtualchain.pdf).

The consensus hash is the mechanism by which virtualchain enforces fork * -consistency.  Due to the way it's
calculated, a consensus hash is a compact representation of the process's operation log.  By embedding the
*current* consensus hash in each input submitted to the blockchain, a process can indicate the state of its
operation log to other processes.  In doing so, an application can easily recognize an input from another process
 in its deployment:  the input must include a consensus hash from the last *S* blocks (where *S* is a staleness
parameter defined by the application that determines how old the consensus hash is allowed to be before
being considered invalid).

### Scanning the Blockchain 

Virtualchain applications generate transactions that contain a small arbitrary data payload.  In Bitcoin,
this payload is embedded as an `OP_RETURN` output script.  The payload is prefixed with a small sequence of "magic bytes"
and one of a set of "opcodes."  The magic bytes make it easy for virtualchain to sift through transactions to find
the ones that could possibly be inputs.  Opcodes are one-byte codes that provide a simple way for applications
to assign a type to their inputs.  Virtualchain will automatically decode and present the application
with the input's given opcode and associated payload.

When processing each transaction, virtualchain will additionally keep track of which public key(s) sent
the given transaction, which public key(s) received funds, and how much of the blockchain's token was spent.
This information is also stored on the blockchain, as part of the blockchain's native token-tracking functionality.
This is often useful to applications as well, since it is often the case that its processes and clients have unique
public keys, and this information tells the application who sent the message and who received the message.
For example, Blockstack uses the sender and recipient information to keep track of name ownership and ownership
transfers.

As mentioned, virtualchain processes the blockchain in a block-oriented fashion.  Once it identifies a block's
transactions that have the application's magic bytes, it will feed them into the application in the following
sequence of steps:

1.  **Parsing**.  Virtualchain will first have the application try to parse each input's payload, given the transaction's
opcode, its sender(s) and recipient(s), and its fee.  The magic bytes and opcode alone do not guarantee that the
transaction is well-formed; if the application cannot parse the payload, virtualchain will drop it from its consideration.

2.  **Prepare-commit**.  Before asking the application to validate each input, it gives the application the chance to consider
all parsed inputs from the block.  For example, Blockstack
uses this step to break ties when two or more people tried to register the same name in the same block.  The application is
not expected to change any state in this step.

3.  **Vote-to-commit**.  Virtualchain then asks the application whether or not it will accept each input.  It discards the
inputs the application rejects.  Again, the application is not expected to change any state in this step.

4.  **Commit**.  After having the application select its valid transactions, virtualchain then feeds them back into the application
in order for the application to commit them.  The application executes its state-transition logic for each input in this step.

5.  **Save**.  Once the application has committed each input, virtualchain updates its persistent-storage bookkeeping atomically
to make the state-transitions permanent.

6.  **Continue**.  After the block's inputs are processed, virtualchain asks the application if it would like to continue to the next block.
The application takes this opportunity to do things like abort on inconsistency, or exit on user request.

### API

Virtualchain provides a callback-oriented interface to applications.  The application gives virtualchain an
*implementation*, which can be a class object, module, etc.  For example, in Blockstack Core, this is
a module called `blockstack/lib/nameset/virtualchain_hooks.py`.

Broadly speaking, the implementation must implement two sets of methods:  the methods
that provide virtualchain with some basic information about how to do 
some bookkeeping and how to recognize inputs from the blockchain, and the methods that
carry out state transitions.

The "basic information" methods are:

* `get_virtual_chain_name()`:  Returns the name of the implementation (e.g. `blockstack-server`).

* `get_virtual_chain_version()`:  Returns the version number of the implementation.

* `get_first_block_id()`:  Returns the block number that contains the application's first input.  For Blockstack,
this is Bitcoin block 373601.

* `get_db_state()`:  Returns an opaque object that represents the application-specific RSM state.

* `get_magic_bytes()`:  Returns the list of bytes that prefix an input payload.  For Blockstack, this is `id`.

* `get_opcodes()`:  Get the list of bytes that describe the input operations this RSM accepts.  For example, in Blockstack, this includes.
`?`, `:`, `+`, `>`, `~`, `*`, `&`, and `!`.

The "state transition" methods are:

* `db_parse( block_id, opcode, op_payload, senders, inputs, outputs, fee, db_state )`:  Returns the parsed operation state
as an opaque object on success.  Returns None if the application could not parse the input.  This is called for each
input in the **parsing** step.  `db_state` is a handle to the application's opaque state, from `get_db_state()`.

* `db_scan_block( block_id, op_list, db_state )`:  Does not return anything, but gives the application a chance to prepare to 
commit (the **prepare-commit** step).

* `db_check( block_id, opcode, op, txid, vtxindex, checked, db_state )`:  This is the **vote-to-commit** step for an input.  The application
decides whether or not the input is valid.  Here, `txid` is the blockchain transaction ID, `vtxindex` is the *logical* index into the block's
inputs at which this input occurres (i.e. this is *i* for the *ith* successfully-parsed input from `db_parse()`), and `checked` is the list 
of previously-committed inputs.  Returns True if the input is to be committed; False if not.

* `db_commit( block_id, opcode, op, txid, vtxindex, db_state )`:  This is the **commit** step for the input.  The application executes the state-transition
for this input and records resulting state in the opaque `db_state` object.

* `db_save( block_id, filename, db_state )`:  This is the **save** step, where the application writes out its current state to the given `filename`.  Virtualchain
selects the filename and will atomically move it into place to finish processing the block.

* `db_continue( block_id, consensus_hash )`:  This is the **continue** step, where the application is given a chance to either stop processing or continue processing.
`consensus_hash` is the virtualchain-calculated consensus hash for the given block.

### Code Overview

In `virtualchain/lib/indexer.py`, there is a class called `StateEngine` that contains the core virtualchain state-processing logic. 
A `StateEngine` instance is what drives the application, via its `build()` method.  Applications should define their RSM state logic
by creating a subclass of `StateEngine`, in order to gain access to its prior consensus hashes.

TODO finish example
