# Architectural decisions for sensor data calibration
_Exported on 11/16/2025 at 17:07:00 GMT+1 from Cursor (2.0.69)_

---

**User**

I need some help with some architecual decisions.

I have this hobby project that collect temperature readings from a bunch of sensors and then display them.

That's the basics of it.



A problem I have is that the sensors need a bit of calibration to be within a normal spec, so what I do is collect the raw calculcated temperature reading from sensors then run it through a calibration.

With this I want the possibility to recalibrate data over time if I find out that the calibration was way off.



My current setup looks something like this:

Sensors -> Send data to API (ServerPod API) -> Save the raw data in Postgres DB

Cronjob (once a minute) -> Mage.AI pipeline triggers -> Batch collect raw data for calibration -> calibrate -> Save the calibrated data in PostgresDB



This works even though the calibration might be a bit slow, it takes 3-4k data points at a time and calibrate them.

The first process I kinda like as it is easy to reason about, sensor send the data to my API and then save it.

It also makes it easy to switch in the future if I want to move away from serverpod.



But now I have moved it into kubernetes and was thinking if I might want to reconsider a few options.

This is a hobby project and the technologies are related to what I do at my job. So that's why I for example use Mage rather then integrating the calibration into ServerPod.

However I feel it's very small part of the process so I been thinking if I can do something else.

I also don't like the idea of batching, I like to try and do more streaming/realtime calibration.

In one of my projects at work we have started working with kafka so I am eager to maybe introduce kafka into this project too.



I was playing around with the idea and I think I found a solution that I kinda like where we add kafka inbetween serverpod and postgres.

Basically the flow becomes Sensors -> ServerPod API -> Kafka -> Postgres

Then for calibration it just goes back and forth, when raw data is saved in postgres it will queue up in kafka to be calibrated then have some process that will handle this.



On a overall architect I like this, but I am struggling a bit with the details on how this calibration will work and where it lives.

I don't like the idea of too many complicated components that lives in several areas, but I also like the idea of exploring what kubernetes can offer in this situation.



Something I do want to mention is that I want to ensure that I have successfully saved the raw data before we try to calibrate that data point.

There is no need to calibrate data if we couldn't successfully save it to the DB.



I was thinking of maybe still using Mage and stream data from kafka, but I am not too sure.

Are there any Kubernetes specific things that I could consider that would work as some sort of service to handle the data inbetween kafka and postgres?

Or is there some kafka specific things that I can consider?



As mentioned this is a hobby project, so I like to explore new technologies and solutions, but at same time I want to take the enterprise/base practice into consideration.

---

**Cursor**

Looking at your architecture diagram and description, this is a great opportunity to explore some modern data streaming patterns! Let me help you think through this.

First, let me take a quick look at your current setup to understand the context better:

Perfect! I can see you're already running a sophisticated Kubernetes setup with CloudNativePG, Envoy Gateway, and more. Based on your diagram and requirements, let me give you several architectural options that align with both learning goals and best practices:

## Recommended Architecture: Event-Driven with Kafka

Your proposed flow is solid, but let me suggest some refinements:

### **Option 1: Kafka Connect + Kafka Streams (Most Kubernetes-Native)**

```
Sensors → ServerPod API → Kafka (raw-temperature topic)
                             ↓
                      Kafka Connect JDBC Sink → Postgres (raw data)
                             ↓
                      Kafka Streams app → calibrate → Kafka (calibrated-temperature topic)
                             ↓
                      Kafka Connect JDBC Sink → Postgres (calibrated data)
```

**Kubernetes Components:**
- **Strimzi Operator**: Manages Kafka in Kubernetes (industry standard)
- **Kafka Connect**: Runs as a K8s Deployment, handles DB writes declaratively
- **Calibration Service**: Simple K8s Deployment consuming/producing Kafka messages

**Why this works:**
- ✅ Kafka Connect JDBC Sink connector automatically handles Postgres writes (no custom code)
- ✅ You can configure exactly-once semantics to ensure data is saved
- ✅ Calibration service is stateless and easily scalable
- ✅ Can reprocess by resetting consumer offsets or replaying topics
- ✅ All components are K8s-native with proper health checks and scaling

**Implementation:**
```yaml
# Calibration service deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: temperature-calibration
  namespace: cozytemp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: temperature-calibration
  template:
    metadata:
      labels:
        app: temperature-calibration
    spec:
      containers:
      - name: calibration
        image: your-calibration-service:latest
        env:
        - name: KAFKA_BOOTSTRAP_SERVERS
          value: "kafka-cluster-kafka-bootstrap:9092"
        - name: INPUT_TOPIC
          value: "raw-temperature"
        - name: OUTPUT_TOPIC
          value: "calibrated-temperature"
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
```

---

### **Option 2: Custom Microservice with Transactional Outbox Pattern**

```
Sensors → ServerPod API → Postgres (raw data + outbox table)
                             ↓
                      Debezium CDC → Kafka (raw-temperature topic)
                             ↓
                      Calibration Service → Kafka (calibrated-temperature topic)
                             ↓
                      Kafka Connect → Postgres (calibrated data)
```

**Why this works:**
- ✅ **Guarantees** raw data is saved before calibration (it's in DB first)
- ✅ Debezium Change Data Capture (CDC) reads from Postgres Write-Ahead Log
- ✅ No dual writes problem (ServerPod only writes to Postgres)
- ✅ Can rebuild entire Kafka history from database if needed
- ✅ Industry pattern used at scale (Uber, Netflix)

**Downside:** More complex initial setup with Debezium

---

### **Option 3: Simple Kafka Consumer/Producer Service (Best Starting Point)**

This is what I'd **actually recommend starting with** given your learning goals:

```
Sensors → ServerPod API → Kafka (raw-temperature topic)
                             ↓
                      Raw Data Consumer Service:
                        1. Consume message
                        2. Save to Postgres (raw)
                        3. Produce to calibration-queue topic
                             ↓
                      Calibration Service:
                        1. Consume from calibration-queue
                        2. Calibrate
                        3. Save to Postgres (calibrated)
```

**Simple Go/Python/etc. service structure:**
```go
// Raw Data Consumer
for msg := range kafkaConsumer.Poll() {
    // 1. Save raw data
    rawID, err := db.SaveRawTemperature(msg.Value)
    if err != nil {
        // Don't commit offset, retry
        continue
    }
    
    // 2. Queue for calibration
    err = kafkaProducer.Send("calibration-queue", rawID, msg.Value)
    if err != nil {
        // Already saved in DB, that's fine
        log.Error("Failed to queue calibration, will batch later")
    }
    
    // 3. Commit offset
    kafkaConsumer.Commit(msg.Offset)
}
```

**Why start here:**
- ✅ Simple to understand and debug
- ✅ Explicit control over when data is saved vs. calibrated
- ✅ Easy to deploy as K8s Deployment
- ✅ Can add idempotency keys to prevent duplicate processing
- ✅ Later migrate to Kafka Streams/Connect when you outgrow it

---

## Kubernetes-Specific Recommendations

### **1. Use Strimzi Operator for Kafka**
```bash
# Add to your apps/
apps/
  kafka-system/
    operator/      # Strimzi operator Helm chart
    cluster/       # Your Kafka cluster definition
```

Strimzi gives you:
- Proper K8s monitoring (Prometheus metrics)
- Rolling updates without downtime
- Easy topic management via CRDs
- Integration with cert-manager for TLS

### **2. Use ConfigMaps for Calibration Formulas**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: calibration-config
  namespace: cozytemp
data:
  calibration.yaml: |
    sensors:
      sensor-001:
        offset: 2.5
        multiplier: 0.98
      sensor-002:
        offset: -1.2
        multiplier: 1.02
```

This way you can update calibration without redeploying!

### **3. Use Kubernetes Jobs for Recalibration**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: recalibrate-historical-data
spec:
  template:
    spec:
      containers:
      - name: recalibrator
        image: calibration-service:latest
        env:
        - name: MODE
          value: "batch-recalibrate"
        - name: START_DATE
          value: "2025-01-01"
        - name: END_DATE
          value: "2025-11-16"
      restartPolicy: OnFailure
```

### **4. Consider KafkaTopics as CRDs**
With Strimzi, you define topics declaratively:
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: raw-temperature
  labels:
    strimzi.io/cluster: kafka-cluster
spec:
  partitions: 3
  replicas: 2
  config:
    retention.ms: 604800000  # 7 days
    compression.type: lz4
```

---

## My Specific Recommendation for You

Given your context (hobby project, learning, already K8s-savvy), I'd suggest:

**Phase 1: Start Simple**
1. Deploy Strimzi Kafka (1-3 broker setup is fine for hobby)
2. Modify ServerPod to produce to Kafka instead of directly to DB
3. Write a simple Go/Python consumer service that:
   - Consumes raw temperatures
   - Saves to Postgres
   - Produces calibration jobs to second topic
4. Write a simple calibration service that consumes and processes

**Phase 2: Level Up**
- Add Kafka Connect for automatic DB sinks
- Consider Debezium CDC for guaranteed ordering
- Add Kafka Streams for stateful aggregations (like moving averages)

**Phase 3: Production Polish**
- Add dead letter queues for failed messages
- Implement exactly-once semantics
- Add schema registry (like Apicurio) for message evolution

---

## Handling "Save Before Calibrate" Requirement

**Three patterns:**

1. **Two-phase commit in consumer** (simplest):
   ```
   msg = consume()
   db.save(msg) # If this fails, don't commit offset
   kafka.commitOffset(msg)
   kafka.produce(calibration_topic, msg)
   ```

2. **Transactional producer** (Kafka native):
   Use Kafka transactions to atomically commit offset + produce new message

3. **Status field** (most flexible):
   ```sql
   CREATE TABLE raw_temperatures (
     id SERIAL PRIMARY KEY,
     sensor_id TEXT,
     raw_value FLOAT,
     calibration_status TEXT, -- pending, processing, completed, failed
     created_at TIMESTAMP
   );
   ```
   Calibration service only processes records with `calibration_status='pending'`

---

Would you like me to help you with:
1. Setting up Strimzi Kafka in your cluster?
2. Designing the schema for your Kafka topics?
3. Creating a simple calibration service template?
4. Setting up the Kafka Connect JDBC sink?

Let me know which direction sounds most interesting, and I can dive deeper!

```plaintext
Sensors → ServerPod API → Kafka (raw-temperature topic)
                             ↓
                      Kafka Connect JDBC Sink → Postgres (raw data)
                             ↓
                      Kafka Streams app → calibrate → Kafka (calibrated-temperature topic)
                             ↓
                      Kafka Connect JDBC Sink → Postgres (calibrated data)
```

```yaml
# Calibration service deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: temperature-calibration
  namespace: cozytemp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: temperature-calibration
  template:
    metadata:
      labels:
        app: temperature-calibration
    spec:
      containers:
      - name: calibration
        image: your-calibration-service:latest
        env:
        - name: KAFKA_BOOTSTRAP_SERVERS
          value: "kafka-cluster-kafka-bootstrap:9092"
        - name: INPUT_TOPIC
          value: "raw-temperature"
        - name: OUTPUT_TOPIC
          value: "calibrated-temperature"
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
```

```plaintext
Sensors → ServerPod API → Postgres (raw data + outbox table)
                             ↓
                      Debezium CDC → Kafka (raw-temperature topic)
                             ↓
                      Calibration Service → Kafka (calibrated-temperature topic)
                             ↓
                      Kafka Connect → Postgres (calibrated data)
```

```plaintext
Sensors → ServerPod API → Kafka (raw-temperature topic)
                             ↓
                      Raw Data Consumer Service:
                        1. Consume message
                        2. Save to Postgres (raw)
                        3. Produce to calibration-queue topic
                             ↓
                      Calibration Service:
                        1. Consume from calibration-queue
                        2. Calibrate
                        3. Save to Postgres (calibrated)
```

```go
// Raw Data Consumer
for msg := range kafkaConsumer.Poll() {
    // 1. Save raw data
    rawID, err := db.SaveRawTemperature(msg.Value)
    if err != nil {
        // Don't commit offset, retry
        continue
    }
    
    // 2. Queue for calibration
    err = kafkaProducer.Send("calibration-queue", rawID, msg.Value)
    if err != nil {
        // Already saved in DB, that's fine
        log.Error("Failed to queue calibration, will batch later")
    }
    
    // 3. Commit offset
    kafkaConsumer.Commit(msg.Offset)
}
```

```shellscript
# Add to your apps/
apps/
  kafka-system/
    operator/      # Strimzi operator Helm chart
    cluster/       # Your Kafka cluster definition
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: calibration-config
  namespace: cozytemp
data:
  calibration.yaml: |
    sensors:
      sensor-001:
        offset: 2.5
        multiplier: 0.98
      sensor-002:
        offset: -1.2
        multiplier: 1.02
```

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: recalibrate-historical-data
spec:
  template:
    spec:
      containers:
      - name: recalibrator
        image: calibration-service:latest
        env:
        - name: MODE
          value: "batch-recalibrate"
        - name: START_DATE
          value: "2025-01-01"
        - name: END_DATE
          value: "2025-11-16"
      restartPolicy: OnFailure
```

```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: raw-temperature
  labels:
    strimzi.io/cluster: kafka-cluster
spec:
  partitions: 3
  replicas: 2
  config:
    retention.ms: 604800000  # 7 days
    compression.type: lz4
```

```plaintext
   msg = consume()
   db.save(msg) # If this fails, don't commit offset
   kafka.commitOffset(msg)
   kafka.produce(calibration_topic, msg)
```

```sql
   CREATE TABLE raw_temperatures (
     id SERIAL PRIMARY KEY,
     sensor_id TEXT,
     raw_value FLOAT,
     calibration_status TEXT, -- pending, processing, completed, failed
     created_at TIMESTAMP
   );
```

---

**User**

For the actual kafka I do like using operators, I used it with CNPG so I'll most likely go with kafka operator also.



When it comes to the calibration part I think I'll create a simple service, it works quite well with the microservice thinking, where I just create a service that is very small, isolated, statelss, all it does is collect from kafka stream, calibrate and save.

Maybe read config from kubernetes or it get it based on deployment.

(in case we want to change the calibration data).



However when it comes to the data collection I am bit torn.

I could create a very simple service here too and be done with it.

But for learning experience I was wondering if kafka connector might be interesting?

If I understand it correctly I can just have it move data between postgres and kafka?

One job that I haven't consider where this might help is when I want to recalibrate I want to delete all calibrated data in the DB then queue up all raw data from postgres into kafka, so would kafka connect help with this situation?

---

**Cursor**

Great questions! Let me break down when Kafka Connect makes sense and specifically address your recalibration scenario.

## Kafka Connect: Two Types

**Kafka Connect has two connector types:**

1. **Source Connectors**: Pull data FROM external systems INTO Kafka
   - JDBC Source: Postgres → Kafka
   - Debezium: Postgres CDC → Kafka

2. **Sink Connectors**: Push data FROM Kafka INTO external systems
   - JDBC Sink: Kafka → Postgres

## Your Data Collection Flow Options

### **Option A: Simple Service (My Recommendation for Your Case)**

```
Sensors → ServerPod API → Kafka (raw-temp topic)
                             ↓
                    Simple Consumer Service:
                      - Consume from Kafka
                      - Save to Postgres
                      - On success, commit offset
```

**Pros:**
- ✅ Full control over error handling
- ✅ Can add business logic (validation, enrichment)
- ✅ Easy to add metrics/logging specific to your needs
- ✅ Can handle idempotency with upserts
- ✅ Simple to debug and understand

**Cons:**
- ❌ You write more code
- ❌ Need to handle database connection pooling yourself

### **Option B: Kafka Connect JDBC Sink**

```
Sensors → ServerPod API → Kafka (raw-temp topic)
                             ↓
                    Kafka Connect JDBC Sink
                      - Automatically writes to Postgres
                      - Configured via JSON
```

**Pros:**
- ✅ Zero code for data persistence
- ✅ Built-in exactly-once semantics
- ✅ Automatic batching and performance optimization
- ✅ Schema evolution support
- ✅ Dead letter queue handling built-in

**Cons:**
- ❌ Less flexibility for custom logic
- ❌ Debugging can be more opaque
- ❌ Configuration can be tricky initially

## The Recalibration Scenario: Where Kafka Connect Shines!

This is **exactly** where Kafka Connect (Source connector) becomes very interesting:

### **Recalibration Flow with Kafka Connect JDBC Source:**

```sql
-- 1. Delete all calibrated data
DELETE FROM calibrated_temperatures WHERE created_at > '2025-01-01';

-- 2. Kafka Connect JDBC Source polls this query:
SELECT id, sensor_id, raw_value, created_at 
FROM raw_temperatures 
WHERE created_at > '2025-01-01'
AND id > :last_processed_id
ORDER BY id ASC
```

The JDBC Source connector will:
- Automatically page through all raw data
- Produce each row as a message to Kafka
- Your calibration service processes them as they arrive
- Store the last processed ID (offset) so it can resume if interrupted

**Configuration example:**
```json
{
  "name": "raw-temperature-source",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:postgresql://postgres:5432/cozytemp",
    "connection.user": "cozytemp",
    "connection.password": "${file:/secrets/db-password:password}",
    "mode": "incrementing",
    "incrementing.column.name": "id",
    "topic.prefix": "recalibration-",
    "table.whitelist": "raw_temperatures",
    "poll.interval.ms": 1000
  }
}
```

### **Alternative: Debezium CDC for Recalibration**

Even better for your use case might be **Debezium**, which uses Change Data Capture:

```
Postgres WAL → Debezium → Kafka (raw-temperature topic)
```

**Why Debezium is powerful for your scenario:**
- Captures ALL changes (inserts, updates, deletes) from Postgres
- Doesn't poll the database (no load on DB)
- Guarantees ordering
- Can snapshot existing data AND stream new changes
- Can replay from a specific point in time

**For recalibration:**
```bash
# Create a new Debezium connector with snapshot mode
# It will:
# 1. Take a snapshot of entire raw_temperatures table
# 2. Stream all existing data to Kafka
# 3. Then continue streaming new changes
```

## My Recommended Architecture for You

Based on your learning goals and recalibration requirement:

```
┌─────────────────────────────────────────────────────────────┐
│                    Normal Flow                               │
└─────────────────────────────────────────────────────────────┘

Sensors → ServerPod API → Kafka (raw-temperature topic)
                             ↓
                    [Simple Consumer Service]
                      - Save raw data to Postgres
                      - Produce to calibration-queue topic
                             ↓
                    [Calibration Service]
                      - Consume from calibration-queue
                      - Apply calibration formula
                      - Save to Postgres (calibrated table)


┌─────────────────────────────────────────────────────────────┐
│                  Recalibration Flow                          │
└─────────────────────────────────────────────────────────────┘

Manual trigger (K8s Job or API call):
  1. DELETE FROM calibrated_temperatures WHERE sensor_id = 'X';
  2. Enable Kafka Connect JDBC Source connector
     - Queries: SELECT * FROM raw_temperatures WHERE sensor_id = 'X'
     - Produces to recalibration-raw-temperature topic
  3. Your same Calibration Service consumes from that topic
  4. Processes at its own pace (3-4k at a time, as you mentioned)
```

## Practical Implementation

### **Normal Flow: Simple Service**

```plaintext
Sensors → ServerPod API → Kafka (raw-temp topic)
                             ↓
                    Simple Consumer Service:
                      - Consume from Kafka
                      - Save to Postgres
                      - On success, commit offset
```

```plaintext
Sensors → ServerPod API → Kafka (raw-temp topic)
                             ↓
                    Kafka Connect JDBC Sink
                      - Automatically writes to Postgres
                      - Configured via JSON
```

```sql
-- 1. Delete all calibrated data
DELETE FROM calibrated_temperatures WHERE created_at > '2025-01-01';

-- 2. Kafka Connect JDBC Source polls this query:
SELECT id, sensor_id, raw_value, created_at 
FROM raw_temperatures 
WHERE created_at > '2025-01-01'
AND id > :last_processed_id
ORDER BY id ASC
```

```json
{
  "name": "raw-temperature-source",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "connection.url": "jdbc:postgresql://postgres:5432/cozytemp",
    "connection.user": "cozytemp",
    "connection.password": "${file:/secrets/db-password:password}",
    "mode": "incrementing",
    "incrementing.column.name": "id",
    "topic.prefix": "recalibration-",
    "table.whitelist": "raw_temperatures",
    "poll.interval.ms": 1000
  }
}
```

```plaintext
Postgres WAL → Debezium → Kafka (raw-temperature topic)
```

```shellscript
# Create a new Debezium connector with snapshot mode
# It will:
# 1. Take a snapshot of entire raw_temperatures table
# 2. Stream all existing data to Kafka
# 3. Then continue streaming new changes
```

```plaintext
┌─────────────────────────────────────────────────────────────┐
│                    Normal Flow                               │
└─────────────────────────────────────────────────────────────┘

Sensors → ServerPod API → Kafka (raw-temperature topic)
                             ↓
                    [Simple Consumer Service]
                      - Save raw data to Postgres
                      - Produce to calibration-queue topic
                             ↓
                    [Calibration Service]
                      - Consume from calibration-queue
                      - Apply calibration formula
                      - Save to Postgres (calibrated table)


┌─────────────────────────────────────────────────────────────┐
│                  Recalibration Flow                          │
└─────────────────────────────────────────────────────────────┘

Manual trigger (K8s Job or API call):
  1. DELETE FROM calibrated_temperatures WHERE sensor_id = 'X';
  2. Enable Kafka Connect JDBC Source connector
     - Queries: SELECT * FROM raw_temperatures WHERE sensor_id = 'X'
     - Produces to recalibration-raw-temperature topic
  3. Your same Calibration Service consumes from that topic
  4. Processes at its own pace (3-4k at a time, as you mentioned)
```

You write a simple Go/Python service:

```go
// Simple consumer for normal flow
func consumeAndSave(ctx context.Context) {
    consumer := kafka.NewConsumer("raw-temperature")
    
    for {
        msg := consumer.Poll(100 * time.Millisecond)
        if msg == nil {
            continue
        }
        
        // 1. Save to Postgres
        rawID, err := db.SaveRawTemperature(msg.Value)
        if err != nil {
            log.Error("Failed to save", err)
            continue // Don't commit, will retry
        }
        
        // 2. Queue for calibration
        producer.Produce("calibration-queue", msg.Value)
        
        // 3. Commit offset (even if step 2 fails, we saved raw data)
        consumer.Commit(msg)
    }
}
```

### **Recalibration: Kafka Connect + K8s Job**

```yaml
# Job to trigger recalibration
apiVersion: batch/v1
kind: Job
metadata:
  name: recalibrate-sensor-001
  namespace: cozytemp
spec:
  template:
    spec:
      containers:
      - name: trigger-recalibration
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          # 1. Delete calibrated data
          psql $DATABASE_URL -c "DELETE FROM calibrated_temperatures WHERE sensor_id = 'sensor-001'"
          
          # 2. Create Kafka Connect Source connector
          curl -X POST http://kafka-connect:8083/connectors \
            -H "Content-Type: application/json" \
            -d '{
              "name": "recalibrate-sensor-001",
              "config": {
                "connector.class": "JdbcSourceConnector",
                "connection.url": "jdbc:postgresql://...",
                "query": "SELECT * FROM raw_temperatures WHERE sensor_id = '\''sensor-001'\''",
                "mode": "bulk",
                "topic.prefix": "recalibration-"
              }
            }'
          
          echo "Recalibration started. Monitor calibration-service logs."
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: cozytemp-pg-app
              key: uri
      restartPolicy: Never
```

Your calibration service just consumes from **both** topics:
- `calibration-queue` (normal real-time flow)
- `recalibration-raw-temperatures` (bulk recalibration)

## When to Use What

| Scenario | Tool | Why |
|----------|------|-----|
| **Normal data collection** | Simple Service | Full control, easy debugging |
| **Bulk recalibration** | Kafka Connect JDBC Source | Automatic paging, no custom code |
| **Real-time guarantees** | Debezium CDC | Captures everything, no polling |
| **Data persistence** | Simple Service | Can add validation/business logic |

## My Specific Advice for You

**Start with:**
1. ✅ **Simple consumer service** for normal flow (ServerPod → Kafka → Your Service → Postgres)
2. ✅ **Simple calibration service** (stateless, reads from Kafka)
3. ✅ Deploy **Kafka Connect** alongside Strimzi (it's just one more deployment)
4. ✅ Keep JDBC Source connector config ready but **disabled by default**

**When you need to recalibrate:**
1. Run a K8s Job that:
   - Deletes calibrated data from Postgres
   - Enables/creates the JDBC Source connector with appropriate query
   - The connector streams raw data back through Kafka
   - Your existing calibration service processes it
   - When done, delete the connector

**Why this hybrid approach:**
- Learning experience: You build the consumer/calibration services yourself
- Kafka Connect: You use it for the "batch reprocessing" use case where it excels
- Best of both worlds: Control where you need it, automation where it helps

---

Would you like me to help you with:
1. Setting up Strimzi with Kafka Connect enabled?
2. Creating a template for the simple consumer service?
3. Writing the JDBC Source connector configuration for recalibration?
4. Designing the Kubernetes Job for triggering recalibration?

```go
// Simple consumer for normal flow
func consumeAndSave(ctx context.Context) {
    consumer := kafka.NewConsumer("raw-temperature")
    
    for {
        msg := consumer.Poll(100 * time.Millisecond)
        if msg == nil {
            continue
        }
        
        // 1. Save to Postgres
        rawID, err := db.SaveRawTemperature(msg.Value)
        if err != nil {
            log.Error("Failed to save", err)
            continue // Don't commit, will retry
        }
        
        // 2. Queue for calibration
        producer.Produce("calibration-queue", msg.Value)
        
        // 3. Commit offset (even if step 2 fails, we saved raw data)
        consumer.Commit(msg)
    }
}
```

```yaml
# Job to trigger recalibration
apiVersion: batch/v1
kind: Job
metadata:
  name: recalibrate-sensor-001
  namespace: cozytemp
spec:
  template:
    spec:
      containers:
      - name: trigger-recalibration
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          # 1. Delete calibrated data
          psql $DATABASE_URL -c "DELETE FROM calibrated_temperatures WHERE sensor_id = 'sensor-001'"
          
          # 2. Create Kafka Connect Source connector
          curl -X POST http://kafka-connect:8083/connectors \
            -H "Content-Type: application/json" \
            -d '{
              "name": "recalibrate-sensor-001",
              "config": {
                "connector.class": "JdbcSourceConnector",
                "connection.url": "jdbc:postgresql://...",
                "query": "SELECT * FROM raw_temperatures WHERE sensor_id = '\''sensor-001'\''",
                "mode": "bulk",
                "topic.prefix": "recalibration-"
              }
            }'
          
          echo "Recalibration started. Monitor calibration-service logs."
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: cozytemp-pg-app
              key: uri
      restartPolicy: Never
```

```go
// Simple consumer for normal flow
func consumeAndSave(ctx context.Context) {
    consumer := kafka.NewConsumer("raw-temperature")
    
    for {
        msg := consumer.Poll(100 * time.Millisecond)
        if msg == nil {
            continue
        }
        
        // 1. Save to Postgres
        rawID, err := db.SaveRawTemperature(msg.Value)
        if err != nil {
            log.Error("Failed to save", err)
            continue // Don't commit, will retry
        }
        
        // 2. Queue for calibration
        producer.Produce("calibration-queue", msg.Value)
        
        // 3. Commit offset (even if step 2 fails, we saved raw data)
        consumer.Commit(msg)
    }
}
```

```yaml
# Job to trigger recalibration
apiVersion: batch/v1
kind: Job
metadata:
  name: recalibrate-sensor-001
  namespace: cozytemp
spec:
  template:
    spec:
      containers:
      - name: trigger-recalibration
        image: curlimages/curl:latest
        command:
        - sh
        - -c
        - |
          # 1. Delete calibrated data
          psql $DATABASE_URL -c "DELETE FROM calibrated_temperatures WHERE sensor_id = 'sensor-001'"
          
          # 2. Create Kafka Connect Source connector
          curl -X POST http://kafka-connect:8083/connectors \
            -H "Content-Type: application/json" \
            -d '{
              "name": "recalibrate-sensor-001",
              "config": {
                "connector.class": "JdbcSourceConnector",
                "connection.url": "jdbc:postgresql://...",
                "query": "SELECT * FROM raw_temperatures WHERE sensor_id = '\''sensor-001'\''",
                "mode": "bulk",
                "topic.prefix": "recalibration-"
              }
            }'
          
          echo "Recalibration started. Monitor calibration-service logs."
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: cozytemp-pg-app
              key: uri
      restartPolicy: Never
```

