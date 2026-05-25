# Data Tables

Data tables provide structured data storage that is directly accessible from Amazon Connect contact flows, eliminating the need for Lambda functions for simple data lookups and writes. They are ideal for storing configuration data, business rules, customer preferences, and other reference data that flows need at runtime.

## Overview

Data tables act as key-value stores within your Connect instance. Each table has defined attributes (columns) and rows of data. Contact flows interact with tables via the **Data Table** flow block -- no external compute required.

**When to use data tables vs Lambda:**
- **Data tables** -- simple lookups, configuration data, business rules, small reference datasets
- **Lambda** -- complex logic, external API calls, large datasets, joins across multiple sources

## Flow Block Operations

The **Data Table** flow block supports three operations:

### Evaluate (Read)

Look up a row by its key and return attribute values for use in the flow:

- Specify the table name and the key value to look up
- Returned attributes are stored as contact attributes for downstream blocks
- Supports exact-match lookups on the primary key

**Use case:** Look up a customer's tier level by account number to route to the appropriate queue.

### List

Retrieve multiple rows that match filter criteria:

- Filter by attribute values
- Returns a list of matching rows
- Useful for populating dynamic menus or checking membership in a set

**Use case:** List all active promotions for a product category to play in the IVR.

### Write

Insert or update rows in the data table from within a flow:

- Set attribute values for a given key
- Creates the row if it does not exist, updates if it does (upsert behavior)
- Enables flows to persist data without Lambda

**Use case:** Record a customer's language preference during an IVR interaction for future calls.

## Table Structure

Each data table consists of:

| Component | Description |
|---|---|
| **Table name** | Unique identifier within the instance |
| **Primary key** | The attribute used for lookups (string) |
| **Attributes** | Named columns with data types |
| **Rows** | Individual data records |

### Attribute Types

| Type | Description |
|---|---|
| String | Text values |
| Number | Numeric values |
| Boolean | True/false values |

## Management APIs

### Table Operations

| API | Purpose |
|---|---|
| `CreateDataTable` | Create a new data table with its schema |
| `DescribeDataTable` | Get table metadata and schema |
| `ListDataTables` | List all tables in the instance |
| `UpdateDataTable` | Modify table metadata |
| `DeleteDataTable` | Delete a table and all its data |

### Attribute Operations

| API | Purpose |
|---|---|
| `CreateDataTableAttribute` | Add a new attribute (column) to a table |
| `UpdateDataTableAttribute` | Modify an attribute definition |

### Data Operations

| API | Purpose |
|---|---|
| `EvaluateDataTableValues` | Look up a row by key (same as flow block Evaluate) |
| `BatchCreateDataTableValue` | Insert multiple rows in a single call |
| `BatchUpdateDataTableValue` | Update multiple rows in a single call |
| `BatchDeleteDataTableValue` | Delete multiple rows in a single call |

### Example -- Create and Populate a Table

```javascript
import {
  ConnectClient,
  CreateDataTableCommand,
  BatchCreateDataTableValueCommand,
  EvaluateDataTableValuesCommand,
} from "@aws-sdk/client-connect";

const client = new ConnectClient({ region: "us-east-1" });

// Create table
await client.send(new CreateDataTableCommand({
  InstanceId: instanceId,
  DataTableName: "CustomerTiers",
  PrimaryKeyAttribute: {
    Name: "accountNumber",
    Type: "String",
  },
  Attributes: [
    { Name: "tierLevel", Type: "String" },
    { Name: "discountPercent", Type: "Number" },
    { Name: "priorityRouting", Type: "Boolean" },
  ],
}));

// Populate with data
await client.send(new BatchCreateDataTableValueCommand({
  InstanceId: instanceId,
  DataTableName: "CustomerTiers",
  Values: [
    {
      PrimaryKeyValue: "ACC-001",
      Attributes: {
        tierLevel: "Gold",
        discountPercent: "15",
        priorityRouting: "true",
      },
    },
    {
      PrimaryKeyValue: "ACC-002",
      Attributes: {
        tierLevel: "Silver",
        discountPercent: "10",
        priorityRouting: "false",
      },
    },
  ],
}));

// Look up a customer's tier
const result = await client.send(new EvaluateDataTableValuesCommand({
  InstanceId: instanceId,
  DataTableName: "CustomerTiers",
  PrimaryKeyValue: "ACC-001",
}));
// result.Attributes = { tierLevel: "Gold", discountPercent: "15", priorityRouting: "true" }
```

## Access Control

Data table access is managed through Connect security profiles:

- **Read access** -- allows using the Evaluate and List operations in flows and via API
- **Write access** -- allows using the Write operation in flows and Batch APIs
- **Admin access** -- allows creating, modifying, and deleting tables and attributes

Assign permissions at the security profile level to control which users can manage tables vs. which flows can read/write data.

## Contact Flow Integration

In the contact flow designer:

1. Add a **Data Table** block
2. Select the operation (Evaluate, List, or Write)
3. Configure the table name and key/filter values (can use contact attributes as dynamic values)
4. Map output attributes to contact attributes for use in subsequent blocks
5. Handle success and error branches

**Error handling:** The Data Table block has `Success` and `Error` branches. Common error cases:
- Key not found (Evaluate)
- Table does not exist
- Insufficient permissions
- Throttling (rate limit exceeded)

## Key Considerations

- **Size limits** -- data tables are designed for reference data, not large transactional datasets. Consider DynamoDB or RDS for high-volume data.
- **Latency** -- lookups are low-latency (single-digit milliseconds) since data is stored within the Connect infrastructure
- **No joins** -- tables are independent; you cannot join across tables in a single operation. Use multiple Data Table blocks or Lambda for cross-table logic.
- **Schema changes** -- new attributes can be added at any time; existing rows will have null for the new attribute until updated
- **Concurrency** -- batch operations are atomic per row but not across rows. For multi-row consistency, use Lambda.
- **Region** -- data tables are regional resources tied to the Connect instance
