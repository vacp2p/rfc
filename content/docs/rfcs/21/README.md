---
slug: 21
title: 21/WAKU2-FAULT-TOLERANT-STORE
name: Waku v2 Fault-Tolerant Store
status: draft
editor: Sanaz Taheri <sanaz@status.im>
contributors:
---

 Reliability of `13/WAKU2-STORE` protocol heavily relies on the fact that full nodes i.e., those who persist messages being always online and not missing any messages. In this specification we provide a method to alleviate this high availability and uptime requirements as well as make store protocol resilient in presence of faulty nodes. Relying on this method, full nodes can go offline for some time and still be able to fix the gap in their message history when get back online. Another advantage would be that nodes with lower availability and uptime would be able to participate in the store protocol as a full node and reliably provide store protocol services.

# Method description 
 As the first step towards fault-tolerant `13/WAKU2-STORE` protocol, we extend it with a new type of time-based query through which nodes fetch message history from each others based on a time window. This method operates based on the assumption that the querying node is aware of a list of other store nodes which have been online for the intended period of time.  

# Security Consideration

The main security consideration to take into account while using this method is that a querying node have to reveal its offline time to the queried node. This will gradually result in the extraction of node's activity pattern which can lead to inference attacks. 

# Wire Specification
We introduce two fields of `StartTime` and `EndTime` to the [HistoryQuery](/spec/13#payloads) protobuf to signify the time range to be queried. 

## Payloads

```diff
syntax = "proto3";

message HistoryQuery {
  // the first field is reserved for future use
  string pubsubtopic = 2;
  repeated ContentFilter contentFilters = 3;
  PagingInfo pagingInfo = 4;
  + double StartTime = 5;
  + double EndTime = 6;
}

```
  
### HistoryQuery

RPC call to query historical messages.
- `StartTime`: this field MAY be filled out to signify the starting point of the queried time window. 
  This field holds the Unix epoch time.  
  The `messages` field of the corresponding [`HistoryResponse`](/spec/13#HistoryResponse) MUST contain historical waku messages whose [`timestamp`](/spec/14#Payloads) is larger than or equal to `StartTime`.
- `EndTime` this field MAY be filled out to signify the ending point of the queried time window. 
  This field holds the Unix epoch time. 
  For a valid time-based query, `EndTime` is larger than or equal to `StartTime`. 
  The `messages` field of the corresponding [`HistoryResponse`](/spec/13#HistoryResponse) MUST contain historical waku messages whose [`timestamp`](/spec/14#Payloads) is less than or equal to `StartTime`.

If both `StartTime` and `EndTime` are ommited then no time-window filter takes place. 
Note that `HistoryQuery` preserves `AND` operation among the queried criteria which are valid. As such,  The `messages` field of the corresponding [`HistoryResponse`](/spec/13#HistoryResponse) MUST contain historical waku messages that satisfy the queried  `pubsubtopic` AND `contentFilters` AND the time range [`StartTime`, `EndTime`]. 

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
