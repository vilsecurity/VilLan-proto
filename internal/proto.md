We want a replicated data store:
*  With scalability (i.e. O(n) packets per event MAXIMUM, but O(poly(n)) allowed amongst the master nodes)
*  Which cannot be reverted, even if all but one of the storage nodes are captured.
*  With Byzantine fault tolerance
*  Without some huge compute power or node size (\*cough\* Bitcoin \*cough\*)

What I came up with (spiderweb), uses the following algorithm:

Everyone needs to agree on these constants:

| Constant    | Used value | Description                                                           |
| ----------- | ---------- | --------------------------------------------------------------------- |
| DRIFT_TIME  | 1 second   | The tolerance to clock drift that the system has                      |
| SHARE_TIME  | 2 seconds  | The maximum time a node can wait before sharing it's unseen commits   |
| SUBMIT_TIME | 2 seconds  | The maximum time a node can wait before submitting its epoch          |
| FINAL_TIME  | 2 seconds  | The maximum sensible time taken for a node to finalise its ecpoh      |
| EPOCH_TIME  | 20 seconds | The time between each epoch                                           |

Please note that `SHARE_TIME + SUBMIT_TIME + FINAL_TIME + 2 * DRIFT_TIME <= EPOCH_TIME` MUST hold!

There are 3 types of node, each with all the capabilities of the one below it.

| Type   | Description                                        |
| ------ | -------------------------------------------------- |
| Root   | A node which may legally sign any message it likes |
| Master | A node which may contribute towards elections      |
| Worker  | A node that may only submit commits                |

A node can also be set as a store, which means it must store all the completed epochs it receives.


### Notes before getting started
*  In this protocol description, you will see phrases such as "A worker cannot vote",
   and "sharing a minority viewpoint is criminal". These are not political statements.
*  Be aware that it is perfectly possible for data to be referenced in an epoch, but not actually exist.
   The converse also holds.
   We only guarantee that data cannot be removed after it has been stored properly AND referenced in an epoch.
   However, this should be very rare, and a spike would indicate an attack!
*  Any node can store any request it likes, but can only apply them if they form part of a valid epoch
*  Old epochs can be deleted (or moved to cold storage), and the cluster will continue to function.
   However, storage nodes are allowed to delete unreferenced data, so make sure to back up that as well!

### Defintions
1. A "view" is a master node's collecion of commits that have been seen since the last epoch.
1. An epoch is "complete" if it is signed by a majority of the master nodes, and "incomplete" otherwise.
1. An epoch is "open" if it was created less than EPOCH_TIME ago, and is "closed" otherwise.
1. A set of epochs are "consistent" if they are semantically equivalent, and are "inconsistent" otherwise.
1. In the context of a master node in selection state, a single epoch is "consistent" if it is semantically equivalent
   to the nodes view of that epoch
1. Something is "legal" if it does not contradicts the laws. Illegal messages should be discarded.
1. Something is "criminal" if it indicates a severe fault with the cluster.
   This may be a serious configuration error (such as clock drift > DRIFT_TIME, or misconfiguration of the constants),
   or may represent an attack on the cluster.

### Laws
These laws are in order of priority, and in any conflict, the law that comes first takes precendent.
1. If something is signed by a root node, it is legal.
1. A worker cannot vote.
1. An epoch cannot be received before its `created` time
1. An epoch cannot be created until EPOCH_TIME from its designated `previous_epoch`.
1. An epoch can only be signed by nodes that were master nodes BEFORE the epoch in question.
1. If two valid consistent closed complete epochs are seen, see "Panic mode".
1. If two valid consistent open complete epochs are seen, the one with the lowest priority should be discarded.
   The priority rules are (in descending order):
    1. Root signed.
    1. The lowest hash value (big-endian)
1. Attempting to access a service you don't have access to (i.e. a worker trying to send `share` rpcs)
   is criminal.

### Running a worker node
1. When there is data to be committed, ALL the current master nodes MUST be notified (with a `notify_commit` rpc),
   and at least an attempt to store it MUST be given to ALL the storage nodes (with a `store_commit` rpc).
1. When a previously unseen complete epoch is received, or one with higher priority,
   the node's state must be adjusted to incorporate it.

### Running a storage node
Data must be written to persistent storage before any acknowlegement is sent.

In order of (descending) priority:
1. If a closed complete epoch is seen with some new unstored data, the storage node MUST iterate through
   the other storage nodes, one at a time, to try and get a copy of the data, until either the set of storage nodes
   is exhausted, or it is found. If the latter is the case, it must be written to persisten storage immediately.
1. If a StoreCommit is received, write ALL the data to a persistent storage before sending a response

### Running a master node
1. Do everything a worker does.
1. After EPOCH_TIME from the previous epoch, the node enters a "selection" state.
1. Pick a random time during the sharing window (between 0 seconds and SHARE_TIME after entering selection),
   and broadcast all the valid commits that have not been seen before that point in a `share` rpc.
1. The sharing deadline is another DRIFT_TIME from the end of the sharing window.
   After this point, any `share` rpcs are criminal.
1. Pick a random time during the submission window (between the sharing deadline and SUBMIT_TIME after it),
   and if a consistent epoch is not seen before that point, the node should bundle its view into an epoch,
   and broadcast an `submit` rpc. Make sure to exclude any requests you do not explicitly agree with,
   such as a `*_add`, `*_remove` rpc. A node sharing a minority viewpoint is criminal.
1. The submission deadline is another DRIFT_TIME from the end of the submission window.
   After this point, any `submit` rpcs are criminal.
1. After the submission deadline, select the highest priority consistent epoch,
   and broadcast it with a `selected` rpc.
   This is the only strictly superlinear (O(n^2)) broadcast.
1. If there is at least one completed epoch, select the highest priority one, and use it.
1. If there isn't one after the finalisation deadline (FINAL_TIME after the submission deadline), see "Panic mode".

### Running a root node
There is a good chance you don't need one of these. If you can co-ordinate all the nodes to do something,
then there is no point in having a root. It should only be used in testing enironemnts, or when there are
a very small number of completely trusted individuals (like a one-person apartment). Even then, it may
be avoided. Since the node has a vast amount of power, it should be made as difficult to access as possible,
and be kept disconnected from the cluster whenever possible.

### TODOS:
*  Storage quotas! This nullifies a significant DoS attack.
*  Add master/root data to EVERY epoch! Then we can delete all the data, without breaking it.

### Panic mode
If you get here, one of a few things has happened
1.  A majority of the nodes has been compromised (unlikely, but possible).
1.  A network segment has formed, and you are on the wrong side (even more unlikely).
1.  Someone messed up their programming (probably you) (very likely).

In any case, there's very little you can do. Hopefully the other nodes send some form of error message,
but if you have any ability to do so, signal an error. As loudly as possible.
Beeps, flashing lights, VHF channel 16 mayday's, and and using airburst nuclear weapons as signal flares are all recommended.
