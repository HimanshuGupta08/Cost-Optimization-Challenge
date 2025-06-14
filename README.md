# Cost-Optimization-Challenge

Managing Billing Records in Azure Serverless Architecture

We have a serverless architecture in Azure, where one of our services stores billing records in Azure Cosmos DB. The system is read-heavy, but records older than three months are rarely accessed.

Over the past few years, the database size has significantly grown, leading to increased costs. We need an efficient way to reduce costs while maintaining data availability.

Current System Constraints
Record Size: Each billing record can be as large as 300 KB.

Total Records: The database currently holds over 2 million records.

Access Latency: When an old record is requested, it should still be served, with a response time in the order of seconds.

Solution Requirements

Please propose a detailed solution to optimize costs while ensuring the following
Simplicity & Ease of Implementation – The solution should be straightforward to deploy and maintain.

No Data Loss & No Downtime – The transition should be seamless, without losing any records or requiring2 service downtime.

No Changes to API Contracts – The existing read/write APIs for billing records must remain unchanged

Bonus Points
Include an architecture diagram illustrating your proposed solution.

Provide pseudocode, commands, or scripts for implementing core logic (such as data archival, retrieval, and cost optimization st rategies).

Please share the solution in a Github repo. If you are using chatGPT or other LLMs to solve please share the conversation.

-------------------------------------------------------------------------------------------------------------------------------------------------------------------
# SOLUTION STARTS HERE

## **Challenge Summary**

You have:

* **Azure Serverless architecture**
* **Cosmos DB** (2M+ records, 300 KB each)
* **Read-heavy workload**
* **Old records (3+ months) are rarely accessed**

---

## ✅ **Goals**

1. **Reduce storage cost** (Cosmos DB is expensive)
2. **Ensure old records are still retrievable**
3. **No API changes**
4. **No downtime or data loss**
5. **Simple to deploy and maintain**
6. (Bonus) Architecture diagram + implementation pseudocode

---

## 💡 **Proposed Solution Overview**

### ✅ Step 1: **Tiered Storage Strategy**

Move old records (older than 3 months) from Cosmos DB to **Azure Blob Storage**.

* **Hot data (≤ 3 months)** → stay in Cosmos DB (fast access)
* **Cold data (> 3 months)** → move to Blob Storage (cheaper)
* Implement a **data access layer** that:

  * First queries Cosmos DB
  * If not found, queries Blob Storage (cold path)

### ✅ Step 2: **Use Azure Functions for Automation**

* Use a **Timer Triggered Azure Function** that runs daily or weekly
* This function moves cold data from Cosmos DB to Blob Storage
* Once archived, delete old record from Cosmos DB (after verifying it's saved)

### ✅ Step 3: **Data Access Adapter (No API change)**

* Wrap existing Cosmos DB read operation with a **fallback to Blob Storage**
* This maintains the **existing API contract**

---

## 🧱 **Architecture Diagram**

Here's a simplified architecture you can present:

```
                       +---------------------------+
                       |     API / App Layer       |
                       +------------+--------------+
                                    |
                                    v
                       +---------------------------+
                       | Cosmos DB (≤ 3 months)     |
                       +------------+--------------+
                                    |
                 If Not Found      |
                                    v
                       +---------------------------+
                       |  Azure Blob Storage       |
                       | (cold records > 3 months) |
                       +---------------------------+
                                    ^
                                    |
                +----------------------------------------+
                | Azure Function (scheduled archival job)|
                +----------------------------------------+
```

---

## 🧪 **Step-by-Step Implementation**

### 🔹 1. Azure Function for Archival

**Trigger**: Timer (e.g., run daily)

**Logic**:

* Query Cosmos DB for records older than 3 months
* Store each in Blob Storage (use JSON format, organized by date)
* Delete record from Cosmos DB once confirmed archived

#### 🔸 Pseudocode:

```python
from azure.cosmos import CosmosClient
from azure.storage.blob import BlobServiceClient
import datetime
import json

# Cosmos DB setup
cosmos_client = CosmosClient(COSMOS_URI, COSMOS_KEY)
container = cosmos_client.get_database_client('BillingDB').get_container_client('Records')

# Blob setup
blob_service_client = BlobServiceClient.from_connection_string(BLOB_CONN_STRING)
container_client = blob_service_client.get_container_client("archived-records")

cutoff_date = datetime.datetime.utcnow() - datetime.timedelta(days=90)

def archive_old_data():
    old_records = container.query_items(
        query="SELECT * FROM c WHERE c.timestamp < @cutoff",
        parameters=[{"name": "@cutoff", "value": cutoff_date.isoformat()}],
        enable_cross_partition_query=True
    )

    for record in old_records:
        blob_name = f"{record['id']}.json"
        blob_content = json.dumps(record)
        container_client.upload_blob(blob_name, blob_content, overwrite=True)
        container.delete_item(record, partition_key=record['partitionKey'])

```

---

### 🔹 2. Modify Data Access Logic

If your API fetches by record ID:

```python
def get_billing_record(record_id):
    try:
        # Try Cosmos DB first
        return container.read_item(record_id, partition_key=record_id)
    except:
        # Fallback to Blob Storage
        blob_client = container_client.get_blob_client(f"{record_id}.json")
        if blob_client.exists():
            blob_data = blob_client.download_blob().readall()
            return json.loads(blob_data)
        else:
            raise Exception("Record not found")
```

> **Note**: This logic can live behind your existing data access interface to avoid changing API contracts.

---

### 🔹 3. Blob Storage Cost Optimization

* Use **Cool** or **Archive tier** for blobs older than 3 months.
* Lifecycle rule in Blob Storage:

  ```json
  {
    "rules": [
      {
        "name": "tier-down",
        "enabled": true,
        "definition": {
          "filters": {
            "blobTypes": ["blockBlob"]
          },
          "actions": {
            "baseBlob": {
              "tierToCool": {
                "daysAfterModificationGreaterThan": 30
              },
              "tierToArchive": {
                "daysAfterModificationGreaterThan": 90
              }
            }
          }
        }
      }
    ]
  }
  ```

---

## 📌 Notes

* **No downtime**: Archive job runs in background
* **No data loss**: Archive before delete
* **No API changes**: Access layer handles fallback
* **Simplicity**: Server, less components (Functions + Blob)


The ChatGPT prompts that were used in this solution - 

1. What are the best cosmosDB strict cost optimization ideas.
2. Great and best idea to implement the suggested solution.
3. Python scripts for the implementation.
4. I don't want any downtime so provide the solution according to the requirements.
