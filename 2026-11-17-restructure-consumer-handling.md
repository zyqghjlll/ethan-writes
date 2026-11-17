# What is a Kafka consumer actually responsible for?

Imagine two Kafka consumers in the **same codebase**. They do essentially the **same thing**. Here are two simplified, illustrative sketches.

```java
void consumeA(Record record) {
    if (alreadyProcessed(record)) return;

    try {
        handle(record);
        commitOffset();
    } catch (Exception e) {
        logFailure(e);
    }
}
```

```java
void consumeB(Record record) {
    if (alreadyProcessed(record)) return;
    if (isBlank(record.value())) return;

    try {
        Message message = parse(record.value());
        handle(message);
    } catch (Exception e) {
        logFailure(e);
    } finally {
        commitOffset();
    }
}
```

Looking at them side by side, I notice that one checks for a blank payload and the other leaves that detail out of the listener. One attempts the commit after handling succeeds. The other attempts it in `finally`.

Some differences may be intentional. What interests me is how hard it can be to see the shared structure.

---

## The real problem is not a wrong line of code

It is tempting to fix these one method at a time — align the checks, standardize the logging, move a line or two.

I find it helpful to step back and ask a more basic question.

**What is a consumer actually responsible for?**

Without a shared answer, I can end up improvising the structure each time I write a listener.

---

## Finding the orchestrator and its collaborators

When I face a piece of code whose responsibilities feel tangled, I use one simple lens:

> I look for the **orchestrator** and its **collaborators**.
> The orchestrator owns the main flow.
> Collaborators each handle one specific responsibility.
> The orchestrator coordinates their work.

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

Looking back at the two sketches, some collaborators are already visible. `handle` gives the business logic a name. In the second listener, I can also see where parsing happens. Naming those responsibilities helps me read both methods at the same level of detail.

Here’s how I’d sketch the main flow. I’m focusing on the structure here, leaving the details of retries, transactions, and offset management outside the example.

```java
public void consume(ConsumerRecord<String, String> record, Consumer<?, ?> consumer) {
    if (idempotencyChecker.isDuplicate(record)) return;  // Collaborator: idempotency

    Message message = parser.parse(record);              // Collaborator: parser

    handler.handle(message);                             // Collaborator: business

    consumer.commitAsync();                              // Collaborator: commit
}
```

The two consumers can now share a recognizable shape. The differences move into the collaborators, where they belong — and each collaborator can be tested, swapped, and reasoned about on its own.

---

## Starting with the structure

When a listener feels messy, I come back to one question: **Who is responsible for what?** Finding the orchestrator and naming its collaborators helps me decide which details to move and which steps I want to keep visible in the listener.
