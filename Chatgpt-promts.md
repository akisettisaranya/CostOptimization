
To derive this solution, the following key questions and prompts were explored:

   1. How can we reduce Azure Cosmos DB costs for a system storing 2 million+ billing records of ~300 KB each, while ensuring the data is available?

   2. Only the latest 3 months of data are frequently accessed. Older records are rarely used but must be available when needed. What's the best storage strategy?

   3. Can we have a simple, transparent architecture that allows serving both hot and cold data through the same API, without breaking existing clients?

   4. We prefer minimal operational complexity: no changes to API contracts, no downtime, and use of native Azure services only.

   5. Please provide the architecture diagram as ASCII code that can be embedded in a README.

   6. Summarize everything clearly, provide estimated cost savings, and include a simple code snippet showing cold fallback logic.
