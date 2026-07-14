# 78 — Blob Storage — Object Storage at Scale
## Category: HLD Components

---

## What is this?

**Blob storage** (also called **object storage**) is a giant, flat, effectively infinite key → bytes store. You hand it a string key like `users/42/avatar.jpg` and a pile of bytes, and it stores them, forever, cheaply, over HTTP. Amazon S3 is the canonical example; Google Cloud Storage, Azure Blob Storage, and Cloudflare R2 are the same idea.

Think of it as **the world's largest coat check**. You hand over a coat (any size, any shape — the attendant doesn't care what's inside), you get a ticket (the key), and later you present the ticket to get the exact same coat back. The coat check has infinite hangers, never loses a coat, and charges you almost nothing per coat per month. What it *won't* do is let you re-sew a button while the coat hangs there — you take the whole coat out, fix it, and hand back a whole new coat.

"BLOB" originally stood for **B**inary **L**arge **OB**ject. In practice it means: big immutable files — images, videos, PDFs, database backups, log archives, ML training datasets.

---

## Why does it matter?

Every real product stores files. Profile pictures, uploaded videos, generated invoices, CSV exports, nightly database dumps. The moment a system handles files, you have a decision to make about where those bytes live — and getting it wrong is one of the most expensive, hardest-to-undo mistakes in backend engineering.

**If you don't understand this:**
- You'll store a 2MB image as a `BLOB` column in Postgres, and eighteen months later your database is 4TB, your nightly backup takes 9 hours, your cache hit rate has collapsed, and you're paying 30x more per gigabyte than you need to.
- You'll route every byte of every 5GB video upload through your Node.js API server, and one user on hotel wifi will hold an Express worker hostage for 40 minutes.
- You'll serve images directly from your origin server and wonder why your bandwidth bill is bigger than your compute bill.

**In interviews:** Blob storage shows up in *almost every* HLD question. "Design Instagram" — where do photos go? "Design YouTube" — where do video segments go? "Design Dropbox" — the whole question is blob storage. Interviewers specifically listen for two phrases: **"store the file in S3, store the URL in the database"** and **"pre-signed URL so the upload bypasses our servers."** Say those and you've cleared a bar. Miss them and you look junior.

**At work:** You'll design upload flows, set lifecycle policies that quietly save your company six figures a year, and debug why an image 404s for 3 seconds after upload.

---

## The core idea — explained simply

### The Warehouse vs. The Filing Cabinet vs. The Hard Drive

Imagine three ways to store things in a company.

**The hard drive (block storage).** You're given a raw, empty metal box with numbered slots. Slot 1, slot 2, slot 3 … up to slot 200,000,000. There's no concept of a "file" — just numbered slots holding fixed-size chunks. To make it useful, YOU install an organizing system on top (a filesystem: ext4, NTFS). It's blindingly fast because it's a physical disk bolted to your machine, but only *one* machine can have it plugged in at a time, and when it's full, it's full. This is **AWS EBS**, a virtual disk you attach to one EC2 instance.

**The filing cabinet in the shared office (file storage).** A cabinet with drawers, folders inside drawers, folders inside folders. Everyone in the office walks over and opens the same cabinet. There's a real hierarchy, real folders, and normal file operations — open, seek to the middle, overwrite 4 bytes, close. But there's a physical cabinet somewhere, and if 5,000 people try to open the same drawer at once, you have a problem. This is **NFS / AWS EFS**.

**The warehouse with a ticket window (object storage).** There is no cabinet, no drawers, no hierarchy. There is one enormous warehouse and one clerk at a window. You say: "store this crate, ticket number `photos/2026/07/12/sunset.jpg`." He takes it. Later you say the ticket number, he brings the crate. You can't ask him to open the crate and change something inside — you can only replace the entire crate. In exchange, the warehouse is infinite, is accessible from anywhere on earth over HTTP, and costs almost nothing. This is **S3**.

**Mapping the analogy back:**

| Analogy | Technical concept | Real service |
|---|---|---|
| Numbered metal slots, no meaning | Blocks (fixed-size, addressed by number) | AWS EBS, GCP Persistent Disk |
| You install the organizing system | You put ext4/XFS on top of the block device | `mkfs.ext4 /dev/xvdf` |
| One machine plugs it in | One instance attaches the volume | EBS is single-attach (mostly) |
| Shared cabinet, real folders | Hierarchical filesystem over the network | NFS, AWS EFS, Azure Files |
| Ticket → crate, no folders | Flat key → bytes, accessed over HTTP | S3, GCS, Azure Blob, R2 |
| Can't re-sew a button in place | Objects are immutable — replace, don't edit | `PutObject` replaces the whole object |

### The three storage types, side by side

| | **Block storage** | **File storage** | **Object storage** |
|---|---|---|---|
| **Unit** | Fixed-size block | File in a directory tree | Object (key + bytes + metadata) |
| **Access** | SCSI/NVMe, attached device | POSIX (`open`/`read`/`write`/`seek`) | HTTP (`GET`/`PUT`/`DELETE`) |
| **Structure** | None (raw) | Hierarchical folders | **Flat namespace** |
| **Partial edit** | Yes — write any block | Yes — seek and overwrite | **No** — replace whole object |
| **Latency** | ~0.1–1 ms | ~1–5 ms | ~20–150 ms first byte |
| **Shared by** | One machine | Many machines | The whole internet |
| **Max size** | ~64 TB per volume | Petabytes, but managed | **Effectively unlimited** |
| **Cost / GB / mo** | ~$0.08 (gp3) | ~$0.30 (EFS standard) | **~$0.023 (S3 Standard)** |
| **Good for** | Databases, OS disks | Shared home dirs, legacy apps, CMS | Images, video, backups, logs, ML data |
| **Example** | AWS EBS | AWS EFS / NFS | **AWS S3** |

Notice the cost column. Object storage is roughly **3.5x cheaper than block** and **13x cheaper than file** storage — and that's before you compare it to a managed database, where you're really paying for IOPS and RAM, not bytes.

---

## Key concepts inside this topic

### 1. Never store files in your database — do the actual math

This is the single most common junior mistake, and it's worth being able to argue *numerically*, not just by vibes.

Suppose you're building a photo app. 100,000 users, each uploads 10 photos, each photo is 2 MB.

```
Total bytes = 100,000 users × 10 photos × 2 MB = 2,000,000 MB = 2 TB
```

**Option A — store the image bytes in a Postgres `bytea` column:**

| Consequence | Why it hurts |
|---|---|
| DB size goes from ~20 GB to **~2 TB** | 100x growth in a system you were *not* sizing for bytes |
| Nightly `pg_dump` goes from 4 min to **~7 hours** | Backups now dominate your ops calendar |
| Buffer pool / shared_buffers is poisoned | Postgres caches *pages*. A single 2 MB image evicts ~250 pages of hot index data. Your `WHERE user_id = ?` query that was 0.4 ms is now 8 ms because the index isn't cached anymore. |
| Replication lag explodes | Every photo upload ships 2 MB down the WAL stream to every replica |
| Cost | RDS storage ≈ **$0.115/GB/mo** (gp3) → 2 TB ≈ **$236/mo**, and that's just the disk, not the instance |
| You can't put a CDN in front of it | Every image read burns a DB connection |

**Option B — store the image in S3, store the key in Postgres:**

```
DB row:  id | user_id | s3_key                          | width | height | created_at
         77 | 42      | photos/42/9f3a...c1.jpg         | 1080  | 1350   | 2026-07-12
```

| Consequence | Effect |
|---|---|
| DB size | Back to **~20 GB** — it stores strings, not pixels |
| Backup | Back to **~4 minutes** |
| Cache | Buffer pool holds only hot rows and indexes. Queries stay fast. |
| Cost | 2 TB in S3 Standard ≈ **$47/mo**, and if it ages into Infrequent Access, **$25/mo** |
| Serving | Point CloudFront at the bucket — reads never touch your infrastructure |

**$236/mo vs $47/mo is 5x on storage alone**, and the real gap is bigger because RDS charges you for IOPS and memory you're now wasting on binary blobs. Against S3 Glacier the ratio hits 30–50x.

**The rule, memorize it:**

> **The file goes in blob storage. The pointer goes in the database.**

The database is a *fast, transactional index over small structured records*. Every byte you put in it should be a byte you plan to filter, sort, join, or aggregate on. You will never write `WHERE image_bytes LIKE ...`. So the image doesn't belong there.

**The one legitimate exception:** genuinely tiny blobs (< ~10 KB) that are always read together with the row — a thumbnail hash, an SVG icon, an encrypted secret. Even then, prefer a separate table so the main table's rows stay small.

### 2. How S3 works conceptually — buckets, keys, and the folder illusion

Three nouns:

- **Bucket** — a globally-unique namespace, e.g. `acme-user-uploads-prod`. Lives in one region. This is your top-level container; you'll have a handful, not thousands.
- **Key** — the full string identifying an object *within* a bucket: `photos/42/2026/07/sunset.jpg`.
- **Object** — the bytes, plus **metadata** (content-type, content-length, ETag, custom `x-amz-meta-*` headers), plus a storage class and optional version ID.

**The folder illusion — this is an interview favorite.**

There are **no folders in S3.** The key `photos/42/sunset.jpg` is a single flat string that happens to contain `/` characters. S3 does not have a directory called `photos`. The console *renders* a folder tree by grouping on the `/` delimiter, which fools everyone.

```
What you think exists:              What actually exists:
  photos/                             ┌──────────────────────────────────────────┐
    └── 42/                           │ KEY (string)              → BYTES        │
        ├── sunset.jpg                ├──────────────────────────────────────────┤
        └── beach.jpg                 │ "photos/42/sunset.jpg"    → <bytes>      │
                                      │ "photos/42/beach.jpg"     → <bytes>      │
                                      │ "backups/db-2026-07-12"   → <bytes>      │
                                      └──────────────────────────────────────────┘
                                       ONE FLAT HASH MAP. That's it.
```

Consequences that actually bite you:

- **You cannot rename a "folder".** There's nothing to rename. Renaming `photos/42/` to `photos/99/` means copying every object to a new key and deleting the old ones. On a million objects that's a million API calls.
- **Listing is a prefix scan.** `listObjectsV2({ Prefix: 'photos/42/' })` scans keys that start with that string. There's no O(1) "give me this directory."
- **Key design matters.** Prefix with something high-cardinality and evenly distributed (a user ID, a UUID, a hash) so S3 can partition your keyspace. Historically, keys that all started with a timestamp (`2026-07-12-...`) created a hot partition. S3 auto-partitions now, but a well-distributed prefix is still the safe habit.

**Durability: what "eleven nines" actually means.**

S3 Standard advertises **99.999999999% (11 nines) durability of objects over a given year.** In plain English: if you store 10,000,000 objects, you can statistically expect to lose **one object every 10,000 years**.

How? **Erasure coding.** Your object is split into `k` data shards, and `m` parity shards are computed from them (think of parity as very clever mathematical checksums, like RAID but generalized). Any `k` of the `k+m` shards can reconstruct the original. Those shards are then scattered across different disks, different racks, and **at least three Availability Zones** (physically separate datacenters, kilometers apart, separate power and network).

```
        Object "sunset.jpg"  (6 MB)
                  │
         split + compute parity
                  │
    ┌─────┬─────┬─────┬─────┬─────┬─────┐
    │ D1  │ D2  │ D3  │ D4  │ P1  │ P2  │   (4 data + 2 parity, illustrative)
    └──┬──┴──┬──┴──┬──┴──┬──┴──┬──┴──┬──┘
       │     │     │     │     │     │
     AZ-a  AZ-a  AZ-b  AZ-b  AZ-c  AZ-c
     
  Lose ANY 2 shards (a dead disk, a burning rack, a whole AZ)
  → the remaining 4 mathematically reconstruct the object.
```

**Durability is NOT availability. Be precise about this — interviewers probe it.**

| | Durability | Availability |
|---|---|---|
| **Question it answers** | "Will my bytes still exist?" | "Can I read my bytes *right now*?" |
| **S3 Standard SLA** | 99.999999999% (11 nines) | **99.99%** (4 nines) |
| **Failure looks like** | Object permanently gone | `503 Slow Down` / timeout |
| **Budget** | ~1 object lost per 10M per 10,000 years | **~52 minutes of downtime per year** |

Your data is essentially never lost. But roughly an hour a year, you may not be able to reach it. Design for that: retries with exponential backoff, and a CDN in front so cached reads survive an S3 blip.

### 3. Pre-signed URLs — the single most valuable pattern here

**The naive flow (what everyone does first, and it's wrong):**

```
Client ──── 5 GB video ────▶ Your Node API ──── 5 GB video ────▶ S3
```

Why this is bad, concretely:

- **Every byte crosses your server twice** (in and out). A 5 GB upload = 10 GB of your bandwidth, billed.
- **Memory / disk pressure.** Even with streaming, you're holding buffers and file handles for the duration.
- **Request timeouts.** Your load balancer has a 60s idle timeout. Your Express server has a body-size limit. A user on 2 Mbps hotel wifi needs **~5.5 hours** for 5 GB. Your infrastructure will kill that request long before it finishes.
- **Concurrency collapse.** Node is single-threaded per process. 50 concurrent slow uploads and your event loop is drowning in stream backpressure while your `/api/feed` endpoint starves.
- **Scaling the wrong thing.** You'd have to scale API servers for *bandwidth*, not for *logic*.

**The correct flow — pre-signed URLs:**

A **pre-signed URL** is a normal S3 URL with a cryptographic signature baked into the query string. The signature encodes: which bucket, which key, which HTTP method, and an expiry timestamp — all signed with your AWS secret key. S3 verifies the signature itself. **Anyone holding that URL can perform exactly that one operation, until it expires.** Your server never sees the bytes; it only *authorizes* the transfer.

```
┌────────┐                                                    ┌────────────┐
│        │  1. POST /api/uploads                              │            │
│        │     { filename, contentType, size }                │  Your API  │
│ Client │ ─────────────────────────────────────────────────▶ │  (Node)    │
│        │                                                    │            │
│        │  2. { uploadUrl: "https://bucket.s3...?X-Amz-      │  - authN   │
│        │       Signature=abc&X-Amz-Expires=900", key }      │  - authZ   │
│        │ ◀───────────────────────────────────────────────── │  - quota   │
│        │     (no bytes moved — just a signed string)        │  - DB row  │
└───┬────┘                                                    └─────┬──────┘
    │                                                               │
    │  3. PUT <uploadUrl>                                           │ 4. S3 Event
    │     Body: the raw 5 GB of video bytes                         │    Notification
    │     ──── DIRECT TO S3, BYPASSING YOUR SERVER ────▶            │    (s3:ObjectCreated)
    │                                                               │
    ▼                                                               ▼
┌─────────────────────────────────────────┐              ┌──────────────────┐
│                  S3                     │─────────────▶│  SQS / Lambda /  │
│  verifies signature, expiry, method     │   event      │  your webhook    │
│  writes object, returns 200             │              │                  │
└─────────────────────────────────────────┘              │ marks upload     │
                                                         │ COMPLETE in DB,  │
                                                         │ enqueues         │
                                                         │ transcode job    │
                                                         └──────────────────┘
```

**Node.js — generating a pre-signed upload URL:**

```javascript
// npm i @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { randomUUID } from 'node:crypto';

const s3 = new S3Client({ region: 'ap-south-1' });
const BUCKET = 'acme-user-uploads-prod';

const ALLOWED_TYPES = new Set(['image/jpeg', 'image/png', 'video/mp4']);
const MAX_BYTES = 5 * 1024 * 1024 * 1024; // 5 GB

/**
 * The client tells us what it WANTS to upload. We decide whether it may,
 * pick the key ourselves (never trust a client-supplied path — that's how
 * you get someone writing to `photos/../../backups/prod.dump`), and hand
 * back a signature that is valid for exactly one PUT, for 15 minutes.
 */
export async function createUploadUrl({ userId, contentType, sizeBytes }) {
  if (!ALLOWED_TYPES.has(contentType)) throw new Error('Unsupported content type');
  if (sizeBytes > MAX_BYTES) throw new Error('File too large');

  const key = `uploads/${userId}/${randomUUID()}`;

  const command = new PutObjectCommand({
    Bucket: BUCKET,
    Key: key,
    ContentType: contentType,
    // ContentLength pins the size INTO the signature: a client that signs for
    // 2 MB cannot then push 50 GB through the same URL.
    ContentLength: sizeBytes,
  });

  const uploadUrl = await getSignedUrl(s3, command, { expiresIn: 900 }); // 15 min

  // Record the intent BEFORE the bytes land, so an upload that never completes
  // is visible as a PENDING row we can garbage-collect later.
  await db.uploads.insert({ key, userId, status: 'PENDING', createdAt: new Date() });

  return { uploadUrl, key };
}

/** Symmetric idea for reading a private object: a short-lived GET signature. */
export async function createDownloadUrl(key, ttlSeconds = 300) {
  const command = new GetObjectCommand({ Bucket: BUCKET, Key: key });
  return getSignedUrl(s3, command, { expiresIn: ttlSeconds });
}
```

**And the browser side — note that no AWS credentials ever touch the client:**

```javascript
async function uploadFile(file) {
  const res = await fetch('/api/uploads', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ contentType: file.type, sizeBytes: file.size }),
  });
  const { uploadUrl, key } = await res.json();

  // The 5 GB goes browser → S3. Your API server is already free, serving
  // other requests. It never allocated a single byte of this file.
  await fetch(uploadUrl, {
    method: 'PUT',
    headers: { 'Content-Type': file.type, 'Content-Length': String(file.size) },
    body: file,
  });

  return key; // hand this to /api/photos to attach it to a post
}
```

**Closing the loop — S3 event notifications.** The client could lie ("I finished!") or crash mid-upload. So don't trust it. Configure the bucket to fire an `s3:ObjectCreated:*` event into SQS or Lambda when the object actually lands. Your worker then flips the DB row from `PENDING` to `COMPLETE`, reads real metadata, and kicks off thumbnailing or transcoding. **S3 is the source of truth for "did the file arrive."**

### 4. Multipart upload — parallelism and retries for big files

A single `PUT` of a 5 GB file is one long, fragile TCP conversation. If the connection drops at 97%, you upload all 5 GB again. And you're limited to one connection's throughput.

**Multipart upload** splits the object into parts (5 MB–5 GB each, up to 10,000 parts), uploads them **in parallel over separate connections**, and lets S3 stitch them back together server-side.

```
  5 GB video
       │  split into 500 parts × 10 MB
       ▼
 ┌────┬────┬────┬────┬─── … ───┬─────┐
 │ P1 │ P2 │ P3 │ P4 │         │P500 │
 └─┬──┴─┬──┴─┬──┴─┬──┘         └──┬──┘
   │    │    │    │  (8 in flight at a time)
   ▼    ▼    ▼    ▼               ▼
 ══════════ parallel PUTs to S3 ══════════
   │    │    │    │               │
   └────┴────┴────┴───────────────┘
         each returns an ETag
                 │
                 ▼
      CompleteMultipartUpload([{PartNumber, ETag}, ...])
                 │
                 ▼
        S3 assembles ONE object
```

Why it matters:
- **Parallelism** — 8 concurrent parts can saturate a link that one stream cannot. Uploads get several times faster.
- **Retry granularity** — part 217 fails? Retry *part 217* (10 MB), not the whole 5 GB.
- **Resumability** — the upload ID persists. A mobile app that loses signal in a tunnel resumes with the parts it hasn't sent.
- **Required above 5 GB** — a single `PutObject` caps at 5 GB. Multipart goes to 5 TB.

```javascript
import { S3Client } from '@aws-sdk/client-s3';
import { Upload } from '@aws-sdk/lib-storage';
import { createReadStream } from 'node:fs';

const s3 = new S3Client({ region: 'ap-south-1' });

/**
 * lib-storage's Upload handles the whole dance: CreateMultipartUpload,
 * chunking, N concurrent UploadPart calls, per-part retries, and
 * CompleteMultipartUpload. Use it for server-side uploads (e.g. a worker
 * pushing a transcoded video back to S3).
 */
export async function uploadLargeFile(localPath, key) {
  const uploader = new Upload({
    client: s3,
    params: { Bucket: 'acme-media-prod', Key: key, Body: createReadStream(localPath) },
    queueSize: 8,               // 8 parts in flight — tune to your bandwidth
    partSize: 10 * 1024 * 1024, // 10 MB parts
    leavePartsOnError: false,   // clean up orphaned parts if we give up
  });

  uploader.on('httpUploadProgress', (p) => {
    console.log(`${((p.loaded / p.total) * 100).toFixed(1)}%`);
  });

  return uploader.done();
}
```

**The billing trap everyone hits:** if a multipart upload is never completed or aborted, the uploaded parts **sit in your bucket, invisible in the object listing, and you are billed for them.** Always add a lifecycle rule: `AbortIncompleteMultipartUpload after 7 days`. Companies have found five figures a month of ghost parts this way.

For browser uploads of big files, the same pattern works: your API pre-signs *each part's* URL, the browser uploads parts directly to S3 in parallel, then calls your API with the list of ETags so the server can issue `CompleteMultipartUpload`.

### 5. Storage classes and lifecycle policies — where the money is

Not all bytes are equally hot. A photo is viewed 200 times in its first week and then, statistically, never again. A compliance log is written once and read only if regulators come knocking. Paying hot-storage prices for cold bytes is pure waste.

**Storage classes trade retrieval speed and retrieval cost for storage cost:**

| Class | Storage $/GB/mo | Retrieval latency | Retrieval cost | Min. duration | Use for |
|---|---|---|---|---|---|
| **S3 Standard** | ~$0.023 | milliseconds | free (you pay egress) | none | Active images, hot video, anything user-facing |
| **S3 Intelligent-Tiering** | $0.023 → $0.004 | ms (auto-managed) | free | none | Unknown/unpredictable access patterns |
| **S3 Standard-IA** | ~$0.0125 | milliseconds | **~$0.01/GB** | 30 days | Backups read a few times a year |
| **S3 Glacier Instant Retrieval** | ~$0.004 | milliseconds | ~$0.03/GB | 90 days | Archives you'd want *immediately* if ever |
| **S3 Glacier Flexible Retrieval** | ~$0.0036 | **minutes to 12 hours** | ~$0.01/GB | 90 days | Old backups, legal archives |
| **S3 Glacier Deep Archive** | **~$0.00099** | **12–48 hours** | ~$0.02/GB | 180 days | 7-year compliance retention, tape replacement |

Read that bottom row again. **Deep Archive is ~23x cheaper than Standard.** The catch is right there in the latency column: you ask for the object and it shows up *tomorrow*. Cheap storage means slow, expensive retrieval. That's the whole trade.

**Worked cost example — a media startup with 500 TB of user video:**

```
All in S3 Standard:
  500,000 GB × $0.023        = $11,500 / month = $138,000 / year

Realistic access pattern: 10% is hot (recent), 90% is basically never touched.

With a lifecycle policy:
   50,000 GB (hot)     × $0.023   =    $1,150
  150,000 GB (30d+ IA) × $0.0125  =    $1,875
  300,000 GB (1y+ Glacier FR)     
                       × $0.0036  =    $1,080
                                    ─────────
                                      $4,105 / month = $49,260 / year

SAVED: ~$89,000 / year — from one YAML file nobody had to write code for.
```

**Lifecycle policies** are declarative rules the bucket applies automatically, by object age:

```javascript
// Applied once via the SDK/console/Terraform. S3 then enforces it forever,
// with no cron job, no worker, and no code of yours running.
import { PutBucketLifecycleConfigurationCommand } from '@aws-sdk/client-s3';

await s3.send(new PutBucketLifecycleConfigurationCommand({
  Bucket: 'acme-media-prod',
  LifecycleConfiguration: {
    Rules: [
      {
        ID: 'tier-down-old-videos',
        Status: 'Enabled',
        Filter: { Prefix: 'videos/' },
        Transitions: [
          { Days: 30,  StorageClass: 'STANDARD_IA' },   // cools after a month
          { Days: 365, StorageClass: 'GLACIER' },        // archives after a year
        ],
        // Expiration: { Days: 2555 },  // uncomment for a 7-year delete policy
      },
      {
        // The ghost-parts rule from section 4. Add this to EVERY bucket.
        ID: 'kill-orphaned-multipart-parts',
        Status: 'Enabled',
        Filter: { Prefix: '' },
        AbortIncompleteMultipartUpload: { DaysAfterInitiation: 7 },
      },
    ],
  },
}));
```

**Rule of thumb:** if you don't know the access pattern, use **Intelligent-Tiering** — it watches access and moves objects between tiers for you, for a small monitoring fee. If you *do* know the pattern, hand-written lifecycle rules are cheaper.

### 6. Versioning, immutability, and soft deletes

Objects are immutable — but keys are not. `PUT` the same key twice and the second write silently obliterates the first. That is a terrifying property for a bucket holding your customers' data.

**Enable versioning** and the semantics change completely:

- Every `PUT` to an existing key creates a **new version** with a fresh version ID; old versions remain, addressable by version ID.
- **`DELETE` does not delete.** It writes a zero-byte **delete marker** as the newest version. `GET` returns 404, but the real bytes are still there. Remove the delete marker and the object comes back. This is a **soft delete** — your undo button, and your ransomware insurance.
- A *real* delete requires `DELETE` with an explicit `versionId`.

```javascript
import { ListObjectVersionsCommand, DeleteObjectCommand } from '@aws-sdk/client-s3';

/** "Undelete": find the delete marker hiding the object and remove it. */
export async function restoreDeletedObject(bucket, key) {
  const { DeleteMarkers = [] } = await s3.send(
    new ListObjectVersionsCommand({ Bucket: bucket, Prefix: key })
  );
  const marker = DeleteMarkers.find((m) => m.Key === key && m.IsLatest);
  if (!marker) return { restored: false, reason: 'not deleted' };

  // Deleting the delete marker re-exposes the previous real version.
  await s3.send(new DeleteObjectCommand({
    Bucket: bucket, Key: key, VersionId: marker.VersionId,
  }));
  return { restored: true };
}
```

The cost of versioning: **you now pay for every version.** A 100 MB file overwritten daily for a year is 36 GB, not 100 MB. Pair versioning with a lifecycle rule like `NoncurrentVersionExpiration: { NoncurrentDays: 30 }`.

**Object Lock (WORM — Write Once, Read Many)** goes further: it makes an object *undeletable*, even by the root account, until a retention date passes.

- **Governance mode** — a user with a special IAM permission can override. Good for "don't delete this by accident."
- **Compliance mode** — **nobody** can delete it, including AWS root, until the clock runs out. This is what banks and hospitals use to satisfy SEC 17a-4 / HIPAA retention. It's also the single best defence against ransomware that has stolen your admin credentials.

### 7. Serving blobs to users — always via a CDN

Never let users hit your S3 bucket directly at scale. Two reasons: **latency** (a user in Mumbai fetching from `us-east-1` eats ~200 ms of round-trip before the first byte) and **cost** (S3 egress is ~$0.09/GB; CDN egress is cheaper and, on a cache hit, S3 isn't touched at all).

Put a CDN (CloudFront, Fastly, Cloudflare) in front of the bucket. Recall from [60 — CDN](./60-cdn.md) that a CDN caches your object at hundreds of edge locations near users; the first request in Mumbai pulls from S3 and warms the edge, and the next 10,000 are served in ~15 ms without S3 seeing a single request.

```
User (Mumbai) ──▶ CloudFront edge (Mumbai) ──MISS──▶ S3 (ap-south-1)
                        │                                  │
                        │◀────────── object ───────────────┘
                        │  (cached at edge for max-age)
                        ▼
                  User gets bytes (~15 ms on subsequent hits)
```

**The versioned-key trick — the most useful operational habit in this doc.**

The obvious approach is a stable key: `avatars/42.jpg`. When the user changes their avatar, you overwrite that key — and now every CDN edge on earth is serving the *old* picture until its TTL expires. So you call a cache invalidation. Invalidations are slow (minutes), rate-limited, and cost money.

Instead, **make the key content-dependent and never overwrite anything:**

```javascript
import { createHash } from 'node:crypto';

/**
 * Content-addressed keys: the key IS a fingerprint of the bytes.
 * Different bytes → different key → different URL → the CDN has never
 * seen it → guaranteed fresh. Same bytes → same key → we skip the upload
 * entirely (free deduplication).
 */
function contentKey(buffer, ext) {
  const hash = createHash('sha256').update(buffer).digest('hex').slice(0, 32);
  return `avatars/${hash}.${ext}`;
}

async function setAvatar(userId, buffer, ext) {
  const key = contentKey(buffer, ext);
  await putIfAbsent(key, buffer);            // no-op if those exact bytes exist
  await db.users.update(userId, { avatarKey: key }); // ← the DB switch is the "publish"
  return `https://cdn.acme.com/${key}`;
}
```

Now you can set `Cache-Control: public, max-age=31536000, immutable` (cache for a year — this URL's content can never change). **You never invalidate anything, ever.** Updating the avatar means writing a *new* key and flipping one small DB column. The DB row is the mutable pointer; the blob is immutable. This is exactly how build tools produce `app.9f3ac1.js` filenames, and it is the same idea.

### 8. Consistency — and the historical footgun

**Since December 2020, S3 provides strong read-after-write consistency** for all operations, in all regions, at no extra cost. Write an object, immediately read it, and you get the new bytes. List a bucket right after a write and the new key is there.

**Why you must still know the history:** for its first 14 years, S3 was *eventually consistent* for overwrites and deletes. Write a new version of a key and a `GET` a second later could legitimately return the **old** bytes. This shaped an entire generation of architectures — Hadoop had a whole subsystem (`S3Guard`) whose only purpose was compensating for it, and the standard mitigation was exactly the versioned-key trick from section 7 (if you never overwrite a key, eventual consistency on overwrites can't hurt you).

Older interview prep material — and possibly your interviewer — still says "S3 is eventually consistent." The strong answer is:

> "S3 has been strongly read-after-write consistent since December 2020. Before that, new-object writes were read-after-write consistent but *overwrites and deletes* were eventually consistent, which is why immutable content-addressed keys became the standard pattern. That pattern is still worth keeping — it makes CDN caching trivial."

Two things that are still *not* strongly consistent and are worth naming: **cross-region replication is asynchronous** (a write in `us-east-1` is not instantly readable from the `eu-west-1` replica bucket), and **CDN caches are, by definition, stale** until their TTL expires. The blob store is consistent; the layers you bolt on top of it are not.

---

## Visual / Diagram description

### Diagram 1: The complete upload → process → serve pipeline

This is the diagram to draw on the whiteboard whenever a design involves user-uploaded media.

```
                                   ┌──────────────────────────────┐
   ┌──────────┐   1. "I want to    │       API SERVERS (Node)     │
   │          │      upload"       │                              │
   │  CLIENT  │ ─────────────────▶ │  - authenticate the user     │
   │ (browser │                    │  - check quota / file type   │
   │  or app) │ ◀───────────────── │  - generate PRE-SIGNED URL   │
   │          │   2. pre-signed    │  - INSERT uploads(PENDING)   │
   └────┬─────┘      URL + key     └──────────────┬───────────────┘
        │                                         │
        │ 3. PUT the raw bytes                    │ writes metadata only
        │    DIRECT to S3                         ▼
        │    (multipart, parallel)        ┌───────────────┐
        │    ✗ never touches API          │  POSTGRES     │
        │                                 │  photos(      │
        ▼                                 │   id, user_id,│
 ┌─────────────────────────────┐          │   s3_key,     │◀──┐
 │        BLOB STORAGE (S3)    │          │   status)     │   │
 │                             │          └───────────────┘   │
 │  uploads/42/9f3a-...  →     │                              │
 │      <bytes>                │                              │ 6. mark
 │  Bucket: versioning ON      │                              │  COMPLETE
 │  Lifecycle: →IA 30d →Glacier│                              │  + attach
 └──────┬───────────────▲──────┘                              │  metadata
        │               │                                     │
        │ 4. s3:Object  │ 5b. worker writes                   │
        │    Created    │     derivatives back                │
        │    EVENT      │     (thumb.jpg, 720p.mp4)           │
        ▼               │                                     │
 ┌──────────────┐   ┌───┴──────────────────┐                  │
 │  SQS QUEUE   │──▶│   WORKER FLEET       │──────────────────┘
 │              │   │  5a. transcode /     │
 │              │   │      thumbnail /     │
 │              │   │      virus scan      │
 └──────────────┘   └──────────────────────┘

  ─────────────────── SERVING PATH (completely separate) ──────────────────

  ┌────────┐  GET cdn.acme.com/thumbs/9f3a.jpg  ┌──────────┐  MISS  ┌──────┐
  │ CLIENT │ ────────────────────────────────▶  │   CDN    │ ─────▶ │  S3  │
  │        │ ◀────────────────────────────────  │  (edge)  │ ◀───── │      │
  └────────┘   ~15 ms on a cache HIT            └──────────┘        └──────┘
               API servers: not involved. At all.
```

**What this shows.** Two paths that never touch your API servers with bytes. On the write path, the API's only job is to *authorize* and *record* — it hands out a signed permission slip and writes one small row. The bytes go straight from client to S3. S3 then tells your system (via an event) that the file really arrived, and a worker fleet does the CPU-heavy derivative work asynchronously, writing results *back* to S3. On the read path, a CDN sits in front and absorbs essentially all traffic.

The load-bearing insight: **your API servers scale with requests-per-second, not with gigabytes.** That's what lets a small team serve petabytes.

### Diagram 2: Block vs File vs Object, physically

```
  BLOCK (EBS)                 FILE (EFS/NFS)              OBJECT (S3)
 ┌──────────────┐          ┌───────────────┐          ┌──────────────────┐
 │  EC2 instance│          │ EC2 │ EC2 │EC2│          │  Any HTTP client │
 └──────┬───────┘          └──┬──┴──┬──┴─┬─┘          │  anywhere on the │
        │ attached            │     │    │            │  internet        │
        │ (one machine)       └─────┼────┘            └────────┬─────────┘
        ▼                           │ NFS mount                │ HTTPS
 ┌──────────────┐                   ▼                          ▼
 │ /dev/xvdf    │            ┌─────────────┐          ┌──────────────────┐
 │ ┌──┬──┬──┬──┐│            │  /shared    │          │  GET /bucket/key │
 │ │b0│b1│b2│b3││            │   ├─ docs/  │          │                  │
 │ └──┴──┴──┴──┘│            │   │   └ a.md│          │  flat map:       │
 │ raw blocks   │            │   └─ img/   │          │  "docs/a.md"→bytes│
 │ you mkfs it  │            │       └ b.png│         │  "img/b.png"→bytes│
 └──────────────┘            └─────────────┘          └──────────────────┘
  overwrite byte 500:         overwrite byte 500:      overwrite byte 500:
      YES                          YES                  NO — rewrite whole
                                                            object
```

---

## Real world examples

### Instagram

Photos and videos never live in Instagram's databases. The metadata — user, caption, timestamp, dimensions, the storage key — lives in Postgres (and Cassandra for feeds); the actual pixels live in blob storage, served through a CDN. When you upload, the app sends bytes toward object storage while the backend records a row; a worker pipeline then generates the multiple resolutions your feed actually requests (a phone doesn't download the 4000px original to render a 400px thumbnail). This split is precisely why a single Postgres cluster can index billions of photos: it stores a few hundred bytes per photo, not a few megabytes. See [100 — HLD: Instagram](./100-hld-instagram.md).

### YouTube

The uploaded original is a large blob. Multipart upload is the only sane way to get a 40 GB 4K master file across the internet reliably. Once it lands, an event kicks off a transcoding fleet that produces a *ladder* of renditions (144p through 4K) chopped into small segments — and every one of those segments is written back to blob storage as its own object with its own key. HLS/DASH playback is then, quite literally, a client fetching a manifest and then `GET`-ing a sequence of small objects from a CDN. Old, rarely-watched videos are strong candidates for cooler storage tiers; the original master, once transcoded, is a perfect archive candidate. See [101 — HLD: YouTube](./101-hld-youtube.md).

### Dropbox — content-addressed chunks and deduplication

Dropbox's design is the purest illustration of the pattern. A "file" in Dropbox is **not** stored as a file. It's split into ~4 MB **chunks**; each chunk is hashed; the hash **is** the storage key; and a metadata database stores the ordered list of chunk hashes that reconstitute the file.

```
  report.pdf (12 MB)
        │  split into 4 MB chunks, hash each
        ▼
  ┌──────────┬──────────┬──────────┐
  │ chunk A  │ chunk B  │ chunk C  │
  │ sha=9f3a │ sha=c1d0 │ sha=77ba │
  └────┬─────┴────┬─────┴────┬─────┘
       ▼          ▼          ▼
   BLOB STORE (key = the hash):
     "9f3a" → <bytes>     "c1d0" → <bytes>     "77ba" → <bytes>

   METADATA DB:
     report.pdf  →  [9f3a, c1d0, 77ba]      (owner: alice)
     report.pdf  →  [9f3a, c1d0, 77ba]      (owner: bob — SAME chunks!)
```

Three enormous wins fall out of this for free:

1. **Deduplication.** If 10,000 employees have the same 50 MB company handbook, the bytes are stored **once**. Every copy is just another metadata row pointing at the same chunk hashes. Dropbox stores far less data than its users think they've uploaded.
2. **Delta sync.** Edit one paragraph of a 12 MB PDF and only the chunks that actually changed get re-uploaded. Chunk A and C are unchanged — the client already knows their hashes exist server-side, so it uploads only chunk B.
3. **Free integrity checking.** The key is a hash of the content, so verifying a chunk is just re-hashing it.

The trade-off: your metadata database now does much more work (it holds the file→chunks mapping for every file, every version, every user), and reconstructing a file means many blob reads instead of one. Dropbox decided that trade was overwhelmingly worth it — and famously migrated off S3 onto their own "Magic Pocket" object store once their scale made the per-GB math favour owning hardware.

---

## Trade-offs

| Storing files in **the database** | Storing files in **blob storage** |
|---|---|
| ✅ Transactional — file and row commit together, atomically | ❌ Two systems to keep in sync (you can leak orphaned objects) |
| ✅ One system to back up, one to secure | ❌ Separate bucket policies, separate backup story |
| ❌ 5–50x more expensive per GB | ✅ Dirt cheap; tiering makes it cheaper still |
| ❌ Backups and restores become brutally slow | ✅ DB stays small and fast to restore |
| ❌ Destroys buffer-pool cache locality | ✅ Hot rows and indexes stay in RAM |
| ❌ Cannot put a CDN in front | ✅ Trivially CDN-able |
| ❌ Doesn't scale past a few hundred GB in practice | ✅ Scales to exabytes without you thinking about it |

| Blob storage: what you gain | Blob storage: what you give up |
|---|---|
| Effectively infinite capacity, zero capacity planning | **Higher latency** (~20–150 ms vs ~1 ms for a local disk) |
| 11 nines durability without you designing anything | **No partial writes** — change one byte, re-upload the object |
| Pay only for what you store; tiering cuts it further | **No filesystem semantics** — no `seek`, no append, no rename |
| Direct client↔storage transfer via pre-signed URLs | **Eventual consistency in the layers around it** (CDN, cross-region replication) |
| Built-in versioning, lifecycle, encryption, object lock | **Per-request costs** — millions of tiny objects can cost more in API calls than in storage |
| Reachable from anywhere over HTTP | **Egress is expensive** (~$0.09/GB out of AWS) — a real lock-in force |

**Rule of thumb:**
- **Big, immutable, read-often, written-once** (images, video, backups, logs, ML datasets) → **object storage.** Always. Not negotiable.
- **Small, structured, queried, transactional** (users, orders, comments) → **your database.**
- **Needs a real filesystem and low latency from one machine** (a Postgres data directory, a build cache) → **block storage.**
- **Many machines need the same mutable filesystem** (legacy apps, shared home directories) → **file storage** — and be a little suspicious of the design that requires it.
- **Millions of objects under 100 KB?** Batch them. Per-request cost and per-object overhead will eat you alive. Pack them into larger objects (that's exactly what [91 — Data Lakes and Warehouses](./91-data-lakes-and-warehouses.md) does with Parquet files).

---

## Common interview questions on this topic

### Q1: "A user uploads a 5 GB video. Walk me through what happens."
**Hint:** Do **not** stream it through your API server. Client calls `POST /uploads`; your API authorizes, picks a key, and returns pre-signed URLs (multipart — one signed URL per part). The client uploads parts **directly to S3 in parallel**, retrying individual failed parts. Client calls `POST /uploads/:id/complete` with the part ETags; your API issues `CompleteMultipartUpload`. S3 fires an `ObjectCreated` event → SQS → a worker fleet transcodes into a rendition ladder and writes segments back to S3 → the DB row flips to `READY`. Serving happens through a CDN. Say the words "pre-signed URL" and "the bytes never touch my API servers."

### Q2: "Why not just store the images in Postgres? It's simpler — one system."
**Hint:** Grant the simplicity point honestly, then bury it with numbers. 2 TB of images makes your DB 2 TB: backups go from minutes to hours, replication lag balloons, and — the killer — the buffer pool gets poisoned, so a single 2 MB image evicts hundreds of pages of hot index data and your fast queries get slow. It costs ~5x more per GB, and you can't put a CDN in front of it. **File in blob storage, URL in the DB.**

### Q3: "S3 claims 11 nines of durability. Does that mean my app has 11 nines of uptime for images?"
**Hint:** No — and this is the trap. **Durability ≠ availability.** 11 nines means the bytes won't be *lost* (erasure-coded shards across 3+ AZs). Availability is **99.99%** — about **52 minutes a year** where you may get a 503 or a timeout. Mitigate with retries + exponential backoff, and a CDN in front so cached objects survive an S3 blip.

### Q4: "How do you invalidate a cached image when a user changes their avatar?"
**Hint:** You don't — you make invalidation unnecessary. Never overwrite a key. Use a **content-addressed key** (`avatars/<sha256-of-bytes>.jpg`), set `Cache-Control: immutable, max-age=31536000`, and treat the DB column holding the key as the mutable pointer. Changing the avatar writes a *new* object and updates one small DB row. New URL → guaranteed CDN miss → guaranteed fresh. Zero invalidations, and identical bytes dedupe for free.

### Q5: "You're paying $11,500/month for 500 TB in S3. Cut the bill."
**Hint:** Measure the access pattern first — media access is almost always a steep power law. Then a lifecycle policy: hot stays Standard, 30 days → Standard-IA, 1 year → Glacier. Roughly $11.5k → $4.1k. Also: add `AbortIncompleteMultipartUpload` (ghost parts you're silently billed for), expire noncurrent versions, and check whether egress — not storage — is the real line item, in which case the fix is a CDN, not a storage class.

### Q6: "Is S3 eventually consistent?"
**Hint:** **Not since December 2020** — it's now strongly read-after-write consistent for all operations, everywhere, free. It *was* eventually consistent for overwrites and deletes before that, which is why immutable content-addressed keys became the industry pattern (and why a lot of prep material still gets this wrong). Note that cross-region replication and CDN caches remain eventually consistent regardless.

---

## Practice exercise

### Design the upload pipeline for a video course platform

You're building a Udemy-style platform. Instructors upload lecture videos (typically 200 MB – 4 GB, occasionally 20 GB). Students stream them. Assume **50,000 lectures**, growing by **3,000/month**, average lecture **800 MB**. Old courses are rarely watched but must never be deleted.

**Part A — the math (show your arithmetic):**
1. Total storage today, and after 3 years of growth.
2. Monthly S3 Standard cost for the year-3 figure.
3. Design a lifecycle policy (which classes, at what ages) and recompute the year-3 cost. State the saving in dollars and as a percentage. Assume ~15% of lectures are "hot" at any time.

**Part B — the diagram:**
Draw the full upload path on paper, with every arrow labelled. It must include: the client, your API server, the blob store, the event notification, the queue, the transcoding workers, the metadata DB, and the CDN. Mark clearly, with a bold line, **which arrows carry video bytes** — and convince yourself that none of them touch your API servers.

**Part C — the code:**
Write `createMultipartUpload(userId, filename, sizeBytes)` in Node.js. It should: validate the file type and size; generate an S3 key; call `CreateMultipartUpload`; compute how many 25 MB parts are needed; pre-sign a `UploadPart` URL for each; and return `{ uploadId, key, partUrls: [...] }`. Then write the matching `completeUpload(uploadId, key, parts)`. (`@aws-sdk/client-s3` exports `UploadPartCommand` and `CompleteMultipartUploadCommand` — you can pre-sign `UploadPartCommand` exactly like `PutObjectCommand`.)

**Part D — the hard questions:**
1. A student's browser dies at part 340 of 500. What happens to the 340 uploaded parts? What must you do so this doesn't quietly cost you money forever?
2. An instructor re-uploads a corrected version of lecture 7. Students have the old version cached in a CDN with a 24-hour TTL. How do you make sure everyone sees the new video **immediately**, without issuing a CDN invalidation?
3. A malicious user calls `POST /uploads` 10,000 times in a minute. What breaks, and what guard do you add?

**Produce:** the arithmetic, the labelled diagram, the two functions, and a short written answer to each of the three hard questions.

---

## Quick reference cheat sheet

- **Blob / object storage** = a flat, infinite `key → bytes` map accessed over HTTP. S3, GCS, Azure Blob, R2.
- **The three types:** **block** (raw disk, one machine, low latency, EBS) · **file** (shared hierarchical FS, EFS/NFS) · **object** (flat, HTTP, infinite, S3).
- **Objects are immutable.** You replace the whole object; you never edit a byte in the middle.
- **The one rule:** **file in blob storage, URL/key in the database.** A 2 MB image in Postgres poisons the buffer pool, wrecks backups, and costs 5–50x more.
- **"Folders" in S3 are an illusion.** `a/b/c.jpg` is one flat string containing slashes. There is no directory. Listing is a prefix scan; renaming a "folder" means copying every object.
- **Pre-signed URL** = a short-lived, cryptographically signed URL granting exactly one operation on one key. **The bytes go client → S3 directly; your API only authorizes.** This is the highest-value pattern in the doc.
- **S3 event notifications** close the loop — S3, not the client, tells you the file really landed.
- **Multipart upload:** split into parts, upload in **parallel**, retry only failed parts, resume after network drops. Required above 5 GB. **Always set `AbortIncompleteMultipartUpload`** or you pay for invisible ghost parts.
- **Durability (11 nines, erasure-coded shards across 3+ AZs) ≠ availability (99.99%, ~52 min/year).** Retry with backoff.
- **Storage classes:** Standard → Standard-IA → Glacier → Deep Archive. **Cheap storage = slow, expensive retrieval.** Deep Archive is ~23x cheaper and takes 12–48 hours to read.
- **Lifecycle policies** transition objects by age automatically — often a 60–70% bill cut for zero code.
- **Versioning** makes `DELETE` a soft delete (a delete marker). **Object Lock (compliance mode)** makes objects undeletable even by root — ransomware and audit insurance.
- **Always serve through a CDN**, and use **content-addressed / versioned keys** (`avatars/<hash>.jpg` + `Cache-Control: immutable`) so you never invalidate anything.
- **S3 is strongly read-after-write consistent since Dec 2020.** Older material saying "eventually consistent" is out of date — but cross-region replication and CDN caches still aren't.
- **Content-addressed chunking** (Dropbox) gives deduplication, delta sync, and integrity checking for free — because the key *is* the hash of the bytes.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [61 — Databases: SQL vs NoSQL](./61-databases-sql-vs-nosql.md) — the store for small structured records; blob storage is what you use for everything that *doesn't* belong there |
| **Next** | [60 — CDN](./60-cdn.md) — blob storage holds the bytes, the CDN is how you actually get them to users fast and cheaply |
| **Related** | [100 — HLD: Instagram](./100-hld-instagram.md) — photo upload, derivative generation, and CDN serving, end to end |
| **Related** | [101 — HLD: YouTube](./101-hld-youtube.md) — multipart upload of huge masters, transcoding ladders, and segment objects |
| **Related** | [97 — HLD: Pastebin](./97-hld-pastebin.md) — the simplest possible "metadata in the DB, content in the blob store" split |
| **Related** | [91 — Data Lakes and Warehouses](./91-data-lakes-and-warehouses.md) — an analytics data lake is literally object storage plus a table format |
