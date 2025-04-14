
# 📘 Assignment: Cost Optimization Challenge – Managing Billing Records in Azure Serverless Architecture

## 🔎 Problem Statement

We have a serverless architecture in Azure where one of our services stores billing records in **Azure Cosmos DB**. Over time, the data volume has significantly increased, leading to **escalating operational costs**. Although the system is **read-heavy**, records older than **three months are rarely accessed**. These older records still need to be served with **acceptable latency**.

---

## ✅ Current Constraints

- **Record Size**: Up to 300 KB per billing record
- **Total Records**: Over 2 million
- **Read Pattern**: Read-heavy; minimal writes/updates
- **Availability Requirement**: Older records must be accessible within a few seconds
- **Implementation Requirements**:
  - ❌ No changes to API contracts
  - ❌ No downtime or data loss
  - ✅ Must be simple, scalable, and easy to maintain

---

## 🎯 Proposed Solution – Tiered Storage Strategy

### 💡 Core Idea

Implement a **hot-cold tiered storage model** by segregating recent (active) and historical (archived) data.

| Tier       | Storage Type         | Description                                  |
|------------|----------------------|----------------------------------------------|
| Hot Tier   | Azure Cosmos DB      | Holds records from the last 90 days          |
| Cold Tier  | Azure Blob Storage   | Stores records older than 90 days (as JSON)  |

An **Azure Timer Trigger Function** handles daily archival. A **read abstraction layer** (via Azure Function or APIM policy) seamlessly retrieves data from either Cosmos DB or Blob Storage.

---

## ⚙️ Technical Components

### 1. Archival Function (Timer Trigger)

- Scheduled to run daily
- Moves records older than 90 days from Cosmos DB to Blob Storage
- Ensures successful transfer before deletion from Cosmos DB

### 2. Read Handler Function

- Attempts to read from Cosmos DB first
- Falls back to Blob Storage if record not found
- Returns consistent data structure to client

---

## 🧪 Implementation Snippets

### 🔄 Archival Function (Python)

```python
from datetime import datetime, timedelta
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
import json

def main():
    client = CosmosClient("<COSMOS_URI>", "<KEY>")
    container = client.get_database_client("billingdb").get_container_client("records")

    blob_service = BlobServiceClient.from_connection_string("<BLOB_CONN>")
    blob_container = blob_service.get_container_client("billing-archive")

    threshold = (datetime.utcnow() - timedelta(days=90)).isoformat()
    query = f"SELECT * FROM c WHERE c.timestamp < '{threshold}'"
    old_records = container.query_items(query, enable_cross_partition_query=True)

    for record in old_records:
        blob_name = f"{record['id']}.json"
        blob_container.upload_blob(blob_name, json.dumps(record), overwrite=True)
        container.delete_item(record, partition_key=record['id'])
```

---

### 🔍 Read Handler Function (Python)

```python
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
import json

def get_billing_record(record_id):
    cosmos_result = try_cosmos(record_id)
    if cosmos_result:
        return cosmos_result
    return try_blob_storage(record_id)

def try_cosmos(record_id):
    try:
        container = CosmosClient("<COSMOS_URI>", "<KEY>")             .get_database_client("billingdb")             .get_container_client("records")
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

## 📈 Benefits

- 💰 **Reduced storage cost** by offloading infrequently accessed records to Blob Storage
- 🔁 **No impact on existing APIs or user experience**
- ⚙️ **Seamless, automated archival and retrieval**
- ☁️ **Scalable, serverless architecture**

---

## 🗂 Project Structure

```text
/
├── archive_old_records/        # Timer Function to archive old billing data
├── get_billing_record/         # Function to retrieve record (from Cosmos or Blob)
├── requirements.txt            # Python dependencies
├── README.md                   # Deployment guide
└── Assignment.md               # Documentation (this file)
```

---

## 📝 Additional Notes

- The archival process is **idempotent** — ensures no duplication or premature deletion
- Azure Blob Storage supports **cool/archive tiers** and **lifecycle management policies** for additional cost savings
- Can be enhanced to support encryption, compression, or indexing if needed

---

## 👨‍💻 Contributor

**Jayaramakrishna Puvvada**  
📞 9963060631  
📧 jayaramakrishnapuvvada@gmail.com  
