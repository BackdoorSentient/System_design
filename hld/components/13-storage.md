# 06_storage.md — Storage

---

**Q: What are the three primary storage paradigms — block, file, and object storage? How do they differ?**

**Block Storage**:
- Data is stored as fixed-size blocks (typically 512B–4KB). The storage system presents raw blocks to the OS, which formats them with a filesystem (ext4, NTFS, XFS).
- Analogous to a physical hard drive or SSD attached to a server.
- Access: Low-level, via OS/filesystem calls. Extremely low latency (~0.1–1ms).
- Characteristics: No built-in metadata beyond what the filesystem provides. Fast random reads/writes. Stateful — typically attached to one server at a time.
- Examples: AWS EBS (Elastic Block Store), GCP Persistent Disk, Azure Managed Disk.
- Use cases: Boot volumes, databases (MySQL, PostgreSQL data directories), any application needing a traditional filesystem.

**File Storage (NAS — Network Attached Storage)**:
- Data is stored in a hierarchical directory structure (folders/files). Clients access via NFS (Linux) or SMB/CIFS (Windows) network protocols.
- Analogous to a shared network drive.
- Multiple servers can mount the same filesystem simultaneously.
- Examples: AWS EFS (Elastic File System), Azure Files, on-prem NAS appliances.
- Use cases: Shared configuration files, CMS media libraries, home directories, applications that need shared filesystem access across multiple instances.
- Latency: ~1–10ms (network adds overhead vs local block).

**Object Storage**:
- Data stored as flat objects (key → value), where the value is a binary blob (any file) and the key is a globally unique identifier (typically a URL-like path).
- No hierarchy (though keys can include `/` to simulate directories).
- Access: Via REST API (`GET`, `PUT`, `DELETE` HTTP calls).
- Examples: AWS S3, GCP Cloud Storage, Azure Blob Storage, MinIO (self-hosted).
- Characteristics: Infinitely scalable (no practical size limit). Designed for high durability (S3: 11 nines — 99.999999999%). Eventual consistency (read-after-write for new objects; may be eventual for overwrites).
- Latency: ~10–100ms per request (network round trip to object store).
- Use cases: Static assets, user-uploaded files (photos, videos), backups, data lake (raw log files, ML training data), artifacts.

---

**Q: Deep-dive on Amazon S3 — how does it work internally and what are its key features?**

S3 is a distributed object store where objects are stored in **buckets** (globally unique name). Objects can be up to 5TB each. Total storage is effectively unlimited.

**Internal architecture (simplified)**:
- Objects are divided into chunks and distributed across many storage nodes.
- Multiple copies are maintained across AZs (at least 3 AZs within a region).
- Index (mapping key → location) is maintained separately for fast lookups.
- Background replication continuously verifies and repairs checksums.

**Key features**:

**Storage classes** (tiered pricing based on access frequency):
| Class | Use case | Retrieval latency | Cost |
|---|---|---|---|
| S3 Standard | Frequently accessed | Milliseconds | $$$ |
| S3 Standard-IA | Infrequent access | Milliseconds | $$ (+ retrieval fee) |
| S3 One Zone-IA | Non-critical, infrequent | Milliseconds | $ |
| S3 Glacier Instant | Archive with quick access | Milliseconds | $ |
| S3 Glacier Flexible | Archive | 1–12 hours | Very cheap |
| S3 Glacier Deep Archive | Long-term archive | 12–48 hours | Cheapest |

**Lifecycle policies**: Automatically transition objects between storage classes or delete them after a configurable number of days. Example: Move logs to Glacier after 90 days, delete after 365 days.

**Versioning**: Store multiple versions of the same key. Protects against accidental deletion and overwrites. Enables point-in-time recovery.

**Multipart upload**: For objects >100MB, upload in parallel chunks (each 5MB–5GB). If a part fails, retry only that part. Enables faster uploads and handling of unreliable networks.

**Pre-signed URLs**: Generate a time-limited URL that grants temporary access to a private object without requiring the requester to have AWS credentials. Used for: client-side uploads directly to S3 (bypass your server), private file downloads.

**S3 Event Notifications**: Trigger Lambda functions, SQS, or SNS when objects are created/deleted. Used for: auto-processing uploaded files (transcoding, virus scanning, thumbnail generation).

**S3 Select**: Run SQL queries on CSV/JSON/Parquet objects stored in S3. Reduces data transfer by filtering server-side.

**Eventual consistency note**: As of Dec 2020, S3 offers strong read-after-write consistency for all operations (including overwrites). This was a major improvement.

---

**Q: What is the difference between blob storage and object storage?**

These terms are often used interchangeably. The distinction is vendor terminology:

- **Object storage** is the general term for the paradigm: flat key-value storage of binary blobs with metadata.
- **Blob storage** is Microsoft Azure's term for their object storage service: Azure Blob Storage.

Functionally, AWS S3, Azure Blob Storage, and GCP Cloud Storage are all implementations of the same object storage paradigm. The differences are in specific features, pricing, and ecosystem integration.

**Azure Blob Storage specifics**:
- Three blob types: Block Blob (large files), Append Blob (log data — only append writes), Page Blob (VMs, random read/write, 512-byte aligned — used for Azure managed disks).
- Access tiers: Hot, Cool, Archive (similar to S3 storage classes).

---

**Q: How do you design a scalable file upload system using object storage?**

**Naive approach (antipattern)**: Client uploads to your server, server saves to S3. Problems: your server is a bottleneck; large files exhaust server memory/bandwidth; your server pays double network costs.

**Production approach: Direct-to-S3 upload with pre-signed URLs**:

1. Client requests an upload URL from your API: `POST /files/upload-url`.
2. Your server generates an S3 pre-signed PUT URL (valid for 15–60 minutes) and returns it.
3. Client uploads the file directly to S3 using the pre-signed URL (multipart for large files).
4. S3 stores the file. Your server's bandwidth is not consumed.
5. S3 triggers a Lambda/SNS notification on upload completion.
6. Your backend processes the file (virus scan, thumbnail, metadata extraction) and updates your DB with the final S3 key.

**Security considerations**:
- Pre-signed URL enforces: exact bucket, key prefix, file size limit (`Content-Length-Range`), content type, and expiry.
- Don't let clients choose the S3 key directly — generate a UUID key to prevent path traversal attacks and overwriting other users' files.
- Set S3 bucket policy to block all public access. Serve files via CloudFront with signed URLs (not public S3 URLs).
- Scan for malware before making uploaded files available to other users.

**Large file handling**:
- Use S3 multipart upload for files >100MB.
- Client splits file into 10MB chunks, uploads each chunk in parallel, then calls CompleteMultipartUpload.
- If upload fails mid-way, resume from the last successfully uploaded part.
- Libraries: AWS S3 Transfer Acceleration + SDK handles multipart automatically.

---

**Q: When do you use block storage vs object storage for a database?**

**Always use block storage for databases**. Here's why:

Databases require:
- **Low latency random I/O**: A PostgreSQL query scanning an index jumps to arbitrary byte offsets. AWS EBS delivers <1ms latency for SSD-backed volumes. S3 delivers 10–100ms per GET request.
- **POSIX filesystem semantics**: `fsync`, `flock`, file locking, atomic renames — required for WAL (write-ahead logging) durability. S3 doesn't support these.
- **Random write support**: Databases do frequent small random writes. S3 requires rewriting the entire object to modify any byte.

Object storage is read-optimized for large, sequential, immutable objects. It is fundamentally incompatible with transactional database workloads.

**Exception**: Some modern "serverless" databases (Neon, CockroachDB Serverless, Aurora) separate storage from compute. They store data in a custom distributed storage layer (or S3-compatible) with their own POSIX-like abstraction on top. This is a specialized architecture, not standard S3 usage.

**Backup to S3**: Databases write backups to S3 — this is a sequential write of large files, not a transactional workload. Entirely appropriate.

---

**Q: What is RAID and how does it compare to cloud storage redundancy?**

RAID (Redundant Array of Independent Disks) provides redundancy and/or performance at the physical disk level.

**Common RAID levels**:
- **RAID 0** (striping): Data split across 2+ disks. Read/write performance doubles. Zero redundancy — one disk failure loses all data.
- **RAID 1** (mirroring): Exact copy on 2+ disks. Survives one disk failure. Reads can be parallelized. Write performance is the bottleneck disk.
- **RAID 5**: Striping with parity across 3+ disks. Can survive 1 disk failure. One disk's worth of capacity used for parity. Good balance of performance, capacity, and redundancy.
- **RAID 10** (RAID 1+0): Mirror pairs that are then striped. Excellent performance and survives multiple failures (one per mirror pair). Requires 4 disks minimum, 50% storage efficiency.

**Cloud storage redundancy**: In cloud block storage (EBS, Persistent Disk), redundancy is handled transparently by the cloud provider using distributed replication across multiple physical drives and potentially multiple AZs. You don't need to configure RAID — the redundancy is built in.

**For databases on cloud**:
- Single-AZ EBS: ~99.999% durability (multiple copies within one AZ).
- Multi-AZ RDS: Synchronous standby replica in another AZ. Automatic failover in ~1–2 minutes.
- Aurora: Stores 6 copies of data across 3 AZs. Tolerates loss of 2 copies for writes and 3 for reads.

---

**Q: How do CDNs integrate with object storage for media delivery?**

Object storage (S3) is not optimized for serving files to end users globally — you pay per request, and data transfer from S3 to the internet is expensive, and latency is high for geographically distant users.

**Pattern: S3 as origin + CloudFront as CDN**:
1. Store files in S3 (private bucket).
2. Create a CloudFront distribution with S3 as the origin.
3. CloudFront edge nodes cache files at ~300+ PoPs globally.
4. Users request files via CloudFront. First request per PoP fetches from S3 (cache miss); subsequent requests are served from the PoP (cache hit, <20ms latency).

**Benefits**:
- Global low latency (CloudFront PoP is near the user).
- Reduced S3 request costs (S3 is only called on cache miss).
- HTTPS termination at the edge.
- Signed URLs for private content access control.
- Data transfer from CloudFront to internet is cheaper than S3 to internet.

**Cache TTL tuning**:
- Static assets with versioned filenames (e.g., `logo.v5.png`): `Cache-Control: max-age=31536000` (1 year). Never expires in practice because the filename changes on update.
- Dynamic content (user avatars): Shorter TTL (1 hour) or use CloudFront invalidation on upload.
- Video files: Long TTL with Range request support for partial content.
