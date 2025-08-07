# Azure Cosmos DB Cost Optimization Using Data Tiering

## Problem Statement
An organization uses Azure's serverless architecture and stores billing records in Azure Cosmos DB. The data is heavily read, but older records (over 3 months) are rarely accessed. As the number of records grows (over 2 million, each ~300KB), Cosmos DB costs are rising significantly. The goal is to optimize cost without compromising data availability or breaking existing API contracts.

## Objectives
- Reduce Cosmos DB storage and RU/s cost.
- Ensure high availability of all data (hot + cold).
- Maintain existing API contracts (no changes for API consumers).
- Implement with minimal complexity and no downtime.

## Solution Overview
To optimize costs while maintaining performance and availability, we implement a **data tiering strategy** by categorizing billing records into **hot** and **cold** data:

### 🔹 Hot Data
- **Definition**: Billing records generated within the last 90 days.
- **Storage**: Stored in **Azure Cosmos DB**, which provides low-latency access and high throughput.
- **Usage**: Frequently accessed for customer support, reporting, or audit.

### 🔹 Cold Data
- **Definition**: Records older than 90 days that are accessed infrequently.
- **Storage**: Moved to **Azure Blob Storage** (Cool Tier), which is highly cost-effective for large datasets that are rarely read.
- **Usage**: Archived for compliance and long-term retention; still available on-demand with slightly higher read latency (~2–5 seconds).

### 🛠 How the Architecture Works
- **Step 1: Ingestion**  
  All incoming billing records are written to **Cosmos DB** via the existing API infrastructure.

- **Step 2: Tiering Scheduler**  
  A **scheduled Azure Function or Azure Data Factory pipeline** runs daily to identify and move records older than 90 days to Azure Blob Storage. This includes:
  - Copying the data to Blob
  - Validating successful migration
  - Optionally deleting old records from Cosmos DB (after backup)

- **Step 3: Unified Data Access Layer**  
  The backend service (Billing API) implements a transparent fallback mechanism:
  - First attempts to read from Cosmos DB
  - If not found, retrieves the record from Blob Storage
  - This logic is abstracted from API consumers, maintaining existing API contracts

### 🔐 Key Properties of the Solution
| Feature               | Implementation Detail                                   |
|------------------------|--------------------------------------------------------|
| **No API Changes**     | Read/Write interfaces remain unchanged                 |
| **No Downtime**        | Migration happens asynchronously in the background     |
| **No Data Loss**       | Data is archived with verification before deletion     |
| **Easy Maintenance**   | Built using Azure-native services (Functions, ADF)     |
| **Scalable**           | Supports millions of records with minimal effort       |

## Architecture
1. **API writes and reads** continue to use Cosmos DB.
2. **Scheduled job** (Azure Function or Data Factory) moves records older than 90 days to Blob Storage.
3. The **API backend service** implements a fallback mechanism:
   - Try reading from Cosmos DB.
   - If not found, try reading from Blob Storage.

This ensures transparent cold data access without changing API contracts.

### Architecture Diagram
```
             ┌────────────────────────────┐
             │   Frontend / External API  │
             └────────────┬───────────────┘
                          │
                          ▼
           ┌────────────────────────────┐
           │     Existing Billing API    │  ◄── No change to endpoints
           └───────┬─────────────────────┘
                   ▼
      ┌──────────────────────────────┐
      │ Unified Billing Service Layer│
      └──────┬──────────┬────────────┘
             ▼          ▼
     ┌────────────┐  ┌────────────────┐
     │ Cosmos DB  │  │ Azure Blob     │
     │ (Hot Tier) │  │ Storage (Cold) │
     └────────────┘  └────────────────┘
             ▲
             │
      ┌──────────────┐
      │ Azure Timer  │ ────► Archive data > 90d daily
      │ Function / ADF│
      └──────────────┘
```
- Cosmos DB stores hot (active) records.
- Azure Blob Storage holds cold (archived) records.
- Azure Function/Data Factory transfers data older than 90 days.
- API transparently serves from hot or cold based on availability.

## Benefits
- ✅ **No API change**: The API surface remains identical.
- ✅ **No data loss**: Data is moved only after successful backup.
- ✅ **No downtime**: Cosmos DB remains operational throughout.
- ✅ **Cost optimization**: Cold data stored in Blob reduces Cosmos DB storage and RU costs.
- ✅ **Cold data availability**: Cold data can be retrieved within 2–5 seconds.

## Technologies Used
- Azure Cosmos DB
- Azure Blob Storage (Cool Tier)
- Azure Functions (for data movement)
- ASP.NET Core Web API

## Example Cold Record Access Logic
```csharp
public async Task<BillingRecord> GetBillingRecordAsync(string id)
{
    var record = await _cosmosDbRepo.GetRecordAsync(id);
    if (record != null)
        return record;

    return await _blobRepo.GetRecordAsync(id);
}
```

## Cost Reduction Analysis

By implementing data tiering, significant cost savings can be achieved in two main areas:

### 1. Storage Cost Reduction

| Tier         | Storage Type               | Cost per GB/month (approx) | Notes                            |
|--------------|----------------------------|-----------------------------|----------------------------------|
| **Hot Tier** | Azure Cosmos DB            | ~$0.25–$0.30 per GB         | Highly available, fast, expensive |
| **Cold Tier**| Azure Blob Storage (Cool)  | ~$0.01–$0.02 per GB         | Optimized for infrequent access  |

**Example Estimate:**
- **Record size**: 300 KB  
- **Cold records (older than 90 days)**: 1.5 million  
- **Total cold data size** ≈ 1.5M × 300 KB = **~430 GB**

**Cost before tiering:**  
All data in Cosmos DB → 2M × 300 KB = ~572 GB  
Monthly cost ≈ 572 GB × $0.25 = **$143**

**Cost after tiering:**  
- Cosmos DB (hot) = 0.5M × 300 KB ≈ 143 GB → **$36**  
- Blob (cold) = 1.5M × 300 KB ≈ 429 GB → **$6.50**

✅ **Monthly savings: ~$100 (70% reduction in storage cost)**

### 2. RU/s (Request Unit) Cost Optimization

Cold records are **rarely accessed**, yet they consume:
- Indexing costs  
- RU charges on reads (even infrequent)

By moving cold data out:
- Cosmos DB indexes less data → **lower RU consumption**  
- API reads rarely trigger RU usage on old records

✅ **Potential RU reduction: 30–50%**, depending on query patterns

### ✅ Summary

| Metric          | Before Tiering | After Tiering | Savings        |
|------------------|----------------|----------------|----------------|
| **Storage Cost** | ~$143/month    | ~$42/month     | ~$101/month    |
| **RU Consumption** | Full           | Hot data only  | Up to 50%      |

📉 **Overall cost reduction: ~70%** without any service/API downtime or architectural complexity.

## Conclusion
This data tiering strategy provides a seamless, cost-effective, and scalable solution for managing large volumes of semi-archival data in Azure without impacting user experience or existing integrations.
