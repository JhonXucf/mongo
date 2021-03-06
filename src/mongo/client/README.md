# Internal Client

## Replica set monitoring and host targeting
The internal client driver responsible for routing a command request to a replica set must determine which member to target. Host targeting involves finding which nodes in a topology satisfy the $readPreference. Node eligibility depends on the type of a node (i.e primary, secondary, etc.) and its average network latency round-trip-time (RTT). For example, { $readPreference: secondary } requires the client to know which nodes are secondaries and, of those nodes, which nodes fit within a delta of the node with the smallest RTT. A FailedToSatisfyReadPreference error occurs when there is a host selection time out and no eligible nodes are found. 

Nodes in a topology are discovered and monitored through replica set monitoring. Replica set monitoring entails periodically refreshing the local view of topologies for which the client needs to perform targeting. The client has a ReplicaSetMonitor for each replica set it needs to target in the cluster. So, if a mongos needs to target 2 shards for a query, it either has or creates a ReplicaSetMonitor for each of the the corresponding shards.

The ReplicaSetMonitorInterface supports two replica set monitoring protocols. The older "scanning" protocol involves regularly performing sequential scans over the nodes in a topology, waiting some period between each scan. A scan re-checks each node by calling isMaster. Through the isMaster responses, the client's perception of the topology is refreshed. The "scanning" protocol is off by default but remains implemented through the ScanningReplicaSetMonitor.

The new replica set monitoring protocol supports the "awaitable isMaster" command feature and abides by the Server Discovery and Monitoring (SDAM) specifications. The "awaitable isMaster" command feature allows the isMaster command to wait for a significant topology change or timeout before replying. Two different versions of the new protocol are supported - "sdam", which does not support awaitable isMaster with exhaust, and "streamable", which does support exhaust and is on by default. Clients who enable the awaitable isMaster (with or without exhaust) will learn much sooner about stepdowns, elections, reconfigs, and other events.

In the new protocol, the StreamableReplicaSetMonitor is used to gather and maintain information regarding the client's local topology description. The topology description holds the learned states of each member in the replica set. Since the new protocol supports exhaust, the RTT is measured by sending a 'ping' to each node in the topology at a fixed frequency rather than through the isMaster response latency. Aside from the RTT, the remaining information for satisfying read preferences is gathered through awaitable isMaster commands asynchronously sent to each node in the topology.


#### Code references
* [**Read Preference**](https://docs.mongodb.com/manual/core/read-preference/)
* [**ReplicaSetMonitorInterface class**](https://github.com/mongodb/mongo/blob/v4.4/src/mongo/client/replica_set_monitor_interface.h)
* [**ReplicaSetMonitorManager class**](https://github.com/mongodb/mongo/blob/v4.4/src/mongo/client/replica_set_monitor_manager.h)
* [**RemoteCommandTargeter class**](https://github.com/mongodb/mongo/blob/v4.4/src/mongo/client/remote_command_targeter.h)
* [**ServerDiscoveryMonitor class**](https://github.com/mongodb/mongo/blob/v4.4/src/mongo/client/server_discovery_monitor.cpp)
* [**ServerPingMonitor class**](https://github.com/mongodb/mongo/blob/v4.4/src/mongo/client/server_ping_monitor.h)
* The specifications for
[**Server Discovery and Monitoring (SDAM)**](https://github.com/mongodb/specifications/blob/master/source/server-discovery-and-monitoring/server-discovery-and-monitoring.rst)
* [**TopologyDescription class**](https://github.com/mongodb/mongo/blob/v4.4/src/mongo/client/sdam/topology_description.h)
---
