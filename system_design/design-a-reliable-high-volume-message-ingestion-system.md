# Design a Reliable High-Volume Message Ingestion System

## Problem
Design a backend that receives telemetry from many devices,
stores accepted messages, and processes them asynchronously.

## Scope
- Receive messages
- Persist accepted messages
- Process messages asynchronously

## Questions to clarify
- What does a successful response guarantee?
- What are the average and peak message rates?
- Are duplicates and out-of-order messages acceptable?

## Workloads
- How many devices we need to collect?
- How frequently devices send data?
- What is the size of each messages and the format of message?
- What's the requirement about lantency for processing messages?
- What happens if some devices outage, or loss messages?
- What happens if message duplicate?
- Does messages order matter?

## Non-function requeirments
- High aviable. 