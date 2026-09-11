# Qdrant — RAG/indexing/vector subsystem

## Selection rationale

**FACT:** Qdrant was selected because it exposes a mature, independently deployable vector database with source-visible REST/gRPC schemas, collections, points, payload filters, shards, WAL/segments, snapshots, distributed consensus, RBAC, audit logging, Docker build, and extensive Rust integration tests. **INFERENCE:** It best illuminates the replacement boundary between a platform’s RAG orchestration and a vector/index service.

## Architecture and stack

**FACT:** Qdrant is primarily Rust, organized into crates/modules for API, collection, segment, storage, consensus, WAL, edge/object-storage access, and GPU support under [`lib`](../../../repos/qdrant/lib:1). The REST schema defines vector search/query/point contracts in [`schema.rs`](../../../repos/qdrant/lib/api/src/rest/schema.rs:1210). **FACT:** The Dockerfile uses multi-stage Cargo builds, cross-platform compilation, optional GPU features, and a release binary in [`Dockerfile`](../../../repos/qdrant/Dockerfile:1). **INFERENCE:** This is a database service/runtime, not a Python embedding library or complete RAG ingestion pipeline.

## Repository structure

**FACT:** [`lib/api`](../../../repos/qdrant/lib/api:1) contains REST/gRPC API models; [`lib/collection`](../../../repos/qdrant/lib/collection:1) contains collection/query/shard behavior; [`lib/segment`](../../../repos/qdrant/lib/segment:1) contains segment/index structures; [`lib/storage`](../../../repos/qdrant/lib/storage:1) contains persistence, cluster metadata, RBAC, snapshots, and dispatch; [`tests`](../../../repos/qdrant/tests:1) and per-crate integration tests exercise operations; [`openapi`](../../../repos/qdrant/openapi:1) exposes generated API documentation assets; and [`config`](../../../repos/qdrant/config:1) contains runtime configuration.

## Core data model

**FACT:** A point has an ID, one or more vectors, and optional JSON payload in [`PointStruct`](../../../repos/qdrant/lib/api/src/rest/schema.rs:1427). Batch writes support upsert/insert-only/update-only modes, optional shard keys, and update filters in [`PointsBatch`](../../../repos/qdrant/lib/api/src/rest/schema.rs:1457). **FACT:** Search accepts named vectors, filters, search parameters, limit/offset, payload/vector projection, and score threshold in [`SearchRequestInternal`](../../../repos/qdrant/lib/api/src/rest/schema.rs:1210). **FACT:** The storage layer models collection access lists and read/read-write/points-read-write modes in [`rbac/mod.rs`](../../../repos/qdrant/lib/storage/src/rbac/mod.rs:34). **INFERENCE:** Collections and payloads are the natural isolation primitives; Qdrant does not define application users, projects, conversations, documents, or memory records.

## API/interface boundary

**FACT:** REST and gRPC APIs are generated from source models/protobufs, including point/query types in [`schema.rs`](../../../repos/qdrant/lib/api/src/rest/schema.rs:1427) and [`qdrant.rs`](../../../repos/qdrant/lib/api/src/grpc/qdrant.rs:1395). **FACT:** Internal collection operations use typed update/query enums and shard selectors, as shown in [`collection_test.rs`](../../../repos/qdrant/lib/collection/tests/integration/collection_test.rs:56). **INFERENCE:** A platform can wrap Qdrant behind an internal repository interface that translates chunk IDs, tenant/project payload filters, embeddings, and citations into Qdrant operations.

## Deployment and persistence

**FACT:** Qdrant builds a standalone container with Cargo and supports multi-platform/GPU builds in [`Dockerfile`](../../../repos/qdrant/Dockerfile:6). **FACT:** Local collections use WAL, segments, shard distribution, snapshots, and recovery paths; integration tests exercise snapshot restore in [`collection_restore_test.rs`](../../../repos/qdrant/lib/collection/tests/integration/collection_restore_test.rs:40). **FACT:** The repository includes object-storage-backed edge shard tooling for S3/GCS in [`shard_query/src/main.rs`](../../../repos/qdrant/lib/edge/tools/shard_query/src/main.rs:1). **INFERENCE:** Production operations require volume/object-storage planning, shard/replica configuration, backup/snapshot policy, and cluster coordination.

## Security and multi-tenancy assumptions

**FACT:** Qdrant’s `Auth` context records access, subject, remote, auth type, tracing ID, and API path, and emits audit events around checks in [`auth.rs`](../../../repos/qdrant/lib/storage/src/rbac/auth.rs:7). **FACT:** Collection RBAC distinguishes global access from collection-specific access and validates operation requirements in [`rbac/mod.rs`](../../../repos/qdrant/lib/storage/src/rbac/mod.rs:126). **FACT:** Collection metadata operations and point/update operations are checked through typed access requirements in [`ops_checks.rs`](../../../repos/qdrant/lib/storage/src/rbac/ops_checks.rs:27). **INFERENCE:** Qdrant has service/database authorization, not SaaS application tenancy. Collection-level credentials or a trusted platform gateway should prevent users from choosing arbitrary collection names or filters.

## Extension/plugin model

**FACT:** Qdrant is extended primarily through configuration, API models, storage backends/features, Rust crate boundaries, and deployment integrations rather than a runtime plugin marketplace. The crate workspace and features are defined in [`Cargo.toml`](../../../repos/qdrant/Cargo.toml:1). **INFERENCE:** Adding a new vector distance/index/storage capability may require Rust source integration; selecting collections, payload indexes, shard keys, or object storage is configuration/API-level.

## Failure, retry, and background behavior

**FACT:** Collections use WAL/segments and recovery/snapshot flows, and cluster dispatch uses consensus/shard distribution in [`dispatcher.rs`](../../../repos/qdrant/lib/storage/src/dispatcher.rs:73). **FACT:** Integration tests cover resharding, snapshot recovery, WAL clocks, replication, and transient/abort cases under [`lib/collection/tests/integration`](../../../repos/qdrant/lib/collection/tests/integration:1). **INFERENCE:** Qdrant provides durable database write/recovery semantics, but it does not schedule document parsing, chunking, embedding jobs, or application-level retry workflows.

## Observability and testing

**FACT:** The repository contains extensive unit, integration, benchmark, compatibility, RBAC, quota/snapshot, and REST/gRPC schema tests, including RBAC operation tests in [`ops_checks.rs`](../../../repos/qdrant/lib/storage/src/rbac/ops_checks.rs:382). **FACT:** Audit logging is integrated with request auth checks in [`auth.rs`](../../../repos/qdrant/lib/storage/src/rbac/auth.rs:109). **UNKNOWN:** This wave did not run cluster-scale benchmarks or validate operational SLOs on the captured checkout.

## Upgradeability and coupling

**FACT:** Client-facing REST/gRPC schemas are typed and can be consumed independently, but the storage engine, segment formats, WAL, shard distribution, consensus, and persistence are deeply internal Rust components. **INFERENCE:** Using Qdrant over HTTP/gRPC is highly replaceable; linking against internal crates or relying on on-disk formats is a source-level coupling/fork risk. Payload schema and collection naming conventions become application coupling even through the API.

## Integration level and fit

- **Likely integration level:** **1 — HTTP/API** for the recommended boundary; **3 — shared infrastructure/database** because Qdrant is a stateful service; **4 — source integration** only for custom index/storage behavior.
- **Fit:** Strong replaceable vector/index subsystem behind a platform-owned RAG repository. Keep chunk/document metadata, parser status, embedding model version, tenant/project authorization, citation assembly, and ingestion jobs in the platform/RAG service; store immutable IDs and filterable ownership payloads in Qdrant.
- **Not provided:** **FACT/UNKNOWN:** Qdrant does not provide document upload/parsing/chunking, embedding generation, LLM retrieval orchestration, citations policy, conversation memory, workflow execution, SaaS billing, or application tenant lifecycle.

## Modification difficulty experiments

| Experiment | Score | Reason |
|---|---:|---|
| Add another vector collection/index | 0–1 | REST/gRPC collection and point APIs. |
| Tenant/project filtering | 1–2 | Payload filters and collection RBAC exist, but host must derive filters safely. |
| Replace vector backend | 1 at API boundary; 4 if internal | HTTP abstraction is clean; internal segment/storage replacement is invasive. |
| Add custom distance/index | 4 | Rust segment/query/storage internals and persistence compatibility. |
| Add document ingestion pipeline | 3 | External service/queue/parser/embedding components required; not a Qdrant extension. |
| Cross-region durable replication | 3–4 | Cluster/consensus/backup topology, not a simple client adapter. |

## Bottom line

**INFERENCE:** Qdrant is an excellent stateful infrastructure subsystem and a poor product foundation. Use the public API, not internal Rust crates or its collection names as the application domain model, to preserve replaceability against Weaviate, pgvector, Milvus, or another index.
