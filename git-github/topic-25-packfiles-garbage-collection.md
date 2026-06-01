# Topic 25 — Packfiles, Garbage Collection & Storage Optimization
## How Git Compresses History · What `.git/objects/pack/` Contains · How GC Works

---

## RULE 1 — ELI5 Anchor

Imagine you have a novel manuscript. You keep every draft by photocopying the entire
600-page book each time you change one sentence. After 1,000 drafts, you've got
600,000 pages of photocopies, and 99.9% of that paper is identical text repeated
over and over.

That's what Git does at first — it creates a new *full copy* of every file every time
you change it (loose objects, Topic 02). For small repos with few files, this is fine.
But left unchecked, a repo with 10,000 commits touching a 100 KB file would store
1 GB of data for something whose *actual changes* total maybe 50 KB.

Git's solution is **packfiles** — the compression department:

| Naive approach (loose objects) | Packfile approach |
|---|---|
| One file per object, full content | All objects in one binary file |
| `blob "hello world v1"` (12 bytes) | Store only the *difference* from v1 |
| `blob "hello world v2"` (12 bytes) | `delta: change 'v1' to 'v2'` (10 bytes) |
| Each file zlib-compressed individually | All deltas zlib-compressed together |
| `objects/ab/cdef12...` per object | `objects/pack/pack-SHA.pack` |
| Slow to transfer (many small files) | Fast to transfer (one big file) |

**Garbage collection (GC)** is the janitor that:
1. Packs loose objects into packfiles
2. Discards objects that nothing points to anymore (unreachable objects)
3. Re-packs existing packfiles to remove redundancy
4. Expires old reflog entries (Topic 20) so their objects can be pruned

The three tools that make this happen:
- `git gc` — the all-in-one maintenance command
- `git repack` — just the packing step
- `git prune` — just the deletion step

---

## RULE 2 — Physical Architecture First

### The `.git/objects/` Directory: Before and After GC

**BEFORE GC (fresh repo with some commits):**
```
.git/
├── objects/
│   ├── info/                        ← pack index metadata (usually empty)
│   ├── pack/                        ← empty until first pack is created
│   ├── 1a/
│   │   └── 2b3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c  ← loose blob object
│   ├── 2b/
│   │   └── 3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d  ← loose tree object
│   ├── 3c/
│   │   └── 4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e  ← loose commit object
│   ├── ... (one directory per unique first-2-chars of SHA)
│   └── ff/
│       └── 0011223344556677889900aabbccddeeff001122  ← loose blob object
│
└── refs/
    ├── heads/
    │   └── main          ← contains: "3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d3e\n"
    └── tags/
```

**AFTER `git gc` (all objects packed):**
```
.git/
├── objects/
│   ├── info/
│   │   └── packs                    ← text file listing pack files:
│   │                                    "P pack-abc123def456....pack\n"
│   ├── pack/
│   │   ├── pack-abc123def456....idx  ← pack INDEX (lookup table, binary)
│   │   ├── pack-abc123def456....pack ← pack DATA (all objects, binary)
│   │   └── pack-abc123def456....bitmap ← reachability bitmap (optional)
│   └── (no loose object directories — all packed)
│
├── refs/
│   ├── heads/              ← may be emptied into packed-refs
│   └── tags/               ← may be emptied into packed-refs
└── packed-refs             ← all refs consolidated here:
                                 "# pack-refs with: peeled fully-peeled sorted
                                  3c4d5e6f... refs/heads/main
                                  7f8a9b0c... refs/tags/v1.0.0
                                  ^d4e5f6a7... ← peeled tag: the commit SHA"
```

### Full `.git/objects/pack/` Layout

```
objects/pack/
├── pack-{40-char SHA}.idx     ← the INDEX file (how to find objects fast)
├── pack-{40-char SHA}.pack    ← the PACK file (actual object data)
└── pack-{40-char SHA}.bitmap  ← BITMAP file (reachability speedup, optional)

The 40-char SHA in the filename is the SHA-1 of the PACK file's contents.
The .idx and .pack always have the SAME base name.

After git repack with bitmaps:
  pack-1a2b3c...idx      ← index for pack 1
  pack-1a2b3c...pack     ← data for pack 1
  pack-1a2b3c...bitmap   ← bitmap for pack 1 (at most one bitmap per repo)

After cloning a large repo (multiple packs from incremental fetches):
  pack-aaa111...idx
  pack-aaa111...pack
  pack-bbb222...idx
  pack-bbb222...pack
  pack-ccc333...idx
  pack-ccc333...pack
  multi-pack-index       ← MIDX: single index spanning all packs (modern Git)
```

### The `packed-refs` File

```
# cat .git/packed-refs
# pack-refs with: peeled fully-peeled sorted
3c4d5e6f7a8b9c0d1e2f3a4b5c6d7e8f9a0b1c2d refs/heads/develop
7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a refs/heads/main
d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3 refs/tags/v1.0.0
^9a0b1c2d3e4f5a6b7c8d9e0f1a2b3c4d5e6f7a8b  ← peeled: dereferenced tag object → commit

Parsing rules:
  ├── Lines starting with # are comments
  ├── Each non-comment line: "<sha> <refname>"
  ├── Lines starting with ^ are "peeled" entries
  │   └── The SHA after ^ is what the preceding tag tag-object points to
  └── File is sorted by refname for binary search
```

---

## RULE 3 — Exact Data Flow

### Data Flow 1: `git gc` — What Happens Step By Step

```
COMMAND: git gc

─────────────────────────────────────────────────────────────────────
PHASE 1: Expire reflog entries
─────────────────────────────────────────────────────────────────────

READS:    .git/logs/HEAD
          .git/logs/refs/heads/* (all branch reflogs)
          .git/logs/refs/stash  (stash reflog)

CHECKS:   For each reflog entry, is the timestamp older than:
          - gc.reflogExpire         (default: 90 days) for reachable refs
          - gc.reflogExpireUnreachable (default: 30 days) for unreachable

WRITES:   Removes expired entries from .git/logs/* files

WHY:      Expired reflog entries no longer pin their objects.
          Objects they pointed to become "unreachable" (eligible for pruning).

─────────────────────────────────────────────────────────────────────
PHASE 2: Pack loose objects
─────────────────────────────────────────────────────────────────────

READS:    All files in .git/objects/??/ (loose objects)

COMPUTES: 1. Sort all object SHAs
          2. Group similar objects for delta compression:
             - Objects of the same TYPE and similar SIZE are candidates
             - Heuristic: file names from tree objects (e.g., "src/app.js")
             - More recent versions = bases; older = deltas from newer
          3. For each delta candidate:
             - Try to express object B as: "take base A, apply these changes"
             - If delta(A→B) is smaller than raw B → use delta
          4. Zlib-compress the entire pack stream

WRITES:   .git/objects/pack/pack-{SHA}.pack  (new binary file)
          .git/objects/pack/pack-{SHA}.idx   (index for the pack)
          .git/objects/info/packs            (updated to list new pack)

DELETES:  .git/objects/??/{38-char-file}     (all loose objects now in pack)

─────────────────────────────────────────────────────────────────────
PHASE 3: Prune unreachable objects (if older than gc.pruneExpire, default 2 weeks)
─────────────────────────────────────────────────────────────────────

READS:    All refs: .git/refs/**, .git/packed-refs
          All reflogs: .git/logs/**
          All FETCH_HEAD, ORIG_HEAD, CHERRY_PICK_HEAD, etc.

COMPUTES: Reachability graph:
          Starting from every ref tip and reflog entry:
          - Follow commit → parent → parent → parent...
          - Follow commit → tree → subtree → blob
          - Mark every reachable object
          
          Result: a set of "live" SHAs
          
          Every SHA in the pack that is NOT in "live" AND is older than
          gc.pruneExpire is marked for deletion.

WRITES:   New pack with unreachable objects removed (if --prune=now)
          OR defers deletion (default: objects must be > 2 weeks old)

─────────────────────────────────────────────────────────────────────
PHASE 4: Pack existing refs into packed-refs
─────────────────────────────────────────────────────────────────────

READS:    .git/refs/heads/*, .git/refs/tags/*, etc.

WRITES:   .git/packed-refs (consolidated file)
DELETES:  Individual ref files from .git/refs/heads/ and .git/refs/tags/
          (they've been merged into packed-refs)

─────────────────────────────────────────────────────────────────────
BEFORE/AFTER STATE:
─────────────────────────────────────────────────────────────────────

BEFORE:
.git/objects/
  1a/2b3c...   ← 847 bytes (loose blob)
  3c/4d5e...   ← 234 bytes (loose blob)
  7f/8a9b...   ← 1,203 bytes (loose commit)
  ... (1,847 loose object files)
Total: ~8.4 MB in ~1,847 files

AFTER:
.git/objects/pack/
  pack-abc123.pack   ← 1.2 MB (all objects, delta-compressed)
  pack-abc123.idx    ← 94 KB (lookup table)
Total: ~1.3 MB in 2 files
```

### Data Flow 2: How Git Reads an Object from a Packfile

```
COMMAND: git cat-file -p abc123def456...

─────────────────────────────────────────────────────────────────────
STEP 1: Check loose objects first
        READS: .git/objects/ab/c123def456...
        Result: File not found → move to step 2

STEP 2: Check packed-refs is not relevant (that's for refs, not objects)

STEP 3: Open the pack index (.idx file)
        READS: .git/objects/pack/pack-{SHA}.idx

STEP 4: Binary search in the index
        The .idx file has a "fan-out table" (256 entries):
        fan_out[0xAB] = number of objects whose SHA starts with 0x00..0xAA
        fan_out[0xAC] = number of objects whose SHA starts with 0x00..0xAB
        
        So fan_out[0xAB+1] - fan_out[0xAB] = count of objects starting with 0xAB
        Binary search within that range for the full SHA "abc123..."
        
        Found at index position N → look up the 4-byte offset at position N
        in the offset table: offset = 0x004F2A3B (= byte 5,189,179 in the .pack)

STEP 5: Seek to the offset in the .pack file
        READS: .git/objects/pack/pack-{SHA}.pack at byte 5,189,179

STEP 6: Read the object header at that position
        First byte: 0b10110011
          Bits 6-4: type = 0b101 = 5 = OBJ_BLOB
          Bit 7:    MSB = 1 → more size bytes follow
          Bits 3-0: first 4 bits of size

        Continue reading variable-length size encoding...
        Result: type=blob, uncompressed_size=4,821 bytes

STEP 7: Is this a base object or a delta?
        Type 6 (OBJ_OFS_DELTA) or Type 7 (OBJ_REF_DELTA) → it's a delta
        Type 1-4 (commit/tree/blob/tag) → it's a base object

        IF BASE OBJECT:
          Zlib-decompress the data stream → raw object bytes
          Done.

        IF DELTA (OBJ_OFS_DELTA):
          Read the negative offset: base is at (current_offset - delta_offset)
          Recursively read the base object at that position
          Apply the delta instructions to the base → reconstructed object
          (May chain through multiple deltas: D1 ← D2 ← D3 ← base)

STEP 8: Return the decompressed object data
─────────────────────────────────────────────────────────────────────
```

### Data Flow 3: `git gc --auto` — The Trigger Logic

```
COMMAND: git gc --auto
(This is called automatically by: git commit, git fetch, git merge, etc.)

STEP 1: Count loose objects
        ls .git/objects/?? | wc -l  (approximate)
        
        If count < gc.auto (default: 6700) → EXIT immediately
        "gc.auto" is actually the loose object threshold in hundreds:
        gc.auto=6700 means: trigger when ~6700 loose objects exist

STEP 2: Count existing pack files
        ls .git/objects/pack/*.pack | wc -l
        
        If count < gc.autoPackLimit (default: 50) → EXIT
        "gc.autoPackLimit=50" means: trigger when >50 pack files exist

STEP 3: If EITHER threshold exceeded → run git gc in background
        (Uses background process so the triggering command doesn't block)
        
        Output: "Auto packing the repository for optimum performance."

STEP 4: Runs full gc:
        git pack-refs --all --prune
        git reflog expire --expire=90d --expire-unreachable=30d --all
        git repack -d -l
        git prune --expire=2.weeks.ago
        git worktree prune --expire=3.months
        git rerere gc
```

---

## RULE 4 — ASCII Architecture Diagrams

### Diagram 1: Loose Objects vs Packfile

```
LOOSE OBJECTS (before gc):
────────────────────────────────────────────────────────────────────

.git/objects/
│
├── 1a/2bcd...    "blob 12\0hello world\n"    (zlib compressed)
│                  14 bytes on disk
├── 3e/4f5a...    "blob 15\0hello world v2\n"  (zlib compressed)
│                  17 bytes on disk
├── 7b/8c9d...    "blob 14\0hello world v3\n"  (zlib compressed)
│                  16 bytes on disk
└── ...            (thousands more files)

Problems:
- One inode per object (filesystem overhead)
- No delta compression between versions
- Slow git fetch/clone (thousands of HTTP requests or pack-objects calls)

────────────────────────────────────────────────────────────────────

PACKFILE (after gc):
────────────────────────────────────────────────────────────────────

pack-abc123.pack (one binary file):
┌─────────────────────────────────────────────────────────┐
│ HEADER: "PACK" magic + version(2) + object_count(3)     │  12 bytes
├─────────────────────────────────────────────────────────┤
│ OBJECT 1: blob "hello world v3" (most recent = BASE)    │  10 bytes
│   type=blob, size=15, data=zlib("hello world v3\n")     │
├─────────────────────────────────────────────────────────┤
│ OBJECT 2: OFS_DELTA from object 1                       │   6 bytes
│   base_offset=10, delta=["replace 'v3' with 'v2'"]      │  (saved 11 bytes!)
├─────────────────────────────────────────────────────────┤
│ OBJECT 3: OFS_DELTA from object 1                       │   5 bytes
│   base_offset=10, delta=["replace 'v3' with ''"]        │  (saved 12 bytes!)
├─────────────────────────────────────────────────────────┤
│ ...more objects...                                      │
├─────────────────────────────────────────────────────────┤
│ TRAILER: SHA-1 of everything above                      │  20 bytes
└─────────────────────────────────────────────────────────┘

pack-abc123.idx (lookup table):
┌─────────────────────────────────────────────────────────┐
│ MAGIC: \377tOc + version(2)                             │   8 bytes
├─────────────────────────────────────────────────────────┤
│ FAN-OUT TABLE: 256 × 4-byte entries                     │  1024 bytes
│   fan_out[N] = count of objects with SHA[0] <= N        │
│   (cumulative count for binary search range narrowing)  │
├─────────────────────────────────────────────────────────┤
│ SHA TABLE: N × 20 bytes (sorted list of all SHA-1s)     │  N×20 bytes
├─────────────────────────────────────────────────────────┤
│ CRC32 TABLE: N × 4 bytes (CRC of compressed obj data)   │  N×4  bytes
├─────────────────────────────────────────────────────────┤
│ OFFSET TABLE: N × 4 bytes (byte offset in .pack file)   │  N×4  bytes
│   If MSB is 1, it's an index into the large-offset table│
├─────────────────────────────────────────────────────────┤
│ LARGE OFFSET TABLE: M × 8 bytes (for packs > 2 GB)     │  M×8  bytes
├─────────────────────────────────────────────────────────┤
│ TRAILER: SHA-1(pack file) + SHA-1(index file)           │  40 bytes
└─────────────────────────────────────────────────────────┘
```

### Diagram 2: Delta Chain

```
Git stores objects as DELTAS FROM A BASE (like a diff, but binary):

                 ┌──────────────────────────────────┐
                 │  BASE OBJECT (full content)       │
                 │  blob: "function login() {       │
                 │    return auth.check(user);      │
                 │  }"                               │
                 │  (stored as full zlib data)       │
                 └──────────────────┬───────────────┘
                                    │  "to reconstruct v2,
                                    │   insert 2 lines after line 1"
                 ┌──────────────────▼───────────────┐
                 │  DELTA 1 (OFS_DELTA)              │
                 │  base_offset: -147 bytes          │
                 │  instructions:                    │
                 │    copy(0, 25)        ← keep line 1 │
                 │    insert("  // TODO: rate limit")│
                 │    copy(25, 40)       ← keep rest │
                 └──────────────────┬───────────────┘
                                    │
                 ┌──────────────────▼───────────────┐
                 │  DELTA 2 (OFS_DELTA)              │
                 │  base_offset: -89 bytes           │
                 │  instructions:                    │
                 │    copy(0, 65)       ← keep all  │
                 │    insert("  logger.info(...)")   │
                 └──────────────────────────────────┘

TO READ DELTA 2:
  1. Read Delta 2 header → "base is 89 bytes before me in the pack"
  2. Seek back 89 bytes → read Delta 1 (also a delta!)
  3. Delta 1 says "base is 147 bytes before it"
  4. Seek back 147 bytes → read Base Object (raw content)
  5. Apply Delta 1 to Base → intermediate content
  6. Apply Delta 2 to intermediate → final content

MAX DELTA DEPTH: git limits chains to ~50 levels (pack.depth config)
Deeper chains = smaller pack, slower read. Git tunes this automatically.

TWO DELTA TYPES:
  OBJ_OFS_DELTA (type 6): base is at (current_offset - stored_offset)
  └─ base must be in the SAME pack file
  └─ more compact (4-byte negative offset)

  OBJ_REF_DELTA (type 7): base is identified by full 20-byte SHA
  └─ base can be in a DIFFERENT pack or loose
  └─ used when packs are built without full history (e.g., partial clones)
```

### Diagram 3: Reachability and Pruning

```
GIT OBJECT GRAPH (before gc):

       HEAD
        │
        ▼
  ┌──────────┐     ┌──────────┐
  │ commit D │────►│ commit C │
  │ (main)   │     │          │─────────────────────────────┐
  └──────┬───┘     └──────┬───┘                             │
         │ tree            │ tree                            │
         ▼                 ▼                                 │
  ┌──────────┐     ┌──────────┐                             │
  │  tree D  │     │  tree C  │                             │
  └──────┬───┘     └──────┬───┘                             │
         │ blob            │ blob                            ▼
         ▼                 ▼                        ┌──────────────────┐
  ┌──────────┐     ┌──────────┐                     │  ORPHAN COMMIT   │
  │ blob D   │     │ blob C   │                     │ (was on deleted  │
  └──────────┘     └──────────┘                     │  branch 'exp')   │
                                                     └──────────┬───────┘
  ┌──────────┐    ◄── reflog entry for               │ tree     │
  │ commit B │        HEAD@{3} (14 days ago)         ▼          │
  │(expired  │                               ┌──────────┐       │
  │ reflog)  │                               │  tree X  │       │
  └──────────┘                               └──────────┘       │
                                                                 ▼
                                                        ┌──────────────┐
                                                        │   blob X     │
                                                        └──────────────┘

REACHABILITY ANALYSIS:
  Starting from: HEAD, refs/heads/main, refs/tags/*, FETCH_HEAD, ORIG_HEAD
  AND: all non-expired reflog entries

  Reachable (KEEP):     commit D, commit C, tree D, tree C, blob D, blob C
  Pinned by reflog:     commit B (reflog entry still within 90 days) → KEEP
  Unreachable + old:    Orphan commit, tree X, blob X → PRUNE

AFTER git prune (or git gc):
  Objects for orphan commit, tree X, blob X are DELETED from disk.
  Once deleted, they CANNOT be recovered (unless you have the SHA
  written down somewhere and it hasn't been GC'd yet).
```

### Diagram 4: Bitmap Files

```
BITMAP FILE (.bitmap) — for fast reachability queries during clone/fetch:

WITHOUT BITMAP:
  git clone repo → server must walk entire commit graph
  For 100,000 commits, touching 10 commits/second = 2.8 hours
  (It's actually faster, but the graph walk is the bottleneck for large repos)

WITH BITMAP:
  Server pre-computes: "which objects are reachable from main@tip?"
  Stores as a compressed bitset (one bit per object in the pack):
    bit N = 1 → object N is reachable from this commit

  ┌────────────────────────────────────────────────────────────┐
  │  BITMAP FILE STRUCTURE                                     │
  ├────────────────────────────────────────────────────────────┤
  │  Magic: "BITM" + version                                   │
  │  Flags: has_hash_cache, has_name_hash_cache               │
  │  Nr. commits: 32-bit count of bitmap entries              │
  │  ─────────────────────────────────────────────────────   │
  │  For each "selected" commit (typically branch tips):       │
  │    commit SHA (20 bytes)                                   │
  │    XOR offset: link to "parent bitmap" (incremental!)      │
  │    bitmap data: EWAH-compressed bitset                     │
  │      bit 0: is object[0] in the pack reachable?            │
  │      bit 1: is object[1] in the pack reachable?            │
  │      bit N: is object[N] in the pack reachable?            │
  └────────────────────────────────────────────────────────────┘

CLONE WITH BITMAP:
  Client wants: everything reachable from main
  Server: loads bitmap for main@tip → bitset of reachable objects
  Server: streams only those objects → no graph traversal needed
  
  Result: clone of linux kernel repo: 90 seconds → 12 seconds

GENERATE BITMAPS:
  git repack -a -d --write-bitmap-index
  OR
  git config pack.writeBitmaps true  (automatic on git gc)
```

### Diagram 5: Multi-Pack Index (MIDX)

```
PROBLEM: After many incremental fetches, you have dozens of pack files:
  pack-aaa.pack  pack-bbb.pack  pack-ccc.pack  ... pack-zzz.pack

Finding an object requires searching each .idx file separately.

SOLUTION: Multi-Pack Index (MIDX)
  .git/objects/pack/multi-pack-index

┌────────────────────────────────────────────────────────────┐
│  MULTI-PACK INDEX                                          │
├────────────────────────────────────────────────────────────┤
│  Header: magic + version + hash_version + chunk_count      │
│  ─────────────────────────────────────────────────────    │
│  PACK NAMES CHUNK: list of all .pack filenames            │
│    "pack-aaa111.pack\0pack-bbb222.pack\0..."               │
│  ─────────────────────────────────────────────────────    │
│  OID FANOUT CHUNK: 256 entries (like regular .idx)        │
│  ─────────────────────────────────────────────────────    │
│  OID LOOKUP CHUNK: sorted SHA list (across ALL packs)     │
│  ─────────────────────────────────────────────────────    │
│  OBJECT OFFSETS CHUNK: for each SHA:                      │
│    - which pack file it's in (index into PACK NAMES)      │
│    - byte offset within that pack                         │
└────────────────────────────────────────────────────────────┘

BENEFIT: One binary search across all packs, instead of N searches.
GENERATE: git multi-pack-index write
          git maintenance run --task=incremental-repack
```

---

## RULE 5 — Hands-On Proof Commands

### See Loose Objects and Pack Files

```bash
# Count loose objects
find .git/objects -type f | grep -v '/pack/' | grep -v '/info/' | wc -l

# See the size of all loose objects on disk
du -sh .git/objects/ --exclude=.git/objects/pack

# List pack files
ls -lh .git/objects/pack/

# Count objects IN a pack file
git verify-pack -v .git/objects/pack/pack-*.idx | tail -1
# Output: non delta: 1234 objects
#         chain length = 1: 567 objects
#         chain length = 2: 89 objects

# See the top 10 biggest objects (by uncompressed size) in a pack
git verify-pack -v .git/objects/pack/pack-*.idx \
  | sort -k 3 -n -r \
  | head -10
# Output: SHA type size offset [depth base-SHA]
```

### Inspect Pack File Contents

```bash
# Show ALL objects in the pack with their details
git verify-pack -v .git/objects/pack/pack-*.idx
# Columns: SHA-1, type, uncompressed-size, pack-size, offset, [depth, base-SHA]

# Find the 10 largest objects by uncompressed size
git verify-pack -v .git/objects/pack/pack-*.idx \
  | grep -v chain \
  | sort -k3 -rn \
  | head -10

# For each large object, find which file it corresponds to
# (Git doesn't store filenames in pack — we trace through tree objects)
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | sort -k3 -rn \
  | head -20
```

### Run GC Manually and Measure Impact

```bash
# BEFORE: measure repo size
du -sh .git/

# See what git gc would do without doing it
git gc --dry-run

# Run gc with verbose output
git gc --verbose
# Output: Counting objects: 1,847, done.
#         Delta compression using up to 8 threads.
#         Compressing objects: 100% (1,623/1,623), done.
#         Writing objects: 100% (1,847/1,847), done.
#         Total 1,847 (delta 891), reused 0 (delta 0)
#         Removing duplicate objects: 100% (3/3), done.

# AFTER: measure repo size
du -sh .git/
# Typically 60-90% smaller for mature repos

# Aggressive GC (slower, better compression)
git gc --aggressive
# Uses more CPU, tries harder delta compression
# Recommended: run monthly on large repos, rarely on small ones
```

### Inspect the Pack Index Binary Format

```bash
# Hex dump the first 32 bytes of the .idx file (header + fan-out start)
xxd .git/objects/pack/pack-*.idx | head -4
# 00000000: ff74 4f63 0000 0002 0000 0000 0000 0000  .tOc............
# ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
# ff 74 4f 63 = magic bytes "\377tOc" (version 2 pack index)
# 00 00 00 02 = version 2
# Then: fan-out table starts (256 × 4-byte entries)

# Get the total object count (last entry of fan-out table = total objects)
# Fan-out table: bytes 8 to 8+(256*4)=1032
# Last 4-byte entry at offset 1028:
python3 -c "
import struct
with open('.git/objects/pack/' + __import__('os').listdir('.git/objects/pack/')[0], 'rb') as f:
    f.seek(1028 + 8)  # skip magic(4) + version(4) + 255 fan-out entries
    total = struct.unpack('>I', f.read(4))[0]
    print(f'Total objects in pack: {total}')
"
```

### Check Reachability and Prune

```bash
# Find objects that are NOT reachable from any ref or reflog
git fsck --unreachable
# Output: unreachable blob abc123...
#         unreachable commit def456...

# Show dangling objects (not pointed to by anything, but not chained from others)
git fsck --dangling

# Prune objects older than 2 weeks
git prune --expire=2.weeks.ago --verbose
# Output: Removing unreachable: blob abc123...

# Prune ALL unreachable objects immediately (DANGEROUS: no 2-week grace)
git prune --expire=now
# WARNING: Only do this if you're sure no concurrent clone/fetch is happening.
# A concurrent clone reads objects that aren't yet in the pack; pruning them
# mid-clone would corrupt the clone.

# Check integrity of all packed objects
git verify-pack -v .git/objects/pack/pack-*.idx | grep -v "^$" | wc -l
```

### Repack with Bitmaps

```bash
# Full repack: consolidate all packs into one, generate bitmaps
git repack -a -d --write-bitmap-index
# -a: pack all reachable objects
# -d: delete redundant packs after repackaging
# --write-bitmap-index: generate .bitmap file for fast clone/fetch

# Verify bitmap was created
ls -lh .git/objects/pack/*.bitmap

# Incremental repack (only new objects since last full repack)
git repack -d
# Packs only loose objects into a new pack (doesn't consolidate existing packs)
```

### Use `git maintenance` (Modern Replacement for ad-hoc gc)

```bash
# Start background maintenance (runs tasks on a schedule)
git maintenance start
# Creates OS-specific cron jobs / systemd timers / launchd plists

# Run all maintenance tasks right now
git maintenance run --auto

# Run specific tasks:
git maintenance run --task=gc              # full gc
git maintenance run --task=commit-graph    # update commit-graph file
git maintenance run --task=fetch           # background fetch from remotes
git maintenance run --task=loose-objects   # pack loose objects
git maintenance run --task=incremental-repack  # repack without full gc
git maintenance run --task=pack-refs       # pack loose refs into packed-refs

# Stop background maintenance
git maintenance stop

# See registered repos for background maintenance
git maintenance list
```

---

## RULE 6 — Syntax Breakdown (Deep)

### `git gc`

```
git gc [--aggressive] [--auto] [--prune=<date>] [--no-prune] [--quiet] [--verbose]
│   │   │              │        │                 │             │          │
│   │   │              │        │                 │             │          └─ print progress and stats
│   │   │              │        │                 │             └─ suppress all output
│   │   │              │        │                 └─ skip pruning (just pack)
│   │   │              │        └─ prune objects older than <date>
│   │   │              │           default: "2.weeks.ago"
│   │   │              │           "now" = prune everything unreachable immediately
│   │   │              └─ only run if loose object count or pack count exceeds threshold
│   │   │                 (same thresholds as automatic trigger)
│   │   └─ slower but better compression:
│   │      - uses --depth=250 instead of default 50 for delta chains
│   │      - uses --window=250 instead of default 10 for delta candidates
│   │      WARNING: can be 10-100x slower; don't run in CI
│   └─ the "garbage collect" subcommand
└─ git binary
```

### `git repack`

```
git repack [-a] [-A] [-d] [-f] [-F] [-l] [-n] [-q] [--depth=<n>] [--window=<n>]
│   │       │    │    │    │    │    │    │    │    │               │
│   │       │    │    │    │    │    │    │    │    │               └─ window for delta search
│   │       │    │    │    │    │    │    │    │    │                  (how many objects to compare against)
│   │       │    │    │    │    │    │    │    │    └─ max delta chain depth
│   │       │    │    │    │    │    │    │    └─ quiet
│   │       │    │    │    │    │    │    └─ do not update server info (for dumb HTTP servers)
│   │       │    │    │    │    │    └─ local only (don't pack objects from alternates)
│   │       │    │    │    │    └─ force writing all deltas even if existing ones are fine
│   │       │    │    │    └─ force delta recomputation (--force)
│   │       │    │    └─ delete redundant packs after creating new one
│   │       │    └─ like -a but unreachable objects → kept as loose (not packed)
│   │       │       prevents "git prune" from deleting recently-orphaned objects
│   │       └─ pack ALL reachable objects into a single pack
│   │          without -a: only packs loose objects (doesn't consolidate existing packs)
│   └─ the "repack" subcommand
└─ git binary

COMMON COMBINATIONS:
  git repack -d          ← just pack loose objects, delete redundant packs
  git repack -a -d       ← full repack into one pack, delete old packs
  git repack -a -d --write-bitmap-index   ← full repack + bitmap for fast clone
  git repack -a -d -f --depth=250 --window=250  ← maximum compression (same as --aggressive)
```

### `git prune`

```
git prune [--dry-run] [--verbose] [--expire=<date>] [--] [<head>...]
│   │      │            │            │                 │    │
│   │      │            │            │                 │    └─ additional tips to treat as reachable
│   │      │            │            │                 └─ end of options
│   │      │            │            └─ only prune objects older than <date>
│   │      │            │               (default from gc.pruneExpire, typically "2.weeks.ago")
│   │      │            └─ show which objects are being removed
│   │      └─ show what would be deleted without deleting
│   └─ the "prune" subcommand
└─ git binary

WARNING: Never run git prune while a clone or fetch is in progress.
The 2-week grace period exists specifically to prevent this race condition.
```

### `git pack-refs`

```
git pack-refs [--all] [--no-prune] [--auto]
│   │          │        │             │
│   │          │        │             └─ only pack if beneficial
│   │          │        └─ pack refs but don't delete the loose ref files
│   │          └─ pack ALL refs (tags + branches + remote-tracking)
│   │             without --all: only packs tags and already-packed branches
│   └─ the "pack-refs" subcommand
└─ git binary
```

### Key Configuration Variables

```bash
# gc.auto
#   Number of loose objects before auto-gc triggers
#   0 = disable auto-gc entirely
git config gc.auto 6700              # default

# gc.autoPackLimit
#   Number of pack files before auto-gc triggers (to consolidate)
git config gc.autoPackLimit 50       # default

# gc.reflogExpire
#   How long to keep reflog entries for reachable refs
git config gc.reflogExpire 90        # "90 days" (default)
git config gc.reflogExpire never     # never expire (useful for important repos)

# gc.reflogExpireUnreachable
#   How long to keep reflog entries for unreachable refs
git config gc.reflogExpireUnreachable 30   # "30 days" (default)

# gc.pruneExpire
#   Grace period for unreachable objects before deletion
git config gc.pruneExpire "2.weeks.ago"   # default
git config gc.pruneExpire never            # never auto-prune

# pack.depth
#   Maximum delta chain depth (default: 50, --aggressive: 250)
git config pack.depth 50

# pack.window
#   Delta compression window (how many candidates to consider per object)
git config pack.window 10            # default (--aggressive uses 250)

# pack.writeBitmaps / pack.writeBitmapIndex
#   Auto-generate bitmap on gc/repack
git config pack.writeBitmaps true    # recommended for servers

# pack.threads
#   Number of threads for delta compression (0 = auto = CPU count)
git config pack.threads 0

# gc.bigPackThreshold
#   Packs larger than this are not repacked during incremental repack
git config gc.bigPackThreshold "2g"
```

---

## RULE 7 — Beginner → Production Example

### Beginner: Watch Your Repo Shrink After GC

```bash
# Create a repo with some history
mkdir gc-demo && cd gc-demo
git init

# Create a large-ish file and make many changes to it
python3 -c "
import os
os.system('git init')
for i in range(100):
    with open('bigfile.txt', 'w') as f:
        f.write('line ' * 1000 + f'version {i}\n' * 100)
    os.system(f'git add bigfile.txt && git commit -m \"version {i}\"')
"

# Check size BEFORE gc
du -sh .git/objects/
# Output: ~8.5M  .git/objects/

# Count loose objects
find .git/objects -type f | grep -v pack | grep -v info | wc -l
# Output: ~300 (commits + trees + blobs for 100 versions)

# Run gc
git gc --verbose

# Check size AFTER gc
du -sh .git/objects/
# Output: ~180K  .git/objects/  ← 97% smaller!

# The magic: bigfile.txt is nearly identical across versions
# Delta compression stores only the diffs
git verify-pack -v .git/objects/pack/pack-*.idx \
  | sort -k3 -rn | head -5
# You'll see the base blob is large, but all deltas are tiny
```

---

### Production: Managing a Large Monorepo (50 GB `.git/`)

```
SCENARIO: You're at a company with a 5-year-old monorepo.
  - 180,000 commits
  - 50 engineers active daily  
  - .git/ is 50 GB
  - git clone takes 35 minutes
  - git fetch takes 4 minutes
  - Engineers are complaining that git is slow

DIAGNOSIS:
```

```bash
# Step 1: How big is the object database?
du -sh .git/objects/

# Step 2: How many pack files do we have?
ls .git/objects/pack/*.pack | wc -l
# Output: 847  ← terrible! 847 incremental packs from 5 years of fetches

# Step 3: What are the biggest objects?
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1 == "blob" { print $3, $4 }' \
  | sort -rn \
  | head -20
# Output: 
# 104857600 assets/marketing/video-2019.mp4   ← 100 MB
# 52428800  assets/old-logo-source.psd        ← 50 MB
# 31457280  database/dump-2020-01-01.sql.gz   ← 30 MB
# ...

# Step 4: Is a bitmap file present?
ls .git/objects/pack/*.bitmap
# Output: ls: no matches  ← no bitmap! This is why clone is slow
```

```
ROOT CAUSES identified:
1. 847 pack files → MIDX and bitmap absent → every clone walks entire graph
2. Large binary files (video, PSDs, DB dumps) committed directly → inflates repo
3. No scheduled maintenance → packs have never been consolidated
```

```bash
# IMMEDIATE FIX 1: Full repack with bitmap
# (Run during off-hours; this will take 30-60 minutes for a 50 GB repo)
git repack -a -d --write-bitmap-index --threads=16
# -a: consolidate all 847 packs into 1
# -d: delete the 847 old packs
# --write-bitmap-index: generate bitmap for fast clone
# --threads=16: use all server CPUs

# VERIFY: Check we now have 1 pack + 1 bitmap
ls .git/objects/pack/
# pack-abc123.pack  pack-abc123.idx  pack-abc123.bitmap

# RESULT: Clone time drops from 35 min → 4 min (9x faster)
```

```bash
# IMMEDIATE FIX 2: Set up git maintenance for ongoing health
git maintenance start
# Registers cron: runs incremental-repack nightly, loose-objects hourly

# Tune for this large repo:
git config maintenance.loose-objects.auto 100     # pack loose objects more aggressively
git config maintenance.incremental-repack.auto 5  # repack when >5 new packs exist
git config gc.bigPackThreshold 2g                 # don't repack the big base pack
git config pack.writeBitmaps true
```

```bash
# LONG-TERM FIX 3: Remove the big binary files from history
# (Coordinate with team; requires force push; see Topic 26 for git filter-repo)
git filter-repo --path assets/marketing/video-2019.mp4 --invert-paths
git filter-repo --path assets/old-logo-source.psd --invert-paths
git filter-repo --path database/dump-2020-01-01.sql.gz --invert-paths

# Then add Git LFS for future large files:
git lfs install
git lfs track "*.mp4" "*.psd" "*.sql.gz"
git add .gitattributes
git commit -m "chore: configure Git LFS for large binary files"

# Full repack after history rewrite
git reflog expire --expire=now --all
git gc --prune=now --aggressive
# .git/ size: 50 GB → 3 GB (after removing the large files from history)
```

```bash
# MONITORING: Add to your CI healthcheck pipeline
# Run weekly, alert if any of these thresholds are exceeded:

LOOSE=$(find .git/objects -type f | grep -v pack | grep -v info | wc -l)
PACKS=$(ls .git/objects/pack/*.pack 2>/dev/null | wc -l)
GIT_SIZE=$(du -sb .git/objects | cut -f1)

echo "Loose objects: $LOOSE (alert if >1000)"
echo "Pack files:    $PACKS (alert if >20)"
echo "Objects size:  $(($GIT_SIZE / 1024 / 1024)) MB (alert if >10000)"

[ $LOOSE -gt 1000 ] && echo "WARNING: Too many loose objects, run git gc"
[ $PACKS -gt 20   ] && echo "WARNING: Too many pack files, run git repack -a -d"
```

---

## RULE 8 — Mistakes → Root Cause → Fix → Prevention

### MISTAKE 1: Running `git prune` While a Clone Is In Progress

**The mistake:**
```bash
# On the server, a clone is currently running (in another terminal):
# git clone https://github.com/org/repo.git  ← clone in progress

# Meanwhile, you (admin) run:
git prune --expire=now
```

**Root Cause:** When `git clone` starts, the server sends objects to the client. 
The client is receiving objects and storing them as loose objects. These loose objects 
are NOT yet reachable from any ref (the refs haven't been updated yet — that happens at 
the END of the clone). If you prune while this is happening:

1. Objects the clone is currently receiving are not yet in any pack
2. Prune sees them as "unreachable" and deletes them
3. The clone fails with: `error: Server does not allow request for unadvertised object`

**Root Cause (deeper):** This is why the default `gc.pruneExpire` is "2.weeks.ago" — 
it's a grace period so that objects created by in-progress operations are never pruned.
The 2-week window ensures that any object that has been "orphaned" for less than 2 weeks
is kept, protecting in-progress clones/fetches that can run for hours on slow connections.

**Fix:**
```bash
# If a clone is currently corrupted, restart it:
rm -rf /path/to/partial-clone
git clone ... /path/to/partial-clone  # start fresh

# Check for running clone/fetch processes before pruning:
ps aux | grep git
lsof | grep .git/objects  # see if any process has objects files open

# Always use the grace period (never --expire=now on a shared server):
git prune --expire=2.weeks.ago   # safe default
```

**Prevention:**
- Never use `git prune --expire=now` on a server that accepts clones/pushes
- Use `git gc` (which uses a 2-week grace) rather than calling `git prune` directly
- For automated cleanup: configure `gc.pruneExpire=2.weeks.ago` in server git config

---

### MISTAKE 2: `git gc --aggressive` in CI

**The mistake:**
```yaml
# .github/workflows/maintenance.yml
- name: Clean up repo
  run: git gc --aggressive --prune=now
```

**Root Cause:** `git gc --aggressive` uses `--window=250 --depth=250`, which means:
- For each object, compare it against 250 other objects to find the best delta
- Allow delta chains 250 levels deep
- On a repo with 50,000 objects: 50,000 × 250 comparisons = 12.5 million operations
- Runtime: can take hours and use multiple GB of RAM
- CPU: 100% for the entire duration

Additionally: `--prune=now` removes the 2-week grace period. Any object not reachable
from a ref — including objects that are being fetched/pushed right at that moment —
gets permanently deleted.

**Fix:**
```yaml
# For CI: only pack loose objects, no aggressive compression
- name: Light maintenance
  run: git gc --auto   # only runs if thresholds exceeded; safe and fast

# For scheduled server maintenance (off-hours, dedicated maintenance window):
- name: Full maintenance (scheduled, not CI)
  run: |
    git maintenance run --task=loose-objects
    git maintenance run --task=incremental-repack
  # This is safer and more controlled than git gc --aggressive
```

**Prevention:**
- Reserve `git gc --aggressive` for: one-time history cleanup, after `git filter-repo`, on local dev machines
- Use `git maintenance` with scheduled tasks for servers
- Set `gc.auto=0` in repos that use `git maintenance` (to avoid double-runs)

---

### MISTAKE 3: Assuming `git gc` Is the Only Way Objects Get Deleted

**The mistake:**
```
"I'll create this branch, do some experimental commits, delete the branch,
then run git gc to clean up. No problem."
```

Wait 30 days and check if the objects are gone. They're still there. WHY?

**Root Cause:** After you delete the branch, the commits become unreachable from refs —
BUT they're still referenced by reflog entries in `.git/logs/refs/heads/experiment`.
The reflog pins those objects for `gc.reflogExpireUnreachable` days (default: 30 days).

The full deletion sequence is:
1. Delete branch → commits unreachable from refs ✓
2. Wait 30 days → reflog entries for that branch expire ✓
3. Run `git gc` → prune checks that objects are also older than `gc.pruneExpire` (2 weeks) ✓
4. Only NOW are the objects deleted

Effectively: objects survive for **max(reflogExpireUnreachable, pruneExpire) + time-until-gc**.

**Fix:**
```bash
# To immediately delete: expire the reflog first, then prune
git reflog expire --expire=now --expire-unreachable=now --all
git gc --prune=now

# WARNING: After this, git reflog can no longer recover those commits
```

**Prevention:**
- Understand that `git reflog` (Topic 20) is the REASON objects aren't deleted immediately
- This is intentional — it's your safety net
- Only bypass it deliberately when you need to reclaim disk space

---

### MISTAKE 4: Committing Large Binary Files (The Blob That Couldn't Be Delta-Compressed)

**The mistake:**
```bash
git add assets/mockup-v1.psd    # 50 MB Photoshop file
git commit -m "add mockup"
# (make changes in Photoshop)
git add assets/mockup-v1.psd    # 50 MB, slightly different
git commit -m "update mockup"
# (repeat 20 times)
```

**Root Cause:** Delta compression works by diffing the *binary content* of objects.
For many binary formats (Photoshop PSD, SQLite, ZIP, compiled binaries), the entire
file changes even for tiny logical edits, because:
- PSD files have compressed internal layers
- SQLite has B-tree pages that get entirely rewritten for small data changes
- ZIP files have offsets embedded that change when anything moves

Result: Git can't find a good delta (the "delta" is nearly as big as the full file),
so it stores the full 50 MB blob for each commit. After 20 commits: 1 GB in `.git/`.
Delta compression gives 0% benefit for incompressible binary files.

**Diagnosis:**
```bash
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1 == "blob" && $3 > 10000000 { print $3/1000000 " MB", $4 }' \
  | sort -rn
# Output:
# 50.3 MB assets/mockup-v1.psd
# 50.1 MB assets/mockup-v1.psd   ← same filename, different version
# 49.8 MB assets/mockup-v1.psd
# ... (20 entries)
```

**Fix:**
```bash
# Remove the file from ALL history (see Topic 26 for full details)
git filter-repo --path assets/mockup-v1.psd --invert-paths

# Switch to Git LFS (see Topic 26)
git lfs install
git lfs track "*.psd"
# Now .psd files are stored as 200-byte pointer files in git,
# actual binary content in LFS object store (GitHub's CDN or your own)
```

**Prevention:**
- Add a pre-commit hook (Topic 19) to reject files > 10 MB
- Configure `.gitattributes` to mark binaries early: `*.psd binary`
- Use Git LFS for any binary assets from day one of the project

---

### MISTAKE 5: Forgetting That `git push` Can Re-Inflate a Pruned Repo

**The mistake:**
```
Alice: "Great, I ran git gc --aggressive --prune=now on the server.
        Repo went from 50 GB to 8 GB. Problem solved."

Two weeks later:
Alice: "The repo is 50 GB again?! What happened?"
```

**Root Cause:** `git gc` on the SERVER removes loose/unreachable objects. But when
developers `git push`, Git sends new loose objects (the delta between their local work
and what the server has). These arrive as loose objects on the server, waiting for the
next GC.

More importantly: if ANY developer still has a local clone containing the "removed"
large binary blobs and they do `git push --force` or `git push` with those objects
in their history, Git will re-upload them to the server.

GC removes objects from a SINGLE repository. It doesn't prevent them from coming back.

**Fix:**
```bash
# After aggressive cleanup of a large file from server history:
# 1. Notify ALL developers to re-clone (their local copies still have the blobs)
# 2. OR have them run:
git filter-repo --path assets/mockup-v1.psd --invert-paths
git remote set-url origin <same-url>
git push origin --force --all --tags

# 3. Set up branch protection (Topic 24) to require PR merges
#    so no one can accidentally push the large file back directly

# 4. Add a pre-receive hook on the server:
# .git/hooks/pre-receive (server-side):
# Check for blobs > 10 MB and reject the push
```

**Prevention:**
- Use Git LFS from the start
- Add a server-side `pre-receive` hook that rejects objects over a size limit
- Document the "no large files in git" policy and enforce it via hooks

---

## RULE 9 — Byte-Level Internals

### The Pack File Binary Format (v2)

```
PACK FILE FORMAT (version 2):
─────────────────────────────────────────────────────────────────────

HEADER (12 bytes):
  Bytes 0-3:   "PACK"               ← magic bytes: 0x50 0x41 0x43 0x4B
  Bytes 4-7:   version              ← big-endian uint32: 0x00 0x00 0x00 0x02
  Bytes 8-11:  object_count         ← big-endian uint32: number of objects

OBJECT ENTRIES (variable length, repeated object_count times):
  For each object:
  ┌───────────────────────────────────────────────────────────┐
  │ SIZE-TYPE HEADER (1 or more bytes, variable-length int)   │
  │                                                           │
  │ First byte:                                               │
  │   bit 7 (MSB): 1 = more bytes follow; 0 = last byte      │
  │   bits 6-4: OBJECT TYPE                                   │
  │     001 = OBJ_COMMIT                                      │
  │     010 = OBJ_TREE                                        │
  │     011 = OBJ_BLOB                                        │
  │     100 = OBJ_TAG                                         │
  │     110 = OBJ_OFS_DELTA  ← delta with negative offset    │
  │     111 = OBJ_REF_DELTA  ← delta with 20-byte base SHA   │
  │   bits 3-0: first 4 bits of UNCOMPRESSED SIZE             │
  │                                                           │
  │ Subsequent bytes (if MSB was 1):                          │
  │   bit 7: 1 = more bytes; 0 = last byte                   │
  │   bits 6-0: next 7 bits of size                           │
  │                                                           │
  │ SIZE ENCODING EXAMPLE: uncompressed size = 300 bytes      │
  │   300 = 0b100101100                                       │
  │   First byte:  type=3(blob), bits 3-0 = 1100 = 0xC       │
  │     With MSB=1 (more follows): 0b1_011_1100 = 0xBC       │
  │   Second byte: bits 6-0 = 0b0000010 = 2, MSB=0 (last)   │
  │     0b0_0000010 = 0x02                                    │
  │   Read: size = 0b0000010_1100 = 44 → WRONG               │
  │   Actual: 4-bit chunk from byte0, 7-bit chunks after:     │
  │   size = 1100 | (0000010 << 4) = 0b00000101100 = 44      │
  │   (The chunks are concatenated low-bits first)            │
  ├───────────────────────────────────────────────────────────┤
  │ IF OBJ_OFS_DELTA: negative offset (variable-length int)   │
  │   A variable-length integer encoding the distance back    │
  │   in the pack file to find the base object.               │
  │   base_offset = current_pos - decoded_neg_offset          │
  ├───────────────────────────────────────────────────────────┤
  │ IF OBJ_REF_DELTA: 20 bytes                                │
  │   The SHA-1 of the base object (can be in any pack/loose) │
  ├───────────────────────────────────────────────────────────┤
  │ COMPRESSED DATA: zlib-deflate compressed stream           │
  │   For base objects: the raw object content               │
  │   For deltas: delta instructions (see below)              │
  └───────────────────────────────────────────────────────────┘

TRAILER (20 bytes):
  SHA-1 of all bytes preceding the trailer
  (This is also the pack's "name" — matches the filename)
```

### The Delta Instruction Format

```
DELTA INSTRUCTIONS (applied to base object to produce result):

Header of delta data (after zlib decompression):
  source_length:  variable-length int (size of the base object)
  target_length:  variable-length int (size of the result object)

Instructions (repeated until target_length bytes produced):
  COPY instruction (bit 7 = 1):
  ┌──────────────────────────────────────────────────────┐
  │ First byte: 0b1_ooo_ssss                             │
  │   bit 7:    1 = this is a COPY instruction           │
  │   bits 6-4: offset encoding bits (which bytes follow)│
  │   bits 3-0: size encoding bits (which bytes follow)  │
  │                                                      │
  │ Followed by 0-4 bytes for offset, 0-3 bytes for size │
  │   offset: byte position in BASE to start copying from│
  │   size:   how many bytes to copy (0 = 65536 bytes)   │
  │                                                      │
  │ Meaning: "copy <size> bytes from BASE at <offset>"   │
  └──────────────────────────────────────────────────────┘

  INSERT instruction (bit 7 = 0):
  ┌──────────────────────────────────────────────────────┐
  │ First byte: 0b0_nnnnnnn                              │
  │   bit 7:    0 = this is an INSERT instruction        │
  │   bits 6-0: n = number of literal bytes to insert    │
  │                                                      │
  │ Followed by exactly n bytes of literal data          │
  │                                                      │
  │ Meaning: "insert these n literal bytes into result"  │
  └──────────────────────────────────────────────────────┘

EXAMPLE: Diff between "hello world v1" and "hello world v2"
  Base:   "hello world v1"  (14 bytes)
  Result: "hello world v2"  (14 bytes)

  Instructions:
  1. COPY: offset=0, size=13  → copy "hello world v" from base
  2. INSERT: n=1, data="2"    → insert literal "2"
  
  Delta size: ~5 bytes vs 14 bytes for the full content
  Savings: 64%
```

### The Commit-Graph File

```
.git/objects/info/commit-graph    ← introduced in Git 2.18

PROBLEM: git log, git merge-base, git push all need to traverse the commit graph.
For a repo with 100,000 commits, walking the graph means reading 100,000 files.

SOLUTION: commit-graph file pre-computes and caches:
  - parent SHA(s) for each commit (avoid reading the commit object)
  - tree SHA for each commit (fast tree access)
  - commit date (for date-ordering without reading objects)
  - generation number (topological level in the DAG)

STRUCTURE:
  Header: magic "CGPH" + version + hash_version + chunk_count
  Chunks:
    OID FANOUT:    256-entry fan-out (like .idx files)
    OID LOOKUP:    sorted SHA list of all commits
    COMMIT DATA:   per commit: tree_sha(20) + date(8) + parent_positions(4×)
    GENERATION:    topological generation number per commit
    BLOOM FILTERS: (optional) which files changed per commit → fast "git log -- path"

GENERATE:
  git commit-graph write --reachable
  git maintenance run --task=commit-graph   ← updates incrementally

BENEFIT:
  git log --oneline: 3.2 seconds → 0.4 seconds (on linux kernel repo)
  git merge-base:    1.1 seconds → 0.08 seconds
```

---

## RULE 10 — Mental Model Checkpoint

**1.** You have a repo with 10,000 loose objects. `git gc` runs and produces one `.pack` file. You then make 5 new commits. Where do the 5 new commits live?

**2.** What is the difference between `git gc` and `git prune`? When would you run `git prune` directly?

**3.** An object is unreachable from all refs but is less than 2 weeks old. Will `git gc` delete it? Why or why not?

**4.** What is a delta object of type `OBJ_OFS_DELTA`? What does the stored "offset" represent, and where does the base object have to be?

**5.** You run `git repack -a -d --write-bitmap-index`. Then a developer pushes 50 new commits. Is the bitmap still valid? Will it be used for the next `git clone`?

**6.** What is `packed-refs` and why does it exist? What triggers a loose ref to be moved into `packed-refs`?

**7.** A `.bitmap` file exists in `.git/objects/pack/`. How does this speed up `git clone`? What operation is eliminated?

**8.** You want to find out which file in git history is consuming the most disk space. What command sequence would you use?

**9.** After running `git filter-repo` to remove a large file, you check `.git/` and the file is still large. What step did you forget?

**10.** What is `gc.bigPackThreshold` and why would you set it on a large repo?

---

**Answers:**

1. The 5 new commits become **loose objects** — individual files in `.git/objects/??/`. They won't be packed until the next GC run (triggered automatically when the loose count exceeds `gc.auto`).

2. `git gc` is the all-in-one command: packs loose objects, expires reflogs, prunes unreachable objects, packs refs. `git prune` only does the deletion step (removes unreachable objects). You'd run `git prune` directly if you only want to clean up unreachable objects without repacking — rare; most users should use `git gc`.

3. **No.** The default `gc.pruneExpire` is "2.weeks.ago". Any unreachable object younger than 2 weeks is kept. This protects in-progress operations and provides a recovery window.

4. `OBJ_OFS_DELTA` is a delta object whose base is identified by a **negative byte offset** within the same pack file. The offset says "go backwards N bytes from my position to find my base." The base MUST be in the same pack file (unlike `OBJ_REF_DELTA` which uses a SHA and can span packs).

5. The bitmap is based on the state of the pack at the time it was generated. The 50 new commits are **loose objects** or in a new incremental pack. The bitmap is still used for what it covers (the existing pack), but the new commits won't be in the bitmap. Git will combine the bitmap lookup with a live graph walk for the new commits. The bitmap is regenerated on the next `git gc` or `git repack -a`.

6. `packed-refs` consolidates all branch and tag refs (normally one file per ref) into a single sorted text file. It exists for performance: repos with 10,000 tags would have 10,000 small files in `.git/refs/tags/`. Reading them all is slow. `packed-refs` makes ref lookup O(log n) via binary search. `git pack-refs --all` moves loose refs into it; `git gc` does this as one of its steps.

7. The bitmap eliminates the **commit graph walk** during clone. Normally the server must walk every commit to determine which objects to send. The bitmap is a pre-computed bitset: "which of the N objects in the pack are reachable from main@tip?" — the server just loads the bitmap and streams those objects directly.

8. 
```bash
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '$1 == "blob"' \
  | sort -k3 -rn \
  | head -20
```

9. After `git filter-repo`, the old objects still exist as **unreachable** objects (now pointed to by reflog entries). You must expire the reflogs and then prune:
```bash
git reflog expire --expire=now --all
git gc --prune=now
```

10. `gc.bigPackThreshold` (e.g., `2g`) tells git to skip repacking any pack file larger than 2 GB during incremental maintenance. Without it, every `git gc` would repack the huge base pack along with the small new packs, taking hours. With it, only small new packs are repacked; the big one is left alone. The new packs are consolidated into a "geometric series" of progressively larger packs, not one monolithic pack.

---

## RULE 11 — Connect Everything Backwards

**→ Topic 02 (Git Objects):** Packfiles are just a more efficient way to store the exact same four object types (blob, tree, commit, tag) introduced in Topic 02. The object content and SHA computation are identical — only the storage format changes. A blob in a packfile has the same SHA-1 as it would as a loose object.

**→ Topic 03 (The DAG):** The commit DAG from Topic 03 is what the reachability algorithm traverses during GC. An object is "reachable" if there's a path through the DAG from any ref tip to that object. Unreachable objects (not connected to the DAG from any ref) are candidates for deletion.

**→ Topic 20 (git reflog):** The reflog is the primary reason objects aren't deleted immediately after becoming unreachable. Each reflog entry holds a reference to a commit SHA; that SHA and all its descendants (via tree/blob pointers) are kept alive. The 90-day and 30-day expiry windows in `gc.reflogExpire` are the interface between reflog (Topic 20) and GC (Topic 25).

**→ Topic 10 (Remotes):** `git fetch` sends pack data as pack files over the wire — the network protocol is literally `git pack-objects | network | git index-pack` on the receiving side. When your server has bitmaps, `git clone` is faster because the server can compute "objects to send" in seconds using the bitmap instead of minutes by graph traversal.

**→ Topic 14 (Tags):** Tags in `packed-refs` have a special `^` line showing the "peeled" SHA — the commit the tag ultimately points to (through the tag object). This allows fast `git describe` and `git log --simplify-by-decoration` without having to read every tag object.

**→ Topic 22 (git blame, git log):** The commit-graph file (`.git/objects/info/commit-graph`) dramatically speeds up `git log` and blame operations because Git can traverse commit history without reading individual commit objects — parent SHAs, generation numbers, and dates are pre-cached in the commit-graph. `git log --oneline -n 1000` on the Linux kernel repo: 3.2s without commit-graph → 0.4s with it.

**→ Topic 26 (next topic — Performance & Large Repos):** Packfiles are the foundation of everything in Topic 26. Shallow clones (`--depth=N`) produce a special pack with "shallow" boundary markers. Sparse checkouts work with existing packs but check out fewer tree objects. Git LFS replaces large blobs in packs with tiny pointer files. All of these optimize around the fundamental packfile model explained in this topic.

---

## RULE 12 — Quick Reference Card

### Commands

| Command | What It Does |
|---|---|
| `git gc` | All-in-one: pack loose objects, expire reflogs, prune unreachable, pack refs |
| `git gc --auto` | Run gc only if thresholds exceeded (safe to call frequently) |
| `git gc --aggressive` | Maximum compression (slow; use offline only) |
| `git gc --prune=now` | Skip 2-week grace period (dangerous on shared servers) |
| `git repack -d` | Pack loose objects, delete redundant packs |
| `git repack -a -d` | Full repack into single pack + delete old packs |
| `git repack -a -d --write-bitmap-index` | Full repack + generate bitmap file |
| `git prune` | Delete unreachable objects older than gc.pruneExpire |
| `git prune --expire=now` | Delete ALL unreachable objects immediately |
| `git pack-refs --all` | Move all loose refs into packed-refs |
| `git maintenance start` | Enable scheduled background maintenance |
| `git maintenance run` | Run all maintenance tasks once |
| `git maintenance run --task=gc` | Run just the gc task |
| `git verify-pack -v pack-*.idx` | Show all objects in a pack with details |
| `git commit-graph write --reachable` | Generate/update commit-graph file |
| `git multi-pack-index write` | Generate MIDX spanning all pack files |
| `git fsck --unreachable` | List objects not reachable from any ref |
| `git count-objects -v` | Count loose objects and pack file stats |

### Configuration

| Config Key | Default | What It Controls |
|---|---|---|
| `gc.auto` | 6700 | Loose object count before auto-gc triggers |
| `gc.autoPackLimit` | 50 | Pack count before auto-gc triggers |
| `gc.reflogExpire` | 90 days | Keep reachable reflog entries for N days |
| `gc.reflogExpireUnreachable` | 30 days | Keep unreachable reflog entries for N days |
| `gc.pruneExpire` | 2.weeks.ago | Grace period before deleting unreachable objects |
| `pack.writeBitmaps` | false | Auto-generate bitmap on repack |
| `pack.depth` | 50 | Max delta chain depth |
| `pack.window` | 10 | Delta search window (candidates per object) |
| `pack.threads` | 0 (auto) | Threads for delta compression |
| `gc.bigPackThreshold` | 0 (disabled) | Skip packs larger than this during incremental repack |

### Object Types in a Pack

| Type ID | Name | Content |
|---|---|---|
| 1 | OBJ_COMMIT | commit object (full) |
| 2 | OBJ_TREE | tree object (full) |
| 3 | OBJ_BLOB | blob object (full) |
| 4 | OBJ_TAG | tag object (full) |
| 6 | OBJ_OFS_DELTA | delta from object at negative offset in same pack |
| 7 | OBJ_REF_DELTA | delta from object identified by 20-byte SHA |

---

## RULE 13 — When Would I Use This At Work?

### Scenario 1: "Our CI Clones Are Taking 8 Minutes"

```
SYMPTOM: git clone takes 8 minutes on every CI run.
COST: 20 engineers × 10 PRs/day × 8 minutes = 1,600 engineer-minutes/day wasted.

DIAGNOSIS:
  ls .git/objects/pack/*.bitmap
  # No output → no bitmap file!

  ls .git/objects/pack/*.pack | wc -l
  # Output: 234  → too many pack files

FIX:
  # On the git server (or bare clone used as cache):
  git repack -a -d --write-bitmap-index

RESULT: Clone time: 8 minutes → 45 seconds (10x improvement).
LESSON: Generate bitmaps on any server-side repo used for cloning.
```

### Scenario 2: "New Hire Accidentally Committed the Production Database Backup"

```
It happens. Someone ran "git add ." and included a 2 GB SQL dump.
It was caught in code review, but the commit was already pushed.

IMMEDIATE STEPS:
  1. Revoke any credentials in the database dump (treat as leaked)
  2. Remove from history with git filter-repo (see Topic 26)
  3. Force-push to GitHub (coordinate with team via Slack/Teams)
  4. All engineers must:
     git fetch --all
     git reset --hard origin/main   # or re-clone

  5. After everyone has updated:
     On the server / GitHub's GC (happens automatically within 24h):
     git reflog expire --expire=now --all
     git gc --prune=now

  6. Verify it's gone:
     git log --all --full-history -- database-backup.sql
     # Should show no results

LESSON: A pre-commit hook (Topic 19) that rejects files > 10 MB would have
prevented this entirely.
```

### Scenario 3: Running Git on a Resource-Constrained Server

```
SCENARIO: Self-hosted GitLab on a 4 GB RAM server. git gc runs every push
and is making the server unresponsive.

DIAGNOSIS:
  # Check if git gc is running during user operations:
  ps aux | grep "git gc"   # if this shows up during pushes, it's --auto triggering

FIX: Decouple gc from user operations
  # Disable auto-gc on the server:
  git config --global gc.auto 0
  git config --global gc.autoPackLimit 0

  # Set up a dedicated nightly cron at 3am:
  # /etc/cron.d/git-maintenance:
  0 3 * * * git-user /usr/bin/git -C /path/to/repo maintenance run --task=loose-objects
  0 3 * * 0 git-user /usr/bin/git -C /path/to/repo maintenance run --task=gc

RESULT: Users no longer experience lag from background GC.
       Repo is maintained weekly (Sunday 3am) without impacting operations.
```

### Scenario 4: Pre-Release Audit — "How Big Will This Tag Be to Distribute?"

```
SCENARIO: You're about to cut a release. The legal team wants to know the
download size for the source archive. You also want to audit for any
accidentally committed secrets or large files before the release is public.

COMMANDS:
  # How big is the full repo download (git clone)?
  git count-objects -vH
  # Output:
  # count: 0
  # size: 0
  # in-pack: 182453
  # packs: 1
  # size-pack: 247.48 MiB    ← this is the download size
  # prune-packable: 0
  # garbage: 0
  # size-garbage: 0

  # Find top 20 largest blobs in the entire history
  git rev-list --objects --all \
    | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
    | awk '$1 == "blob"' \
    | sort -k3 -rn \
    | head -20
  
  # Find any files that look like secrets in history
  git log -p --all | grep -E "(password|secret|api.key|token)" | head -20

RESULT: You can audit the full repo history and spot problems before
they become public release problems.
```

### Scenario 5: "git log Is Slow — Fixing It Without a Full GC"

```
SCENARIO: git log --oneline -n 100 takes 4 seconds on a large repo.
You can't run a full gc right now (it would take 2 hours).

FAST FIX: Generate the commit-graph file (takes ~30 seconds):
  git commit-graph write --reachable --changed-paths
  # --reachable: include all commits reachable from any ref
  # --changed-paths: also compute bloom filters for which paths changed per commit
  
  # Now git log is fast (reads commit-graph instead of individual commit objects):
  time git log --oneline -n 100
  # Before: 4.2s
  # After:  0.3s

  # Keep it updated after every fetch (add to post-fetch script):
  git commit-graph write --reachable --append
  # --append: incremental update (only new commits added)

LESSON: The commit-graph is the single highest-impact optimization
for read-heavy repos (lots of git log, git blame, CI pipelines).
```

---

*Topic 25 complete — 25/26 topics done.*
