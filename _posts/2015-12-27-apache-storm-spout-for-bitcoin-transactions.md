---
layout: post
title:  "Apache Storm Spout for Bitcoin transactions processing"
date:   2015-12-27 17:52:00
---

This topology has been written as my final project for "Real-Time Analytics with Apache Storm" course at [Udacity](https://www.udacity.com) and doesn't have much practical sense as the maximum throughput of Bitcoin network is only about 7 transactions/second. It's unlikely to be changed soon, at least until the number of transaction in the network approaches this constraint. Thus, it doesn't make a lot of sense to use Apache Storm for processing. You can analyse Bitcoin transactions even with a single script (application) without a need to distribute computations unless you perform some really heavy and CPU consuming calculations. Nonetheless, it can be a good starting point for your own spout or WebSocket client in Java. 

There are a couple of Java libraries available for this purpose. `org.eclipse.jetty.websocket` seems more mature, powerful and better maintaining, but I chose `javax.websocket-client-api` as sufficient for me
{% highlight xml %}
<dependency>
    <groupId>javax.websocket</groupId>
    <artifactId>javax.websocket-client-api</artifactId>
    <version>1.1</version>
</dependency>
{% endhighlight %}

and `tyrus-client` as a reference implementation of the API.

{% highlight xml %}
<dependency>
    <groupId>org.glassfish.tyrus</groupId>
    <artifactId>tyrus-client</artifactId>
    <version>1.10</version>
</dependency>
<dependency>
    <groupId>org.glassfish.tyrus</groupId>
    <artifactId>tyrus-container-grizzly-client</artifactId>
    <version>1.10</version>
</dependency>
{% endhighlight %}

In order to turn our [POJO](https://en.wikipedia.org/wiki/Plain_Old_Java_Object) into WebSocket client endpoint, we need to declare it with the corresponding annotation:
{% highlight java %}
@ClientEndpoint
public class BlockchainInfoClient {
    ...
}
{% endhighlight %}

The connection is established right in the constructor.

`@OnOpen` annotation makes a method to be invoked right after a connection is established. In our case, it sets a few session parameters and subscribes for unconfirmed transactions.
{% highlight java %}
@OnOpen
public void onOpen(final Session session) throws IOException {
    session.setMaxIdleTimeout(0);
    session.setMaxBinaryMessageBufferSize(16384);
    session.setMaxTextMessageBufferSize(16384);

    JSONObject subscriptionMessage = new JSONObject();
    subscriptionMessage.put("op", "unconfirmed_sub");
    session.getBasicRemote().sendText(subscriptionMessage.toString());

    LOG.info(subscriptionMessage.toString());

    userSession = session;

    LOG.info("Subscribed for unconfirmed transactions from Blockchain.info.");
}
{% endhighlight %}

We will also need a handler for the incoming messages, which must be decorated with `@OnMessage`:
{% highlight java %}
@OnMessage
public void onMessage(final String message, boolean isLastPartOfMessage) {
    messageBuffer.append(message);
    if (isLastPartOfMessage) {
        try {
            if (messageHandler != null) {
                messageHandler.handleMessage(messageBuffer.toString());
            }
        } catch (Exception e) {
            LOG.error(e.getMessage(), e);
        } finally {
            messageBuffer = new StringBuilder();
        }
    }

    LOG.info("Message received from Blockchain.info");
}
{% endhighlight %}

To access messages from the spout define MessageHandler interface and a method to inject an implementation of this interface to our class:
{% highlight java %}
public void addMessageHandler(final MessageHandler msgHandler) {
    messageHandler = msgHandler;
}

public static interface MessageHandler {
    public void handleMessage(String message);
}
{% endhighlight %}

The spout extends `BaseRichSpout` abstract class
{% highlight java %}
public class TransactionSpout extends BaseRichSpout {
}
{% endhighlight %}

and override some of it's methods.

`open` is called once a spout is initialized within a worker. We need it to store a collector, initialize a queue of incoming messages and instantiate a previously created client with a handler putting messages to our queue.
{% highlight java %}
@Override
public void open(Map conf, TopologyContext context, SpoutOutputCollector spoutOutputCollector) {
    collector = spoutOutputCollector;
    queue = new LinkedBlockingQueue<String>(1000);

    client = new BlockchainInfoClient();
    client.addMessageHandler(new BlockchainInfoClient.MessageHandler() {
        @Override
        public void handleMessage(String message) {
            queue.offer(message);
        }
    });
}
{% endhighlight %}

Then we need to override the main method of each spout -- `nextTuple`, which will either emit new tuple into a topology or simply return if there are no new transactions in the queue.
{% highlight java %}
@Override
public void nextTuple() {
    String ret = queue.poll();

    if (ret==null)
    {
        Utils.sleep(50);
        return;
    }

    collector.emit(new Values(ret));
}
{% endhighlight %}

And at last, describe the output schema in `declareOutputFields`
{% highlight java %}
@Override
public void declareOutputFields(OutputFieldsDeclarer declarer) {
    declarer.declare(new Fields("transaction"));
}
{% endhighlight %}

The full source code as well as usage example can be found in [GitHub repository](https://github.com/Sundrique/bitcoin-trending-addresses).