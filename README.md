# POC Mule Kafka Batch Listener

A proof-of-concept MuleSoft application demonstrating Apache Kafka batch message consumption with different acknowledgment modes.

## Purpose

This project serves as a **practical demonstration** of how to implement batch message processing from Apache Kafka in MuleSoft applications. It's designed to help developers understand:

- **The difference between auto and manual commit strategies** and when to use each
- **How batch processing works** in the MuleSoft Kafka connector
- **Best practices for handling Kafka offsets** in enterprise integration scenarios
- **Logging and monitoring patterns** for batch message consumption

## What This Project Demonstrates

### 1. **Batch Message Consumption**

Instead of processing Kafka messages one at a time, this POC shows how to consume multiple messages in a single batch operation. This approach:
- **Improves throughput** by reducing the number of polling operations
- **Enables batch-oriented processing** where you need to process related messages together
- **Optimizes resource utilization** by grouping I/O operations

### 2. **Auto-Commit vs Manual-Commit Strategies**

The project implements both acknowledgment modes side-by-side, demonstrating the trade-offs:

#### **Auto-Commit Mode** (`auto-commit` flow)
- **Use case**: When you can tolerate occasional message re-processing
- **Behavior**: Offsets are automatically committed after successful processing
- **Pros**: Simpler implementation, less code to manage
- **Cons**: Risk of message loss if the application crashes before auto-commit occurs
- **Best for**: Non-critical data, idempotent operations, high-throughput scenarios

#### **Manual-Commit Mode** (`manual-commit` flow)
- **Use case**: When you need guaranteed message processing
- **Behavior**: Offsets are committed only after explicit confirmation
- **Pros**: Full control over exactly when offsets are committed, prevents message loss
- **Cons**: More complex, requires careful error handling
- **Best for**: Financial transactions, critical business data, exactly-once processing requirements

### 3. **Real-World Batch Processing Patterns**

The `logging-subflow` demonstrates common batch processing patterns:
- **Batch-level operations**: Actions performed once per batch (logging batch size, batch commit key)
- **Item-level operations**: Actions performed for each message in the batch (logging individual offsets, partition info)
- **Metadata tracking**: Accessing partition numbers, offsets, and commit keys for observability

### 4. **Kafka Consumer Configuration and Batch Behavior**

The project showcases how Kafka consumer configurations control batch behavior:

#### **How Batching Actually Works**

The batch listener operates on a polling mechanism with the following behavior:

- **If 20 records arrive quickly**: The batch listener emits all 20 immediately (record limit reached)
- **If fewer records are available**: The Kafka **broker** waits to accumulate ~10 KB of data OR until 10 seconds elapse, then returns whatever it has
- **If no records are available**: The broker returns empty after the timeout, and the batch listener does **not** emit anything (no flow execution)

This is the **standard and desirable pattern** for event-driven sources that should only trigger flows when data actually exists.

#### **Key Configuration Parameters**

- **Record Limit (20 records)**: Maximum batch size - prevents unbounded memory usage
- **Fetch Minimum Size (10 KB)**: The Kafka broker waits to accumulate at least this much data before responding (unless timeout is reached)
- **Fetch Maximum Wait Timeout (10 seconds)**: Maximum time the broker waits; returns available data after this period
- **Isolation Level (READ_COMMITTED)**: Only consume fully committed messages, ensuring transactional consistency
- **Topic Patterns**: Target specific topics for focused message consumption

## Key Learning Outcomes

After exploring this POC, you'll understand:

1. **How to configure MuleSoft Kafka batch listeners** with different acknowledgment modes
2. **When to choose auto-commit vs manual-commit** based on your business requirements
3. **How to access and use Kafka metadata** (partitions, offsets, commit keys) in your flows
4. **How to structure reusable sub-flows** for batch processing logic
5. **How to implement proper logging** for debugging and monitoring batch operations
6. **How batch size and fetch parameters impact performance** and reliability

## Real-World Applications

This pattern is valuable for scenarios such as:

- **High-Volume Event Processing**: Processing thousands of events per second from distributed systems
- **ETL Operations**: Extracting, transforming, and loading data in batches from Kafka to databases or data warehouses
- **Aggregation & Analytics**: Collecting multiple events to perform calculations or generate reports
- **Order Processing**: Handling multiple orders together for efficient batch fulfillment
- **IoT Data Collection**: Aggregating sensor data from multiple devices before processing
- **Audit Logging**: Batch-writing audit events to persistent storage for compliance

## Architecture

The application consists of:

- **Two Kafka Consumer Configurations**: 
  - `KafkaConnection-1`: For auto-commit mode listening to `batch-topic-1`
  - `KafkaConnection-2`: For manual-commit mode listening to `batch-topic-2`
- **Two Main Flows**:
  - `auto-commit`: Processes batches with automatic offset commitment
  - `manual-commit`: Processes batches with manual offset commitment
- **Shared Sub-flow**: `logging-subflow` for consistent batch processing and logging

## Prerequisites

- **MuleSoft Runtime**: Version 4.9.0 or higher
- **Java**: JDK 17
- **Apache Kafka**: Running instance with topics `batch-topic-1` and `batch-topic-2`
- **Maven**: For building and dependency management

## Configuration

### Kafka Configuration

The application is configured to connect to a local Kafka instance:

- **Bootstrap Server**: `127.0.0.1:9094`
- **Topics**: 
  - `batch-topic-1` (auto-commit)
  - `batch-topic-2` (manual-commit)
- **Batch Settings**:
  - Record limit: 20 messages per batch
  - Fetch minimum size: 10 KB
  - Fetch maximum wait timeout: 10 seconds
  - Isolation level: READ_COMMITTED

### Dependencies

- **Mule HTTP Connector**: v1.10.3
- **Mule Sockets Connector**: v1.2.6
- **Mule Kafka Connector**: v4.11.1

## Building the Application

```bash
# Clean and compile the application
mvn clean compile

# Package the application
mvn clean package

# Install dependencies and package
mvn clean install
```

## Deployment

### Local Development

1. Import the project into Anypoint Studio
2. Ensure Kafka is running locally on port 9094
3. Create the required topics:
   ```bash
   kafka-topics.sh --create --topic batch-topic-1 --bootstrap-server localhost:9094
   kafka-topics.sh --create --topic batch-topic-2 --bootstrap-server localhost:9094
   ```
4. Run the application from Anypoint Studio

### CloudHub Deployment

```bash
mvn clean package deploy -DmuleDeploy
```

## Usage

### Testing the Application

1. **Start the application**: Deploy and run the Mule application
2. **Produce test messages**: Send messages to both Kafka topics
   ```bash
   # Send messages to auto-commit topic
   kafka-console-producer.sh --topic batch-topic-1 --bootstrap-server localhost:9094
   
   # Send messages to manual-commit topic
   kafka-console-producer.sh --topic batch-topic-2 --bootstrap-server localhost:9094
   ```
3. **Monitor logs**: Check the application logs to see batch processing information

### Expected Log Output

```
======================================
A new batch of 5 elements
partition number: 0, offset: 100, msg commitKey: [commit-key]
partition number: 0, offset: 101, msg commitKey: [commit-key]
...
batch commitKey: [batch-commit-key]
```

## Flow Descriptions

### Auto-Commit Flow (`auto-commit`)

**Purpose**: Demonstrates the simpler, fire-and-forget approach to message consumption.

- **Trigger**: Batch message listener with `AUTO` acknowledgment mode
- **Processing**: Delegates to `logging-subflow` for batch inspection
- **Commitment**: Offsets are automatically committed by the Kafka connector after successful processing
- **Use Case Example**: Processing non-critical system logs or metrics where occasional message loss is acceptable

**Key Insight**: This flow shows how automatic offset management simplifies code but requires trust in the automatic commit timing.

### Manual-Commit Flow (`manual-commit`)

**Purpose**: Demonstrates precise control over offset commitment for critical message processing.

- **Trigger**: Batch message listener with `MANUAL` acknowledgment mode
- **Processing**: Delegates to `logging-subflow` for batch inspection
- **Commitment**: Explicitly commits offsets using the `kafka:commit` operation with the batch commit key
- **Use Case Example**: Processing financial transactions where you must guarantee each message is processed before acknowledging it

**Key Insight**: This flow shows how to implement exactly-once semantics by controlling the commit timing after successful processing.

### Logging Sub-flow (`logging-subflow`)

**Purpose**: Centralized batch processing logic that can be reused across different flows.

This sub-flow demonstrates the typical structure of batch processing:

1. **Batch Start Marker**: Logs a separator for readability in log files
2. **Batch Metadata**: Reports the batch size (`sizeOf(payload)`)
3. **Individual Message Processing**: Iterates through each message using `foreach` to:
   - Access message-level metadata (partition, offset, commit key)
   - Process or log each individual message
   - Demonstrate how to work with the batch payload structure
4. **Batch Completion**: Logs the batch-level commit key from `attributes.consumerCommitKey`

**Key Insight**: This pattern shows how batch payloads contain both individual message data and batch-level metadata, essential for proper batch processing and error handling.

## Performance Considerations & Tuning Explained

### Understanding Batch Behavior

The batch listener uses a **pull-based polling model**. Each poll cycle works as follows:

1. **Poll Request**: The listener requests up to 20 records from Kafka broker
2. **Broker-Side Wait Logic**: 
   - If 20+ records are available → broker returns 20 immediately
   - If fewer records are available → broker waits to accumulate ~10 KB OR until 10 seconds elapse
   - After waiting → broker returns whatever data is available (even if < 10 KB)
   - If no records at all → broker returns empty response
3. **Listener Behavior**: 
   - If broker returns data (1-20 records) → batch listener emits the batch and triggers the flow
   - If broker returns empty → batch listener does **not** emit anything, no flow execution, polling continues

### Configuration Parameters Explained

#### **Record Limit: 20 messages**
- **What it does**: Sets the maximum batch size per poll
- **Behavior impact**: 
  - Acts as an upper bound to prevent excessive memory consumption
  - If more than 20 messages are available, only 20 are fetched per poll
  - Next poll will fetch the remaining messages
- **Tuning guidance**: 
  - Increase for high-throughput, small-message scenarios
  - Decrease for large messages or memory-constrained environments

#### **Fetch Minimum Size: 10 KB**
- **What it does**: Instructs the Kafka broker to wait until at least 10 KB of data is available before responding
- **Behavior impact**: 
  - The broker accumulates messages until reaching this threshold (or until timeout)
  - Improves efficiency by avoiding tiny batches when messages trickle in slowly
  - Works in conjunction with the timeout - won't wait forever
- **Trade-off**: May add latency in low-volume scenarios as the broker waits to accumulate data
- **Tuning guidance**: Lower for latency-sensitive applications; higher for throughput optimization

#### **Fetch Maximum Wait Timeout: 10 seconds**
- **What it does**: Maximum time the broker waits to accumulate data before responding
- **Behavior impact**: 
  - Ensures the broker doesn't block indefinitely when message flow is slow
  - After 10 seconds, broker returns whatever data is available (even if < 10 KB)
  - If no data at all, broker returns empty and the batch listener does not trigger the flow
- **Trade-off**: Balances between batch completeness and processing latency
- **Tuning guidance**: Lower for near-real-time processing; higher to maximize batch sizes

#### **Isolation Level: READ_COMMITTED**
- **What it does**: Only reads messages that have been committed by transactional producers
- **Behavior impact**: 
  - Prevents consuming messages that might be rolled back
  - Essential for transactional consistency in the data pipeline
- **When to use**: Always use for financial, order processing, or any system requiring data integrity
- **Alternative**: `READ_UNCOMMITTED` for non-transactional scenarios requiring maximum throughput

### Practical Batch Behavior Examples

**Example 1: High-Volume Scenario**
- 100 messages (each 1 KB) arrive in 1 second
- **Result**: 
  - Broker immediately returns 20 messages (record limit reached)
  - Batch listener emits batch of 20, flow executes
  - Next poll: broker returns another 20 immediately
  - Process repeats until all 100 are consumed

**Example 2: Low-Volume Scenario**
- 5 messages (each 1 KB, total 5 KB) arrive over 8 seconds
- **Result**: 
  - Broker waits for 10 KB OR 10 seconds
  - After 10 seconds (timeout reached), broker returns the 5 messages
  - Batch listener emits batch of 5, flow executes

**Example 3: No Messages**
- No messages arrive at all
- **Result**: 
  - After 10 seconds, broker returns empty
  - Batch listener does **not** emit anything, flow doesn't execute
  - Listener continues polling (next poll cycle begins)

**Example 4: Steady Stream**
- Messages arrive at 2 per second (each 0.5 KB)
- **Result**: 
  - After 10 seconds: 20 messages accumulated (10 KB reached)
  - Broker returns 20 messages
  - Batch listener emits batch of 20, flow executes

**Example 5: Large Messages**
- 3 messages (each 5 KB, total 15 KB) arrive in 2 seconds
- **Result**: 
  - 15 KB exceeds the 10 KB minimum
  - Broker returns 3 messages immediately (no need to wait)
  - Batch listener emits batch of 3, flow executes

## Error Handling Patterns (For Production Enhancement)

This POC focuses on the happy path, but production implementations should consider:

### For Auto-Commit Mode:
- **Challenge**: Messages might be auto-committed even if processing fails
- **Solution**: Implement idempotent processing or dead letter queues for failed messages
- **Pattern**: Try-catch around processing with logging of failures

### For Manual-Commit Mode:
- **Challenge**: If processing fails before commit, messages will be re-delivered
- **Solution**: Store processed message IDs to detect and skip duplicates
- **Pattern**: Database-backed deduplication or state management

### General Best Practices:
- Custom error handlers for Kafka connectivity issues (network failures, authentication errors)
- Dead letter queue processing for messages that consistently fail
- Retry mechanisms with exponential backoff for transient failures
- Circuit breakers for downstream service failures

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add some amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is a proof-of-concept and is provided as-is for educational and demonstration purposes.

## Contact

- **Author**: stn1slv
- **Repository**: [poc-mule-kafka-batch-listener](https://github.com/stn1slv/poc-mule-kafka-batch-listener)

## Additional Resources

- [MuleSoft Kafka Connector Documentation](https://docs.mulesoft.com/kafka-connector/latest/)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [MuleSoft Documentation](https://docs.mulesoft.com/)