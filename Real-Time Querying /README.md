# 📊 Real-Time Querying 

## 📋 Prerequisites
* 💻 Basic Linux command-line skills
* 🗄️ Understanding of SQL fundamentals
* 📄 Familiarity with JSON data format
* 🔄 Basic knowledge of streaming concepts

---

## 🎯 Learning Objectives
By completing this lab, you will:
* 🛠️ Set up Apache Kafka for streaming data ingestion
* ⚙️ Configure Apache Druid for real-time analytics
* 📥 Insert streaming data into a real-time data store
* ⚡ Execute low-latency queries on streaming data
* 📈 Monitor real-time query performance

---

## 🏗️ Environment Setup

### 🖥️ System Requirements
* Linux machine with at least 4GB RAM
* Java 11 or higher
* Python 3.8+
* Internet connection for downloading packages

### 🔧 Step 1: Install Required Tools
```bash
# Update system packages
sudo apt update

# Install Java
sudo apt install -y openjdk-11-jdk wget curl

# Verify Java installation
java -version

# Install Python and pip
sudo apt install -y python3 python3-pip

# Install Kafka Python client
pip3 install kafka-python
```

### 📦 Step 2: Download and Setup Apache Kafka
```bash
# Download Kafka
cd ~
wget https://apache.org
tar -xzf kafka_2.13-3.5.1.tgz
cd kafka_2.13-3.5.1

# Start Zookeeper (in background)
bin/zookeeper-server-start.sh config/zookeeper.properties > ~/zookeeper.log 2>&1 &

# Wait 10 seconds for Zookeeper to start
sleep 10

# Start Kafka broker (in background)
bin/kafka-server-start.sh config/server.properties > ~/kafka.log 2>&1 &

# Wait 15 seconds for Kafka to start
sleep 15
```

### 🚀 Step 3: Download and Setup Apache Druid
```bash
# Download Druid
cd ~
wget https://apache.org
tar -xzf apache-druid-28.0.0-bin.tar.gz
cd apache-druid-28.0.0

# Start Druid in micro-quickstart mode (in background)
./bin/start-micro-quickstart > ~/druid.log 2>&1 &

# Wait 60 seconds for Druid to initialize
sleep 60
```

---

## 📥 Task 1: Insert Streaming Data

### 🛣️ Step 1: Create Kafka Topic
```bash
cd ~/kafka_2.13-3.5.1

# Create a topic for sensor data
bin/kafka-topics.sh --create \
  --topic sensor-events \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1

# Verify topic creation
bin/kafka-topics.sh --list --bootstrap-server localhost:9092
```

### 🐍 Step 2: Create Data Producer Script
Open your terminal editor to build the publisher script:
```bash
nano ~/sensor_producer.py
```

Add the following starter code and complete the `TODO` sections:
```python
from kafka import KafkaProducer
import json
import time
import random
from datetime import datetime

def create_sensor_event():
    """
    Generate a sensor event with timestamp, sensor_id, temperature, and humidity.
    
    Returns:
        dict: Sensor event data
    """
    # TODO: Create a dictionary with the following fields:
    # - timestamp: current ISO format timestamp
    # - sensor_id: random choice from ['sensor_1', 'sensor_2', 'sensor_3']
    # - temperature: random float between 15.0 and 35.0
    # - humidity: random float between 30.0 and 80.0
    # - location: random choice from ['warehouse_a', 'warehouse_b', 'warehouse_c']
    pass

def send_events(producer, topic, num_events=100):
    """
    Send sensor events to Kafka topic.
    
    Args:
        producer: KafkaProducer instance
        topic: Kafka topic name
        num_events: Number of events to send
    """
    # TODO: Loop num_events times
    # TODO: Create event using create_sensor_event()
    # TODO: Send event to Kafka using producer.send()
    # TODO: Print confirmation message
    # TODO: Sleep for 0.5 seconds between events
    pass

if __name__ == "__main__":
    # TODO: Initialize KafkaProducer with:
    # - bootstrap_servers=['localhost:9092']
    # - value_serializer to convert dict to JSON bytes
    
    # TODO: Call send_events() with producer, topic='sensor-events', num_events=100
    
    # TODO: Close producer
    pass
```

### ⚙️ Step 3: Configure Druid Ingestion Spec
Open your editor to set up how Apache Druid parses incoming streams:
```bash
nano ~/sensor-ingestion-spec.json
```

Add the ingestion configuration map below:
```json
{
  "type": "kafka",
  "spec": {
    "ioConfig": {
      "type": "kafka",
      "consumerProperties": {
        "bootstrap.servers": "localhost:9092"
      },
      "topic": "sensor-events",
      "inputFormat": {
        "type": "json"
      },
      "useEarliestOffset": true
    },
    "tuningConfig": {
      "type": "kafka",
      "maxRowsPerSegment": 5000000
    },
    "dataSchema": {
      "dataSource": "sensor_data",
      "timestampSpec": {
        "column": "timestamp",
        "format": "iso"
      },
      "dimensionsSpec": {
        "dimensions": [
          "sensor_id",
          "location"
        ]
      },
      "metricsSpec": [
        {
          "type": "doubleSum",
          "name": "total_temperature",
          "fieldName": "temperature"
        },
        {
          "type": "doubleSum",
          "name": "total_humidity",
          "fieldName": "humidity"
        },
        {
          "type": "count",
          "name": "event_count"
        }
      ],
      "granularitySpec": {
        "type": "uniform",
        "segmentGranularity": "HOUR",
        "queryGranularity": "MINUTE",
        "rollup": true
      }
    }
  }
}
```

### 📤 Step 4: Submit Ingestion Task to Druid
```bash
# Submit the ingestion spec to Druid
curl -X POST -H 'Content-Type: application/json' \
  -d @~/sensor-ingestion-spec.json \
  http://localhost:8888/druid/indexer/v1/supervisor

# Verify supervisor is running
curl http://localhost:8888/druid/indexer/v1/supervisor
```

### 🟢 Step 5: Start Data Streaming
```bash
# Run the producer script
python3 ~/sensor_producer.py
```
> **💡 Expected Output:** Continuous screen prints verifying events are successfully transmitting to the Kafka cluster broker.

---

## ⚡ Task 2: Run Real-Time Queries

### 🔍 Step 1: Verify Data Ingestion
Allow 30 seconds for the ingestion layer to pipe initial segments, then verify availability:
```bash
# Check available datasources
curl http://localhost:8888/druid/coordinator/v1/datasources
```

### 🛠️ Step 2: Execute Basic Real-Time Query
Create the Python reporting consumer script:
```bash
nano ~/query_realtime.py
```

Add the blueprint template below and complete the requested functions:
```python
import requests
import json
from datetime import datetime, timedelta

DRUID_BROKER = "http://localhost:8888"

def execute_query(query):
    """
    Execute a Druid SQL query.
    
    Args:
        query: SQL query string
    
    Returns:
        Query results as JSON
    """
    # TODO: Send POST request to DRUID_BROKER/druid/v2/sql
    # TODO: Set headers to {'Content-Type': 'application/json'}
    # TODO: Send query in format: {"query": query}
    # TODO: Return response.json()
    pass

def query_recent_events(minutes=5):
    """
    Query events from the last N minutes.
    
    Args:
        minutes: Number of minutes to look back
    
    Returns:
        Query results
    """
    query = f"""
    SELECT 
        __time,
        sensor_id,
        location,
        SUM(total_temperature) / SUM(event_count) as avg_temperature,
        SUM(total_humidity) / SUM(event_count) as avg_humidity,
        SUM(event_count) as count
    FROM sensor_data
    WHERE __time >= CURRENT_TIMESTAMP - INTERVAL '{minutes}' MINUTE
    GROUP BY __time, sensor_id, location
    ORDER BY __time DESC
    LIMIT 20
    """
    # TODO: Call execute_query() with the query
    # TODO: Return results
    pass

def query_aggregated_by_location():
    """
    Query aggregated metrics by location for the last 10 minutes.
    
    Returns:
        Aggregated results by location
    """
    # TODO: Write SQL query to:
    # - Select location
    # - Calculate average temperature and humidity
    # - Count total events
    # - Filter for last 10 minutes
    # - Group by location
    # TODO: Execute and return results
    pass

def query_sensor_anomalies(temp_threshold=30.0):
    """
    Find sensors with temperature above threshold.
    
    Args:
        temp_threshold: Temperature threshold
    
    Returns:
        Sensors exceeding threshold
    """
    # TODO: Write SQL query to find sensors where
    # average temperature > temp_threshold in last 5 minutes
    # TODO: Execute and return results
    pass

if __name__ == "__main__":
    print("=== Recent Events ===")
    # TODO: Call query_recent_events() and print results
    
    print("\n=== Aggregated by Location ===")
    # TODO: Call query_aggregated_by_location() and print results
    
    print("\n=== Sensor Anomalies ===")
    # TODO: Call query_sensor_anomalies() and print results
```

---

## 🛠️ Troubleshooting & Health Checks

### 🔍 System Port Mapping Reference
If components fail to bind, confirm these default loopback ports are completely free:
* **`2181`**: Apache Zookeeper Coordination Service
* **`9092`**: Apache Kafka Broker Stream Interface
* **`8888`**: Apache Druid Micro-Quickstart Router Interface

### 📑 Log Analysis Commands
If pipelines fail silently, look inside these logs to find error tracks:
```bash
# Check Kafka execution trails
tail -n 50 ~/kafka.log

# Check Druid ingestion runtime maps
tail -n 50 ~/druid.log
```

---

