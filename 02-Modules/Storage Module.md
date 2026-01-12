---
title: Storage Module
tags:
  - module
  - storage
  - persistence
created: 2025-01-12
type: documentation
links:
  - [[Architecture Overview]]
  - [[Index Module]]
  - [[Query Module]]
---

# Storage Module

**Storage Module**(`/graphrag/storage/`)은 로컬 파일 시스템, Azure Blob Storage, Azure Cosmos DB 등 다양한 스토리지 백엔드 간의 데이터 지속성을 위한 유연한 추상화 계층을 제공합니다.

## Overview

스토리지 모듈은 공통 인터페이스를 사용하는 팩토리 패턴을 따르며, 스토리지 구현 간의 원활한 전환을 가능하게 합니다.

```python
from graphrag.storage import StorageFactory

storage = StorageFactory.create_storage(
    storage_type="file",
    kwargs={"base_dir": "./output"}
)
```

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Storage Factory                         │
│  ┌────────────┐  ┌────────────┐  ┌────────────────────┐    │
│  │   File     │  │    Blob    │  │     CosmosDB       │    │
│  │  Storage   │  │  Storage   │  │      Storage       │    │
│  └─────┬──────┘  └─────┬──────┘  └─────────┬──────────┘    │
└────────┼───────────────┼──────────────────┼─────────────────┘
         │               │                  │
         └───────────────┼──────────────────┘
                         │
                ┌────────▼────────┐
                │ PipelineStorage │
                │   (Interface)   │
                └─────────────────┘
```

## Storage Backends

### 1. File Storage (`file`)

**Purpose**: Local filesystem storage

**Configuration**:
```yaml
output:
  type: file
  base_dir: "./output"
```

**Features**:
- Async file operations with aiofiles
- Cross-platform path handling
- Pattern-based file finding
- Encoding/decoding control

**Usage**:
```python
# Write
await storage.set("data.json", '{"key": "value"}')

# Read
data = await storage.get("data.json")

# Find
import re
for path, metadata in storage.find(re.compile(r"\.json$")):
    print(path)
```

### 2. Azure Blob Storage (`blob`)

**Purpose**: Azure cloud object storage

**Configuration**:
```yaml
output:
  type: blob
  connection_string: ${BLOB_CONNECTION_STRING}
  container_name: "graphrag"
  base_dir: "output"
```

**Features**:
- Auto-container creation
- ABFS URL generation
- DataFrame support (JSON/Parquet)
- Path prefix support

**Authentication**:
- Connection string
- Managed Identity (DefaultAzureCredential)

### 3. Azure Cosmos DB (`cosmosdb`)

**Purpose**: NoSQL document database with vector search

**Configuration**:
```yaml
output:
  type: cosmosdb
  cosmosdb_account_url: https://account.documents.azure.com
  connection_string: ${COSMOS_CONNECTION_STRING}
  database_name: "graphrag"
  container_name: "data"
```

**Features**:
- Automatic database/container creation
- Document-based storage
- Partitioned by document ID
- Bulk operations

## Storage Interface

### PipelineStorage

모든 스토리지 구현은 이 인터페이스를 구현합니다:

```python
class PipelineStorage(ABC):
    async def find(
        self,
        pattern: re.Pattern,
        file_filter: dict[str, Any] | None = None,
        limit: int | None = None,
    ) -> Iterable[tuple[str, dict[str, Any]]]:
        """Find files matching pattern"""

    async def get(
        self,
        key: str,
        as_bytes: bool = False
    ) -> str | bytes:
        """Get file content"""

    async def set(
        self,
        key: str,
        value: str | bytes,
        encoding: str | None = None
    ):
        """Write file content"""

    async def has(self, key: str) -> bool:
        """Check if file exists"""

    async def delete(self, key: str):
        """Delete file"""

    async def clear(self):
        """Clear all files"""

    def child(self, name: str) -> PipelineStorage:
        """Create nested storage scope"""
```

## Factory Usage

### Creating Storage

```python
from graphrag.storage import StorageFactory

# File storage
file_storage = StorageFactory.create_storage(
    storage_type="file",
    kwargs={"base_dir": "./output", "encoding": "utf-8"}
)

# Blob storage
blob_storage = StorageFactory.create_storage(
    storage_type="blob",
    kwargs={
        "connection_string": "...",
        "container_name": "data",
        "base_dir": "output"
    }
)

# Cosmos DB storage
cosmos_storage = StorageFactory.create_storage(
    storage_type="cosmosdb",
    kwargs={
        "cosmosdb_account_url": "https://...",
        "connection_string": "...",
        "database_name": "graphrag",
        "container_name": "data"
    }
)
```

### Custom Storage

```python
from graphrag.storage import StorageFactory, PipelineStorage

class S3Storage(PipelineStorage):
    # Implement interface methods
    async def get(self, key, as_bytes=False):
        # S3 get logic
        pass
    # ... implement other methods

# Register custom storage
StorageFactory.register("s3", S3Storage)

# Use it
storage = StorageFactory.create_storage(
    storage_type="s3",
    kwargs={"bucket": "my-bucket"}
)
```

## Data Persistence Patterns

### File Storage
- Written directly to filesystem
- UTF-8 encoding by default
- Atomic writes with async I/O

### Blob Storage
- Uploaded to Azure containers
- Handles strings and bytes
- Path-based organization

### Cosmos DB Storage
- Optimized for structured data
- Document partitioning
- Prefix-based grouping

## Related Topics

- [[Storage Module]] - Vector database storage
- [[Index Module]] - Pipeline usage
- [[Configuration Module]] - Storage configuration

---
*See also: [[Architecture Overview]], [[Storage Module]], [[Configuration Module]]*
