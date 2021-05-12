---
slug: 21
title: 21/WAKU2-FAULT-TOLERANT-STORE
name: Waku v2 Fault-Tolerant Store
status: raw
editor: Sanaz Taheri <sanaz@status.im>
contributors:
---

 The reliability of `13/WAKU2-STORE` protocol heavily relies on the fact that full nodes i.e., those who persist messages have high availability and uptime and do not miss any messages. 
 If a node goes offline, then it will risk missing all the messages transmitted in the network during that time. 
 In this specification, we provide a method that makes the store protocol resilient in presence of faulty nodes. 
 Relying on this method,  nodes that have been offline for a time window will be able to fix the gap in their message history when getting back online. 
 Moreover, nodes with lower availability and uptime can leverage this method to reliably provide the store protocol services as a full node.

# Method description 
 As the first step towards making the `13/WAKU2-STORE` protocol fault-tolerant, we introduce a new type of time-based query through which nodes fetch message history from each other based on their desired time window. 
 This method operates based on the assumption that the querying node knows some other nodes in the store protocol which have been online for that targeted time window.  

# Security Consideration

The main security consideration to take into account while using this method is that a querying node has to reveal its offline time to the queried node. 
This will gradually result in the extraction of the node's activity pattern which can lead to inference attacks. 

# Wire Specification
We extend the [HistoryQuery](/spec/13#payloads) protobuf message with two fields of `startTime` and `endTime` to signify the time range to be queried. 

## Payloads

```diff
syntax = "proto3";

message HistoryQuery {
  // the first field is reserved for future use
  string pubsubtopic = 2;
  repeated ContentFilter contentFilters = 3;
  PagingInfo pagingInfo = 4;
  + double startTime = 5;
  + double endTime = 6;
}

```
  
### HistoryQuery

RPC call to query historical messages.
- `startTime`: this field MAY be filled out to signify the starting point of the queried time window. 
  This field holds the Unix epoch time.  
  The `messages` field of the corresponding [`HistoryResponse`](/spec/13#HistoryResponse) MUST contain historical waku messages whose [`timestamp`](/spec/14#Payloads) is larger than or equal to the `startTime`.
- `endTime` this field MAY be filled out to signify the ending point of the queried time window. 
  This field holds the Unix epoch time. 
  A time-based query is considered valid if its `endTime` is larger than or equal to the `startTime`. 
  The `messages` field of the corresponding [`HistoryResponse`](/spec/13#HistoryResponse) MUST contain historical waku messages whose [`timestamp`](/spec/14#Payloads) is less than or equal to the `endTime`.

If both `startTime` and `endTime` are omitted then no time-window filter takes place. 
Note that `HistoryQuery` preserves `AND` operation among the queried attributes. 
As such,  The `messages` field of the corresponding [`HistoryResponse`](/spec/13#HistoryResponse) MUST contain historical waku messages that satisfy the indicated  `pubsubtopic` AND `contentFilters` AND the time range [`startTime`, `endTime`]. 

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).