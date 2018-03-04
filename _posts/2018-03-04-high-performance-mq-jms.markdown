---
layout: post
title:  "High performance IBM MQ & JMS applications"
date:   2018-03-04 21:56:00
categories: Java
background: "/images/mq-jms.jpg"
language: en
---

Using the JMS API to do messaging over IBM MQ is rather easy, but writing programs that 
perform well can be a bit tricky. In this post, I'm going to share a few tips I've gathered here
and there that might help you write faster MQ+JMS applications.

My examples are based on MQ 9 and JMS 2.0 but most of the tips will likely work with
other versions of these products, and some of them can even apply to other message brokers.

<!-- more -->

## Tuning the client part

### Cache stuff

The first thing to do is make sure that you are reusing JMS objects as much as possible. 
Creating a connection or a session usually requires a network exchange, which means it can
introduce a significant overhead. While the creation of producers and consumers has less
overhead, it's good to know that these objets were *designed to be reused*, so you might as
well take advantage of this and cache/reuse instances as much as possible.

If you are using Spring JMS, [CachingConnectionFactory](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jms/connection/CachingConnectionFactory.html)
can wrap your connection factory to reuse a single connection, cache sessions and producers.

If your code is running in a Java EE application server, chances are that [connections and sessions
are already automatically cached](https://developer.jboss.org/wiki/ShouldICacheJMSConnectionsAndJMSSessions?_sscc=t),
and calling `close()` on one of these objects will put it back in a pool.

### Use non-persistent messages

Performance often comes at the cost of reliability. Depending on the requirements your
application has, it can be acceptable to lose messages if the queue manager crashes. In that
case, your application might expect a response to the lost message, not receive it and decide
to send it again. This kind of scenario is most likely privileging performance over reliability,
while still having a plan B in case something doesn't go as expected.

To improve performance in such scenarios, it is best to use non-persistent messages. Persistent
messages are used by queue managers to *guarantee message delivery*. In practice, this means that a
message will be synchronously written to disk *at the same time it is sent* by the producer. As you can guess, the
previous sentence makes it clear that persistent messages impact performance.

Now, to debunk a common misconception: **there is no such thing as a persistent queue**. Persistency
is configured *per message*, and if a given message has no `Persistence` field set in its message
descriptor (MQMD), then [a queue can define a **default** persistence flag](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.0.0/com.ibm.mq.dev.doc/q023070_.htm).

To send non persistent messages, [specify a `DeliveryMode` when using a producer](https://docs.oracle.com/cd/E19798-01/821-1841/bncfy/index.html):

{% highlight java %}
// on the whole producer
producer.setDeliveryMode(DeliveryMode.NON_PERSISTENT);

// on a single message
producer.send(message, DeliveryMode.NON_PERSISTENT, 3, 10000);
{% endhighlight %}

Be aware that [JMS' default mode is `DeliveryMode.PERSISTENT`](https://docs.oracle.com/javaee/7/api/javax/jms/MessageProducer.html#setDeliveryMode-int-),
which means that even if you define your queue's default persistence as “not persistent”, 
JMS will not care and send persistent messages when calling `producer.send(message)`.

If you are using [Spring JMS' JmsTemplate](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/jms/core/JmsTemplate.html),
you can enable explicit QOS and set a default delivery mode:

{% highlight java %}
@Bean
public JmsTemplate jmsTemplate() throws NamingException {
    JmsTemplate jmsTemplate = new JmsTemplate();

    jmsTemplate.setConnectionFactory(queueConnectionFactory());
    jmsTemplate.setExplicitQosEnabled(true);
    jmsTemplate.setDeliveryPersistent(false);

    return jmsTemplate;
}
{% endhighlight %}

If you are using [a `.bindings` file to look up JMS objects from a JNDI context](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.0.0/com.ibm.mq.con.doc/q123210_.htm),
you can configure JMSAdmin to set a default delivery mode on producers using the `persistence` property:

{% highlight text %}
define q(MY.JNDI.Q) queue(MY_Q) qmgr(MY_QMGR) persistence(NON)
{% endhighlight %}

### Lose the transaction

JMS `Session`s can be transactional, but this has a cost. If your application does not need to do
multiple operations in a single atomic transaction, then you can make your code a bit faster using
non-transactional sessions:

{% highlight java %}
// transactions are configured using the first parameter
Session session = connection.createQueueSession(false, Session.AUTO_ACKNOWLEDGE);
{% endhighlight %}

### Change acknowledgement modes

When using non-transactional sessions, the queue manager has to find an alternative to `commit()` to make
sure that a message was successfully consumed by a client application. This is achieved using aknowledgements.
When a message is acknowledged, the queue manager knows that the delivery has been guaranteed and that this message 
can be discarded. JMS supports several acknowledgement modes:

* In the default `AUTO_ACKNOWLEDGE` mode, a message will be automatically acknowledged when
`consumer.receive()` returns a message, of when a [`MessageListener`](https://docs.oracle.com/javaee/7/api/javax/jms/MessageListener.html)
was successfully executed on a message.

* In `CLIENT_ACKNOWLEDGE` mode, you have to manually call `message.acknowledge()`. If your application does
more work after having received a message, you could acknowledge it immediately to relieve the queue manager of its duty
and allow it to process the next message.

* In `DUPS_OK_ACKNOWLEDGE` mode, you tell the queue manager that it's okay for your application to reveive the same
message more than once. Now that the 'once and only once' constraint is gone, the queue manager might take a few shortcuts
to enhance performance, which could lead to a message being sent twice on rare occasions.

In theory, `DUPS_OK_ACKNOWLEDGE` should be the most performant mode, but in practice it means that you have to design
your application to support duplicate messages. This could add complexity on multi-threaded or multi-instances applications.

### Use read ahead and put async

[“Read ahead”](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.0.0/com.ibm.mq.dev.doc/q032570_.htm) 
is a concept specific to IBM MQ. When this functionality is enabled and you are using *non-transactional sessions*,
non-persistent messages can be pre-fetched in an internal buffer so that your application can consume the next message faster.

When you call the `receive()` method, the IBM MQ classes for JMS will fetch a list of messages, store them in
this internal buffer, and return the first message to your application. The next calls to `receive()` will then take
messages from this buffer instead of querying the queue manager. This can work because no transactions and non-persistent 
messages are involved, which means there is no hard requirement for guaranteed delivery and consistency.

Read ahead can be enabled on a queue like this:

{% highlight java %}
((MQQueue) myQueue).setReadAheadAllowed(CommonConstants.WMQ_READ_AHEAD_ALLOWED_ENABLED);
{% endhighlight %}

Similar to read ahead that allows faster `MQGET`s, IBM MQ has a concept of [“put async”](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.0.0/com.ibm.mq.dev.doc/q032560_.htm) that allows faster `MQPUT`s.
In normal conditions, the queue manager sends back a confirmation that the request was processed. In put async mode,
your application can put the next message without having to wait for this confirmation. This can be useful for
custom injectors, in which you might want to inject messages as fast as possible.

Put async can be enabled on a queue like this:

{% highlight java %}
((MQQueue) myQueue).setPutAsyncAllowed(CommonConstants.WMQ_PUT_ASYNC_ALLOWED_ENABLED);
{% endhighlight %}

### Use bindings mode

IBM MQ supports two transport modes, i.e. two ways to connect a client application to a queue manager.
The most common one is probably the client mode, which uses TCP/IP connections to connect to a server channel.
They can be fast on loopback interfaces, but can easily make your application feel "slower" on physical
network interfaces.

The second mode is called [“bindings mode”](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.0.0/com.ibm.mq.dev.doc/q031720_.htm)
(not to be confused with [JNDI bindings](https://www.ibm.com/support/knowledgecenter/en/prodconn_1.0.0/com.ibm.scenarios.wmqwasha.doc/topics/ins_mq_JNDI.htm?cp=SSFKSJ_9.0.0)!).
In bindings mode, the client application uses JNI to connect to the queue manager directly using [IPC](https://en.wikipedia.org/wiki/Inter-process_communication).
This means that your application *must* run on the same machine as the one running the queue manager, because it will be working
“in memory” instead of using network connections.

The transport mode has to be configured on the connection factory:

{% highlight java %}
MQConnectionFactory qcf = new MQQueueConnectionFactory();

// no need to specify a host/port/channel here
qcf.setTransportType(CommonConstants.WMQ_CM_BINDINGS);
{% endhighlight %}

## Tuning the server part

Before mentioning what can be configured on the server side, here's a  reminder of the terminology related to MQ connections.
An MQ ‘server’ consists of one or more *queue managers* (QM), each one managing one or more *queues*. 

To access a queue from a JMS application, you have to get a `Connection` to the QM using a `QueueConnectionFactory`. 
When the ‘client’ transport is used (i.e. TCP/IP connections), connections are made to a specific [*channel*](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.0.0/com.ibm.mq.explorer.doc/e_channels.htm). When an application connects
to a channel, the QM creates what is called a *channel instance*.

### Increase the number of channel instances

One way to increase the throughput of your application is to listen for incoming messages in more than one thread.
This can be easily achieved in Spring JMS using [`concurrency`](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/jms/listener/DefaultMessageListenerContainer.html#setConcurrency-java.lang.String-),
but you should be aware that there are several parameters in IBM MQ that control (limit) the number of network connections 
(channel instances) client applications can make.

A channel can define two parameters related to [channel instances](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.0.0/com.ibm.mq.con.doc/q015640_.htm):

* `MAXINST`: maximum number of channel instances
* `MAXINSTC`: maximum number of channel instances *per client application*

These two parameters can be easily viewed and modified using MQ Explorer, in the channel properties page.

A [third —less obvious— parameter](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.0.0/com.ibm.mq.ref.dev.doc/q102490_.htm) can be configured at the queue manager level:

* `MaxChannels`: maximum number of channel instances all channels combined

By default, 100 channel instances (network sockets) are allowed on a given queue manager. As we will see in the next section, 
it is important to be aware of this parameter. To change it, add/modify the following section in `/var/mqm/qmgrs/MyQueueManager/qm.ini`:

{% highlight text %}
Channels:
    MaxChannels=200
{% endhighlight %}

If a channel defines `MaxInst=200` but the QM defines `MaxChannels=100`, then only 100 channel instances will be allowed.

### Don't share conversations

IBM MQ has a notion of [“shared conversations”](https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_9.0.0/com.ibm.mq.dev.doc/q032530_.htm), 
meaning that multiple JMS connections/sessions can be shared on the same channel instance (TCP/IP connection). By default, this parameter is set to 10, 
but you can change the value in MQ Explorer in the channel properties page. Sharing conversations means that less network sockets will be used, 
which can sometimes be a good thing because it removes the overhead introduced each time a new network connection is opened. On the other end, this 
can also have a performance cost on applications that do long running operations on the same `Session`, like message listeners, because data from different 
sessions has to be ‘mixed’ in a same socket, then ‘unmixed’ on the other end.

Setting `SHARECNV` to 1 can improve performance dramatically in some cases, because the channel instance will be dedicated to a given `Session`.
Beware though that it can lead to ‘connection refused’ errors very fast. As we saw in the previous section, queue managers only allow 100 channel 
instances by default. Assuming you don't share conversations and your application is the only one using that queue manager, this means that if you 
start 50 consumers on 2 queues, you will end up with 1 JMS connection + 2*50 sessions = 101 channel instances, so the last thread won't be able 
to start correctly.

When using `SHARECNV=1`, it is recommended to carefully determine the number of channel instances that will be used by your application,
and adapt the parameters we saw in the previous section accordingly.

Of course, because shared conversations and channel instances apply to *network connections*, you don't have to worry about all these
parameters when using ‘bindings’ transport.

## Links

[Best practices to improve performance in JMS](http://www.precisejava.com/javaperf/j2ee/JMS.htm)

[High performance messaging](ftp://public.dhe.ibm.com/software/solutions/soa/pdfs/IBM_WebShere_MQ_Performance_Report.pdf)

[Guaranteed messaging with JMS](http://www2.sys-con.com/itsg/virtualcd/Java/archives/0604/chappell/index.html)

[Programming a WebSphere MQ JMS Client for performance](https://www.ibm.com/developerworks/community/blogs/messaging/entry/programming_a_websphere_mq_jms_client_for_performance7?lang=en)

[Avoiding run-away numbers of channels](https://www.ibm.com/developerworks/community/blogs/aimsupport/entry/avoiding_runaway_numbers_of_mq_channels?lang=en)