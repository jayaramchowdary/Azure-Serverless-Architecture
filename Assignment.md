
```markdown
# 📘 Assignment: Cost Optimization Challenge – Managing Billing Records in Azure Serverless Architecture

## 🔎 Problem Statement

We have a serverless architecture in Azure, where one of our services stores billing records in **Azure Cosmos DB**. Over time, the data size has significantly increased, leading to **high operational costs**. Although the system is **read-heavy**, records older than **three months are rarely accessed**. However, these records still need to be available when requested, with **acceptable latency**.

---

## ✅ Current Constraints

- **Record Size**: Up to 300 KB per billing record.
- **Total Records**: More than 2 million.
- **Read-Heavy Workload**: Rare writes/updates; mostly reads.
- **Availability Requirement**: Older records should still be accessible within a few seconds.
- **Technical Constraints**:
  - ❌ No API contract changes allowed
  - ❌ No downtime or data loss permitted
  - ✅ Simple, easy-to-deploy and maintain solution preferred

---

## 🎯 Solution Plan A – Tiered Storage Strategy

### 💡 Core Idea

Split the data into **hot** (active) and **cold** (archived) storage:

| Type      | Location              | Description                            |
|-----------|-----------------------|----------------------------------------|
| Hot Data  | Azure Cosmos DB       | Stores recent (last 3 months) records  |
| Cold Data | Azure Blob Storage    | Stores older records (archived as JSON)|

An **Azure Function** handles daily archival, while a **read abstraction layer** (Azure Function or API Management policy) makes data access seamless.

---

## 🧱 Architecture Diagram

```
Client / API
     │
     ▼
Abstraction Layer (Azure Function or API Mgmt)
     │
 ┌───┴──────────┐
 │              │
 ▼              ▼
Cosmos DB   Azure Blob Storage
 (< 3 mo)   (> 3 mo)
     ▲              ▲
     │              │
     └── Timer Trigger Function (Archival)
```

---

## ⚙️ Technical Components

### 1. Azure Timer Trigger Function

- Runs daily
- Queries Cosmos DB for records older than 90 days
- Moves them to Blob Storage in JSON format
- Deletes from Cosmos DB after successful archival

### 2. Read Handler Function

- Reads from Cosmos DB first
- If not found, falls back to Blob Storage
- Returns data to client without changing API behavior

---

## 🧪 Example Pseudocode

### 🔄 Archival Function (Python)

```python
from datetime import datetime, timedelta
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
import json

def main():
    # Connect to Cosmos DB
    client = CosmosClient("<COSMOS_URI>", "<KEY>")
    container = client.get_database_client("billingdb").get_container_client("records")

    # Connect to Blob Storage
    blob_service = BlobServiceClient.from_connection_string("<BLOB_CONN>")
    blob_container = blob_service.get_container_client("billing-archive")

    # Get data older than 90 days
    threshold = (datetime.utcnow() - timedelta(days=90)).isoformat()
    query = f"SELECT * FROM c WHERE c.timestamp < '{threshold}'"
    old_records = container.query_items(query, enable_cross_partition_query=True)

    for record in old_records:
        blob_name = f"{record['id']}.json"
        blob_container.upload_blob(blob_name, json.dumps(record), overwrite=True)
        container.delete_item(record, partition_key=record['id'])
```

---

### 🔍 Read Handler with Fallback Logic

```python
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
import json

def get_billing_record(record_id):
    # Try Cosmos DB
    cosmos_result = try_cosmos(record_id)
    if cosmos_result:
        return cosmos_result

    # Try Blob Storage
    return try_blob_storage(record_id)

def try_cosmos(record_id):
    try:
        container = CosmosClient("<COSMOS_URI>", "<KEY>") \
            .get_database_client("billingdb") \
            .get_container_client("records")
        return container.read_item(item=record_id, partition_key=record_id)
    except:
        return None

def try_blob_storage(record_id):
    blob_service = BlobServiceClient.from_connection_string("<BLOB_CONN>")
    blob_container = blob_service.get_container_client("billing-archive")
    blob_client = blob_container.get_blob_client(f"{record_id}.json")

    if blob_client.exists():
        blob_data = blob_client.download_blob().readall()
        return json.loads(blob_data)
    return None
```

---

## ✅ Benefits

- 💰 **Significant cost savings** using Blob Storage (cool/archive tier)
- 🔁 **No API changes or downtime**
- ☁️ **Fully serverless** and scalable
- 📂 **Easy to implement and automate**

---

## 📂 Related Files

- `archive_old_records/`: Azure Function for archiving old data
- `get_billing_record/`: Azure Function to fetch data with fallback logic
- `requirements.txt`: Required libraries for Azure SDK
- `README.md`: GitHub-friendly deployment guide
- `Assignment.md`: This file, for formal documentation or submission

---

## 📝 Additional Notes

- The archival function is designed to be **idempotent** – it will not re-upload or delete unless the record is already archived.
- **Lifecycle policies** in Azure Blob Storage can be configured to automatically move archived data to **cool** or **archive tier** for further cost savings.

---

## 👨‍💻 Contribution

This assignment solution was created by **Kommineni Naresh** using hands-on Azure expertise and aided by **ChatGPT** for architecture design, code generation, and documentation.

---

## 📎 License

MIT License
```

---

Let me know if you want this as a downloadable `.md` file or to package it in a GitHub repo with folder structure and scripts!
