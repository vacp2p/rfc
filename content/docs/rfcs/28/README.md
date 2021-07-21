---
slug: 28
title: 28/STATUS-FEATURING
name: Status community featuring using waku v2
status: raw
tags: waku-application
editor: Szymon Szlachtowicz <szymon.s@ethworks.io>
---

This spec is a proposition of voting on community featuring over waku v2.

# Overview

When there is a active community that is seeking new members, current users of community should be able to feature their community so that it will be accessible to larger audience.
Status community curation DApp should provide such a tool.

Rules of featuring:
    - Given community can't be featured twice in a row.
    - Only one vote per user per community (single user can vote on multiple communities)
    - Voting will be done off-chain
    - If community hasn't been featured votes for given community are still valid for the next 4 weeks

Since voting for featuring is similar to polling solutions proposed in this spec could be also used for different applications.

# Voting

Voting for featuring will be done through waku v2. 

Payload of waku message will be :
```ts
type FeatureVote = {
    voter: string // address of a voter
    sntAmount: BigNumber // amount of snt voted on featuring
    communityPK: string // public key of community
    timestamp: number // timestamp of message, must match timestamp of wakuMessage
    sign: string // cryptographic signature of a transaction (signed fields: voterAddress,sntAmount,communityPK,timestamp)
}
```

timestamp is necessary so that votes can't be reused after 4 week period

# Counting Votes

Votes will be counted by the DApp itself.
DApp will aggregate all the votes in the last 4 weeks and calculate which communities should be displayed in the Featured tab of DApp.

Rules of counting:
    - When multiple votes from the same address on the same community are encountered only the vote with highest timestamp is considered valid.
    - If a community has been featured in a previous week it can't be featured in current week.
    - In a current week top 5 (or 10) communities with highest amount of SNT votes up to previous Sunday 23:59:59 UTC are considered featured.

# Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).

