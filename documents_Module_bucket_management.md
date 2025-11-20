# Bucket management
## Core Architecture Concepts
1. Bucket Pattern
   - Group related resources into logical “buckets” so that:
   - Each bucket contains resources that share similar characteristics.
   - Operations such as lookup, permission aggregation, and snapshot generation become faster and more predictable.

2. Outlier Pattern
   - Store resources that do not fit any bucket rule:
   - Prevents buckets from becoming inconsistent.
   - Keeps rare or exceptional resources isolated so bucket search remains efficient.

3. Consistent Hashing
   - Determine the bucket that should contain a given resource:
   - Ensures stable bucket assignment even when buckets evolve or scale.
   - Virtual nodes are used to balance distribution and provide fast in-memory lookup.

4. Cuckoo Filter
  - Enable extremely fast membership checks:
  - Quickly verify whether a resource exists inside a bucket.
  - Reduces the need to scan bucket content.
  - Useful for checking assigned, granted, or revoked resources with minimal memory footprint.

## Summarize
**Bucket management rule set**
1. Hash → bucket (always one)
2. Add to bucket.resources
3. If full → bucket.outliers
4. If outliers too many → split bucket
5. New bucket → update ring + index
6. Lookup always O(1): check resources or outliers
7. Delete always from hashed bucket only
8. Never place resource in "next bucket"
9. Use locks to prevent double-split
10. Always rehash after split to maintain correctness


## Full Architecture Overview 

```mermaid
flowchart LR
    subgraph User Accessibility
        U1[(u1_documents)] --> B1[(u1_documents_1)]
        U1 --> B2[(u1_documents_2)]
        U1 --> B3[(u1_documents_3)]
    end

    subgraph Hashing Layer
        H[Consistent Hashing<br/>Virtual Nodes]
    end

    H --> B1
    H --> B2
    H --> B3

    subgraph Outliers
        O1[(bucket_1.outliers)]
        O2[(bucket_2.outliers)]
    end

    B1 --> O1
    B2 --> O2

    subgraph Split Engine
        S[Split Manager]
        S --> RING[Rebuild Hash Ring]
        S --> NEW[Create New Bucket]
        S --> MIG[Rehash + Migrate Items]
    end

    O1 --> S
    O2 --> S
```
## Routing resource & Hash Ring
```mermaid
flowchart LR
    R[Resource ID] --> H[Consistent Hashing<br/>+ Virtual Nodes]
    H --> B{Target Bucket?}
    B -->|bucket_X| BX[Load Bucket Document]
    BX --> CH{Found?}
    CH -->|resources| OK1[Return<br>bucket.resources ID]
    CH -->|outliers| OK2[Return<br>bucket.outliers ID]
    CH -->|not found| NF[Resource not exist]
```

## Add resource
```mermaid
flowchart LR
    RI[Add Resource] --> HR[Hash Resource]
    HR --> B[Get Bucket]
    B --> CAP{Bucket Full?}
    CAP -->|No| INS[Insert Into resources]
    CAP -->|Yes| OUT[Insert Into outliers]
    OUT --> INC[Increment outlier_count]
    INS --> UPD[Update Bucket Count]
    
    UPD --> DONE
    INC --> CHK{outlier_count >= limit?}
    CHK -->|Yes| SPLIT[Trigger Monitor Split]
    CHK -->|No| DONE[Finish]
```

## Delete resource
```mermaid
flowchart LR
    D[Delete Request] --> HR[Hash Resource]
    HR --> B[Load Target Bucket]
    B --> F{Found in resources?}
    F -->|Yes| DR[Delete from resources]
    F -->|No| FO{Found in outliers?}
    FO -->|Yes| DO[Delete from outliers]
    FO -->|No| DN[Nothing to delete]
    DR --> UP[Update counts]
    DO --> UP

```

## Monitor Trigger split bucket
```mermaid
flowchart TD
    M[Monitor Bucket] --> C1{count >= max_per_bucket?}
    M --> C2{outlier_count >= max_outliers?}
    M --> C3{doc_size >= max_doc_size?}
    
    C1 -->|Yes| SPLIT[Trigger Bucket Split]
    C2 -->|Yes| SPLIT
    C3 -->|Yes| SPLIT

    C1 -->|No| OK1[Healthy]
    C2 -->|No| OK1
    C3 -->|No| OK1

```

## Trigger split bucket
```mermaid
flowchart TD
    TRIGGER[Split Triggered] --> LOCK[Acquire Distributed Lock]
    LOCK --> NB[Create New Bucket]
    NB --> RH[Rebuild Hash Ring<br/>Insert Virtual Nodes]
    RH --> REMAP[Rehash All Items<br/>in Old Bucket]
    
    REMAP --> M1{New Bucket Target?}
    M1 -->|Old Bucket| KEEP[Keep in Old]
    M1 -->|New Bucket| MOVE[Move to New]

    MOVE --> CLEAN[Reset Outliers + Update Counts]
    KEEP --> CLEAN

    CLEAN --> UPDATE[Update Main Bucket Index]
    UPDATE --> UNLOCK[Release Lock]

```

## Evaluate permission
```mermaid
flowchart TD
    REQ[Check Permission<br/>user, resource] --> HR[Hash Resource]
    HR --> B[Load Bucket]
    B --> R{Found permissions?}
    R -->|Yes| BIT[Evaluate Bitmask]
    BIT --> RESULT[ALLOW / DENY]
    R -->|No| RESULT_NO[DENY]

```