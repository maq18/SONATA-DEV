# Sonata: Query-Driven Streaming Network Telemetry

## Overview
Managing and securing networks requires collecting and analyzing measurement
data. Current telemetry systems do not make it easy to do so, typically because 
they separate data collection (e.g., packet capture or flow monitoring) from 
analysis, producing either too much data to answer a general question, or too
little data to answer a detailed question. 

[Image](images/architecture.eps?raw=true)
Sonata is a streaming telemetry system that makes use of programmable data 
plane and scalable streaming processor for scalability.  
The above figure shows Sonata's design: It allows operators to directly express 
queries in a high-level declarative language without worrying about how and 
where the query gets executed. Under the hood, it partitions each query into
a portion that runs on the switch and another that runs on the streaming
analytics platform. For each query, the data plane first processes the packet
before emitting the intermediate result to the stream processor. Sonata's
runtime then uses the result of each query to refine the subsequent packet
processing.

Sonata offers three key contributions:
1. Declarative Query Interface 
2. Query Partitioning 
3. Dynamic Refinement

## Declarative Query Interface
It allows network operators to apply intuitive dataflow operators over 
arbitrary combinations of packet fields. 

### Packet Fields

### Dataflow Operators

| Operator| Description|
| ---------------------| -----------|
| Map(keys, map_keys, map_values,func)| Transform each tuple with function `func` applied over tuples identified by key(`map_keys`) or value (`map_values`) fields.|
| filter(filter_keys, filter_values,func)| Filter packets that satisfy predicate `func` applied over set of key (`filter_keys`) or value (`filter_vals`) fields|
| reduce(keys,func)| Emit result of function `func` applied on key (`keys`) over the input stream.|
| distinct(keys)| Emit tuples with unique combinations of key (`keys`) fields.|
| join(window, new_qid, query)| Join the output of `query` and assign `new_qid` to the resulting joined query. Also, specify whether the `join` operation is applied over the same window interval or over rolling windows.|



### Example Queries

#### Newly Opened TCP Connections
To detect a large number of newly opened TCP connections, Query~\ref{tcp-conn}
first applies a `filter` operation over the entire packet stream to
select TCP packets with just the `SYN` flag set. It then counts the
number of packets observed for each host and reports the hosts
for which this count exceeds threshold `Th` in an epoch.
```python
n_syn = (PacketStream(1)
         .filter(filter_keys=('tcp.flags',), func=('eq', 2))
         .map(keys=('ipv4.dstIP',), map_values=('count',), func=('eq', 1,))
         .reduce(keys=('ipv4.dstIP',), func=('sum',))
         .filter(filter_vals=('count',), func=('geq', Th))
         )
```

#### Slowloris Attacks
The application to detect a slowloris attack has two sub-queries: One
counts the number of unique connections by applying a `distinct` followed 
by a `reduce` operation. It only reports hosts for which the number of 
connections exceeds threshold `Th1`. The other counts the total bytes
transferred for each host. It then joins these sub-queries to compute 
the average transmission rate per connection and reports hosts with an 
average rate below a threshold `Th2`.
```python
n_conns = (PacketStream(1)
           .filter(filter_keys=('ipv4.protocol',), func=('eq', 6))
           .map(keys=('ipv4.dstIP', 'ipv4.srcIP', 'tcp.sport',))
           .distinct(keys=('ipv4.dstIP', 'ipv4.srcIP', 'tcp.sport',))
           .map(keys=('ipv4.dstIP',), map_values=('count',), func=('eq', 1,))
           .reduce(keys=('ipv4.dstIP',), func=('sum',))
           .filter(filter_vals=('count',), func=('geq', Th1))
           )

n_bytes = (PacketStream(2)
           .filter(filter_keys=('ipv4.protocol',), func=('eq', 6))
           .map(keys=('ipv4.dstIP', 'ipv4.totalLen',))
           .map(keys=('ipv4.dstIP',), map_values=('ipv4.totalLen',))
           .reduce(keys=('ipv4.dstIP',), func=('sum',))
           )

slowloris = (n_conns.join(window='Same', new_qid=3, query=n_bytes)
             .map(map_values=('count2',), func=('div',))
             .filter(filter_keys=('count2',), func=('leq', Th2))
             )
```

#### Zorro Attacks
This application also has two sub-queries.
The first part identifies the hosts that receive more than `Th1`
similar-sized telnet packets. It then joins the output of the first 
sub-query with the other over rolling window interval and reports 
hosts that receive more than `Th2` packets with the keyword `zorro` 
in the payload.
```python
brute_telnet = (PacketStream(1)
             .filter(filter_keys=('proto',), func=('eq', 6))
             .map(keys=('ipv4.dstIP', 'ipv4.srcIP', 'ipv4.totalLen'))
             .distinct(keys=('ipv4.dstIP', 'ipv4.srcIP', 'ipv4.totalLen'))
             .map(keys=('ipv4.dstIP', 'ipv4.totalLen'))
             .map(keys=('ipv4.dstIP', 'ipv4.totalLen'), map_values=('count',), func=('eq', 1,))
             .reduce(keys=('ipv4.dstIP', 'ipv4.totalLen'), func=('sum',))
             .filter(filter_vals=('count',), func=('geq', Th1))
             .map(keys=('ipv4.dstIP',))
             )

payload = (PacketStream(2)
          .map(keys=('ipv4.dstIP', 'payload'))
          )

zorro_attack = (payload.join(new_qid=3, query=brute_telnet)
                 .filter(filter_vals=('payload',), func=('eq', 'zorro'))
                 .map(keys=('ipv4.dstIP', 'payload',), map_values=('count',), func=('eq', 1))
                 .reduce(keys=('ipv4.dstIP', 'payload',), func=('sum',))
                 .map(keys=('ipv4.dstIP',))
                 )
```

## Query Partitioning
Reducing the workload on the stream processor requires
determining how to partition input queries between the data-plane and the
streaming  targets.  Sonata's query planner models the decision problem as 
an integer linear program (ILP), which it solves using historical packet traces 
to minimize the load at the stream processor for a set of queries subject to 
the constraints of PISA data-plane targets (e.g., the amount of register memory 
in the switch).

## Dynamic Refinement
Data-plane targets should not waste limited resources, such as memory, on 
portions of the traffic that do not satisfy the queries. By solving an extended 
version of the original ILP for query partitioning, Sonata's query planner 
determines how to progressively zero in on traffic over time (i.e., 
window-by-window, for a fixed-size window duration). Dynamic refinement reduces 
the load on the stream processor at the cost of additional delays of possibly 
multiple time windows to identify the traffic that satisfies the queries.