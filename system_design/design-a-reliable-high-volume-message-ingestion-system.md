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