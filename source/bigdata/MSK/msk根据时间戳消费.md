---
title: msk根据时间戳消费
categories: MSK
tags: 
    - kafka
---

```bash
%%The result of testing on my side, there is no direct command to consume historical data directly by timestamp.
If you need to re-consume the historical data you need to put the data to be consumed in a new consumer group and then consume.%%

$ . /kafka-consumer-groups.sh --bootstrap-server $str --group group group_test12 --topic MSKTutorialTopic --reset-offsets --to-datetime 2022-08-19T00 :00:00.000 -execute

GROUP TOPIC PARTITION NEW-OFFSET
group_test12 MSKTutorialTopic 0 121
$ . /kafka-console-consumer.sh --topic MSKTutorialTopic --bootstrap-server $str --group group group_test12              

group_test12 is the name of the new consumer group and MSKTutorialTopic is the topic that needs to be repeatedly consumed.


```
