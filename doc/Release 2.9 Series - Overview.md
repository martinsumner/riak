# Riak Futures

## What is Riak?

Riak is a data store originally developed around the principles of the [Amazon dynamo](https://www.dynamodbguide.com/the-dynamo-paper/) paper.  As a solution, Riak tends to be targeted at systems wanting to store data with:

- challenging availability requirements, such as with online-only service businesses with a direct relationship between availability and revenue, or systems recognised as critical national infrastructure;
- demanding durability guarantees, often due to a regulatory need as well as a financial need to protect against data loss;
- a workload that can be largely aligned with value fetches against known keys, and where the development team is able to effectively reason about consistency trade-offs and plan indexing strategies;
- applications where limiting high percentile latency response times is more critical than minimising median response times, ensuring all requests get a reasonable response time rather than prioritising that most requests get the fastest possible response time.
- a significant but bounded demand to scale, where scaling beyond 10 nodes is likely, but scaling beyond 100 nodes can be supported by splitting workloads across clusters, not by continuously expanding a single cluster.

Ongoing development of Riak is now managed within the open-source community, with most development activity funded directly by Riak's largest customers. This follows the purchase of Basho's [IPR by bet365](http://lists.basho.com/pipermail/riak-users_lists.basho.com/2017-August/019500.html) one of Riak's biggest users.

## Release Schedule

April 2018 saw the release of [Riak KV 2.2.5](https://github.com/basho/riak/blob/riak-2.2.5/RELEASE-NOTES.md), the first [post-Basho](https://www.theregister.co.uk/2017/07/31/end_of_the_road_for_basho_as_court_puts_biz_into_receivership/) release of the Riak KV store.  This release was deliberately light on features, focusing on stability fixes that could be added with minimal disruption to the existing codebase and minimal risk to existing production workloads.  The release did though establish a smoother path to releasing future changes, by investing significant effort in improving the reliability and usability of the Riak test and release process.

There is now a plan for significant further improvements to Riak.  These improvements will be delivered in two release cycles - Release 2.9 and Release 3.0:

- Release 2.9 is focused on delivering significant database throughput improvements for use-cases which depend on both ordered keys and mid-to-large size objects, an overhaul of the efficiency of managing anti-entropy both within and across database clusters, and reductions in the network-overheads of running a Riak cluster.

- Release 3.0 is focused on providing a future-proof Riak, migrating to an up-to-date OTP platform, and stripping away the accidental complexity of under-used features.  Release 3.0 will also build on some of the foundation improvements in Riak 2.9, to provide for a more efficient and flexible replication solution, and allow for a richer set of query features that can be run at minimal risk to the predictability of performance for the core Key/Value workloads of Riak customers.

Release 2.9 will have an initial private release candidate available in early December 2018, and the release is expected to be generally available by the end of January 2019.  The target for Release 3.0 is to have an initial Release Candidate available in April 2019.

Release 2.9 is intended to be a stepping stone towards migrating to Release 3.0, but for users of features that will be terminated in Release 3.0, it is possible that community-led updates may continue on the 2.9 release branch for some time beyond the availability of Release 3.0.

### Release 2.9

Release 2.9 brings the following two significant, but optional, changes:


- [Tictac Active Anti-Entropy](https://github.com/martinsumner/kv_index_tictactree).

  - This makes two fundamental changes to the way anti-entropy has historically worked in Riak.  The feature changes the nature of the construction of the Merkle Trees used in Anti-Entropy so that they can be built incrementally.  The feature also changes the nature of the underlying anti-entropy key store so that the store can now be key-ordered, whilst still allowing for acceleration of access to keys by either their Merkle tree location or by the last modified date of the object.

  - Previously anti-entropy had required knowledge of all elements of the tree to build the tree, and for a key store to be kept ordered based on the layout of that tree.  These changes allow for:

    - Lower overhead internal anti-entropy based on *cached trees*.

    - Cluster-wide anti-entropy based on *cached trees* and without the need to pause for cluster-wide full synchronisation to be suspended for long periods while AAE trees and stores are rebuilt. Cached trees are kept updated in parallel to the rebuilding of trees and AAE stores.

    - Cross-cluster Merkle trees to be *independent of the internal layout* of the data in the cluster,

    - Folding anti-entropy.  The rapid and efficient production of anti-entropy Merkle *trees of subsets of the store data*, with those subsets definable at run-time based on *bucket*, *key-range* and *modified date* restrictions.  Allowing for more flexible inter-cluster comparisons (other than comparing whole stores).

  - Database statistics and operator helpers.  The anti-entropy keystore stores keys and additional metadata to support potentially helpful queries, without the need to fold over the vnode object store.  This keystore can then also efficiently support ordered folds for unordered backends (e.g. bitcask).  By folding over ranges of keys and metadata, not slowed by loading in all the values off disk - [administrative database queries](https://github.com/martinsumner/riak_kv/blob/develop-2.9/src/riak_kv_clusteraae_fsm.erl#L165-L208) can now be efficiently supported (e.g. object counts, find keys with siblings, find keys with large object sizes, object size histograms, calculate the average object size of items modified in last 24 hours etc).

  - The Tictac AAE feature can be run in additional to or instead of traditional Riak Active Anti-Entropy mechanisms to ease migration from the existing service.

  - Future work on Tictac AAE is planned to handle issues arising from time-based deletion of objects.  Tictac AAE will not currently work efficiently if a significant portion of objects have automatic TTL-based expiry.


- [Leveled backend](https://github.com/martinsumner/leveled).

  - A new database backend to Riak, written entirely in Erlang, and optimised specifically for Riak-style workloads.  Leveled is based on the same [Log Structured Merge (LSM) Tree paper](https://www.cs.umb.edu/~poneil/lsmtree.pdf) as the existing [leveldb](https://github.com/basho/leveldb/wiki) and [hanoidb](https://github.com/krestenkrab/hanoidb) backends, but making specific trade-offs to improve on throughput in some common riak uses cases:

  - LSM-trees supporting larger objects.  Other LSM-tree based these stores have the potential to be bottle-necked by write-amplification (100 fold write amplification has bene seen on large, mature Riak stores using leveldb) when used for storing larger objects (e.g. objects over 4KB).  Leveled splits objects from headers, which was suggested as an option in the original LSM-tree paper, and further explored in the [WiscKey paper](https://www.usenix.org/node/194425) (and implemented in other stores such as Dgraph's [BadgerDB](https://github.com/dgraph-io/badger)).  The full object is stored in a sequence-ordered journal separate to the LSM-tree, which contains only Keys and their metadata.  This reduces write amplification for larger values, as the LSM tree merge events are proportionate to the size of the headers not the objects.

  - Replacing GETs with HEADs.  In all existing riak backends, the cost of extracting the header of an object is broadly equivalent to the cost of extracting the whole object.  However, when resolving a GET request, only the headers of the objects are required to determine if the version vectors of the objects match, and if they do match only one vnode is required to return the actual body.  Likewise the PUT path in Riak only requires to see the version object header not the object body before updating an object.

    - By providing a fast-path to accessing the head of the object, Riak with a leveled backend is able to stop the practice of pulling the desired object N times over the network for each GET, using HEAD requests where possible instead.

    - The response time of the HEAD messages also provides early warning of vnode queues and network latency, so that GET requests can be pushed towards fast responding vnodes, better balancing load across the cluster when one or more nodes slower than other nodes in the cluster.

  - Anti-entropy without a secondary store.  The Tictac AAE solution requires an ordered keystore (As does the current Riak AAE solution), but as the Leveled backend already has a dedicated keystore for holding keys and metadata, this can be reused for AAE purposes.  This means that Tictac AAE can be run in `native` mode - where no secondary store is required, queries can be directed back to the actual backend store.

  - A cluster-wide hot-backup facility, which due to reduced write amplification provides for efficient rsync support in a key-ordered backend.

  - Migrating to the leveled backend requires a riak cluster `replace` operation - there is no in-place transition to leveled from an existing backend.  

  - It is expected that community interest and support in the [bitcask backend](https://github.com/basho/bitcask) within Riak will continue into Riak 3.0 and beyond, as bitcask still offers throughput advantages with some workloads, where there is no demand for secondary indexes.

  - Some [performance testing results and guidance for choosing a backend have been made available to assist with this decision](https://github.com/martinsumner/riak_testing_notes/blob/master/Release%202.9%20-%20Choosing%20a%20Backend.md).  The optimal decision though is driven by too many variables (e.g. object size, number of keys, distribution of requests to keys, mutability of objects, physical server configuration, feature requirements and levels of application concurrency) to make an optimal decision obvious in most uses cases - realistic use-case specific testing is always recommended.

Release 2.9 also brings three building blocks to enable current and future improvements to the management of operational risk:


- [Vnode Soft Limits](https://github.com/martinsumner/riak_kv/blob/develop-2.9/docs/soft-limit-vnode.md)

  - When Riak is in receipt of a PUT request, it must select a vnode to co-ordinate the PUT.  However, when load is high, vnodes may have work queues of varying sizes - and picking a vnode with a large queue will slow the PUT to the pace of that slow vnode.  Vnode soft limits are a resolution to this problem, providing a simple check of the sate of a vnode queue before determining that a particular vnode is a good candidate to coordinate a given PUT.

  - The biggest advantage seen in testing vnode soft limits is with the leveldb backend, where under soak test conditions there is a 50% reduction in the trend-line of 99th percentile PUT measure, and a 80% reduction in the peak 99th percentile PUT time.


- [Core node worker pool](https://github.com/martinsumner/riak_core/blob/develop-2.9/docs/node_worker_pool.md)

  - Riak-backed applications tend to make heavy use of the standard GET/PUT KV operations.  These are short-lived tasks, but sometimes longer-lives tasks are required to either provide information to the operator (e.g. what is the average object size in the store?), or detect otherwise hidden errors (e.g. AAE tree rebuilds).  Each such task has tended to evolve its own mechanism to ensure that the impact of the task can be controlled to avoid inhibiting higher priority database work.  The core node worker pool is a simple mechanism for controlling concurrency of background tasks on a per-node basis.  It allows for either a single node worker pool to manage concurrency, or a series of pools modelled on the [Differentiated Services design pattern](https://en.wikipedia.org/wiki/Differentiated_services).

  - There are other more sophisticated candidate methods which have been proposed in this space (e.g. riak_kv_sweeper and riak_core jobs).  A decision will be made for the Riak 3.0 release which mechanism should be the standard going forward, but the core node worker pool appears to be the simplest of all the proposals at this stage.


- Repl API

  - Multiple customers of Riak have ended up with some form of bespoke extensions to the Riak replication features, normally to avoid some inefficiency in a replication feature by leveraging knowledge of the application (e.g. keys are time-stamped based, some keys are write-once etc).  The repl code itself has expanded complexity to deal with scheduling of jobs, marshalling the use of resource by jobs, managing environmental factors (e.g. NAT, encryption requirements), handling change within the cluster, managing exceptional replication topologies.  Going forward, the preferred approach for handling special customer scenarios is to expose core replication features for customers to manage from outside of the database, rather than extending the internal feature scope for each scenario.

   - There are two repl features to be exposed in Riak 2.9.  The first feature is an API to re-replicate an object: given a key re-replicate this key from the cluster in receipt of the request.  The second feature is the availability of an `aae_fold` API, to give access to cluster-wide AAE trees available as part of the TictacAAE change - as well as the ability to fetch keys and version information from objects within specific segments of the AAE tree.


Following the general availability of Riak 2.9.0, will will continue on the 2.9 stream in parallel to work on Release 3.0.  This parallel work may include:

- Fixes and improvements to riak real-time replication that have already been proven in production with bet365;

- More exposing of riak_repl features - for example a `re-replicate if` feature that will re-replicate an object based on version information that has been returned form an `aae_fold`, as well as additional `aae_fold` queries.

Release [task lists](https://github.com/martinsumner/riak_testing_notes/blob/master/Release%202.9%20-%20%20Task%20Countdown.md), and [test progress](https://github.com/martinsumner/riak_testing_notes/blob/master/Release%202.9%20-%20Test%20Summary.md) are tracked online.

#### Release 2.9.0 RC1

[Release Notice.](http://lists.basho.com/pipermail/riak-users_lists.basho.com/2019-January/039316.html)

#### Release 2.9.0 RC2

Release Candidate 2 has the following changes:

- Customer testing exposed that a mochiweb change made in Riak 2.1.x had altered the [behaviour when handling large headers](https://github.com/basho/riak_api/issues/123) (e.g. PUTs to riak with large index entries).  In Riak 2.0.x headers could be of arbitrary size, but since 2.1 these headers have been restricted to the size of the receive buffer (default 8KB).  Mochiweb and webmachine have now been updated to make the receive buffer configurable, to allow for a workaround should a customer hit this limit.  The receive buffer size is now configurable via the `advanced.config` - adding a stanza like:

``{webmachine,
   [{recbuf, 65536}]},``

- As part of the change above, mochiweb has been brought up-to-date with the mainstream mochi repository.  This brings through all [changes since 2.9.0](https://github.com/basho/mochiweb/blob/master/CHANGES.md).  Users of the HTTP API should consider these changes when testing the release.

- Log level with the leveled backend can now be set through riak.conf, and the log format has been changed to make the logs easier to index.

- An issue discovered in property-based testing (by [Quviq](http://www.quviq.com/)) with object folds in sqn_order has been resolved.

- The process of closing down leveled has been refactored to stop process leaks discovered in property-based testing (by [Quviq](http://www.quviq.com/)).

- A workaround to an issue running a leveled unit test in riak `make test` was leading to a `make test` failure.

- A Protocol Buffers API change made as part of the object touch repl changes was missing from RC1, and this has now been picked up in RC2.

#### Release 2.9.0 RC3

Please ignore this release candidate.

#### Release 2.9.0 RC4

The RC4 changes only have an impact on the behaviour of Riak when used with a leveled backend:

- There are corrections to the Leveled fixes made in RC2 to ensure that the full cache-index situation is handled safely, and a potential deadlock on shutdown between the penciller and an individual sst file is resolved.

- The Riak KV default cache size for leveled is reduced to the leveled default, the maximum size the cache can grow to (via jitter/returned) is reduced, and the number of cache lines are reduced.  This means that in a stalled penciller, the next L0 file is constrained to be an order of magnitude smaller than in RC2.  This may prevent bad behaviour under heavy handoff load.

- The riak_kv_leveled_backend will now pause the vnode in response to a stalling leveled backend.

- The riak_kv_leveled_backend will support v1 objects only, the riak_kv_vnode will never try to write an object as v0 into leveled.

#### Release 2.9.0 RC5

The primary RC5 change is again leveled related.  It was discovered in handoff scenarios in a leveled backend, Riak consumed much more memory than expected.  This was caused by "switched" Level 0 files in the Penciller.  These files have a small memory footprint when garbage collected, but a large footprint uncollected - there is a legacy of all the data being on the LoopState in the `starting` state (but not the `reader` state).  Each file process now does `garbage_collect/1` on self at the point of the switch to free this memory immediately.

There is also a small fix to poolboy to prevent a crash log from appearing on shutdown.

#### Release 2.9.0 RC6

RC6 fixes some security issues within yokozuna, and completes a full run through of the yokozuna tests.  It resolves an issue with HTTP security features crashing Riak which was introduced as part of the RC2 mochiweb uplift to fix the 2i index changes.  It also transitions the eleveldb branch used to point back to the `basho` repository, with a fix that allows eleveldb to be deployed on recent OSX versions.  An OSX-specific issue with `make test` failing on `eper` and `riak_ensemble` unit tests is also resolved.

#### Release 2.9.0 RC7

A customer volume test revealed that the beam would not fully garbage collect memory held by leveled_sst file processes within leveled (leaving 4MB of unlinked memory associated with the process), in some circumstances.  This is similar to the problem partially-resolved in RC5.  A more comprehensive fix has now been added for RC7, with a process-specific GC call now made each time a non-L0 file transitions to the read state, and stays in that state for more than 10s.

#### Transition Configuration Guidance

This section contains some initial notes to assist with planning and configuration for Transition of pre-2.9 releases to 2.9:

- The leveled backend is not compatible with other backends in terms of the serialised disk format.  There is no in-place transition possible from bitcask/eleveldb/hanoidb to leveled.  Transitioning requires a node replace operation.  It is recommended to:

  - First transition to 2.9 with the current backend in-place, minimising the time spent running mis-matched versions in parallel;

  - Then as a second phase run a rolling series of node transfers to replace the nodes with the previous backend, with nodes with the leveled backend.

  - Testing hash shown that higher transfer-limits can be sued safely when running transfers to leveled nodes, by comparison to transfers to eleveldb nodes.

- If upgrading from a release prior to the introduction of version 1 hashing of AAE, and if you intend to eventually move to TictacAAE - then follow [the guidance](http://docs.basho.com/riak/kv/2.2.3/setup/upgrading/version/) to not upgrade to version 1.  This prevents CPU resource bing invested in the upgrade when it is eventually unnecessary.

- Tictac AAE and Legacy AAE may be run in parallel - set both to `active` in `riak.conf`.  The cost of running Tictac AAE in parallel can be reduced by adjusting the `tictacaae_exchangetick` to a higher value.  By default this is is set to 120000 ms (2 minutes).

  - When Tictac AAE has not been run from the initial loading of the node, then the AAE process will not be fully effective until all nodes have undergone an "AAE rebuild".  An increased `tictacaae_exchangetick` is recommended in this period.


- For observability of new features, the stats output from `riak-admin status` have been extended, but also there is a greater focus on use of logs for standard events, on both exit and entry from the event.  Tictac AAE is best observed form indexing the Riak logs (both `console.*.log` and `erlang.*.log`), and `riak-admin aae-status` will no longer offer any information.

- Flushing to disk on every write can be enabled in leveled using `leveled.sync_strategy`.  For Riak 2.9.0, the `riak_sync` mechanism must be used to enable sync, the `sync` mechanism is only valid on later versions of OTP.

- Leveled like levedb continuously compacts the keystore (the LSM-tree).  However, it must separately compact the value store, and compaction of the value store may be scheduled - using `leveled.compaction_runs_perday`, `leveled.compaction_low_hour`, `leveled.compaction_high_hour` and `leveled.max_run_length`.  The following log should help with tuning:

  - "IC003", "Scoring of compaction runs complete with highest score=~w  with run of run_length=~w",

  - If the highest score is increasing over time (and positive), then there is a backlog of compaction activity - so increase either the length of the run or the runs per day.

- The size of the journal files can be changed to align with the size of the objects.  Set the configuration parameter `leveled.journal_size` to be approximately the size of around 100 thousand objects.

- Leveled compression can be either `native` or `lz4`.  lz4 has improved performance in most volume tests, but unless the performance improvement is significant for a use case, sticking with `native` compression is recommended, as this does not create a dependency on an external library. For objects which are already compressed, and may gain little value from compression, it is recommended switching the compression point to be `on_compact` rather than `on_receipt`.

- The code contains a more complete view of startup options for [leveled](https://github.com/martinsumner/leveled/blob/master/priv/leveled.schema) and [tictac_aae](https://github.com/martinsumner/riak_kv/blob/develop-2.9/priv/riak_kv.schema#L28-L96).


### Release 2.9.1

There are some minor feature improvements that have been partially completed, but have not been included in Riak 2.9.0.  These improvements may be added to a minor release, which will be built in parallel to the work to produce Riak 3.0.

These features are:

- Flush puts only at the co-ordinator.  Flushing to disk on each write carries a significant cost, this cost can be marginally reduced, whilst still improving the durability guarantee (of OS-controlled flushes) by only flushing on the co-ordinated write.  This also means that handoff activity is not impacted by having flush enabled.

- Read-repair to primaries only.  Currently during a node failure, the fallback vnode will be be empty, and so each GET in an impacted preflist will prompt a read-repair.  This "fills" the fallback vnode (which offers advantages if the primary is to be unavailable for a long period).  However, this increases load in the cluster at a point when the cluster has reduced capacity, it also lengthens the time that handoff will take when the primary comes back online (replicating old PUTs the primary had).  The alternative would be to only read-repair to primary vnodes.

- Conditional re-replication.  The `aae_fold` feature in Riak 2.9.0 allows keys & clocks to be returned to clients to allow for comparison between clusters.  The client can now compare clocks and call the replicate API should the comparison reveal a need to re-replicate.  It would be preferable that clocks remain opaque to clients, so that clients did not need to interpret the clocks. The replication interface would then support the passing of clocks for conditional replication, and the comparison of clocks and subsequent forwarding decision would be managed internally within Riak.

- Yokozuna separation from riak_kv_vnode.  Currently the riak_kv code contains references to the yokozuna module, and this can be adjusted so that Yokozuna could still be used if required - but if not, Riak KV could be run without loading Yokozuna.

### Release 3.0

Work on Release 3.0 will initially focus entirely on ensuring the OTP uplift is completed, so that Riak can be run on OTP 20 (and preferably also OTP 21).  As part of doing this, under-used features may be removed if insufficient community interest and time is available to maintain them.  This is likely to impact the availability of riak_ensemble beyond Riak 2.9, but it is now expected that sufficient community effort will be available to sustain support for Yokozuna.  It requires significant work to maintain features through new releases, and reducing accidental complexity is an important goal for the Riak community to ensure we can in the future continue to safely manage the codebase.

The initial release of 3.0 is unlikely to contain further features beyond that of the OTP uplift.  However, there are some areas of feature growth expected within the Riak 3.0 release cycle:

- Further extensions to the riak replication API, including the option of direct integration with the off-the-shelf queue technology for real-time replication, and other capabilities to ease replication from Riak to third party resources.

- Commence initial merging and consolidation of the Riak CS and Riak KV code bases, bringing the S3 api and super-cluster concepts to KV.

- Cloud optimisations within the leveled backend, the ability to backup ledger files to S3 as part of rolling them from the write state to the read state.  

- Some consolidation of capabilities between bitcask and leveled, so that where possible leveled capabilities such as hot_backup, snap_prefold and head_only requests can be supported in bitcask.

- Allow for throttled folds over ranges of keys and metadata for functional as well as non-functional reasons.

- An approach to timestamped objects, and timestamped reaping of tombstones, which is consistently implemented across backends and is complimentary to other features (such as anti-entropy).
