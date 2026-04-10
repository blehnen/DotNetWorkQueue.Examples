# INTEGRATIONS.md

## Overview

This repository integrates with five external transport backends: SQL Server, PostgreSQL, SQLite (file-based), LiteDb (file-based), and Redis. Each transport is bound via a dedicated `DotNetWorkQueue.Transport.*` NuGet assembly. Connection strings are read at runtime from `App.config` `appSettings["Connection"]` in each runnable project — the committed values point to the original author's development LAN and local file paths, and must be updated before running.

## Metrics

| Metric | Value |
|--------|-------|
| Transport backends | 5 (SQL Server, PostgreSQL, SQLite, LiteDb, Redis) |
| App.config files with Connection key | 15 (one per runnable project) |
| Network-dependent transports | 3 (SQL Server, PostgreSQL, Redis) |
| File-based transports | 2 (SQLite, LiteDb) |

## Findings

### Connection String Pattern (All Transports)

Every runnable project reads its connection string identically:

```csharp
ConfigurationManager.AppSettings["Connection"]
```

This is passed directly to `QueueConnection(queueName, connectionString)` and thence to the transport layer.

Evidence: `Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/QueueCreation.cs` line 292
Evidence: `Source/Examples/Redis/Producer/RedisProducer/Commands/QueueCreation.cs` line 105

The `App.config` key is always `Connection` under `<appSettings>`. Producer and consumer projects for the same transport use identical connection strings — confirmed by comparing producer vs consumer App.configs for SQL Server, PostgreSQL, SQLite, LiteDb, and Redis.

---

### SQL Server

- **Transport assembly**: `DotNetWorkQueue.Transport.SqlServer` v0.8.0
- **Init type**: `SqlServerMessageQueueInit`
- **Creation type**: `SqlServerMessageQueueCreation`
- **Underlying driver**: `Microsoft.Data.SqlClient` v6.1.3
- **Connection key**: `appSettings["Connection"]`
- **Committed default** (dev machine value, must be changed):
  ```
  Server=192.168.0.58;Application Name=SQLProducer;Database=TestR;Trusted_Connection=True;max pool size=500
  ```
  Evidence: `Source/Examples/SQLServer/Producer/SqlServerProducer/App.config`
  Evidence: `Source/Examples/SQLServer/Consumer/SqlServerConsumer/App.config` (identical value)

- **Projects**: `SqlServerProducer`, `SqlServerConsumer`, `SqlServerConsumerAsync`
  - `Source/Examples/SQLServer/Producer/SqlServerProducer/`
  - `Source/Examples/SQLServer/Consumer/SqlServerConsumer/`
  - `Source/Examples/SQLServer/Consumer/SqlServerConsumerAsync/`

- **Transport-specific setup options** (exposed in `QueueCreation` command):
  - `SetQueueType` — NotRpc / sendRpc / receiveRpc
  - `SetDelayedProcessing` — enable/disable deferred delivery
  - `SetHeartBeat` — enable/disable heartbeat column
  - `SetHoldTransaction` — hold SQL transaction for message lifetime
  - `SetMessageExpiration` — TTL column on queue table
  - `SetPriority` — priority column
  - `SetStatus` — status column (pending/working flag)
  - `SetStatusTable` — separate status-tracking table for external queries
  - `AddColumn` / `AddColumnWithLength` / `AddColumnWithPrecision` — user-defined columns on status table
  - `AddConstraint` / `AddConstraintManyColumns` — user-defined indexes/constraints on status table
  - Evidence: `Source/Examples/SQLServer/Producer/SqlServerProducer/Commands/QueueCreation.cs` lines 62-76

---

### PostgreSQL

- **Transport assembly**: `DotNetWorkQueue.Transport.PostgreSQL` v0.8.0
- **Shared relational assembly**: `DotNetWorkQueue.Transport.RelationalDatabase` v0.8.0
- **Init type**: `PostgreSQLMessageQueueInit` [Inferred from naming pattern; confirmed by namespace in packages.config]
- **Underlying driver**: `Npgsql` v8.0.8
- **Connection key**: `appSettings["Connection"]`
- **Committed default** (dev machine value, must be changed):
  ```
  Server=192.168.0.58;Port=5432;Database=IntegrationTesting;Integrated Security=true;
  ```
  Evidence: `Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/App.config`
  Evidence: `Source/Examples/PostGresSQL/Consumer/PostGresSQLConsumer/App.config` (identical value)

- **Projects**: `PostGreSQLProducer`, `PostGresSQLConsumer`, `PostGreSQLConsumerAsync`
  - `Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/`
  - `Source/Examples/PostGresSQL/Consumer/PostGresSQLConsumer/`
  - `Source/Examples/PostGresSQL/Consumer/PostGreSQLConsumerAsync/`

- **Transport-specific setup options**: Same relational schema options as SQL Server (QueueType, DelayedProcessing, HeartBeat, HoldTransaction, MessageExpiration, Priority, Status, StatusTable, AddColumn variants, AddConstraint variants)
  - Evidence: `Source/Examples/PostGresSQL/Producer/PostGresSQLProducer/Commands/QueueCreation.cs`

---

### SQLite

- **Transport assembly**: `DotNetWorkQueue.Transport.SQLite` v0.8.0
- **Shared relational assembly**: `DotNetWorkQueue.Transport.RelationalDatabase` v0.8.0
- **Underlying driver**: `System.Data.SQLite.Core` v1.0.119.0 + `Stub.System.Data.SQLite.Core.NetFramework` v1.0.119.0
- **Connection key**: `appSettings["Connection"]`
- **Committed default** (local file path, must be changed):
  ```
  Data Source=c:\temp\test.db;Version=3;
  ```
  Evidence: `Source/Examples/SQLite/Producer/SQLiteProducer/App.config`
  Evidence: `Source/Examples/SQLite/Consumer/SQLiteConsumer/App.config` (identical value)

- **Projects**: `SQLiteProducer`, `SQLiteConsumer`, `SQLiteConsumerAsync`
  - `Source/Examples/SQLite/Producer/SQLiteProducer/`
  - `Source/Examples/SQLite/Consumer/SQLiteConsumer/`
  - `Source/Examples/SQLite/Consumer/SQLiteConsumerAsync/`

- **Transport-specific setup options**: Same relational schema options as SQL Server (QueueType, DelayedProcessing, HeartBeat, HoldTransaction, MessageExpiration, Priority, Status, StatusTable, column/constraint options)
  - Evidence: `Source/Examples/SQLite/Producer/SQLiteProducer/Commands/QueueCreation.cs`

- **Note**: File-based; no network server required. The `c:\temp\test.db` path is Windows-specific.

---

### LiteDb

- **Transport assembly**: `DotNetWorkQueue.Transport.LiteDb` v0.8.0
- **Note**: Does NOT pull `DotNetWorkQueue.Transport.RelationalDatabase` — LiteDb is not a relational transport.
- **Underlying driver**: `LiteDB` v5.0.21
- **Connection key**: `appSettings["Connection"]`
- **Committed default** (local file path, must be changed):
  ```
  Filename=c:\temp\test.ldb;Connection=shared;
  ```
  Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/App.config`
  Evidence: `Source/Examples/LiteDb/Consumer/LiteDbConsumer/App.config` (identical value)

- **Projects**: `LiteDbProducer`, `LiteDbConsumer`, `LiteDbConsumerAsync`
  - `Source/Examples/LiteDb/Producer/LiteDbProducer/`
  - `Source/Examples/LiteDb/Consumer/LiteDbConsumer/`
  - `Source/Examples/LiteDb/Consumer/LiteDbConsumerAsync/`

- **Transport-specific setup options**: `QueueCreation` command exposes `CreateQueue` and `RemoveQueue` only — no schema/column customization (LiteDb is document-oriented).
  - Evidence: `Source/Examples/LiteDb/Producer/LiteDbProducer/Commands/QueueCreation.cs`

- **Note**: File-based; no network server required. `Connection=shared` is a LiteDB connection mode for multi-process access.

---

### Redis

- **Transport assembly**: `DotNetWorkQueue.Transport.Redis` v0.8.0
- **Note**: Does NOT pull `DotNetWorkQueue.Transport.RelationalDatabase` or `DotNetWorkQueue.Transport.Shared` — Redis has its own transport model.
  - [Correction: packages.config confirms `DotNetWorkQueue.Transport.Shared` IS present for Redis, but NOT `RelationalDatabase`]
- **Init type**: `RedisQueueInit`
- **Creation type**: `RedisQueueCreation`
- **Underlying driver**: `StackExchange.Redis` v2.10.1
- **Connection key**: `appSettings["Connection"]`
- **Committed default** (dev machine IP, must be changed):
  ```
  192.168.0.2
  ```
  Evidence: `Source/Examples/Redis/Producer/RedisProducer/App.config`
  Evidence: `Source/Examples/Redis/Consumer/RedisConsumer/App.config` (identical value)
  Note: The Redis connection string is just a bare host IP — StackExchange.Redis connection string format; no port, database, or password specified.

- **Projects**: `RedisProducer`, `RedisConsumer`, `RedisConsumerAsync`
  - `Source/Examples/Redis/Producer/RedisProducer/`
  - `Source/Examples/Redis/Consumer/RedisConsumer/`
  - `Source/Examples/Redis/Consumer/RedisConsumerAsync/`

- **Transport-specific setup options**: `QueueCreation` exposes only `RemoveQueue` — no schema creation needed for Redis (queue structure is created implicitly on first use).
  - Evidence: `Source/Examples/Redis/Producer/RedisProducer/Commands/QueueCreation.cs` lines 52-53

---

### No Other External Integrations

- No HTTP/REST API calls detected.
- No message broker integrations other than the DotNetWorkQueue transports above.
- No cloud service SDKs in use by the example projects themselves. (`Azure.Identity` / `Azure.Core` appear only as transitive dependencies of `Microsoft.Data.SqlClient` in the SQL Server packages.config — not directly used.)
  - Evidence: `Source/Examples/SQLServer/Producer/SqlServerProducer/packages.config`

## Summary Table

| Transport | DotNetWorkQueue Package | Driver | Connection Default | Schema Commands |
|-----------|------------------------|--------|-------------------|-----------------|
| SQL Server | `Transport.SqlServer` v0.8.0 | `Microsoft.Data.SqlClient` v6.1.3 | `Server=192.168.0.58;...` | Full (columns, constraints, options) |
| PostgreSQL | `Transport.PostgreSQL` v0.8.0 | `Npgsql` v8.0.8 | `Server=192.168.0.58;Port=5432;...` | Full (same as SQL Server) |
| SQLite | `Transport.SQLite` v0.8.0 | `System.Data.SQLite.Core` v1.0.119.0 | `Data Source=c:\temp\test.db;Version=3;` | Full (same as SQL Server) |
| LiteDb | `Transport.LiteDb` v0.8.0 | `LiteDB` v5.0.21 | `Filename=c:\temp\test.ldb;Connection=shared;` | CreateQueue / RemoveQueue only |
| Redis | `Transport.Redis` v0.8.0 | `StackExchange.Redis` v2.10.1 | `192.168.0.2` | RemoveQueue only |

## Open Questions

- The Redis connection string (`192.168.0.2`) specifies no port, password, or database index — is Redis expected to run on the default port 6379 with no auth? Does DotNetWorkQueue.Transport.Redis accept a full StackExchange.Redis connection string or only a hostname?
- The SQLite and LiteDb paths (`c:\temp\`) are Windows-absolute. Are there any plans to support cross-platform paths?
- `Azure.Identity` appears as a transitive dep of `Microsoft.Data.SqlClient` — does this mean Azure AD / managed identity auth is theoretically available for SQL Server, even though no examples demonstrate it?
