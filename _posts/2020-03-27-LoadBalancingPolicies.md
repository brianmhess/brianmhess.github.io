---
title: Load Balancing Policies and Coordinator Choice Ordering
layout: post
comments: true
tags:
 - Cassandra
---

Applications send queries to Cassandra through a DataStax driver, such as the DataStax Java driver. Due to the masterless architecture of Cassandra, all nodes in the cluster can respond to any query, regardless of whether that node is a replica of the appropriate data or not. This characteristic is one of the main differentiators of Cassandra over other database systems, and allows for high availability and scalability.

When an application sends a query to the cluster, the driver will route the query to a node to handle that request. We call the node that is handling the query as the coordinator for that query. The driver has a choice to make with respect to which node to use as the coordinator for a given query. This is one of the jobs of the Load Balancing Policy.

<!--excerpt-->


## Load Balancing Policies

The Load Balancing Policy is responsible for two primary responsibilities:
1. When the driver makes a connection to the cluster, the Load Balancing Policy will calculate a “distance” to each node in the cluster. If the distance to a node is LOCAL or REMOTE, then the driver will make a direct TCP connection (or more than one) to the node. If the distance is IGNORED, then no direct connection is made, and the node will never be used as a coordinator.
2. For a given query, the Load Balancing Policy will return an ordered list of nodes to use as the coordinator (note: in the 3.x driver it returns an iterator instead). There are some features of the DataStax drivers that will result in needing more than one coordinator, which is why the Load Balancing Policy supplies more than one node (either a list (4.x) or an iterator (3.x)). For example, if a Retry Policy is being used to automatically retry a query, then the driver will choose the next node from the list (or iterator).

The different Load Balancing Policies essentially come up with different approaches to these 2 questions, but mainly the second one, namely how to order the list of nodes to be used as the coordinator.


## Example Scenario
Let us consider a 10-node Cassandra cluster with two 5-node data centers. Nodes 1, 2, 3, 4, and 5 are in DC1 and nodes 6, 7, 8, 9, and 10 are in DC2. We are going to consider a table which has replication set up with 3 replicas in each data center. For discussion purposes, let us assume that nodes 1, 2, 3, 6, 7, and 8 are replicas of the data in question (e.g., the query could be something like `SELECT * FROM ks.tbl WHERE pkey=1`).


### Load Balancing Policy in the Cassandra 3.x driver

As we described above, the primary use for the Load Balancing Policy is to determine which host will act as the coordinator for the query. There are a number of strategies that can be taken to choose the coordinator that have different goals. For example:
* The RoundRobinPolicy attempts to spread out the coordinator duties evenly by simply going around the cluster and choosing each host in a round-robin fashion
* The TokenAwarePolicy attempts to prefer replicas over non-replica hosts, thereby removing one additional network hop (the hop from the coordinator to the replica (NOTE: even with a replica as the coordinator, the coordinator can still choose to not use itself in the case where that replica is slow for some reason)).
* The LatencyAwarePolicy observes the latency for each host it uses as coordinator and tries to avoid coordinators whose latency is notably (configurable) higher than the minimum observed latency.

The way that the Load Balancing Policy returns the choice of coordinator, in the 3.x driver, is by returning an iterator of hosts that the driver will iterate through. Why would the driver need more than one coordinator (as in, why return an iterator of hosts instead of a single host)? Well, in the case of retries or speculative execution you end up using more than one coordinator and the iterator will tell you how to choose the next host for this query.

Some Load Balancing Policies are chainable, that is, they will use a “child” policy and then “wrap” that policy with additional logic. For example, we could wrap the RoundRobinPolicy with the LatencyAwarePolicy. What you get is a convolution of the order of the hosts. So, it is important to understand how each policy will order the hosts before we then consider what happens when they are used in conjunction with each other.


#### Round Robin Policy

In the RoundRobinPolicy, the nodes are put in a canonical order and then the list is rotated by one on every query. So, for the first query, the order might look like:

| 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |

This would choose node 1 as the coordinator. If there was a retry or speculative execution, the second attempt would use node 2, and so on.

For the next query, the order would then be:

|---|---|---|---|---|---|---|---|---|---|
| 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 1 |

This would choose node 2 as the coordinator. If there was a retry or speculative execution, the second attempt would use node 3, and so on. Note that regardless of whether there was a retry or a speculative execution on the first query, the second query will start at node 2.

One upside here is that each node has equal use as coordinator which nicely spreads out the responsibility regardless of the partition(s) being read.

One downside is that this policy will use coordinators from all data centers, which might mean that the coordinator is physically located over the WAN from the client.

One interesting thing to consider is what happens when using the LOCAL consistency levels, such as LOCAL_QUORUM. Depending on which coordinator is used, a different set of replicas will be considered to satisfy the consistency level. In this example, the local replicas for nodes 1, 2, 3, 4, and 5 would be nodes 1, 2, and 3, while the local replicas for nodes 6, 7, 8, 9, and 10 would be nodes 6, 7, and 8.


#### DCAwareRoundRobinPolicy

In the DCAwareRoundRobinPolicy, the nodes are put in a canonical order just like in the RoundRobinPolicy, but there is a “local data center” configuration parameter and only nodes in that local data center will ever be chosen as coordinator. So, if the local data center was “DC1”, for example, for the first query, the order might look like:

| 1 | 2 | 3 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

This list would only have 5 options in it, namely the nodes from DC1.

For the next query, the order would then be:

| 2 | 3 | 4 | 5 | 1 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

The upside here over the RoundRobinPolicy is that only nodes in the local data center will be used, so if properly configured the client is never communicating with a coordinator across the WAN.

Since all nodes in consideration for being a coordinator are in the same data center, the nodes being used as replicas for consideration in the local consistency levels will always be the same. In this example, the replicas will always be nodes 1, 2, and 3.


#### ChainableLoadBalancingPolicy

ChainableLoadBalancingPolicy is an interface that specifies a category of Load Balancing Policies which wrap a “child” Load Balancing Policy. Only RoundRobinPolicy and DCAwareRoundRobinPolicy are not ChainableLoadBalancingPolicy. They can be used as the child policy for any of the following Load Balancing Policies.


#### WhiteListPolicy

The WhiteListPolicy is a simple policy where you can specify which nodes are allowed to be considered as coordinators by explicitly setting them when creating the policy. The WhiteListPolicy has a child policy and the order of that nodes is not altered, but any nodes not in the white list will be removed. For example, if we use the DCAwareRoundRobinPolicy with DC1 and wrap that with a WhiteListPolicy which only has nodes 1, 3, and 5 in the white list, the order of the nodes from the DCAwareRoundRobinPolicy would be:

| 1 | 2 | 3 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

but then we would only keep the nodes in the white list, namely nodes 1, 3, and 5, and would not alter the order in any other way, resulting in this list:

| 1 | 3 | 5 |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

For the next query, the list would from the DCAwareRoundRobinPolicy would be:

| 2 | 3 | 4 | 5 | 1 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

and again after applying the white list the resulting list would be:

| 3 | 5 | 1 |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|


#### HostFilterPolicy

The HostFilterPolicy is a simple policy where you can specify a predicate function that will keep or remove a given host from the list provided by the child policy. When you create the HostFilterPolicy you specify a Predicate<Host> that is applied to the list of nodes from the child policy. It will not reorder the list, but simply (possibly) remove some nodes from the list. An example of the HostFilterPolicy is the WhiteListPolicy.

For illustration purposes, let’s say that the predicate will return true for all nodes greater than “2” (so, nodes 3, 4, and 5) and false for other nodes. Then, if we use the DCAwareRoundRobinPolicy with DC1 as the child policy, the order of the nodes from the DCAwareRoundRobinPolicy would be:

| 1 | 2 | 3 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

but then we would only keep nodes greater than 2, resulting in this list:

| 3 | 4 | 5 |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

or the next query, the list from the DCAwareRoundRobinPolicy would be:

| 2 | 3 | 4 | 5 | 1 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

and again after applying the predicate the resulting list would be:

| 3 | 4 | 5 |   |   |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|


#### TokenAwarePolicy

In the TokenAwarePolicy, the nodes that are the replicas for the given query are brought to the front of the list. All non-replicas are kept in the order the child policy put them in. The order of the replicas can be configured:
* NEUTRAL - the order of the replicas is the order the child policy placed them in
* RANDOM - the replicas are randomized into a random order
* TOPOLOGICAL - the replicas will always come in the order that the ring placed them in

In any choice of the ordering, the replicas will be first in the list. For example, if we use DCAwareRoundRobinPolicy with DC1 as the child policy, then the order of the nodes from the child policy would be:

| 1 | 2 | 3 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

Let us also presume that the ring order of the replicas was 1, 2, and 3.

With the NEUTRAL ordering, the order of nodes would be:

| 1 | 2 | 3 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

With the RANDOM ordering, the order of nodes would be:

| 3 | 2 | 1 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

(Note: the ordering of the first 3 nodes in the list would be some permutation of the nodes 1, 2, and 3; the choice of 3, 2, and 1 was for illustrative purposes only).

With the TOPOLOGICAL ordering, the order of the nodes would be:

| 1 | 2 | 3 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

For the next call, the order of the nodes from the DCAwareRoundRobinPolicy would be:

| 2 | 3 | 4 | 5 | 1 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

With the NEUTRAL ordering, the order of the nodes would be:

| 2 | 3 | 1 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

Notice how node 1 moved from the end of the list to the third position.

With the RANDOM ordering, the order of the nodes would be:

| 1 | 3 | 2 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

Notice again how the three replicas move to the front of the list. (Note: the ordering of the first 3 nodes in the list would be some permutation of the nodes 1, 2, and 3; the choice of 1, 3, and 2 was for illustrative purposes only).

With the TOPOLOGICAL ordering, the order of the nodes would be:

| 1 | 2 | 3 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

Notice how the three replicas move to the front of the list, and they are in the natural ring order of the replicas.

If the DCAwareRoundRobinPolicy returned the following order (just for illustrative purposes):

| 5 | 1 | 3 | 2 | 4 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

then the NEUTRAL ordering would result in:

| 1 | 3 | 2 | 5 | 4 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|
 
the RANDOM ordering would result in:
 
| 3 | 2 | 1 | 5 | 4 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

(Note: again we are using a random ordering of the replicas; here it is 3, 2, and 1 for illustrative purposes only).

The TOPOLOGICAL ordering would result in:

| 1 | 2 | 3 | 5 | 4 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

One thing to note about the TOPOLOGICAL choice is that the replicas will always be in the same order regardless of what a child policy has done to the order of the replicas. This can also create a hotspot for a hot token range since that first replica for that token range will always be chosen first. In general, across an even workload of primary keys, this is less of a concern, and only a concern if there is skew in the workload.


#### LatencyAwarePolicy

The LatencyAwarePolicy observes the latency on each node when it is chosen as coordinator and will move slow nodes to the end of the list. There are a number of parameters which can be set to determine when a node is marked as slow:
* exclusion threshold - a multiplier of how much slower a node has to be on average than the fastest performing node. For example, if the fastest performing node has a latency of 10ms, then if we set the exclusion threshold to 2.5 then any node with average latency greater than 25ms will be deemed as slow. The default is 2.
* retry period - once a node is marked as slow, how long until it will be retried as a coordinator. We need to give a slow node a chance to demonstrate that it is no longer slow, so the retry period is how long a node will be “branded” as slow before being given a chance to be a coordinator again. The default is 10 seconds.
* update rate - how often will the latency for a node be updated.  The default is 100ms.
* minimum measurements - the minimum number of times a node must be used before it can branded as slow. That is, during the warmup period a certain number of uses of a node must occur before it is deemed slow. The default is 50.
* scale - this is a parameter that is used to bias newer latency statistics versus older latency statistics, in a decaying sort of way. The default is 100ms. The formula is given as:

> For a given host, if a new latency l is received at time t, and the previously calculated average is prev calculated at time t', then the newly calculated average avg for that host is calculated thusly:
```
  d = (t - t') / scale
  alpha = 1 - (ln(d+1) / d)
  avg = alpha * l + (1 - alpha * prev)
```

The order of the nodes, aside from moving slow nodes to the end of the list, is retained from the child policy.

For example, if we use DCAwareRoundRobinPolicy with DC1 as the child policy, and let us assume that nodes 4 and 1 are slow, then the order of the nodes from the child policy would be:

| 1 | 2 | 3 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

then the resulting order from the LatencyAwarePolicy would be:

| 2 | 3 | 5 | 1 | 4 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

For the next query, the order from DCAwareRoundRobinPolicy would be:

| 2 | 3 | 4 | 5 | 1 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

and the resulting order from the LatencyAwarePolicy would be:

| 2 | 3 | 5 | 4 | 1 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

For illustrative purposes, let’s consider the next query, where the order from DCAwareRoundRobinPoilcy would be:

| 3 | 4 | 5 | 1 | 2 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

and the resulting order from the LatencyAwarePolicy would be:

| 3 | 5 | 2 | 4 | 1 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|


#### Order of Chained Policies

##### LatencyAwarePolicy(TokenAwarePolicy(DCAwareRoundRobinPolicy))

In this order, after the DCAwareRoundRobinPolicy removes all non-local nodes and puts the local nodes in its round-robin order, the TokenAwarePolicy will move the replicas to the front of the list, and then the LatencyAwarePolicy will move slow nodes to the end of the list.

So, if we use as an example DCAwareRoundRobinPolicy with DC1, and we use replicas 1, 2, and 3, and we say that node 2 and node 4 are currently slow, then the DCAwareRoundRobinPolicy would give the following order:

| 4 | 1 | 3 | 2 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

the TokenAwarePolicy would move the replicas to the front of the list (let’s use NEUTRAL for the ordering):

| 1 | 3 | 2 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

and the LatencyAwarePolicy would move the slow nodes to the end:

| 1 | 3 | 5 | 4 | 2 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

Notice how one of the replicas, node 2, is not in the first 3 spots. In this ordering, node 5, a non-replica, would be preferred over node 2. This may be good, since node 2 is slow right now.


##### TokenAwarePolicy(LatencyAwarePolicy(DCAwareRoundRobinPolicy))

In this order, after the DCAwareRoundRobinPolicy removes all non-local nodes and puts the local nodes in its round-robin order, the LatencyAwarePolicy will move slow nodes to the end of the list, and then the TokenAwarePolicy will move the replicas to the front of the list.

So, using the same example, then the DCAwareRoundRobinPolicy would give the following order:

| 4 | 1 | 3 | 2 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

the LatencyAwarePolicy would move the slow nodes to the end:

| 1 | 3 | 5 | 4 | 2 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

and the TokenAwarePolicy would move the replicas to the front of the list (let’s use NEUTRAL for the ordering):

| 1 | 3 | 2 | 5 | 4 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

Notice how node 2 is in the third spot, but is slow. On the third execution (second retry, for example), node 2, which is slow, would be chosen for the coordinator. This may not be good since node 2 is slow right now.


#### Recommendation (Cassandra driver version 3.x)

In general, it is good to avoid slow nodes, even if they are replicas. For that reason, it is recommended to use the LatencyAwarePolicy(TokenAwarePolicy(DCAwareRoundRobinPolicy)) by default. For the LatencyAwarePolicy, leaving the default parameters is a good place to start.


### Load Balancing Policy in the Cassandra 4.x driver

Load balancing has been greatly simplified in the Cassandra 4.x driver. There is really only one Load Balancing Policy to consider, the DefaultLoadBalancingPolicy and there are only two parameters you can set on it:
* the local data center, which is required, and will restrict the set of nodes that can be chosen as the coordinator to only nodes in the specified data center (just like DCAwareRoundRobinPolicy in version 3.x)
* an optional Predicate<Node> class which can filter which nodes can be chosen as the coordinator (just like the HostFilterPolicy in version 3.x)

This DefaultLoadBalancingPolicy has embedded in it the concepts of data center local nodes (similar to the DCAwareRoundRobinPolicy in version 3.x), token awareness (similar to the TokenAwarePolicy in version 3.x), and latency performance (similar in concept, but significantly different in design, to the LatencyAwarePolicy in version 3.x).

The easiest way to set the host filter is to use the SessionBuilder#withNodeFilter() method. If you had implemented a class that implements the Predicate<Node> interface and included it in your application, you can set it in the configuration file by setting the datastax-java-driver.basic.load-balancing-policy.filter.class to the name of your class.


#### DefaultLoadBalancingPolicy

The general flow of the DefaultLoadBalancingPolicy in determining the order of the nodes to be considered as coordinator is as follows:
1. start with all known nodes in the list; this list of known nodes will already have filtered out any nodes not in the local data center as well as any nodes that do not pass the supplied host filter, if one was supplied
2. move all replicas to the front of the list
3. randomly order the replicas, but leave the order for the non-replicas
4. for each replica, if it has been up for a while and is unhealthy, then put it at the back of the replicas (not the back of the whole list)
5. if the node in the first or second spot is a replica that has recently come online, then flip a coin and 1/4 of the time leave it where it is and 3/4 of the time move it to the back of the replicas (not the back of the whole list)
6. look at the number of in-flight requests in each of the first two nodes in the list and reorder those two so the one with the fewest in-flight requests is first
7. rotate the non-replicas to mimic a round-robin behavior (subsequent calls will rotate one more time than the last)

So, if we consider our 2-DC cluster and use:
* DC1 as the local data center
* nodes 1, 2, 3, 6, 7, 8 as the replicas
* node 1 and node 4 have been up for a while, but are unhealthy
* node 1 has 5 outstanding requests, node 2 has 20 outstanding requests, node 3 has 30 outstanding requests, node 4 has 10 outstanding requests, and node 5 has 40 outstanding requests

Then going through our process outlined above we would start by looking at all live nodes that are in the local data center:

| 1 | 2 | 3 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

Then we move all replicas to the front of the list (they’re already at the front, so this doesn’t actually do anything in this example):

| 1 | 2 | 3 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

Then we randomly permute the replicas, leaving the non-replica order the same (for illustrative purposes, let’s say that the permutation of (1,2,3) is (3,1,2)):

| 3 | 1 | 2 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

Since node 1 has been up for a while and is unhealthy, it is moved to the back of the list of replicas (spot 3). Node 4 has also been up for a while and is unhealthy, but since it is not a replica, it stays where it is:

| 3 | 2 | 1 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

We now look at the first two nodes in the list (nodes 2 and 3) and order them so that the node with fewer outstanding requests is first. Since node 2 and node 3 have 20 and 30 outstanding requests, respectively, we put node 2 before node 3:

| 2 | 3 | 1 | 4 | 5 |   |   |   |   |   |
|---|---|---|---|---|---|---|---|---|---|

The next call would have an identical path as above, but would have a different random permutation of replicas (for illustrative purposes, let’s assume the same permutation is used, namely (3,1,2)) and the non-replicas would rotate by one more, resulting in:

| 2 | 3 | 1 | 5 | 4 |   |   |   |   |   |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |


#### BasicLoadBalancingPolicy

There is another Load Balancing Policy available, namely the BasicLoadBalancingPolicy, but it is mainly a reference policy, though it does provide some basic capabilities and the DefaultLoadBalancingPolicy does derive from it. The BasicLoadBalancingPolicy does the following:
1. start with all known nodes in the list; this list of known nodes will already have filtered out any nodes not in the local data center as well as any nodes that do not pass the supplied host filter, if one was supplied
2. move all replicas to the front of the list
3. randomly order the replicas, but leave the order for the non-replicas
4. rotate the non-replicas to mimic a round-robin behavior (subsequent calls will rotate one more time than the last)

There is little reason to use the BasicLoadBalancingPolicy.


#### DcInferringLoadBalancingPolicy

There is one more Load Balancing Policy, but it is not well documented and intended to be not well publicized, namely the DcInferringLoadBalancingPolicy.

Both the DefaultLoadBalancingPolicy and the BasicLoadBalancingPolicy require that the local data center be specified. This is one big difference between the new Load Balancing Policies and the DCAwareRoundRobinPolicy in version 3.x. 

In the DCAwareRoundRobinPolicy, the local data center can either be specified or inferred. The way that the local data center is inferred is that the driver will go through the supplied contact points in order, and upon first successful connection the driver will determine the data center of this first successful contact point and use that as the local data center. It is possible to supply multiple contact points in the 3.x driver, and if those contact points come from different data centers, it is possible that the DCAwareRoundRobinPolicy could be configured automatically with different inferred local data centers.

In the DcInferringLoadBalancingPolicy in version 4.x, if the local data center has been supplied explicitly, then that will be used as the local data center. Otherwise, the driver will look at the supplied contact points and if they all come from the same data center then that common data center will be used as the local data center. If the contact points come from different data centers, an error will be thrown (which is different behavior than in the 3.x driver).

Use of this Load Balancing Policy is discouraged, but it is necessary/useful when building certain tools, libraries, etc, such as the Spark Cassandra Connector.


#### Recommendation (Cassandra driver version 4.x)

The recommended Load Balancing Policy in version 4.x is the DefaultLoadBalancingPolicy with an explicit local data center specified. If you need/want to further restrict the coordinator choice, you can also supply a host filter, but that is usually not necessary.

