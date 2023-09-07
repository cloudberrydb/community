> Title: [Proposal] Implement Scorll Parallel Retrieve Cursor.<br>
> Date: 08/04, 2023
> GitHub Discussions: https://github.com/orgs/cloudberrydb/discussions/120

### Proposers

@wenchaozhang-123 

### Proposal Status

Under Discussion

### Abstract

To support Ray using data in CBDB to finish ML work, we should implement a new cursor named scroll parallel retrieve cursor which supports rescan on each segment.  This proposal describes the problems in parallel retrieve cursor and the main idea of how to enhance it.

### Motivation

Now, the endpoint of parallel retrieve cursor will be disconnected after retrieves all data in segment. What's more, it can't support the rescan of the table. This is unfriendly to Ray ML work. Hence, we expect to implement a new parallel retrieve cursor, which can support rescan table in segment.

### Implementation

Design and problem of parallel retrieve cursor:
Every parallel retrieve cursor includes one or more endpoints, each endpoint can read tuples. After retrieves all tuples in one endpoint, the connection will be released. We expect Ray to use the parallel retrieve cursor, which needs to rescan the table in one endpoint. However, as we describe above, parallel retrieve cursor doesn't support that. The design of parallel retrieve cursor as follows: 
![parallel retrieve cursor](https://github.com/cloudberrydb/cloudberrydb/assets/53178068/8a43c9b3-f790-4e8a-a23d-49c8c14ec46e)


As a result, we should hold connection after retrieves cursor. What's more, the tuple should be rescanned in the endpoint. To achieve that, we should implement a new cursor which can support rescan the table in the endpoint. The main idea is that we should hold the endpoint connection until SQL command to close it. And we should store the tuple retrieved from the table to the tuplestore, which support the execution of the rescan. The main execution flow is as follows:
![scorll parallel retrieve cursor](https://github.com/cloudberrydb/cloudberrydb/assets/53178068/beb48867-250c-4b07-8381-08ffd50d0340)


[#104](https://github.com/cloudberrydb/cloudberrydb/issues/104)
- [ ] support new grammars
- [ ] implement execution process


### Rollout/Adoption Plan

_No response_

### Are you willing to submit a PR?

- [X] Yes I am willing to submit a PR!