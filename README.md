# 📈 Data Engineering Project: Real-Time Stock Price Data Pipeline

### 🚀 **Goal:**
Build a robust data pipeline that streams live stock prices, processes them, and stores the results for visualization.

---

### 🛠 **Tech Stack:**
- **Programming Language:** Python
- **Data Streaming:** Kafka
- **Data Processing:** Apache Spark
- **Database:** PostgreSQL
- **Cloud Storage:** AWS S3
- **Visualization:** Streamlit

---

## 🧑‍💻 **Project Steps:**

### 1. Set Up the Environment

#### 📥 **Install Dependencies:**
```bash
pip install kafka-python pandas requests psycopg2-binary boto3 apache-spark
```

#### 🔗 **Set Up Kafka Locally (or Use Cloud Services)**
- Download and install Apache Kafka: [Kafka Quick Start](https://kafka.apache.org/quickstart)
- Start a Kafka broker and Zookeeper
- Create a Kafka topic for stock prices:
```bash
bin/kafka-topics.sh --create --topic stock_prices --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

---

### 2. Build the Data Pipeline

#### 📡 **Producer - Fetch Stock Prices & Send to Kafka**
Create a Python script (`producer.py`) to fetch live stock prices using the Yahoo Finance API and send them to Kafka.

```python
from kafka import KafkaProducer
import requests
import json
import time

# Initialize Kafka producer
producer = KafkaProducer(bootstrap_servers='localhost:9092',
                         value_serializer=lambda v: json.dumps(v).encode('utf-8'))

def fetch_stock_price(symbol="AAPL"):
    url = f"https://query1.finance.yahoo.com/v7/finance/quote?symbols={symbol}"
    response = requests.get(url)
    data = response.json()
    return data['quoteResponse']['result'][0]

while True:
    stock_data = fetch_stock_price()
    message = {
        "symbol": stock_data['symbol'],
        "price": stock_data['regularMarketPrice'],
        "time": stock_data['regularMarketTime']
    }
    producer.send('stock_prices', message)
    print(f"Sent: {message}")
    time.sleep(5)  # Fetch every 5 seconds
```

#### 🔄 **Consumer - Process Data Using Apache Spark**
Create a Spark consumer (`consumer.py`) that reads from Kafka, computes a moving average, and stores the result in PostgreSQL.

```python
from pyspark.sql import SparkSession
from kafka import KafkaConsumer
import json
import psycopg2

# Initialize Spark
spark = SparkSession.builder.appName("StockProcessor").getOrCreate()

# Kafka Consumer
consumer = KafkaConsumer('stock_prices', bootstrap_servers='localhost:9092',
                         value_deserializer=lambda x: json.loads(x.decode('utf-8')))

# PostgreSQL Connection
conn = psycopg2.connect(database="stocks_db", user="user", password="password", host="localhost", port="5432")
cursor = conn.cursor()

def process_data():
    for msg in consumer:
        stock_data = msg.value
        symbol = stock_data['symbol']
        price = stock_data['price']
        timestamp = stock_data['time']
        cursor.execute("INSERT INTO stock_prices (symbol, price, timestamp) VALUES (%s, %s, %s)",
                       (symbol, price, timestamp))
        conn.commit()
        print(f"Inserted: {symbol} - ${price} at {timestamp}")

process_data()
```

---

### 3. Store Data in AWS S3
Modify the consumer script (`consumer.py`) to upload raw data to AWS S3 using `boto3`.

```python
import boto3

s3_client = boto3.client('s3', aws_access_key_id="YOUR_ACCESS_KEY", aws_secret_access_key="YOUR_SECRET_KEY")
bucket_name = "stock-price-data"

def upload_to_s3(data):
    file_name = f"{data['symbol']}_{data['time']}.json"
    s3_client.put_object(Bucket=bucket_name, Key=file_name, Body=json.dumps(data))
    print(f"Uploaded {file_name} to S3")

for msg in consumer:
    stock_data = msg.value
    upload_to_s3(stock_data)
```

---

### 4. Visualize the Data with Streamlit
Create a `dashboard.py` file to display live stock trends using Streamlit.

```python
import streamlit as st
import psycopg2
import pandas as pd

# Connect to PostgreSQL
conn = psycopg2.connect(database="stocks_db", user="user", password="password", host="localhost", port="5432")

st.title("📈 Real-Time Stock Price Dashboard")

# Fetch data
df = pd.read_sql("SELECT * FROM stock_prices ORDER BY timestamp DESC LIMIT 50", conn)

st.line_chart(df.set_index("timestamp")['price'])
st.write(df)
```

---

## 🎯 **Key Takeaways:**
- Real-time data streaming with Kafka
- Data processing and analytics using Apache Spark
- Secure and efficient data storage with PostgreSQL and AWS S3
- Interactive data visualization with Streamlit

🛠️ **Ready to run?** Clone the repository and follow the setup instructions to get started!

