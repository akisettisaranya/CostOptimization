# 📦 Azure Cosmos DB Cost Optimization via Data Tiering

## 🚀 Overview
This project helps reduce storage and RU/s costs in Azure Cosmos DB by implementing a **hot–cold data tiering strategy** — keeping recent data in Cosmos DB and archiving older records in Azure Blob Storage.

## 💡 Problem
- All billing records (~2M, 300KB each) were stored in Cosmos DB.
- Older data (90+ days) is rarely accessed but still consumes cost (storage + RUs).
- Need to **reduce cost** without breaking existing API contracts.

## ✅ Solution Summary
- Store recent data (<90d) in Cosmos DB (hot).
- Archive older data (>90d) in Azure Blob (cold).
- Use Azure Function / ADF to move data daily.
- Unified service layer reads from Cosmos DB, falls back to Blob if not found.
- No changes to API endpoints or contracts.

## 🧱 Architecture Diagram
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
         │Function / ADF│
         └──────────────┘
