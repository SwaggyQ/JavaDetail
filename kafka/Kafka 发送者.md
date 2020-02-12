Kafka 发送者



### Cluster

producer分为两个线程，一个向accumulator中堆积record，另一个线程从中获取



### Sender

**KafkaClient client**
**kafka 网络通信客户端，主要封装与 broker 的网络通信。**
**RecordAccumulator accumulator**
**消息记录累积器，消息追加的入口(RecordAccumulator 的 append 方法)。**
**Metadata metadata**
**元数据管理器，即 topic 的路由分区信息。**

sender流程

1. 当线程运行时，始终循环run方法
2. 当线程中断，但非forceClose时，处理剩余请求
3. 当forceClose时，直接中断



在sender中根据事务管理器来控制，尝试发送时，创建一个Node







```java
void run(long now) {
    if (transactionManager != null) {
        try {
            if (transactionManager.shouldResetProducerStateAfterResolvingSequences())
                // Check if the previous run expired batches which requires a reset of the producer state.
                transactionManager.resetProducerId();

            if (!transactionManager.isTransactional()) {
                // this is an idempotent producer, so make sure we have a producer id
                maybeWaitForProducerId();
            } else if (transactionManager.hasUnresolvedSequences() && !transactionManager.hasFatalError()) {
                transactionManager.transitionToFatalError(new KafkaException("The client hasn't received acknowledgment for " +
                        "some previously sent messages and can no longer retry them. It isn't safe to continue."));
            } else if (transactionManager.hasInFlightTransactionalRequest() || maybeSendTransactionalRequest(now)) {
                // as long as there are outstanding transactional requests, we simply wait for them to return
                client.poll(retryBackoffMs, now);
                return;
            }

            // do not continue sending if the transaction manager is in a failed state or if there
            // is no producer id (for the idempotent case).
            if (transactionManager.hasFatalError() || !transactionManager.hasProducerId()) {
                RuntimeException lastError = transactionManager.lastError();
                if (lastError != null)
                    maybeAbortBatches(lastError);
                client.poll(retryBackoffMs, now);
                return;
            } else if (transactionManager.hasAbortableError()) {
                accumulator.abortUndrainedBatches(transactionManager.lastError());
            }
        } catch (AuthenticationException e) {
            // This is already logged as error, but propagated here to perform any clean ups.
            log.trace("Authentication exception while processing transactional request: {}", e);
            transactionManager.authenticationFailed(e);
        }
    }

    long pollTimeout = sendProducerData(now);
    client.poll(pollTimeout, now);
}
```

```java
private void maybeWaitForProducerId() {
    // hasProducerId() 
    // return NO_PRODUCER_ID < producerId;
    while (!transactionManager.hasProducerId() && !transactionManager.hasError()) {
        try {
            // 选择一个有最少request的node用于和broker交流
            Node node = awaitLeastLoadedNodeReady(requestTimeoutMs);
            if (node != null) {
                // 构建Request然后发送
                ClientResponse response = sendAndAwaitInitProducerIdRequest(node);
                InitProducerIdResponse initProducerIdResponse = (InitProducerIdResponse) response.responseBody();
                Errors error = initProducerIdResponse.error();
                // 检查response是否异常
                if (error == Errors.NONE) {
                    // 发送成功，更新producerId
                    ProducerIdAndEpoch producerIdAndEpoch = new ProducerIdAndEpoch(
                            initProducerIdResponse.producerId(), initProducerIdResponse.epoch());
                    transactionManager.setProducerIdAndEpoch(producerIdAndEpoch);
                    return;
                } else if (error.exception() instanceof RetriableException) {
                    log.debug("Retriable error from InitProducerId response", error.message());
                } else {
                    transactionManager.transitionToFatalError(error.exception());
                    break;
                }
            } else {
                log.debug("Could not find an available broker to send InitProducerIdRequest to. " +
                        "We will back off and try again.");
            }
        } catch (UnsupportedVersionException e) {
            transactionManager.transitionToFatalError(e);
            break;
        } catch (IOException e) {
            log.debug("Broker {} disconnected while awaiting InitProducerId response", e);
        }
        log.trace("Retry InitProducerIdRequest in {}ms.", retryBackoffMs);
        time.sleep(retryBackoffMs);
        metadata.requestUpdate();
    }
}
```



```java
private long sendProducerData(long now) {
        Cluster cluster = metadata.fetch();

        // get the list of partitions with data ready to send
        RecordAccumulator.ReadyCheckResult result = this.accumulator.ready(cluster, now);

        // if there are any partitions whose leaders are not known yet, force metadata update
        if (!result.unknownLeaderTopics.isEmpty()) {
            // The set of topics with unknown leader contains topics with leader election pending as well as
            // topics which may have expired. Add the topic again to metadata to ensure it is included
            // and request metadata update, since there are messages to send to the topic.
            for (String topic : result.unknownLeaderTopics)
                this.metadata.add(topic);

            log.debug("Requesting metadata update due to unknown leader topics from the batched records: {}", result.unknownLeaderTopics);

            this.metadata.requestUpdate();
        }

        // remove any nodes we aren't ready to send to
        Iterator<Node> iter = result.readyNodes.iterator();
        long notReadyTimeout = Long.MAX_VALUE;
        while (iter.hasNext()) {
            Node node = iter.next();
            if (!this.client.ready(node, now)) {
                iter.remove();
                notReadyTimeout = Math.min(notReadyTimeout, this.client.pollDelayMs(node, now));
            }
        }

        // create produce requests
        Map<Integer, List<ProducerBatch>> batches = this.accumulator.drain(cluster, result.readyNodes,
                this.maxRequestSize, now);
        if (guaranteeMessageOrder) {
            // Mute all the partitions drained
            for (List<ProducerBatch> batchList : batches.values()) {
                for (ProducerBatch batch : batchList)
                    this.accumulator.mutePartition(batch.topicPartition);
            }
        }

        List<ProducerBatch> expiredBatches = this.accumulator.expiredBatches(this.requestTimeoutMs, now);
        // Reset the producer id if an expired batch has previously been sent to the broker. Also update the metrics
        // for expired batches. see the documentation of @TransactionState.resetProducerId to understand why
        // we need to reset the producer id here.
        if (!expiredBatches.isEmpty())
            log.trace("Expired {} batches in accumulator", expiredBatches.size());
        for (ProducerBatch expiredBatch : expiredBatches) {
            failBatch(expiredBatch, -1, NO_TIMESTAMP, expiredBatch.timeoutException(), false);
            if (transactionManager != null && expiredBatch.inRetry()) {
                // This ensures that no new batches are drained until the current in flight batches are fully resolved.
                transactionManager.markSequenceUnresolved(expiredBatch.topicPartition);
            }
        }

        sensors.updateProduceRequestMetrics(batches);

        // If we have any nodes that are ready to send + have sendable data, poll with 0 timeout so this can immediately
        // loop and try sending more data. Otherwise, the timeout is determined by nodes that have partitions with data
        // that isn't yet sendable (e.g. lingering, backing off). Note that this specifically does not include nodes
        // with sendable data that aren't ready to send since they would cause busy looping.
        long pollTimeout = Math.min(result.nextReadyCheckDelayMs, notReadyTimeout);
        if (!result.readyNodes.isEmpty()) {
            log.trace("Nodes with data ready to send: {}", result.readyNodes);
            // if some partitions are already ready to be sent, the select time would be 0;
            // otherwise if some partition already has some data accumulated but not ready yet,
            // the select time will be the time difference between now and its linger expiry time;
            // otherwise the select time will be the time difference between now and the metadata expiry time;
            pollTimeout = 0;
        }
        sendProduceRequests(batches, now);

        return pollTimeout;
    }
```

sender是一个Runable的类，内部一直循环调用run(time)方法，利用TransactionManager来做事务控制。



 **TransactionManager** 

内部有一个TransactionId，默认为null



