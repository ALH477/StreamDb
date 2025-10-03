# StreamDb Technical Specification
**Version: 1.1**
**Date: 2025-10-03 05:38:48 UTC**
**Author: Original by ALH477, Specification by GitHub Copilot**

## 1. System Architecture

### 1.1 Core Components

#### 1.1.1 Page Layer
```plaintext
PageStructure {
    HEADER {
        uint32 CRC                 // Data integrity check
        int32  Version            // Monotonic version counter
        int32  PrevPageId         // Previous page in chain
        int32  NextPageId         // Next page in chain
        byte   Flags              // Page status flags
        int32  DataLength         // Length of data in page
    }
    DATA {
        byte[4061] Data          // Actual page data
    }
    Constants {
        PAGE_RAW_SIZE = 4096     // Total page size (4KB)
        PAGE_HEADER_SIZE = 35    // Header overhead
        PAGE_DATA_CAPACITY = 4061 // Available data space
    }
}
```

#### 1.1.2 Document Layer
```plaintext
Document {
    Properties {
        Guid   ID                // Unique document identifier
        int    FirstPageId       // Entry point to page chain
        int    CurrentVersion    // Document version number
        List<string> Paths      // Access paths to document
    }
    Limits {
        MaxSize = 256MB         // Maximum single document size
        MaxPaths = Unlimited    // Limited by available storage
    }
}
```

#### 1.1.3 Storage Layer
```plaintext
DatabaseHeader {
    byte[8] MAGIC = [0x55, 0xAA, 0xFE, 0xED, 0xFA, 0xCE, 0xDA, 0x7A]
    VersionedLink IndexRoot      // Document index root
    VersionedLink PathLookupRoot // Path lookup table root
    VersionedLink FreeListRoot   // Free page management
}

StorageLimits {
    MaxPages = 2147483647      // Maximum number of pages
    MaxSize = 8000GB           // Total database size limit
    PageSize = 4KB            // Individual page size
}
```

### 1.2 Data Structures

#### 1.2.1 ReverseTrie (Path Index)
```plaintext
ReverseTrieNode {
    char Value
    int ParentIndex
    int SelfIndex
    Optional<Guid> DocumentId
    Map<char, int> Children
}

Operations {
    Insert(string path, Guid docId)
    Search(string prefix) -> List<string>
    Delete(string path)
    Update(string path, Guid newDocId)
}
```

#### 1.2.2 Free Page Management
```plaintext
FreeListPage {
    Header {
        int32 NextFreeListPage
        int32 UsedEntries
    }
    Data {
        int32[1020] FreePageIds  // Available page IDs
    }
}

FreePagePolicy {
    Strategy: First-Fit
    Reuse: LIFO for recently freed pages
    Consolidation: Automatic when pages empty
    RetentionPolicy: Keep 2 versions, free on 3rd
}
```

## 2. Core Interfaces

### 2.1 Database Interface
```plaintext
interface IDatabase {
    // Document Operations
    Guid WriteDocument(string path, Stream data)
    bool Get(string path, out Stream data)
    bool GetIdByPath(string path, out Guid id)
    void Delete(string path)
    void Delete(Guid documentId)
    
    // Path Operations
    Guid BindToPath(Guid documentId, string path)
    void UnbindPath(Guid documentId, string path)
    IEnumerable<string> Search(string pathPrefix)
    IEnumerable<string> ListPaths(Guid documentId)
    
    // Maintenance
    void Flush()
    void CalculateStatistics(out int totalPages, out int freePages)
    static void SetQuickAndDirtyMode()
}
```

### 2.2 Storage Backend Interface
```plaintext
interface IDatabaseBackend {
    // Document Operations
    Guid WriteDocument(Stream data)
    Stream ReadDocument(Guid id)
    void DeleteDocument(Guid id)
    
    // Path Management
    Guid BindPathToDocument(string path, Guid id)
    Guid GetDocumentIdByPath(string path)
    IEnumerable<string> SearchPaths(string prefix)
    IEnumerable<string> ListPathsForDocument(Guid id)
    
    // Maintenance
    int CountFreePages()
    string GetInfo(Guid id)
    void DeletePathsForDocument(Guid id)
    void RemoveFromIndex(Guid id)
}
```

## 3. Performance Optimizations

### 3.1 Quick Mode
```plaintext
QuickAndDirtyMode {
    Effect: Skips CRC verification on reads
    Performance: ~10x faster read operations
    Risk: Potential undetected data corruption
    Usage: Recommended for read-heavy workloads
           where data integrity is verified externally
}
```

### 3.2 Caching System
```plaintext
Caches {
    PathLookup {
        Type: ReverseTrie<Guid>
        Scope: Document paths to IDs
        InvalidationPolicy: On write/delete
    }
    PageCache {
        Type: LRU<int, BasicPage>
        Size: Configurable
        InvalidationPolicy: On modification
    }
}
```

## 4. Thread Safety

### 4.1 Lock Hierarchy
```plaintext
LockLevels {
    1. PathWriteLock    // Highest priority
    2. FreeListLock     // Page allocation
    3. FileStreamLock   // Lowest priority
}

ThreadingGuarantees {
    - 100% thread-safe within single process
    - Multiple reader support
    - Single writer per path
    - No reader starvation
}
```

## 5. Error Handling

### 5.1 Data Integrity
```plaintext
IntegrityChecks {
    - CRC32 on all pages (unless in QuickMode)
    - Version number verification
    - Page chain validation
    - Database header magic number verification
    - Stream bounds checking
}

RecoveryMechanisms {
    - Automatic page chain repair
    - Orphaned page collection
    - Path index reconstruction
    - Version conflict resolution
}
```

## 6. Implementation Requirements

### 6.1 Platform Requirements
```plaintext
Minimum Requirements {
    - Stream I/O support
    - 64-bit integer support
    - Thread synchronization primitives
    - UUID/GUID generation
    - Basic collection types
}

Recommended Features {
    - Memory-mapped file support
    - Atomic operations
    - Direct I/O capabilities
    - Native CRC32 computation
}
```

### 6.2 Performance Targets
```plaintext
Benchmarks {
    ReadPerformance {
        Standard: 10MB/s minimum
        QuickMode: 100MB/s minimum
    }
    WritePerformance: 5MB/s minimum
    PathLookup: < 1ms average
    PageAllocation: < 0.1ms average
}
```

## 7. Testing Requirements

### 7.1 Test Categories
```plaintext
UnitTests {
    - Page operations
    - Document CRUD
    - Path management
    - CRC verification
    - Cache behavior
}

IntegrationTests {
    - Multi-threading
    - Large documents
    - Database recovery
    - Performance validation
}

StressTests {
    - Concurrent access
    - Resource exhaustion
    - Power failure simulation
    - Corruption recovery
}
```

This specification is designed to be language-agnostic while maintaining the core characteristics of StreamDb: reliability, performance, and simplicity. Implementation details may vary by language, but the core architecture and guarantees should remain consistent.
