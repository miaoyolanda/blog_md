---
title: "WeCron是怎样处理定时任务的"
date: 2018-08-08T21:11:37+08:00
keywords: ["定时", "定时提醒", "微信", "任务队列", "schedule", "cron"]
---

[WeCron](http://wecron.betacat.io/)（微定时）是我开发的一个微信上的定时提醒机器人，它能解析用户输入的语音或者文字，提取其中的时间和事件信息，然后为用户设置提醒。这个服务上线后，经常有用户问我这里定时提醒的实现，因此这篇文章我就打算来谈谈WeCron是怎么实现这个功能的，希望大家以后设计定时调度类系统时能多一个参考。

{{< figure src="https://user-images.githubusercontent.com/2657334/34242455-7c9ae230-e656-11e7-8420-3da003d87ce5.jpeg" width="200px" caption="通过微信设置定时提醒">}}

# 架构

下面是定时提醒部分的架构图：

{{< figure src="/image/wecron-schedule/wecron-schedule-architecture.png" caption="WeCron定时功能架构图" data-edit="https://whimsical.co/Q5KsUWANLBsoz3ks6LTRUg">}}

整个流程还是比较简单的：

1. 用户输入一条提醒语音后，`parser`会把其中的时间和事件信息解析出来，存到数据库中；
2. 同时，定时调度器会被触发，它到数据库中找到一个最近的提醒，计算出到当前时间的时间差，然后sleep相应的秒数；
3. 过段时间，调度器醒来，它会将最近一段时间要发送的提醒任务提交到一个队列里面，由专门发送消息的worker向用户推送消息。这里使用队列的目的，一方面是消峰，另一方面是为了让发送消息的worker能够横向扩展。

# 定时调度器

定时提醒类系统有一个特点，就是它的峰值特别明显。用户设置的提醒大多会集中在整点时间触发，尤其是早晨8、9点。面对这样一个不均衡的分布，固定时间轮训的调度法就显得有些naïïve，它很难在运行效率和提醒的及时性中找到一个平衡点。因此，WeCron中定时调度器被设计为一个事件循环，它能响应两种事件：

1. 数据库有更新或者插入操作
2. 上一次设置的休眠时间到了

每当这个调度器被唤醒，它会首先检查有没有提醒要发送，然后再找到最近一个将要触发的提醒，休眠相应的时间。下面是一个简短的实现：

```python

remind_event = threading.Event()

def event_loop():
    while True:
        wait_seconds = process_jobs()
        # This event wakes up on timeout or someone else sets it
        remind_event.wait(wait_seconds)

def on_remind_update():
    # This will wake up the remind_event
    remind_event.set()

def process_jobs():
    # Send reminds
    reminds = get_unfired_reminds_older_than_now()
    send_notification(reminds)

    # And get the most recent remind
    next_remind = get_unfired_reminds_newer_than_now().limit(1)
    return (next_remind.notify_time - now).seconds
```

另外，为了提高提醒的及时性，这里的调度器会被启动好几份。这时面临的一个问题就是同一个提醒可能会被多个调度器调度到，用户也就有可能重复收到提醒。所以WeCron在选择提醒项的时候使用了`select ... for update`语句，利用（行）锁来同步多个调度器。

# 结束

好了，我就先简单的介绍到这里，希望可以帮助到有需要的人。如果想要看到具体实现，请参考WeCron的[代码库](https://github.com/polyrabbit/WeCron)。如果想要体验这个定时功能，也欢迎扫码关注:stuck_out_tongue_closed_eyes:：
{{< figure src="https://camo.githubusercontent.com/6f87c83d4bb324babcf1fd94751cf1ead16be13e/687474703a2f2f7778332e73696e61696d672e636e2f6d773639302f61633437323334386c793166696c646438686d677a6a323037363037366467612e6a7067" width="150px">}}