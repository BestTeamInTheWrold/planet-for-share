---
layout: post
title: "AWS SQS SNS"
description: "AWS Simple Notification Service, Simple Queue Service,"
---
### SQS & SNS

#### SQS:
Simple Queue Service, used to decouple the components of a cloud application, a buffer. (e.g. spike in traffic)

##### Features：
1. deliver each message at least once(not exactly once), you should design your system to be idempotent, 多次处理同一消息不应该造成影响。
2. 不保证顺序（不一定是先进先出），但自己可以给消息增加顺序相关信息，然后收到后自行reorder。
3. Visibility timeout：接受到这个消息后，这个消息被从queue里隐藏掉的时间，这个时间内其他component是读不到这个消息的(默认值30s，最大12hours)。
4. Delay queue：推迟message能被从queue里读到的时（默认为0，可以设置0-15mins）。
5. Queue size limit: 120，000 messages in flight。
6. Identifiers: queue URLs, message IDs, receipt handles(接受消息的时候会收到，删除消息的时候要用)
7. message attributes: metadata items: timestamps, geospatial data, signatures, identifiers. 可以根据这些值判断消息体需不需要被处理。（最多10个attributes）
8. Long Polling: loop取消息但是队列里没有消息的情况下，为了避免一直循环，减少客户端负担，可以在receiveMessage的时候传waiTimeSeconds参数，最大设置20s. 如果有消息直接返回，如果没有，会等waiTimeSeconds，知道有数据返回或者等待时间到返回。
9. DeadLetterQueue，链接在某个queue上，存放处理失败的消息，以后分析和处理。
10. AccessControl：Queue Policy(可以直接给另外一个account赋权限)
11. Retention Period: 消息在队列里被保留的时间，过了这个时间就删除。默认是4天，可以设置60s到14天。

#### SNS：
Simple Notification Service

发布者-订阅者模式
publisher -> topic -> subscriber

Scenarios:
1. Fanout
2. Application and system alerts
3. Push email and text messaging
4. Mobile push notifications
