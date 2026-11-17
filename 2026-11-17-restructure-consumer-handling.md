# What is a Kafka consumer actually responsible for?

I have seen code like this in real production systems, even at large companies. These two Kafka consumers come from the **same codebase**, and they do essentially the **same thing**.

```java
@KafkaListener(topics = "${order.creation.response.topic}", groupId = "order-consumer-group")
@Transactional(rollbackFor = Exception.class)
public void orderCreationListener(ConsumerRecord<String, String> records, Consumer<?, ?> consumer) {
    String groupId = consumer.groupMetadata().groupId();
    String uniqueKey = madeUniqueKey(groupId, records.topic(), records.partition(), records.offset(), records.key());

    log.info("orderCreation consume start, key:{}", uniqueKey);

    if (isRepeatConsume(uniqueKey)) {
        log.warn("orderCreation duplicate consume detected, key:{}", uniqueKey);
        return;
    }

    long start = System.currentTimeMillis();
    try {
        consumeHandler(records);
        consumer.commitAsync();
    } catch (Exception e) {
        log.error("orderCreation consume error, key:{}, error:{}", uniqueKey, Arrays.toString(e.getStackTrace()));
    } finally {
        long cost = System.currentTimeMillis() - start;
        log.info("orderCreation consume end, key:{}, cost:{} ms", uniqueKey, cost);
    }
}
```

```java
@KafkaListener(topics = "${erp.data.sync.topic}", groupId = "erp-sync-group")
public void erpDataSyncListener(ConsumerRecord<String, String> records, Consumer<?, ?> consumer) {

    String redisKey = String.format("%s#%s#%s", records.topic(), records.partition(), records.offset());
    boolean acquired = redisUtils.lock(redisKey, 1, TimeUnit.HOURS);
    if (!acquired) {
        log.info("Duplicate consume detected, topic:{}|partition:{}|offset:{}|key:{}",
                records.topic(), records.partition(), records.offset(), records.key());
        return;
    }

    if (StringUtils.isBlank(records.value())) {
        return;
    }

    log.info("Consume start, topic:{}|partition:{}|offset:{}|key:{}",
            records.topic(), records.partition(), records.offset(), records.key());
    long start = System.currentTimeMillis();

    try {
        DataSyncDto dataSyncDto = JSON.parseObject(records.value(), DataSyncDto.class);
        dataHandle(dataSyncDto);
        log.info("Consume end, key:{}, cost:{} ms", records.key(), System.currentTimeMillis() - start);
    } catch (Exception e) {
        log.error("Consume error, key:{}|topic:{}|offset:{}|partition:{}|error:{}",
                records.key(), records.topic(), records.offset(), records.partition(),
                e.getMessage());
    } finally {
        consumer.commitAsync();
    }
}
```

Look at them side by side. One wraps everything in `@Transactional`, the other does not. One builds the idempotency key from five fields, the other from three. One checks for a blank payload, the other does not. The logging is different. The error handling is different.

They are inconsistent, fragile, and have no shared structure. And the more consumers a team writes like this, the worse it gets.

---

## The real problem is not a wrong line of code

It is tempting to fix these one method at a time — align the keys, standardize the logging, add the missing null check.

But that misses the point. The inconsistency is a symptom. The root cause is that nobody ever answered a more basic question:

**What is a consumer actually responsible for?**

Until that question has a clear answer, every new consumer will be written slightly differently, because every developer is improvising the structure from scratch.

---

## Find the orchestrator and its collaborators

When I face a piece of code whose responsibilities feel tangled, I use one simple lens:

> Find the **Orchestrator** and its **Collaborators**.
> The orchestrator owns the main flow.
> Collaborators each handle one specific responsibility.
> The orchestrator coordinates — it does not implement.

So let's apply it. What does a Kafka consumer actually need to do?

| Collaborator | Responsibility |
|---|---|
| Idempotency Checker | Has this message been processed before? |
| Message Parser | Deserialize the raw payload into a domain object |
| Business Handler | Execute the actual business logic |
| Offset Committer | Acknowledge successful consumption to Kafka |
| Logger / Tracer | Observability across the pipeline |

Each row is one job. None of them is "the consumer" — together they *are* the consumer. The orchestrator's only role is to call them in the right order.

---

## Diagnosing the original code

Now go back to the two examples with this lens.

The problem becomes obvious: idempotency check, payload validation, parsing, business logic, error handling, and logging are all collapsed into a single method. The orchestrator has drifted deep into collaborator territory — it is not coordinating, it is doing everything itself.

That is exactly why no two consumers look alike. When the orchestrator owns all the logic, there is no shared shape to reuse, so everyone reinvents it.

Once you separate the responsibilities, the orchestrator becomes thin and consistent:

```java
public void consume(ConsumerRecord<String, String> record, Consumer<?, ?> consumer) {
    // Orchestrator: coordinates collaborators, owns no logic itself

    if (idempotencyChecker.isDuplicate(record)) return;  // Collaborator: idempotency

    Message message = parser.parse(record);              // Collaborator: parser

    handler.handle(message);                             // Collaborator: business

    consumer.commitAsync();                              // Collaborator: commit
}
```

Every consumer now has the same shape. The differences move into the collaborators, where they belong — and each collaborator can be tested, swapped, and reasoned about on its own.

---

## Get the structure right first

The fix for messy code is rarely a better one-liner. It is a clearer answer to "who is responsible for what."

Find the orchestrator. Name its collaborators. Keep the orchestrator thin. The structure does most of the work.

And notice that very first collaborator — the idempotency checker. It looks like a single `if` statement, but it hides a surprising amount of depth: at-least-once delivery, deduplication strategy, and a real consistency trade-off.

That one is worth a post of its own. **Next: how to implement an Inbox pattern.**