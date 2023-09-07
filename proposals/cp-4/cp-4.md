> Title: [Proposal] Support SingleNode Deployment<br>
> Date: 09/05, 2023
> GitHub Discussions: https://github.com/orgs/cloudberrydb/discussions/188

### Proposers

- @avamingli 
- @wfnuser

### Proposal Status

In Progress

### Abstract

This proposal adds single node deployment to CBDB. It allows user to start a zero segment (no primary-mirror pairs) but still use, well, most of functionalities of a normal CBDB cluster.

### Motivation

Customers have reported that a singlenode mode may be necessary. Since GPDB isn't fully compatible with Postgres, many features and grammars have been dependent upon. In a scenario where only one node is needed but GPDB is already relied by business logic, a singlenode mode can be used.

### Implementation

Previously @wfnuser already crafted a draft version of singlenode (#77) but was deemed too radical and got rejected. Currently proposed implementation is nearly identical to that one, with the exception that **no** new `gp_role` are added. The implementation will be trifold:
- **Code**
  - `int`-typed global variable `GpSegmentCount` (I start to think this isn't a good name, I'll replace it once I come up with a better one) will be added to cache `getgpsegmentCount` value **at the initialization time**. This value is to support `IS_SINGLENODE` and `IS_UTILITY_OR_SINGLENODE` macros.
  - all branch and assertions w.r.t. `utility` mode will be expanded to `Gp_role == GP_ROLE_UTILITY || IS_SINGLENODE()`
 - **Script**
   - we plan to use the already baked `NUM_PRIMARY_MIRROR_PAIRS=0` to support create singlenode deployment in all scripts and commands (`gpstop`, `gpstart`, etc.)
   - user can reliably detect whether the remote is in singlenode or not by `select case when count(*) = 0 then true else false end as is_singlenode from gp_segment_configuration where content != -1;`
 - **Test**
   - **The most challenging and irksome part of this PR is how to handle tests.** Many tests in `regress` relies on test plan which surely is different under singlenode mode and `isolation2` tests often require concurrent semantic and utility mode (`-1U`, `1U`, etc.) that singlenode does't support. My initial proposal is to make those tests that behave differently on singlenode have their own copy of the expected file and change test suite (e.g., `isolation2/sql_isolation_testcase.py`) to adapt it according to which mode the remote is running at. But I don't know whether this would work. Plus, many details are yet to be determined.

### FAQ
- **Q: How is singlenode differs from the already-supported `utility` mode?**
  **A:** To use `utility` mode one still needs to create a multi-segment cluster, and then connects to one of them using `-c gp_role=utility`. This is different from `singlenode` because the latter requires no segment at all from the ground up. Regarding code changes, as described above, they're mostly identical. There're a few cases where `singlenode` mode behaves different from `utility` mode and thus need special care, but they're generally rare.

### Rollout/Adoption Plan

_No response_

### Are you willing to submit a PR?

- [X] Yes I am willing to submit a PR!